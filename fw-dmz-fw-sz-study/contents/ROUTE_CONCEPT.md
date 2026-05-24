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

# NAT 사용시 고려 사항
| 항목                  | NAT 있는 구조 (IPVS NAT Mode)   | NAT 없는 구조 (DR/DSR/Forwarding)       |
| ------------------- | --------------------------- | ----------------------------------- |
| 기본 개념               | LB 가 NAT + Session State 관리 | LB 는 Backend 선택/전달만 수행              |
| LB 역할               | Stateful NAT 장비             | Forwarding/분산 결정                    |
| 요청 흐름               | Client → LB → Backend       | Client → LB → Backend               |
| 응답 흐름               | Backend → LB → Client       | Backend → Client (Direct Return 가능) |
| Backend 가 보는 Client | LB IP(SNAT 결과)              | 실제 Client IP                        |
| DNAT                | 수행                          | 일부 구조만 수행                           |
| SNAT                | 수행                          | 일반적으로 없음                            |
| Reverse NAT         | 필요                          | 불필요                                 |
| conntrack/state     | LB 중심 관리                    | 상대적으로 적음                            |
| Return Path         | 반드시 LB 복귀 필요                | Direct Return 가능                    |
| Backend Gateway 중요성 | 매우 중요 (LB 향해야 함)            | 상대적으로 덜 중요                          |
| 비대칭 라우팅 영향          | 매우 큼                        | 상대적으로 덜함                            |
| rp_filter 영향        | 자주 문제 발생 가능                 | 상대적으로 적음                            |
| LB 부하               | 큼 (모든 packet 처리)            | 상대적으로 적음                            |
| Throughput 확장성      | LB 병목 가능                    | 확장성 유리                              |
| 성능                  | 상대적으로 낮음                    | 고성능 가능                              |
| 운영 난이도              | 상대적으로 단순                    | routing/ARP 설정 복잡                   |
| Backend 설정          | 단순                          | VIP/ARP/routing 고려 필요               |
| FW(Stateful) 정책     | 중앙 집중 관리 쉬움                 | Return bypass 가능                    |
| FW Session Tracking | 안정적                         | 비대칭 문제 가능                           |
| ACL / IPS / IDS     | 중앙 적용 용이                    | 일부 traffic 우회 가능                    |
| Logging / Audit     | 중앙 수집 용이                    | 분산 가능                               |
| 보안 정책               | LB/FW 중심 통제                 | Backend 측 고려 증가                     |
| 장애 분석               | 비교적 단순                      | 경로 추적 복잡                            |
| Failover 영향         | conntrack sync 중요           | routing consistency 중요              |
| 실무 활용               | Enterprise / DMZ / 보안 중심    | Hyperscale / 고성능 서비스                |
| 대표 기술               | IPVS NAT, FW NAT, Proxy LB  | LVS-DR, DSR, ECMP, Anycast          |

# DMZ 현업 고려사항
| 구간                 | 많이 쓰는 방식          |
| ------------------ | ----------------- |
| Internet ↔ DMZ     | NAT/stateful      |
| FW ↔ LB ↔ WAS      | NAT 기반            |
| 내부 east-west       | routing/eBPF      |
| Kubernetes cluster | overlay + NAT 혼합  |
| CDN edge           | DSR/ECMP/Anycast  |
| 대규모 API            | direct routing 증가 |
