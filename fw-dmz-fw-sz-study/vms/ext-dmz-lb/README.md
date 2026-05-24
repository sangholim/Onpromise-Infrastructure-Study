# EXT-DMZ-LB 구성 요약
### 네트워크 구성
| 구분  | 역할                 | IP 대역                       | 주요 IP        | 비고                   |
| --- | ------------------ | --------------------------- | ------------ | -------------------- |
| DMZ | Backend / Proxy 영역 | 10.10.20.200 ~ 10.10.20.220 | 10.10.20.200 | HOST-ONLY            |
| FW  | Front / VIP 영역     | 10.10.10.200 ~ 10.10.10.220 | 10.10.10.200 | gateway = 10.10.10.2 |
| VIP | 서비스 진입 IP          | FW subnet                   | 10.10.10.221 | Keepalived 관리        |

### VIP / LB 구조 개요
| 항목              | 내용                        |
| --------------- | ------------------------- |
| LB 방식           | IPVS NAT                  |
| HA 구성           | Keepalived VRRP           |
| 스케줄링            | Round Robin (rr)          |
| VIP 위치          | FRONT subnet (10.10.10.x) |
| Backend 위치      | BACK subnet (10.10.20.x)  |
| NAT 특징          | 세션 owner 는 LB             |
| 라우팅 특징          | Backend 응답은 반드시 LB 경유     |
| Backend Gateway | DMZ-LB IP 로 설정 필요         |
| Failover        | Keepalived 기반 VIP 이전 가능   |

### 인터페이스 구성
| NIC    | 용도             | 대역         | 설정            |
| ------ | -------------- | ---------- | ------------- |
| enp0s8 | BACK NIC (DMZ) | 10.10.20.x | never-default |
| enp0s9 | FRONT NIC (FW) | 10.10.10.x | gateway 사용    |

### 네트워크 설정 명령 요약
| 대상          | 주요 설정                                   |
| ----------- | --------------------------------------- |
| DMZ NIC     | `10.10.20.200/24`, `never-default=yes`  |
| FW NIC      | `10.10.10.200/24`, `gateway=10.10.10.2` |
| Route 확인    | `ip route`                              |
| Hostname 설정 | `hostnamectl set-hostname ext-dmz-lb`   |

### 설치 패키지
| 패키지               | 역할               |
| ----------------- | ---------------- |
| iptables-services | NAT / FORWARD 관리 |
| ipvsadm           | IPVS 관리          |
| keepalived        | VIP Failover     |

### sysctl 설정 요약
| 설정                        | 목적                       |
| ------------------------- | ------------------------ |
| `net.ipv4.ip_forward=1`   | L3 Forward 활성화           |
| `net.ipv4.vs.conntrack=1` | IPVS conntrack 연동        |
| `rp_filter=0`             | asymmetric routing 문제 방지 |
| `arp_ignore=1`            | VIP ARP 안정화              |
| `arp_announce=2`          | VIP ARP 충돌 방지            |

### Kernel Module 구성
| 모듈           | 역할          |
| ------------ | ----------- |
| nf_conntrack | 세션 추적       |
| nf_nat       | NAT 처리      |
| ip_vs        | IPVS core   |
| ip_vs_rr     | Round Robin |
| ip_vs_sh     | Source Hash |
| ip_vs_wrr    | Weighted RR |

### NAT / FORWARD 구성
| 항목           | 설정         |
| ------------ | ---------- |
| FORWARD 정책   | ACCEPT     |
| POSTROUTING  | MASQUERADE |
| Outbound NIC | enp0s9     |

### IPVS 설정
| 항목         | 값               |
| ---------- | --------------- |
| VIP        | 10.10.10.221:80 |
| Scheduler  | rr              |
| Backend    | 10.10.20.2:80   |
| Forward 방식 | NAT (`-m`)      |

### Keepalived 설정
| 항목        | 값               |
| --------- | --------------- |
| Instance  | VI_1            |
| 상태        | MASTER          |
| Interface | enp0s9          |
| VRID      | 51              |
| Priority  | 150             |
| VIP       | 10.10.10.221/24 |
| 인증        | PASS / 1234     |

### Firewall 설정
| 항목     | 설정                                            |
| ------ | --------------------------------------------- |
| 허용 서비스 | HTTP                                          |
| 명령     | `firewall-cmd --permanent --add-service=http` |

### 상태 확인 명령
| 목적            | 명령                             |
| ------------- | ------------------------------ |
| IP 확인         | `ip addr`                      |
| Route 확인      | `ip route`                     |
| IPVS 상태       | `ipvsadm -Ln`                  |
| NAT 상태        | `iptables -t nat -L -n -v`     |
| IP Forward 확인 | `sysctl net.ipv4.ip_forward`   |
| VIP 확인        | `ip addr \| grep 10.10.10.221` |
