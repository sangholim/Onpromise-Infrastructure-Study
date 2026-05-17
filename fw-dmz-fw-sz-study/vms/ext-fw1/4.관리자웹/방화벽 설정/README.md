# 관리자웹 설정
- WebGUI → Firewall → Firewall Rules \
  Add new rule
- 방화벽 설정
  - SSH (외부 - NAT)
    ![SSH](ssh.png)
  - SSH (외부 - dmz-bastion)
    ![SSH_DNAT_PORT_FORWADING](ext_fw1_dmz_bastion_port_forwading.png)
    ![SSH_FORWADING](ext_fw1_dmz_bastion_ssh.png)

  - 웹 (외부 - NAT)
    ![관리자 웹](fw_web.png)
  - dmz outbound ( 외부)
    ![관리자 웹](external.png)
  - 목록
    ![방화벽 목록](firewall_rules.png)
  - 재기동
    ``` {shell} \
    sudo /etc/init.d/firewall restart
    ```
- SSH 설정
  - WebGUI → System -> SSH Access
    ![SSH](ssh_access.png)
- 방화벽 정책 확인

- 포트 포워딩  설정 필요