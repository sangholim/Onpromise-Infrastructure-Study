# EXT-DMZ-BASTION 원격 접근 방법

## SSH 원격 접근 필요성
이 구조에서 Bastion Host는 일종의 점프 서버(Jump Server) 역할입니다.

필요한 이유:
- 내부망 서버 직접 노출 방지 (외부 → 내부 직접 접속 차단)
- 접근 경로 단일화 (감사 및 로깅 용이)
- 운영자 접근 통제 (계정/키 기반 관리)

즉, 보안 강화를 위해 오히려 필요한 구성입니다.  
문제는 “SSH를 쓰느냐”가 아니라 “어떻게 쓰느냐”입니다.

---

## 보안상 문제점 (잘못 구성했을 때)

다음 중 하나라도 해당되면 위험합니다:

### ❌ 흔한 취약점
- 22번 포트 전체 인터넷 오픈 (IP 제한 없음)
- 패스워드 로그인 허용
- root 직접 로그인 허용
- SSH key 무관리 (유출, 공유, 회수 안됨)
- Fail2Ban / rate limit 없음 → brute force 공격
- bastion에서 내부 서버로 unrestricted lateral movement
- 로그/감사 미구현

### ⚠️ 실제 공격 시나리오
- 무차별 SSH brute force
- 탈취된 private key로 접근
- bastion 장악 후 내부망 pivot 공격

---

## “해도 되냐?” → 결론
👉 조건부로 OK (권장되는 구조)

단, 아래 조건 필수:
- IP whitelist (회사/관리자 IP만)
- SSH key 기반 인증만 허용
- MFA 적용 (가능하면)
- bastion → 내부 접근 최소화 (Zero Trust)
- 세션 로깅 / 감사

---

## 적용 가이드

### 안전한 SSH 접근 구조 (권장)

```text
외부 사용자
↓ (IP 제한 + key 인증)
Firewall
↓
DMZ Bastion Host
↓ (ProxyJump / SSH Tunnel)
내부 서버

```

---
```bash
# 1. Key 생성 (클라이언트)
ssh-keygen -t ed25519 -C "lsh@internal.com" -f ./lsh_bastion_key

# 2. Bastion Host에 Public Key 등록
sudo adduser lsh

# OS별 그룹 (둘 중 하나만 사용)
sudo usermod -aG sudo lsh     # Ubuntu
# sudo usermod -aG wheel lsh  # CentOS/RHEL

mkdir -p /home/lsh/.ssh
chmod 700 /home/lsh/.ssh
chown lsh:lsh /home/lsh/.ssh

vi /home/lsh/.ssh/authorized_keys

chmod 600 /home/lsh/.ssh/authorized_keys
chown lsh:lsh /home/lsh/.ssh/authorized_keys

# 3. SSH 설정 강화
vi /etc/ssh/sshd_config

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

sudo systemctl restart sshd

# 4. hosts 등록 
## 내부망에서는 일반적으로 호스트를 쓰지 않는데, 가상 host 를 쓰는 경우 (authorized_keys)
## ssh 원격 접속시 prompt 에서 해당 dns 를 찾는 이슈가 있어서, alias domain 설정 필요
vi /etc/hosts
## 아래 라인 추가
127.0.0.1   internal.com

```

---
### sshConfig 설정
```bash
# sshConfig 파일 생성
vi sshConfig

# 파일안에 아래의 정보 입력
Host dmz.internal.com
  HostName 127.0.0.1
  Port 3333
  User lsh@internal.com
  # ex: IdentityFile /Users/mac/Project/Onpromise-Infrastructure-Study/fw-dmz-fw-sz-study/vms/ext-dmz-bastion/ssh/lsh/lsh_bastion_key
  IdentityFile ${lsh 개인키 경로}

ssh -F ${sshConfig 경로} lsh@dmz.internal.com

```