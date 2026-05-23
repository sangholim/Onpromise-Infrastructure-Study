# EXT-DMZ-LB 원격 접근 방법

## SSH 원격 접근 필요성
이 구조에서 DBM-LB 는 DMZ 서버를 고르게 분산하는 역할을 한다.
DMZ Bastion host 를 경유한다.

필요한 이유:
- 부하 분산

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
DMZ - LB

```

---
```bash
# 1. Key 생성 (클라이언트)
ssh-keygen -t ed25519 -C "lsh@internal.com" -f ./lsh_lb_key
# 1.1 파일 생성후 권한 600
chmod 600 ./lsh_lb_key
# 2. DMZ LB Host에 Public Key 등록
sudo adduser lsh

# OS별 그룹 (둘 중 하나만 사용)
# sudo usermod -aG sudo lsh     # Ubuntu
sudo usermod -aG wheel lsh  # CentOS/RHEL

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
# sshConfig 파일 생성 (dmz-bastion 에서 관리하는 sshConfig 파일에 추가한다
vi sshConfig

# 파일안에 아래의 정보 입력
Host dmz.lb.internal.com
# HostName 10.10.20.200 (dmz lb 내부 ip)
  HostName 10.10.20.200
  Port 22
  User lsh
  # ex: IdentityFile /Users/mac/Project/Onpromise-Infrastructure-Study/fw-dmz-fw-sz-study/vms/ext-dmz-lb/ssh/lsh/lsh_lb_key
  IdentityFile ${lsh 개인키 경로}

ssh -F ${sshConfig 경로} lsh@dmz.lb.internal.com

```