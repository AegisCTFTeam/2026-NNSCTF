# EC PZ (Cryptography, 63pts)
- solved by @cooku222
  
## 문제 개요

`chall.sage`는 다음과 같은 동작을 한다:

```python
p = random_prime(2^256)
a, b = [randrange(p) for _ in range(2)]
E = EllipticCurve(GF(p), [a, b])

P = E.random_point()
Q = 2*P
R = 2*Q

F = E.lift_x(Integer(bytes_to_long(flag)))
C = next_prime(0x133713371337) * F
```

즉 `p, a, b`는 전부 비공개이고, 우리에게 주어지는 것은 좌표뿐인 `P, Q, R, C` 네 점이다.
`Q = 2P`, `R = 2Q`, `C = k·F` (단 `k = next_prime(0x133713371337)`, `F`의 x좌표가 flag)라는 관계만 알 수 있다.

목표: 곡선 파라미터 `p, a, b`를 복구한 뒤, `C`로부터 `F`를 역산해서 flag를 얻는 것.

## 1단계 — 소수 p 복구하기

곡선 방정식 `y² = x³ + ax + b (mod p)`는 모든 점에 대해 성립한다.
서로 다른 두 점 `(x1,y1)`, `(x2,y2)`에 대해 방정식을 빼면 `b`가 사라진다:

```
y1² - y2² ≡ (x1³ - x2³) + a(x1 - x2)   (mod p)
```

이건 `a`에 대해 **선형**이다. 세 번째 점을 이용해 같은 식을 하나 더 만들고, 두 식을 연립해서 `a`까지 소거하면:

```
(y1²-y2²)(x1-x3) - (y1²-y3²)(x1-x2)
    ≡ (x1³-x2³)(x1-x3) - (x1³-x3³)(x1-x2)   (mod p)
```

좌변과 우변을 **정수 그대로** 계산해서 뺀 값을 `N`이라 하면, 이 식은 `mod p`에서만 성립하므로 실제로는 `p | N`이다.

`P, Q, R, C` 네 점으로 만들 수 있는 여러 개의 순열 조합에 대해 `N`을 계산하고, 그 값들의 **최대공약수(GCD)** 를 구하면 `p`의 배수가 남는다. 실제로 계산해보면:

```
gcd = 2 × p
```

형태로 나오며, 이를 소인수분해하면 256비트 소수 `p`를 바로 얻을 수 있다.

```python
from math import gcd
import itertools

def N_of_triple(pt1, pt2, pt3):
    x1, y1 = pt1
    x2, y2 = pt2
    x3, y3 = pt3
    lhs = (y1**2 - y2**2) * (x1 - x3) - (y1**2 - y3**2) * (x1 - x2)
    rhs = (x1**3 - x2**3) * (x1 - x3) - (x1**3 - x3**3) * (x1 - x2)
    return lhs - rhs

pts = [P, Q, R, C]
g = 0
for combo in itertools.permutations(pts, 3):
    g = gcd(g, N_of_triple(*combo))

# g = 2 * p  → 소인수분해해서 256비트 소수 p 추출
```

## 2단계 — a, b 복구 및 곡선 검증

`p`를 알았으니, `P`와 `Q` 두 점만으로 앞서 유도한 선형식에서 바로 `a`를 구하고, 곡선 방정식에 대입해 `b`도 구한다.

```
a = (y1² - y2² - (x1³ - x2³)) / (x1 - x2)   mod p
b = y1² - x1³ - a·x1                        mod p
```

이후 `2P == Q`, `2Q == R`가 성립하는지 확인해서 파라미터가 맞는지 검증했다.

## 3단계 — 곡선 위수(order) 계산

`C = k·F`에서 `F`를 얻으려면 `k⁻¹ mod n` (n은 곡선의 위수 또는 점의 위수)이 필요하다.
`p`가 256비트라 위수를 직접 세는 것은 Sage 없이는 번거로워서, **PARI/GP**의 `ellcard`를 사용했다.

```gp
E = ellinit([a, b], p);
n = ellcard(E);
```

## 4단계 — F 복구 및 flag 추출

```gp
k = nextprime(0x133713371337);
kinv = lift(Mod(k, n)^(-1));
F = ellmul(E, C, kinv);
```

`F`의 x좌표를 바이트로 변환하면 flag가 나온다:

```python
x = F[1]  # F의 x좌표
flag_bytes = x.to_bytes((x.bit_length() + 7) // 8, 'big')
print(flag_bytes)
```

## 결과

```
NNS{2_EC_f0r_u_1_gu355}
```

## 핵심 아이디어 요약

| 단계 | 내용 |
|---|---|
| 1 | 여러 점 조합으로 `a, b`를 소거한 식을 만들어 GCD로 소수 `p` 복구 |
| 2 | 두 점으로 선형 연립하여 `a, b` 복구, `2P=Q`, `2Q=R`로 검증 |
| 3 | PARI/GP `ellcard`로 곡선 위수 `n` 계산 |
| 4 | `k⁻¹ mod n`으로 `F = k⁻¹·C` 계산, x좌표를 바이트로 변환해 flag 획득 |
