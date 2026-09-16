# HTB-KEEPER
```bash
OS: Linux
DATE: 26.09.15
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 검사를 위해 전체 포트 검사와 상세 포트 검사를 실시했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80 IP
```
검사 결과 22번 SSH 서비스와 80번 HTTP 서비스가 실행 중이었다. 초기 페이지를 들어가보니 `ticket.keeper.htb` 서브도메인으로 이동하는 하이퍼링크가 있었다. 따라서 `/etc/hosts`에 도메인을 추가해주었다.
```bash
sudo nano /etc/hosts

IP ticket.keeper.htb keeper.htb
```

## Initial Access
서브도메인에 접속해본 결과 request tracker라는 서비스가 실행 중이었다. 대상 시스템의 버전은 4.4.4라는 것을 알게 되었고, 바로 취약점을 검색해보았다. 하지만 딱히 사용할 수 있는 것은 없었다. 따라서 default credential을 검색해보았다. **request tracker** 서비스는 root:passwd가 기본 자격 증명이라는 것을 알게 되어 바로 시도해본 결과 root로 로그인이 가능했다. 서비스를 계속해서 정찰해본 결과 lnorgaard 유저의 패스워드를 알 수 있게 되었다. 따라서 바로 SSH 접속을 시도해본 결과 유저 쉘을 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020260915194816.png)

## Privilege Escalation
처음 쉘을 얻고 유저 디렉토리에 **RT30000.zip**이라는 zip 파일이 있어서 로컬로 옮겨 unzip해주었다.
```bash
scp USER@IP:~/RT30000.zip .
```
```bash
unzip RT30000.zip
```
unzip을 해본 결과 **passcodes.kdbx, KeePassDumpFull.dmp** 두 가지 파일을 얻을 수 있었다. 두 파일이 keepass라는 비밀번호 관리 프로그램과 관련된, 성격이 서로 다른 파일이라는 것을 알게 되었다. 일단 kdbx 데이터베이스 파일부터 살펴보았다. 해당 파일을 Linux에서 실행하기 위해 kdbx를 리눅스에서 돌릴 수 있는 tool을 설치하여 실행해보았지만, 암호가 걸려 있어 아무것도 할 수 없었다. 따라서 KeePassDumpFull.dmp를 살펴보기 위해 똑같이 tool이 필요한지 알았지만, dmp는 메모리 바이트 덩어리이기 때문에 strings 도구로 사람이 읽을 수 있는 문자열만 간단히 추출할 수 있었다. 그런데 너무나도 많은 내용에 알 수 없는 것들이 많아 어떻게 할지 생각해보다가, keepass를 구글에 검색해본 결과 CVE-2023-32784라는 취약점이 있다는 것을 알게 되었다. 해당 취약점은 마스터 패스워드를 입력받는 순간 각 글자의 흔적을 메모리에 남기는 취약점이다. 그래서 그 시점의 메모리, 즉 .dmp를 확보하면 패스워드를 거의 복원할 수 있는 취약점이었다. 따라서 Keepass-Dumper라는 CVE-2023-32784 취약점 PoC를 찾아서 실행해본 결과 dgr(d,e) med flde라는 알 수 없는 문자를 얻을 수 있었다.
```bash
python3 keepass_dump.py -f KeePassDumpFull.dmp --skip --debug

*] 0:  {UNKNOWN}                                              
[*] 2:  d                                                      
[*] 3:  g                                                      
[*] 4:  r                                                      
[*] 6:  <{d, e}>                                               
[*] 7:                                                         
[*] 8:  m                                                      
[*] 9:  e                                                      
[*] 10: d                                                      
[*] 11:                                                        
[*] 12: f                                                      
[*] 13: l                                                      
[*] 15: d                                                      
[*] 16: e                                                      
[*] Extracted: {UNKNOWN}dgr<{d, e}> med flde   
```
따라서 `dgrd med flde`를 먼저 kdbx 도구를 사용하여 마스터키로 시도해본 결과 실패하여, 구글에 검색해본 결과 **rødgrød med fløde**라는 덴마크의 전통적인 과일 푸딩 디저트 이름이라는 것을 알게 되었다. 그래서 바로 다시 마스터키로 시도했지만 또 실패했다. 도구에 문제가 있다고 생각하여 **kpcli**라는 도구도 사용해보았지만, 여전히 공백이나 덴마크 언어를 인식하지 못하는 문제가 있어 **keepassxc**라는 GUI 도구를 사용하여 데이터베이스 접속이 가능했다.

```bash
keepassxc passcodes.kdbx
```
데이터베이스를 살펴보던 중 root 유저 항목에 Putty-User-Key-file이 있었다.
![key](attach_real/Pasted%20image%2020260916193217.png)

해당 키는 .ppk 확장자, 즉 putty private key이다. Putty 계열은 Windows이기 때문에 OpenSSH 계열과 맞지 않아 포맷을 변환해줘야 한다. .ppk를 OpenSSH 포맷으로 변환해주면 ssh -i가 읽을 수 있다. 따라서 puttygen이라는 도구를 사용하여 포맷을 바꿔주었다.
```bash
puttygen root.ppk -O private-openssh -o id_rsa
```
또한 SSH 접속을 위해 권한을 낮춰줘야 한다. 따라서 권한 600으로 id_rsa를 설정하고 SSH를 통해 root로 접속이 가능했다.
```bash
chmod 600 id_rsa
```
```bash
ssh -i id_rsa root@IP
```
이로써 root 쉘을 얻게 되었다.
![root](attach_real/Pasted%20image%2020260916153619.png)


## New Inform
- 취약점을 찾아보기 전에 default credential을 먼저 찾아보자.
- 웹 정찰을 먼저 하자.
- scp는 SSH 위에서 파일을 복사하는 명령어: **scp [옵션] 원본 목적지**
- keepass: 오픈소스 비밀번호 관리 프로그램.
- .kdbx: keepass 데이터베이스 파일(금고 본체). 모든 비밀번호가 저장된 암호화 컨테이너. 마스터 패스워드 없이는 데이터를 볼 수 없다.
- .dmp: 메모리 덤프 파일. 프로그램 실행 중 특정 시점의 RAM 내용을 통째로 파일로 떠낸 것.
- 문자열이 RAM에서 어떻게 남는가? 프로그래밍에서 글자 묶음을 문자열이라고 한다. KeePass는 C#, .NET이라는 언어로 만들어져 있어, 여기에 결정적인 특성이 있다. **.NET의 문자열은 한번 만들어지면 절대 수정할 수 없다.** `Pa`에 글자 하나를 더해 `Pas`로 만들고 싶으면 Pa를 수정하는 게 아니라, `Pas`라는 완전히 새 문자열을 옆에 새로 만든다. `Pa`는 사라지지 않고 RAM에 그대로 남아 아무도 안 쓰는 쓰레기가 된 채로, 가비지컬렉터라는 청소부가 나중에 치운다. 근데 "나중에"가 핵심, 즉시가 아니라 한참 뒤 청소할 때까지 잔재가 RAM에 떠돈다. 따라서 마스터키를 한 글자씩 칠 때마다 KeePass의 입력창이 화면 표시를 갱신하면서 매번 새 문자열을 만든다. 공격자는 .dmp에서 이 쓰레기들을 전부 긁어모아 길이순으로 정렬한다. 이렇게 자리마다 한 글자씩 복원되어 결국 덴마크 언어가 나온 것이다.
- 모르는 게 있으면 바로 구글링하자.
- PPK: PuTTY Private Key. putty라는 프로그램이 개인키를 저장하는 고유 파일 포맷. putty는 Windows에서 SSH 접속할 때 쓰는 대표적인 프로그램.
- puttygen: OpenSSH 포맷 변환기.
- id_rsa 권한 600.