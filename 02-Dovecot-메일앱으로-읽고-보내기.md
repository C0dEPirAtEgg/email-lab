# 02. Dovecot 추가: 메일 앱으로 읽고 보내기

> 목표: Mac의 **Thunderbird**로 `alice@corp-a.test` 계정에 접속해 메일을 읽고, 보낸다.
> 01단계에서는 서버 터미널의 `mail` 명령으로만 메일을 다룰 수 있었다.
> 이번에는 **"사용자가 메일 앱으로 접속하는 입구"**를 만든다.

---

## 0. 이번에 만들 구조

```
                          Mac (호스트)
                       ┌──────────────┐
                       │ Thunderbird  │
                       └──┬────────┬──┘
           ① 읽기: IMAP 143 │        │ ② 보내기: Submission 587 (로그인 필요)
                          ▼        ▼
 ┌────────────── mail-a ──────────────┐            ┌──── mail-b ────┐
 │  Dovecot ◀── 비밀번호 확인 ──▶ Postfix │ ─SMTP 25─▶ │ Postfix        │
 │     │                         │     │            │    │           │
 │     └──── ~/Maildir/ ◀────────┘     │            │ ~/Maildir/     │
 └────────────────────────────────────┘            └────────────────┘
```

| 포트 | 프로그램 | 하는 일 |
|---|---|---|
| 25 (SMTP) | Postfix | **서버 ↔ 서버** 메일 전달 (01단계에서 만듦) |
| **143 (IMAP)** | **Dovecot** | 메일 앱이 메일함을 **읽음** |
| **587 (Submission)** | **Postfix + Dovecot** | 메일 앱이 메일을 **보냄** (로그인한 사람만) |

### 이번 단계에서 바뀌는 것

| 항목 | 01단계 | 02단계 |
|---|---|---|
| 메일함 형식 | `/var/spool/mail/bob` (mbox, 파일 하나) | `~/Maildir/` (메일 1통 = 파일 1개) |
| 메일 읽기 | 서버에서 `mail` 명령 | **어디서든 메일 앱 (IMAP)** |
| 메일 보내기 | 서버에서 `mail` 명령 | **메일 앱 → 587 포트 (로그인 필요)** |
| 비밀번호 | 필요 없었음 | **필요함** (리눅스 계정 비밀번호) |

> ⚠️ 이번 단계는 **암호화 없이(평문)** 만든다. 일부러 그렇게 한다.
> 6장에서 비밀번호가 네트워크에 그대로 보이는 걸 직접 확인하고, **04단계(TLS)**에서 해결한다.

---

## 1. 시작 전: 스냅샷 찍기

설정을 망쳐도 01단계 완료 상태로 바로 돌아갈 수 있게 해 둔다.

1. Fusion에서 `mail-a` 선택 → 메뉴 **Virtual Machine → Snapshots…**
2. **Take Snapshot** → 이름: `01-완료`
3. `mail-b`도 똑같이

> 💡 앞으로 단계를 시작할 때마다 스냅샷을 찍어 두는 습관을 들이자.

---

## 2. 서버 설정 (mail-a, mail-b 둘 다)

> 🔑 **01단계에서 설정을 호스트 이름 기반으로 만들어 둔 덕분에, 아래 명령은 두 서버에서 한 글자도 다르지 않다.**
> mail-a에서 끝까지 진행하고, mail-b에서 **똑같이** 반복하면 된다.

### 2-1. 메일함을 Maildir 형식으로 바꾸기

```bash
# 받은 메일을 각 사용자 홈의 Maildir/ 폴더에 저장 (끝의 / 가 Maildir이라는 뜻)
postconf -e 'home_mailbox = Maildir/'
systemctl restart postfix
```

| 형식 | 구조 | 특징 |
|---|---|---|
| mbox | 메일함 = **파일 1개**에 모든 메일을 이어 붙임 | 단순하지만, 여러 프로그램이 동시에 쓰면 잠금 문제가 생김 |
| **Maildir** | 메일함 = **폴더**, 메일 1통 = **파일 1개** | 동시 접근에 안전, IMAP 서버에서 표준 |

> 01단계에서 받은 메일은 `/var/spool/mail/`에 그대로 남아 있다. 이제부터 오는 메일만 Maildir에 쌓인다.

### 2-2. Dovecot 설치

```bash
dnf install -y dovecot
dovecot --version
```

> ℹ️ 이 문서는 **Dovecot 2.3** 기준이다. 버전이 `2.4.x`로 나오면 아래 2-3 설정의 이름이 일부 다르다.
>
> | 2.3 | 2.4 |
> |---|---|
> | `disable_plaintext_auth = no` | `auth_allow_cleartext = yes` |
> | `mail_location = maildir:~/Maildir` | `mail_driver = maildir` + `mail_path = ~/Maildir` |
>
> 2.4는 설정 파일 맨 위에 `dovecot_config_version` 줄도 필요하다.

### 2-3. Dovecot 설정

Dovecot의 기본 설정 파일은 `/etc/dovecot/dovecot.conf` + `conf.d/` 폴더의 파일 여러 개로 나뉘어 있다.
`dovecot.conf`의 맨 끝에서 **`local.conf`를 마지막으로 읽기** 때문에, 바꿀 설정을 이 파일 하나에 모아 두면 관리가 편하다.

```bash
cat > /etc/dovecot/local.conf <<'EOF'
# ── 02단계: 이메일 실습 설정 ──

# IMAP만 사용 (POP3는 쓰지 않음)
protocols = imap

# 모든 네트워크 주소에서 접속 받기
listen = *

# 메일함 위치: 각 사용자 홈의 Maildir (Postfix의 home_mailbox와 반드시 같아야 함)
mail_location = maildir:~/Maildir

# ⚠️ 실습용: 암호화 없이 비밀번호 로그인 허용 (04단계에서 되돌림)
ssl = no
disable_plaintext_auth = no

# 로그인 방식 (Thunderbird, Outlook 등이 쓰는 표준 방식)
auth_mechanisms = plain login

# Postfix가 "이 비밀번호 맞아?"라고 물어볼 수 있는 통로 (소켓)
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
EOF
```

설정 확인 후 시작:

```bash
doveconf -n        # 기본값과 다른 설정만 출력 → 위에서 넣은 값이 보이면 OK
systemctl enable --now dovecot
```

> 💡 **사용자 정보는 어디서 오나?**
> Dovecot은 기본적으로 **리눅스 계정(PAM)**으로 로그인을 확인한다.
> 그래서 `useradd`로 만든 alice, bob이 그대로 메일 계정이 된다.

### 2-4. Postfix에 "메일 앱용 입구(587)" 만들기

**왜 포트 25를 그대로 쓰지 않나?**

- 포트 25는 **다른 메일 서버가 우리에게 메일을 배달하는 문**이다. 상대 서버는 로그인하지 않는다.
- 포트 587은 **우리 사용자가 메일을 내보내는 문**이다. **반드시 로그인해야** 한다.
- 문을 나누지 않으면, 아무나 우리 서버를 통해 외부로 메일을 보낼 수 있게 된다 (**오픈 릴레이** → 스팸 발송기로 악용됨).

```bash
# ① Postfix가 비밀번호 확인을 Dovecot에게 맡기도록 연결
postconf -e 'smtpd_sasl_type = dovecot'
postconf -e 'smtpd_sasl_path = private/auth'

# ② master.cf에 587 포트(submission) 서비스 추가
postconf -M submission/inet='submission inet n - n - - smtpd'

# ③ 587 포트 전용 규칙: 로그인 켜기 + 로그인한 사람만 통과
postconf -P 'submission/inet/syslog_name=postfix/submission'
postconf -P 'submission/inet/smtpd_sasl_auth_enable=yes'
postconf -P 'submission/inet/smtpd_client_restrictions=permit_sasl_authenticated,reject'
postconf -P 'submission/inet/smtpd_relay_restrictions=permit_sasl_authenticated,reject'
```

| 명령 | 수정하는 파일 | 의미 |
|---|---|---|
| `postconf -e` | `main.cf` | Postfix **전체** 설정 |
| `postconf -M` | `master.cf` | **서비스(포트)** 추가 |
| `postconf -P` | `master.cf` | **특정 서비스에만** 적용할 설정 |

확인 후 재시작:

```bash
postconf -M submission/inet     # submission 서비스 줄이 보이면 OK
postconf -P | grep submission   # 위에서 넣은 4줄이 보이면 OK
postfix check && systemctl restart postfix
ls -l /var/spool/postfix/private/auth   # Dovecot이 만든 소켓 파일이 있으면 OK
```

### 2-5. 방화벽 열기

```bash
firewall-cmd --permanent --add-service=imap              # 143
firewall-cmd --permanent --add-service=smtp-submission   # 587
firewall-cmd --reload
firewall-cmd --list-services     # smtp imap smtp-submission 이 보이면 OK
```

### 2-6. 메일 계정 비밀번호 정하기

01단계에서는 비밀번호 없이 `su -`로만 들어갔다. 이제는 원격 로그인이 필요하다.

```bash
# mail-a 에서
passwd alice       # 예: alice123

# mail-b 에서
passwd bob         # 예: bob123
```

> 실습용이라도 **영문과 숫자만** 쓰자. 4장에서 손으로 base64 인코딩할 때 편하다.

### 2-7. mail-b에서도 똑같이

`mail-b`에 접속해서 **2-1 ~ 2-5**를 그대로 반복한다. (2-6은 위에 적힌 대로 bob만)

---

## 3. 새 메일 하나 보내 두기

Maildir로 바꾼 뒤 메일함이 비어 있으니, 테스트용 메일을 하나 보낸다.

```bash
# mail-a 에서
su - alice
echo "Maildir로 가는 첫 메일" | mail -s "Hello Maildir" bob@corp-b.test
exit
```

```bash
# mail-b 에서: 메일이 파일 1개로 저장됐는지 확인
ls /home/bob/Maildir/new/
cat /home/bob/Maildir/new/*
```

| Maildir 안의 폴더 | 의미 |
|---|---|
| `tmp/` | 받는 중인 메일 (쓰기가 끝나면 `new/`로 이동) |
| `new/` | 아직 아무 메일 앱도 확인하지 않은 메일 |
| `cur/` | 메일 앱이 한 번이라도 확인한 메일 |

---

## 4. 메일 앱이 하는 일을 손으로 해 보기

01단계에서 SMTP를 직접 쳐 봤던 것처럼, 이번에는 **IMAP**과 **로그인이 있는 SMTP**를 직접 쳐 본다.

### 4-1. IMAP으로 메일 읽기 (mail-b에서)

```bash
nc localhost 143
```

IMAP 명령은 앞에 **아무 꼬리표**(`a1`, `a2`…)를 붙인다. 서버는 같은 꼬리표로 답한다.

```
→ * OK [CAPABILITY ...] Dovecot ready.
a1 LOGIN bob bob123
→ a1 OK Logged in
a2 LIST "" "*"
→ * LIST (\HasNoChildren) "." INBOX
a3 SELECT INBOX
→ * 1 EXISTS                      ← 메일 1통
→ a3 OK [READ-WRITE] Select completed.
a4 FETCH 1 (BODY[HEADER.FIELDS (FROM SUBJECT)])
→ From: alice@corp-a.test
→ Subject: Hello Maildir
a5 FETCH 1 BODY[TEXT]
→ Maildir로 가는 첫 메일
a6 LOGOUT
```

다시 `ls /home/bob/Maildir/`를 보면, 메일이 `new/`에서 `cur/`로 옮겨져 있다.
**IMAP으로 한 번 확인했기 때문이다.**

### 4-2. Submission(587)으로 메일 보내기 (mail-a에서)

먼저 **로그인 없이** 보내 본다. 거절돼야 정상이다.

```bash
nc localhost 587
```

```
→ 220 mail.corp-a.test ESMTP Postfix
EHLO test
→ 250-AUTH PLAIN LOGIN           ← "로그인할 수 있어요"라는 뜻
MAIL FROM:<alice@corp-a.test>
RCPT TO:<bob@corp-b.test>
→ 554 5.7.1 ... Client host rejected: Access denied    ← ✅ 로그인 안 하면 거절
QUIT
```

이번에는 **로그인해서** 보낸다. `AUTH PLAIN`은 `\0아이디\0비밀번호`를 base64로 인코딩해서 보낸다.

```bash
printf '\0alice\0alice123' | base64
# → AGFsaWNlAGFsaWNlMTIz
```

```bash
nc localhost 587
```

```
→ 220 mail.corp-a.test ESMTP Postfix
EHLO test
AUTH PLAIN AGFsaWNlAGFsaWNlMTIz
→ 235 2.7.0 Authentication successful    ← ✅ 로그인 성공
MAIL FROM:<alice@corp-a.test>
→ 250 2.1.0 Ok
RCPT TO:<bob@corp-b.test>
→ 250 2.1.5 Ok
DATA
→ 354 End data with <CR><LF>.<CR><LF>
Subject: sent via 587

로그인하고 보낸 메일
.
→ 250 2.0.0 Ok: queued as XXXX
QUIT
```

> 💡 base64는 **암호화가 아니다.** 누구나 되돌릴 수 있다.
> `echo AGFsaWNlAGFsaWNlMTIz | base64 -d` 를 쳐 보자.

---

## 5. Thunderbird로 접속하기 (Mac)

### 5-1. 설치

https://www.thunderbird.net 에서 받아 설치한다.

### 5-2. 계정 추가 (alice)

1. Thunderbird 실행 → **계정 설정 → 새 계정 → 기존 이메일**
2. 이름: `Alice` / 이메일: `alice@corp-a.test` / 비밀번호: `alice123`
3. 자동 설정은 실패한다 (DNS가 없으므로 정상). **수동 설정(Configure manually)** 을 선택한다

| 항목 | 받는 서버 (IMAP) | 보내는 서버 (SMTP) |
|---|---|---|
| 서버 | **mail-a의 IP** (예: `192.168.64.10`) | **mail-a의 IP** |
| 포트 | `143` | `587` |
| 연결 보안 | **없음 (None)** | **없음 (None)** |
| 인증 방식 | **일반 비밀번호** | **일반 비밀번호** |
| 사용자 이름 | `alice` | `alice` |

4. **완료** → "암호화되지 않은 연결" 경고 → **위험을 이해함**에 체크하고 진행

> 💡 Mac에는 `corp-a.test`라는 이름이 등록되어 있지 않아서 **서버 주소는 IP로** 입력한다.
> 메일 주소(`alice@corp-a.test`)와 서버 주소(IP)는 **별개**라는 점이 포인트다.

### 5-3. bob 계정도 추가

같은 방법으로 `bob@corp-b.test`를 추가한다. 서버 IP만 **mail-b의 IP**로 바꾼다.

### 5-4. 주고받기

1. alice 계정에서 **새 메일 쓰기** → 받는 사람: `bob@corp-b.test` → 보내기
2. bob 계정의 받은편지함에 메일이 도착하는지 확인
3. bob 계정에서 **답장**

```
Thunderbird(alice) ─587→ mail-a Postfix ─25→ mail-b Postfix → ~bob/Maildir ─143→ Thunderbird(bob)
```

**한 통의 메일이 포트 3개를 거쳐 간다.** 이 흐름이 이번 단계의 핵심이다.

---

## 6. 비밀번호가 다 보인다 👀

지금 구성의 **가장 큰 문제**를 직접 확인해 본다. mail-a에서:

```bash
dnf install -y tcpdump
tcpdump -i any -A -n 'port 143 or port 587'
```

이 상태에서 Thunderbird의 alice 계정에서 **받기**를 누른다.

```
... a1 LOGIN alice alice123 ...                  ← IMAP: 비밀번호가 그대로
... AUTH PLAIN AGFsaWNlAGFsaWNlMTIz ...          ← SMTP: base64라 바로 해독 가능
```

같은 네트워크(카페 와이파이 등)에 있는 누구든 **비밀번호와 메일 내용을 그대로 볼 수 있다.**
→ 이 문제를 **04단계(TLS)**에서 해결한다.

`Ctrl + C`로 tcpdump를 종료한다.

---

## 7. 문제가 생겼을 때

로그부터 본다.

```bash
journalctl -u dovecot -f        # 로그인 실패, 메일함 오류
journalctl -u postfix -f        # 전송, SASL 오류
```

| 증상 | 확인할 것 |
|---|---|
| Thunderbird: 서버에 연결할 수 없음 | `firewall-cmd --list-services`, `ss -tlnp | grep -E ':143|:587'` |
| IMAP 로그인 실패 (`AUTHENTICATE failed`) | `passwd`로 비밀번호를 설정했는지, `doveconf -n`에 `disable_plaintext_auth = no`가 있는지 |
| `Plaintext authentication disallowed` | `local.conf`의 `ssl = no`, `disable_plaintext_auth = no` 확인 후 `systemctl restart dovecot` |
| 메일함이 비어 보임 | Postfix `home_mailbox = Maildir/` ↔ Dovecot `mail_location = maildir:~/Maildir` 이 같은지 |
| 587에서 `AUTH`가 안 보임 | `postconf -P | grep submission`, `ls -l /var/spool/postfix/private/auth` |
| `SASL authentication failure: cannot connect to Dovecot` | Dovecot이 실행 중인지, 소켓 경로가 `private/auth`로 일치하는지 |
| 보내기: `Relay access denied` | Thunderbird SMTP 포트가 25가 아니라 **587**인지, 인증을 켰는지 |

---

## ✅ 이번 단계에서 배운 것

- 메일 서버에는 **입구가 3개** 있다: 서버끼리(25), 메일 읽기(143), 메일 보내기(587)
- **Postfix는 배달**, **Dovecot은 메일함 열람과 로그인 확인**을 맡는다
- Postfix는 비밀번호를 직접 확인하지 않고 **Dovecot에게 물어본다** (SASL)
- 보내는 문(587)에 로그인을 걸어야 **오픈 릴레이**가 되지 않는다
- 메일함 형식은 **Maildir**(메일 1통 = 파일 1개)가 IMAP과 잘 맞는다
- 지금은 **비밀번호와 메일이 평문**으로 오간다

## ⏭️ 다음 단계 예고

- **03**: `/etc/hosts`를 DNS 서버(BIND) + MX 레코드로 교체하기
- **04**: 사설 CA로 인증서를 만들어 143 → 993(IMAPS), 587에 TLS 적용 → 6장의 엿보기 막기
