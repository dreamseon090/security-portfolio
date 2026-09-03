# HTB-SAUNA
```bash
OS: Windows(AD)
DATE: 26.08.28
DIFFICULTY: Easy
```

## RECON
**nmap**: IP를 받은 후 바로 포트 스캔을 하였다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p PORTS IP
```
여러 포트가 나왔지만 기본 AD 프로토콜인 88(kerberos), ldap 등이 확인되었다. 눈여겨본 포트는 80(http) 서비스였다.

## Initial Access
가장 먼저 접근을 시도한 서비스는 smb였다. 하지만 null credential로는 아무것도 할 수 없어서 http로 방향을 옮겼다. http에 처음 접근했을 때 Egotistical Bank라는 웹서비스를 운영 중인 것으로 보였다. 웹 Recon을 많이 시도해보았지만 디렉토리나 다른 것은 나오지 않았다. 웹사이트를 살펴보던 중 직원 이름이 나와있는 페이지가 있었고, 그 직원 이름으로 username 파일을 만든 후 유저명을 여러 조합으로 만들어주는 tool을 사용하여 많은 username 워드리스트를 만들 수 있었다.
```bash
username-anarchy -i users > real_users.txt
```
`wc -l`을 활용하여 유저명이 88개 만들어진 것을 확인한 후 AS-REP Roasting 공격을 시도해보았다.
```bash
nxc ldap DOMAIN -u real_users.txt -p '' --asreproast asrep.out
```
실행 결과 fsmith라는 유저의 hash가 나왔다. 그 hash로 hashcat을 돌려 크랙한 결과 fsmith 유저의 평문 비밀번호를 얻을 수 있었다.
```bash
hashcat -m 18200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
이후 fsmith 유저 계정으로 winrm을 시도하여 초기 접근에 성공할 수 있었다.

![fsmith](attach_real/Pasted%20image%2020260828164539.png)

## Privilege Escalation
fsmith 계정 접근 이후 여러 가지를 시도해보았다. 그중 첫 번째로 시도한 공격은 Kerberoasting이다. Kerberoasting 공격은 유저 credential 하나만 있어도 시도할 수 있으며, 이 공격으로 다른 유저의 hash를 얻을 수 있다.
```bash
impacket-GetUserSPNs DOMAIN/USER:PASS -dc-ip DC_IP -request -outputfile out
```
하지만 fsmith와 같은 권한을 가진 hsmith라는 유저의 정보가 나와 딱히 쓸모는 없었다.

이후 fsmith 계정을 사용하여 BloodHound 정보를 수집해 활용해보았지만, fsmith에서 Admin으로 가는 path가 존재하지 않아 어려움을 겪었다.

그래서 일단 fsmith 쉘 안에서 권한 상승을 먼저 시도해보았다. 가장 기본적인 취약점인 AutoLogon 자격증명 노출을 시도해보았다.

**AutoLogon**: Windows 계정의 비밀번호를 레지스트리에 평문으로 저장하면 발생하는 취약점
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```
시도한 결과 평문 비밀번호와 평문 유저명 정보가 담겨 있었다.

![AutoLogon](attach_real/Pasted%20image%2020260829010513.png)

자료와 같은 USER 이름으로 여러 프로토콜 접근을 시도해보았지만 접근이 불가능했다. 그래서 BloodHound에 대상 유저명을 검색해보니 svc_loamanager가 아니라 도메인에서는 svc_loanmgr였다. 따라서 다시 시도해본 결과 svc_loanmgr로 접근이 가능했다.

![user2](attach_real/Pasted%20image%2020260829001101.png)

이후 진짜 Domain Admin으로 로그인하기 위해 svc_loanmgr 계정을 BloodHound에서 검색한 결과, Outbound를 확인해보니 GetChanges 권한으로 도메인과 연결되어 있었다.

![bloodhound](attach_real/Pasted%20image%2020260829001529.png)

**GetChanges**: 도메인 컨트롤러끼리 데이터를 동기화하기 위해 만들어진 권한. 이를 악용하면 DCSync 공격이 가능하다.

따라서 DCSync 공격을 시도하였다.
```bash
secretsdump.py DOMAIN/USER:PASS@IP
```
이후 Admin NTLM hash를 얻게 되어, Pass-the-Hash를 사용하여 Admin 권한 상승을 할 수 있었다.

![admin](attach_real/Pasted%20image%2020260829001616.png)

## New Inform
- 유저 이름들로 wordlist를 만들 수 있다.
- BloodHound에서 바로 path가 안 나올 수 있고, Outbound를 항상 확인해야 한다.
- 도메인과 연결되어 있다고 해서 바로 Domain Admin을 얻을 수 있는 것은 아니다. Edge에 따라 달라진다.
- 항상 먼저 AutoLogon을 시도해본다.