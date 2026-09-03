# HTB-ACTIVE
```bash
OS: Windows(AD)
DATE: 26.09.03
DIFFICULTY: Easy
```

## RECON
**nmap**: 가장 먼저 nmap 전체 포트 검사와 상세 포트 검사로 RECON을 시작했다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p PORTS IP
```
포트 검사 결과, 대상 시스템은 53(DNS), 88(Kerberos), 445(SMB), 389(LDAP) 등이 돌아가는 것으로 보아 AD DC IP라는 것을 알게 되었다.

**netexec**: nxc를 이용하여 타겟의 기본 정보를 살펴보았다. 주어진 credential이 없기에 null credential로 접속해본 결과, SMB 공유 목록을 볼 수 있었고 그중 Replication이라는 폴더를 읽을 수 있어 가장 먼저 접근해 보았다.
```bash
nxc smb IP -u '' -p '' --shares
smbclient //IP/Replication -N
```
타겟 SMB 폴더에 접속해본 결과 많은 디렉토리가 있어, `recurse ON` 명령어를 통해 트리 형태로 나열해서 더 보기 편하게 열거해보았다.
```bash
smbclient //IP/Replication -N -c 'recurse ON; ls'
```

## Initial Access
SMB 디렉토리를 열거해본 결과 `\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups` 폴더에 `Groups.xml`이라는 수상한 파일이 있어 다운로드 후 확인해보았다.
해당 파일은 관리자가 여러 컴퓨터의 로컬 계정 비번을 한 번에 설정하려고 만든 파일인데, 그 안에 든 비번은 누구나 아는 열쇠로 암호화되어 있어 GPP 전용 디코더로 crack할 수 있었다.
```python
python3 gpp-decrypt.py -f groups.xml
```
따라서 SVC_TGS라는 유저의 기본 계정을 얻을 수 있었다. user credential을 처음 얻고 가장 먼저 WinRM이나 RDP로 접속할 수 있는지 확인해보았지만 아무것도 접속할 수 없었다. 그래서 SMB에 있는 Users 폴더에 접근해서 user flag를 얻을 수 있었다.
![user](attach_real/Pasted%20image%2020260903182401.png)

## Privilege Escalation
권한 상승을 위해, 그다음 user credential을 이용하여 BloodHound를 실행해보았다.
```bash
bloodhound-python
bloodhound-start
```
하지만 아무런 정보나 유용한 것을 발견하지 못했다. 그다음 kerberoasting 공격을 시도해보았다. 공격을 하기 전, kerberoasting이 가능한 유저부터 확인해보았다.
```bash
GetUserSPNs.py DOMAIN/USER:PASS -dc-ip IP
```
확인해보니 administrator 계정이 나와서 바로 kerberoasting 공격을 실시했다.
```bash
GetUserSPNs.py DOMAIN/USER:PASS -dc-ip IP -request
```
따라서 Kerberos 티켓 해시를 얻을 수 있었다. 바로 hashcat을 사용하여 crack을 시도했다.
```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```
crack에 성공하여 Administrator 계정으로 접속을 시도했다. 하지만 여전히 admin 계정도 WinRM이나 RDP로 접속할 수 없어, OSCP 규정인 인터랙티브 쉘을 얻지 못했다. 따라서 SMB를 활용해 인터랙티브 쉘을 얻을 수 있는 `psexec.py`를 활용하여 최종 접근에 성공할 수 있었다.
```bash
psexec.py DOMAIN/USER:'PASS'@IP
```
![root](attach_real/Pasted%20image%2020260903182028.png)

## New Inform
- SMB 폴더를 tree 형태로 열거해서 꼭 자세히 확인하자.
- 기본 계정을 얻으면 바로 kerberoasting까지 시도하자.
- WinRM이나 RDP로 접속하지 않아도, SMB 기반으로 동작하는 psexec.py 도구를 활용해서 인터랙티브 쉘에 접속하자.
- GPP(Group Policy Preferences)라는 것을 처음 알게 되었다. 이 파일이 나오면 꼭 디코딩해서 크리덴셜을 얻자.