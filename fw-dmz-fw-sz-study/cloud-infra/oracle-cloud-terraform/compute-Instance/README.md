# vcn-compute-Instance

# 정의
- Compute Instance는 클라우드에서 가상 서버(Virtual Machine)를 생성하여 실행하는 컴퓨팅 자원입니다.
# 구축 유의 사항
- A1 Flex Always Free 자원 준수하여 구축

# 구성 설정
| VM         | 역할                |  OCPU |      RAM |
| ---------- | ----------------- | ----: | -------: |
| Router VM  | DMZ ↔ Private 라우팅 |   0.5 |     2 GB |
| DMZ VM     | Web/LB            |   0.5 |     2 GB |
| Private VM | Backend           |     1 |     4 GB |
| **합계**     |                   | **2** | **8 GB** |
