# Oracle Cloud, Terraform 을 이용한 인프라 구축

# 계기
```text
웹 서비스 개발 및 CI/CD 구축 경험을 바탕으로,
서비스가 실제 운영되는 네트워크 및 인프라 계층에 대한 이해를 확장하고자 
클라우드 환경에서 본 프로젝트를 직접 설계·구축하였습니다.
```
# 프로젝트 목표
- 방화벽 장비의 역할 및 실제 동작 방식 이해
- L7 기반 단순 reverse proxy 가 아닌 L3/L4 기반 네트워크 흐름 검증
    - [L3, L7 영역 비교](private-infra/contents/L3_L7_DIFF.md)
- VIP 및 L4 Load Balancer 구조 직접 구현
- DMZ / Server Zone 분리를 통한 내부망 보호 구조 검증
- Stateful Firewall 환경에서의 실제 세션 흐름 이해
- asymmetric routing 및 NAT 환경 문제 분석 경험 확보
