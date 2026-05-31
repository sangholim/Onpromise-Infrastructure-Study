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
