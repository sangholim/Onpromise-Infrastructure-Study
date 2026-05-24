# 내부망 GW 설정 관련 참고
| 항목               | Backend GW 를 VIP 로 잡을 때 문제 가능성 | 이유                        |
| ---------------- | ------------------------------ | ------------------------- |
| VIP 특성           | 안정적인 next-hop 아님               | VIP 는 floating/service IP |
| Failover         | 세션 끊김 가능                       | active node 변경 가능         |
| ARP 문제           | stale ARP 가능                   | VIP MAC 변경 가능             |
| Gratuitous ARP   | 반영 지연 가능                       | 일부 장비/OS cache 유지         |
| Routing 안정성      | real IP 보다 낮음                  | VIP ownership 변경 가능       |
| Stateful session | failover 시 conntrack 유실 가능     | NAT state 는 node local    |
| Keepalived 환경    | standby 전환 시 영향                | VIP 이동 발생                 |
| 운영 관점            | gateway 역할과 service 역할 혼합      | 역할 분리 어려움                 |
| 장애 분석            | 경로 추적 복잡                       | floating IP 기준 동작         |
| Best Practice    | 일반적으로 비권장                      | GW 는 고정 real IP 선호        |
