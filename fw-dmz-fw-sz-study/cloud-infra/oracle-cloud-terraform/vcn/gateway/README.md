# vcn-gateway

# 정의
- VCN과 외부 네트워크 또는 다른 네트워크 사이에서 트래픽을 연결해주는 출입구 역할의 리소스입니다.
# 유형 및 역할
| Gateway                    | 역할                        | 주 사용 영역      |
| -------------------------- | ------------------------- | ------------ |
| **Internet Gateway (IGW)** | VCN ↔ 인터넷 직접 연결           | DMZ / Public |
| **NAT Gateway**            | Private → 인터넷 outbound 연결 | Private      |
| **Service Gateway**        | VCN → OCI 서비스 연결          | Private      |
| **DRG**                    | VCN ↔ 다른 VCN / 온프레미스 연결   | 사설망          |

# 구축 방향
- private -> dmz 나갈수 있게 라우팅룰을 추가할 예정

# 등록 방법
- [internet gateway 가이드](internet-gateway/README.md)
