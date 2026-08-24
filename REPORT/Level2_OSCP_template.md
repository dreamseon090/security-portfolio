# [머신명] - YYYY.MM.DD - Difficulty (Easy/Medium/Hard)

## 1. Recon
- Nmap: `nmap -sC -sV -p- -oA nmap/full <IP>`
- Open ports:
  - PORT/tcp  SERVICE  VERSION
- Notable:
  - (뭐가 수상했나 + 왜 수상한지 해석까지)

## 2. Enumeration
- (스캔/열거 명령어 + 결과)
- Why this direction:
  - (여러 진입점 중 왜 이걸 골랐나 = 나의 사고 흐름)

## 3. Initial Foothold
- Vulnerability: (취약점 이름 + 전환 흐름)
- Payload (단계별 분해):
  - 1단계: `...`  ← 이 줄이 하는 일
  - 2단계: `...`  ← 이 줄이 하는 일
- How it works:
  - (payload가 왜 작동하는지 원리)
- Stuck point:
  - (막힌 부분 + 몇 분 걸렸나 + 뭘로 뚫었나)

## 4. Privilege Escalation
- Enum: (linpeas / find -perm 등)
- 발견: (SUID / sudo / cron / 커널 등)
- Method: `...`
- Why it works: (원리)
- Proof: `cat /root/proof.txt && ip addr`  ← OSCP 규칙

## 5. Lessons Learned
- (~하면 → ~해봐 형태의 재사용 지식)

## 6. Rabbit Holes
- (안 됐던 시도 + 이유 + 날린 시간 + 교훈)

<!-- 한국어 회고 메모 영역 (나만 봄, 최종엔 삭제)
     - 오늘 제일 오래 막힌 부분:
     - 다음에 고칠 습관:
-->