# EXT-DMZ-LB
## 네트워크
### DMZ
- ip 대역 (10.10.20.200 ~ 10.10.20.220)
- HOST-ONLY (10.10.20.200) \
- VIP (10.10.20.221)
### FW
- ip 대역 (10.10.10.200 ~ 10.10.10.220)
- VIP (10.10.10.221)
- HOST-ONLY (10.10.10.200) \
  gateway 10.10.10.2

### VIP 구현
- ipvs nat + keepalvied
- ipvs nat 제약 사항
  - 세션 owner는 LB
  - 동일한 인터페이스 대역들은 (ext-dmz) LB 에 종속된다 \
    ext-dmz 응답은 반드시 LB를 경유해야 정상 (게이트웨이 IP 를 dmz-lb ip 로 할당)
  

``` {shell} \
  # ipv4 할당 및 gateway 설정 (dmz)
  nmcli connection modify enp0s8 \
    ipv4.addresses 10.10.20.200/24 \
    ipv4.method manual \
    ipv4.never-default yes && \
  nmcli connection up enp0s8;

  # ipv4 할당 및 gateway 설정 (fw)
  nmcli connection modify enp0s9 \
    ipv4.addresses 10.10.10.200/24 \
    ipv4.gateway 10.10.10.2 \
    ipv4.method manual && \
  nmcli connection up enp0s9;

  # route 확인
  ip route
```

## 호스트 설정

```shell
sudo hostnamectl set-hostname ext-dmz-lb
```
-------

## ssh 설정 참고
- [ssh 설정가이드](./ssh/README.md)

## 목표
- fw 에서 proxy 로 오는 ip 를 받는 역할, proxy vm 들의 lb 를 담당
- IPVS 구현
- fail over 준비


## 설치
```shell
# =========================================================
# package 설치
# =========================================================

dnf install -y \
  iptables-services \
  ipvsadm \
  keepalived

systemctl enable --now iptables


# =========================================================
# sysctl 설정
# =========================================================

cat <<EOF > /etc/sysctl.d/99-ipvs.conf

# =========================
# IPVS / LB / VRRP 안정 설정
# =========================

# IP Forward
net.ipv4.ip_forward = 1

# conntrack
net.ipv4.vs.conntrack = 1

# Reverse Path Filter 비활성화
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# BACK NIC
net.ipv4.conf.enp0s8.rp_filter = 0

# FRONT NIC
net.ipv4.conf.enp0s9.rp_filter = 0

# ARP 설정 (VIP 안정화)
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2

net.ipv4.conf.enp0s8.arp_ignore = 1
net.ipv4.conf.enp0s8.arp_announce = 2

net.ipv4.conf.enp0s9.arp_ignore = 1
net.ipv4.conf.enp0s9.arp_announce = 2

EOF

# 적용
sysctl --system


# =========================================================
# kernel module
# =========================================================

modprobe nf_conntrack
modprobe nf_nat

modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_sh
modprobe ip_vs_wrr

# module 자동 로드
cat <<EOF > /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_sh
ip_vs_wrr
EOF


# =========================================================
# FORWARD 허용
# =========================================================

iptables -P FORWARD ACCEPT


# =========================================================
# NAT 설정
# FRONT NIC outbound NAT
# =========================================================

iptables -t nat -F

# FW 로 향하는 패킷의 출발지 주소를 enp0s9 의 IP 로 할당하라.
# 응답할때 10.10.20.x 주소는 fw 에서는 알지 못함
iptables -t nat -A POSTROUTING \
  -o enp0s9 \
  -j MASQUERADE

# iptables 영구저장
iptables-save > /etc/sysconfig/iptables

# iptables 부팅시 자동 복원
systemctl status iptables
systemctl enable iptables

# =========================================================
# IPVS 설정
# VIP = FRONT subnet
# DMZ = DMZ subnet
# =========================================================
ipvsadm -C

ipvsadm -A -t 10.10.10.221:80 -s rr

ipvsadm -a -t 10.10.10.221:80 \
  -r 10.10.20.231:80 \
  -m


# 저장
ipvsadm-save > /etc/sysconfig/ipvsadm

systemctl enable ipvsadm --now
# =========================================================
# Keepalived MASTER 설정
# vrrp_instance VI_1
# FRONT NIC = enp0s9
# FW -> DMZ-LB
# 
# vrrp_instance VI_2
# PROXY NIC = enp0s8
# DMZ -> DMZ-LB
# =========================================================

cat <<EOF > /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {

    state MASTER

    interface enp0s9

    virtual_router_id 51

    priority 150

    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        10.10.10.221/24
    }
}

vrrp_instance VI_2 {

    state MASTER

    interface enp0s8

    virtual_router_id 52

    priority 150

    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        10.10.20.221/24
    }
}

EOF


# =========================================================
# Keepalived 서비스 시작
# =========================================================

systemctl enable keepalived --now


# =========================================================
# firewall
# =========================================================

firewall-cmd --permanent --add-service=http
firewall-cmd --reload


# =========================================================
# 상태 확인
# =========================================================

ip addr

ip route

ipvsadm -Ln

iptables -t nat -L -n -v

sysctl net.ipv4.ip_forward

ip addr | grep 10.10.10.221
```