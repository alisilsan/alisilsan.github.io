---
title: "3편. DNS도 되고 3306도 열려 있는데 DB 연결이 실패한다면? — RDS 비밀번호 장애 진단"
date: 2026-11-12 12:00:00 +0900
categories: [Troubleshooting, 백호 튜터링]
tags: [aws, rds, mysql, docker, nestjs, access-denied]
---

다음으로 직접 만들어 본 장애는 RDS Database의 인증정보가 잘못된 상황이다.

사용자가 처음 확인하는 증상은 다음과 같았다.

| 기능 | 상태 |
| --- | --- |
| 웹사이트 화면 | ✅ 정상 |
| 게시글 조회 | ❌ 실패 |
| Database를 사용하는 기능 | ❌ 실패 |

이 장애의 핵심은 Network 연결 실패와 Database 인증 실패를 구분하는 것이었다.

---

## 1. 장애 구성

Backend는 RDS에 접속하기 위해 다음과 같은 정보를 사용한다.

```text
DB_HOST
DB_PORT
DB_USER
DB_PASSWORD
DB_NAME
```

이번에는 이 중 `DB_PASSWORD`만 실제 RDS Password와 다르게 설정하였다.

즉 전체 상황은 다음과 같다.

```text
Backend
  ↓
RDS DNS ✅
  ↓
RDS :3306 ✅
  ↓
MySQL Server ✅
  ↓
Password ❌
```

---

## 2. 먼저 실제 API 장애 확인

사용자가 실패했다고 느끼는 기능은 게시글 조회였다.

따라서 먼저 해당 API를 직접 호출하였다.

```bash
curl -v http://localhost/api/posts
```

응답은 502 또는 API 실패로 나타날 수 있다.

이 단계에서는 아직 원인을 알 수 없다.

알 수 있는 것은 "실제로 API 장애가 발생하고 있다." 정도이다.

---

## 3. Backend 상태 확인

다음으로 Container를 확인하였다.

```bash
docker compose ps
```

Backend Container가 Up이라고 해도 바로 정상이라고 판단하면 안 된다.

```text
Backend Container Up
  ≠ NestJS Application 정상
  ≠ RDS 연결 정상
```

따라서 실제 Backend 로그를 확인해야 한다.

---

## 4. Backend 로그에서 Access denied 발견

다음 명령어를 사용하였다.

```bash
docker compose logs backend --tail 100
```

로그에서 다음과 같은 메시지를 확인할 수 있다.

```text
Access denied for user
```

이 문구는 매우 강한 단서이다.

MySQL Server까지 아예 도달하지 못했다면 `Access denied`라는 응답을 받을 수 없기 때문이다.

즉,

```text
Backend
  ↓
RDS 도달 ✅
  ↓
MySQL 응답 ✅
  ↓
인증 ❌
```

이라고 생각할 수 있다.

---

## 5. 그래도 계층별로 확인해 보기

학습을 위해 DNS, TCP, 인증을 각각 분리해서 확인하였다.

먼저 RDS Endpoint가 정상적으로 해석되는지 확인하였다.

```bash
getent hosts <RDS-ENDPOINT>
```

정상이라면 IP 주소가 반환된다. → **DNS ✅**

다음으로 RDS 3306번 Port까지 연결할 수 있는지 확인하였다.

```bash
nc -zvw 5 <RDS-ENDPOINT> 3306
```

결과:

```text
succeeded
```

즉, **DNS ✅ / TCP :3306 ✅** 이다.

Network 문제일 가능성이 크게 줄어든다.

---

## 6. Backend가 어떤 설정을 사용하고 있는지 확인

Backend의 DB 설정도 확인한다.

```bash
docker compose exec backend printenv | grep -E 'DB_HOST|DB_PORT|DB_USER|DB_NAME'
```

여기서 Host, Port, User, Database Name 등이 의도한 값인지 확인하였다.

단, Password는 민감정보이므로 화면에 그대로 출력하지 않도록 주의하는 것이 좋다.

---

## 7. MySQL에 직접 로그인해 보기

Network와 RDS 자체가 정상인지 확인하기 위해 올바른 계정으로 직접 MySQL 로그인을 시도하였다.

```bash
mysql -h <RDS-ENDPOINT> -P 3306 -u <USER> -p
```

올바른 Password를 입력하면 정상적으로 접속할 수 있었다.

이 결과를 통해 다음을 알 수 있다.

| 확인 항목 | 결과 |
| --- | --- |
| RDS Server | ✅ |
| Network | ✅ |
| User | ✅ |
| 올바른 Password를 사용한 인증 | ✅ |

그런데 Backend에서만 `Access denied`가 발생한다.

따라서 Backend가 사용하는 인증정보를 의심할 수 있다.

---

## 8. 최종 원인

최종 원인은 Backend에 설정된 DB Password가 실제 RDS Password와 다르다는 것이었다.

즉,

```text
RDS 장애             ❌
DNS 장애             ❌
Security Group 장애  ❌
TCP 3306 장애        ❌

Backend DB Password 설정 오류 ✅
```

였다.

올바른 Password로 Backend 설정을 수정하고 Container를 다시 적용한 뒤 게시글 조회 기능을 재확인하였다.

---

## 9. Access denied가 알려주는 것

이번 실습에서 가장 중요했던 부분은 Error Message가 이미 많은 정보를 제공한다는 점이었다.

`Access denied`가 나타났다는 것은 적어도 다음 단계까지는 정상일 가능성이 높다.

```text
DNS
  ↓
TCP
  ↓
MySQL Server
  ↓
Authentication ❌
```

반면 다음과 같은 오류가 발생했다면 조사 위치가 달라질 수 있다.

| Error | 의심할 지점 |
| --- | --- |
| Connection timed out | Network / Security Group 가능성 |
| Connection refused | Port Listener 가능성 |
| Access denied | Authentication 가능성 |
| Unknown database | 연결과 인증 이후 Database Name 문제 가능성 |

즉 Error를 단순한 실패 메시지로 보는 것이 아니라,

> **"이 메시지를 받으려면 앞에서 어떤 단계들이 성공해야 했을까?"**

라고 생각하는 것이 중요했다.

---

## 10. 전체 장애 진단 흐름

이번 장애를 처음부터 정리하면 다음과 같다.

```text
게시글 조회 실패
  ↓
API 직접 호출
  ↓
실패 확인
  ↓
Backend 로그
  ↓
Access denied
  ↓
"RDS까지 못 간 건가? 아니면 갔는데 인증이 실패한 건가?"
  ↓
DNS 확인
  ↓
정상
  ↓
TCP 3306 확인
  ↓
정상
  ↓
MySQL 직접 로그인
  ↓
올바른 Password로 정상
  ↓
Backend 인증정보 문제
  ↓
원인 확정
```

이번 실습을 통해 Database 장애도 단순히 "DB가 안 된다"라고 생각해서는 안 된다는 것을 알 수 있었다.

DNS, TCP, MySQL Server, Authentication, Database Name을 각각 다른 단계로 나누어 확인하면 어디에서 처음 실패했는지를 훨씬 빠르게 찾을 수 있었다.

특히 `Access denied`라는 로그를 보고 Network 장애가 아니라 인증 단계까지 도달한 문제라고 판단하는 과정이 가장 기억에 남았다.
