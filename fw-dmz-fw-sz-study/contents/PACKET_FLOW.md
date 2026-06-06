# http 패킷 흐름
### fw 정책
```text
# fw 정책
source: any 8080, destination: 10.10.1.1 80
# fw cli
conntrack -E -p tcp
# cli 결과

## NEW         : 신규 세션 생성
## 120         : timeout 120초
## SYN_SENT    : SYN 전송 상태
## 10.10.1.1   : Client IP
## 10.10.1.3   : FW NAT IF
## 49570       : Client source port
## 8080        : NAT listening port
## UNREPLIED   : SYN/ACK 미수신
## 10.10.10.221: VIP/NAT address
## reverse tuple 저장 중
[NEW] tcp      6 120 SYN_SENT src=10.10.1.1 dst=10.10.1.3 sport=49570 dport=8080 [UNREPLIED] src=10.10.10.221 dst=10.10.1.1 sport=80 dport=49570

## UPDATE      : 상태 업데이트
## 60          : timeout 60초
## SYN_RECV    : SYN/ACK 수신
## handshake 진행 중
## NAT mapping 유지
[UPDATE] tcp      6 60 SYN_RECV src=10.10.1.1 dst=10.10.1.3 sport=49570 dport=8080 src=10.10.10.221 dst=10.10.1.1 sport=80 dport=49570

## ESTABLISHED : 연결 완료
## 432000      : timeout 5일
## ASSURED     : 양방향 통신 확인
## 원본/NAT tuple 유지
## stateful NAT 동작 중
[UPDATE] tcp      6 432000 ESTABLISHED src=10.10.1.1 dst=10.10.1.3 sport=49570 dport=8080 src=10.10.10.221 dst=10.10.1.1 sport=80 dport=49570 [ASSURED]

## FIN_WAIT    : 연결 종료 시작
## 120         : timeout 120초
## 세션 종료 처리 중
[UPDATE] tcp      6 120 FIN_WAIT src=10.10.1.1 dst=10.10.1.3 sport=49570 dport=8080 src=10.10.10.221 dst=10.10.1.1 sport=80 dport=49570 [ASSURED]

## LAST_ACK    : 마지막 ACK 대기
## 30          : timeout 30초
## 종료 직전 상태
[UPDATE] tcp      6 30 LAST_ACK src=10.10.1.1 dst=10.10.1.3 sport=49570 dport=8080 src=10.10.10.221 dst=10.10.1.1 sport=80 dport=49570 [ASSURED]

## TIME_WAIT   : 종료 후 대기
## 120         : timeout 120초
## 이후 state 제거
[UPDATE] tcp      6 120 TIME_WAIT src=10.10.1.1 dst=10.10.1.3 sport=49570 dport=8080 src=10.10.10.221 dst=10.10.1.1 sport=80 dport=49570 [ASSURED]

```
### dmz-lb 에서 tcpdump
```text
# dmz-lb cli
tcpdump -nn -i any tcp port 80
# cli 결과
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes

## fw -> dmz-lb (VIP)
14:21:29.563465 enp0s9 In  IP 10.10.1.1.49313 > 10.10.10.221.80: Flags [S], seq 92864001, win 65535, options [mss 1460], length 0

## fw -> dmz-lb (DNAT) -> dmz 
14:21:29.563481 enp0s8 Out IP 10.10.1.1.49313 > 10.10.20.2.80: Flags [S], seq 92864001, win 65535, options [mss 1460], length 0

## dmz -> fw 로 향하는 패킷이 NIC enp0s8 에 들어옴
14:21:29.564163 enp0s8 In  IP 10.10.20.2.80 > 10.10.1.1.49313: Flags [S.], seq 4256902853, ack 92864002, win 64240, options [mss 1460], length 0

## dmz-lb (VIP) -> fw 로 향하는 패킷이 NIC enp0s9 로 나감
14:21:29.564177 enp0s9 Out IP 10.10.10.221.80 > 10.10.1.1.49313: Flags [S.], seq 4256902853, ack 92864002, win 64240, options [mss 1460], length 0```
```

### dmz 에서 tcpdump
```text
#dmz cli
tcpdump -nn -i any tcp port 80
# cli 결과
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes

## fw -> dmz 로  향하는 패킷이 enp0s8 에 들어옴
14:45:24.980145 enp0s8 In  IP 10.10.1.1.49397 > 10.10.20.2.80: Flags [S], seq 278656001, win 65535, options [mss 1460], length 0

## dmz -> fw 로  향하는 패킷이 enp0s8 로 나감
14:45:24.980188 enp0s8 Out IP 10.10.20.2.80 > 10.10.1.1.49397: Flags [S.], seq 2327241607, ack 278656002, win 64240, options [mss 1460], length 0
```

### dmz-lb, dmz-lb1 keepalived  (1)
- dmz-lb (MASTER), dmz-lb1 (BACKUP)
- dmz-lb 가 죽으면, dmz-lb1 은 MASTER 상태가 된다.
- dmz-lb 가 다시 살아나면 BACKUP 상태가 된다.

```shell
# dmz-lb MASTER 상태 확인
journalctl -u keepalived | grep "MASTER"
6월 03 15:32:18 ext-dmz-lb Keepalived_vrrp[1075]: (VI_2) Entering MASTER STATE
6월 03 15:32:18 ext-dmz-lb Keepalived_vrrp[1075]: (VI_1) Entering MASTER STATE

# dmz-lb1 BACKUP 상태 확인
root@ext-dmz-lb1:/home/lsh# journalctl -u keepalived -f
6월 03 15:32:24 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_1) Entering BACKUP STATE (init)
6월 03 15:32:24 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_2) Entering BACKUP STATE (init)

# dmz-lb 죽고나서, dmz-lb1 상태 확인
## 원래 MASTER가 VRRP heartbeat를 끊어서, BACKUP이 장애로 판단한 로그, MASTER 상태로 전환
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_2) Receive advertisement timeout
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_2) Entering MASTER STATE
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_2) setting VIPs.
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_2) Sending/queueing gratuitous ARPs on enp0s8 for 10.10.20.221
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: Sending gratuitous ARP on enp0s8 for 10.10.20.221
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_1) Receive advertisement timeout
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_1) Entering MASTER STATE
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_1) setting VIPs.
6월 03 16:12:08 ext-dmz-lb1 Keepalived_vrrp[3012]: (VI_1) Sending/queueing gratuitous ARPs on enp0s9 for 10.10.10.221

# dmz-lb 살리고나서, dmz-lb 상태 확인
## 주소를 재등록하고, 백업 상태로 등록됨 
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: Assigned address fe80::a00:27ff:fe11:aef1 for interface enp0s8
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: Registering gratuitous ARP shared channel
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: (VI_1) removing VIPs.
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: (VI_2) removing VIPs.
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: (VI_1) Entering BACKUP STATE (init)
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: (VI_2) Entering BACKUP STATE (init)
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: VRRP sockpool: [ifindex(  3), family(IPv4), proto(112), fd(12,13) multicast, address(224.0.0.18)]
6월 03 16:19:26 ext-dmz-lb Keepalived_vrrp[1179]: VRRP sockpool: [ifindex(  2), family(IPv4), proto(112), fd(14,15) multicast, address(224.0.0.18)]
6월 03 16:19:26 ext-dmz-lb Keepalived[1029]: Startup complete
6월 03 16:19:26 ext-dmz-lb systemd[1]: Started keepalived.service - LVS and VRRP High Availability Monitor.
```
### dmz-lb, dmz-lb1 keepalived  (2)
- dmz-lb (MASTER, priority: 150), dmz-lb1 (BACKUP, priority: 140, nopreempt)
- dmz-lb, dmz-lb1 VRRP 수신하지 못하는 상태로 만든다.
```shell
# VRRP 패킷 드랍 정책을 추가하여, 모두 MATER 올라가도록 버그 발생 (dmz-lb, dmz-lb1) 
iptables -A OUTPUT -p 112 -j DROP 
iptables -A INPUT -p 112 -j DROP
# 로그 확인하여 모두 MASTER 상태인지 확인 (dmz-lb, dmz-lb1)
## dmz-lb
root@ext-dmz-lb:/home/lsh# journalctl -u keepalived | grep "Entering MASTER"
 6월 06 15:21:12 ext-dmz-lb Keepalived_vrrp[1072]: (VI_2) Entering MASTER STATE
 6월 06 15:21:12 ext-dmz-lb Keepalived_vrrp[1072]: (VI_1) Entering MASTER STATE
 6월 06 15:37:59 ext-dmz-lb Keepalived_vrrp[3331]: (VI_1) Entering MASTER STATE
 6월 06 15:38:17 ext-dmz-lb Keepalived_vrrp[3331]: (VI_2) Entering MASTER STATE
 
 ## dmz-lb1
 root@ext-dmz-lb1:/home/lsh# journalctl -u keepalived | grep "Entering MASTER"
 6월 06 15:37:55 ext-dmz-lb1 Keepalived_vrrp[1171]: (VI_2) Entering MASTER STATE
 6월 06 15:37:55 ext-dmz-lb1 Keepalived_vrrp[1171]: (VI_1) Entering MASTER STATE
 6월 06 15:48:56 ext-dmz-lb1 Keepalived_vrrp[3550]: (VI_2) Entering MASTER STATE
 6월 06 15:48:56 ext-dmz-lb1 Keepalived_vrrp[3550]: (VI_1) Entering MASTER STATE

# VRRP 패킷 드랍 정책을 제거하여, BACKUP, MASTER 상태 동기화 확인  (dmz-lb, dmz-lb1)
iptables -D OUTPUT -p 112 -j DROP
iptables -D INPUT -p 112 -j DROP

# 로그 확인하여 MASTER, BACKUP 상태인지 확인 (dmz-lb, dmz-lb1)
## dmz-lb 는 상태 유지 (dmz-lb1 = BACKUP ,nopreempt, priority 가 더 낮음)
root@ext-dmz-lb:/home/lsh# journalctl -u keepalived | grep "Entering "
 6월 06 15:21:09 ext-dmz-lb Keepalived_vrrp[1072]: (VI_1) Entering BACKUP STATE (init)
 6월 06 15:21:09 ext-dmz-lb Keepalived_vrrp[1072]: (VI_2) Entering BACKUP STATE (init)
 6월 06 15:21:12 ext-dmz-lb Keepalived_vrrp[1072]: (VI_2) Entering MASTER STATE
 6월 06 15:21:12 ext-dmz-lb Keepalived_vrrp[1072]: (VI_1) Entering MASTER STATE
 6월 06 15:37:55 ext-dmz-lb Keepalived_vrrp[3331]: (VI_1) Entering BACKUP STATE (init)
 6월 06 15:37:55 ext-dmz-lb Keepalived_vrrp[3331]: (VI_2) Entering BACKUP STATE (init)
 6월 06 15:37:59 ext-dmz-lb Keepalived_vrrp[3331]: (VI_1) Entering MASTER STATE
 6월 06 15:38:17 ext-dmz-lb Keepalived_vrrp[3331]: (VI_2) Entering MASTER STATE

```

### dmz-lb ipvs
cli: ipvsadm -Lc

| 항목          | 값               | 의미                      |
| ----------- | --------------- | ----------------------- |
| Protocol    | TCP             | TCP 세션                  |
| Expire      | 00:14           | 14초 후 엔트리 만료            |
| State       | TIME_WAIT       | TCP 종료 대기 상태            |
| Source      | 10.10.1.1:49428 | 실제 Client               |
| Virtual     | ext-dmz-lb:http | VIP (`10.10.10.221:80`) |
| Destination | 10.10.20.2:http | 실제 Backend Server       |
