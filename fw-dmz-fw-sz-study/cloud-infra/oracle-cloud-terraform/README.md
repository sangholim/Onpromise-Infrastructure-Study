# 오라클 클라우드 테라폼 구축
# 필수 확인
- 서비스 이용시 반드시 always free tier 항목에서 선정하여 사용
- Pay As You Go 로 상태를 되도록 바꾸지 않는다.(Oracle은 무료 체험이 끝나거나 $300 크레딧을 다 사용해도 Pay As You Go로 업그레이드하지 않으면 Always Free 리소스는 계속 사용할 수 있다고 안내합니다.)

# 구축
```text
Tenancy (내 OCI 전체 계정)
│
├── Compartment: infra
│   ├── VCN
│   ├── Subnet
│   ├── Internet Gateway
│   ├── NAT Gateway
│   ├── VM
│   └── Load Balancer

```
# 순서
- [compartment 설정](compartment/README.md)
- [vcn 설정](vcn/README.md)