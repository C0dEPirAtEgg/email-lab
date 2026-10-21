# email-lab

VMware Fusion과 Rocky Linux 10으로 이메일 인프라를 단계별로 구축하며 배우는 실습 가이드.

## 실습 환경

- 가상화: VMware Fusion (기본 NAT 네트워크)
- OS: Rocky Linux 10
- 도메인: `corp-a.test`, `corp-b.test` (테스트 전용 예약 TLD)

## 단계

| 단계 | 내용 | 문서 |
|---|---|---|
| 01 | Postfix만으로 VM 2대 사이에 메일 주고받기 (VM 1대 구축 후 복제) | [01-가장-가벼운-메일-서버.md](./01-가장-가벼운-메일-서버.md) |
| 02 | Dovecot 추가: Thunderbird로 읽고(IMAP) 보내기(Submission) | [02-Dovecot-메일앱으로-읽고-보내기.md](./02-Dovecot-메일앱으로-읽고-보내기.md) |
| 03 | BIND DNS 서버 + MX 레코드 | 예정 |
| 04 | 사설 CA + TLS 암호화 | 예정 |
| 05 | SPF / DKIM / DMARC | 예정 |
| 06 | Rspamd 스팸 필터 | 예정 |
| 07 | Roundcube 웹메일 (선택) | 예정 |
