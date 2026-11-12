---
title: "2편. 서버 내부에서는 정상인데 외부에서만 Timeout — Security Group 장애 직접 만들어 보기"
date: 2026-11-12 11:00:00 +0900
categories: [Troubleshooting, 백호 튜터링]
tags: [aws, ec2, security-group, nginx, docker, timeout]
---

튜터링 후반부에는 팀원들이 각자 장애를 하나씩 만들어 다른 사람이 원인을 모르는 상태에서 문제를 해결하는 실습을 진행하였다.

내가 만든 장애 중 하나는 EC2의 Security Group에서 HTTP 80번 Port를 차단하는 것이었다.

사용자에게 처음 주어지는 증상은 다음과 같다.

```text
외부 PC에서 사이트 접속
  → 응답 없음
  → Timeout
```

하지만 EC2 내부에서는

```bash
curl http://localhost
```

를 실행하면 정상적으로 `200 OK`가 반환된다.

겉으로는 단순히 사이트가 열리지 않는 문제이지만, 이 두 결과를 비교하면 장애 범위를 크게 줄일 수 있다.

---

## 1. 장애를 어떻게 만들었는가?

EC2 Security Group의 Inbound Rule에서 HTTP 80번 허용 규칙을 제거하였다.

정상 상태에서는 다음과 같은 흐름이다.

```text
Client
  ↓
Security Group
  ↓
EC2 :80
  ↓
Nginx
  ↓
Application
```

하지만 장애 상태에서는 다음과 같이 된다.

```text
Client
  ↓
Security Group ❌
  ↓
EC2
```

외부 Packet이 EC2까지 도달하지 못한다.

---

## 2. 먼저 사용자의 증상을 재현한다

외부에서 서비스에 접근하면 Timeout이 발생한다.

그런데 이것만 보고 Application 문제라고 결론 내리면 안 된다.

먼저 EC2 내부에서 직접 요청한다.

```bash
curl -v http://localhost
```

결과:

```text
HTTP/1.1 200 OK
```

이 결과는 매우 중요한 단서이다.

EC2 내부에서 Application이 정상적으로 응답하고 있다는 뜻이기 때문이다.

---

## 3. Container도 확인한다

다음으로 Container 상태를 확인하였다.

```bash
docker compose ps
```

결과:

```text
nginx      Up
frontend   Up
backend    Up
```

Container는 모두 살아 있다.

하지만 여기서 중요한 점이 있다.

```text
Container Up  ≠  외부 사용자가 서비스에 접근 가능
```

Container는 서버 내부의 상태일 뿐 AWS Network 설정까지 정상이라는 뜻은 아니다.

---

## 4. 실제로 80번 Port가 열려 있는지 확인

다음 명령어를 이용하였다.

```bash
ss -lntp | grep :80
```

결과:

```text
LISTEN 0 4096 0.0.0.0:80
```

즉 Nginx는 실제로 80번 Port에서 요청을 받을 준비가 되어 있다.

여기까지 확인한 결과를 정리하면 다음과 같다.

| 확인 항목 | 결과 |
| --- | --- |
| EC2 | ✅ |
| Docker | ✅ |
| Nginx | ✅ |
| Port 80 LISTEN | ✅ |
| localhost 요청 | ✅ |
| 외부 요청 | ❌ |

따라서 Application 내부보다 EC2 앞의 Network 계층으로 조사 범위를 옮길 수 있다.

---

## 5. 외부 TCP 연결 확인

외부에서 다음 명령어를 사용할 수 있다.

```bash
nc -zv <EC2-IP> 80
```

장애 상태에서는 일정 시간이 지난 뒤 Timeout이 발생한다.

이 결과는 Connection refused와 다르다.

| 결과 | 의미 |
| --- | --- |
| Connection refused | 목적지까지 도달했지만 해당 Port에서 연결을 받아주지 않음 |
| Connection timed out | Packet에 대한 응답 자체를 받지 못함 |

이번에는 EC2 내부에서 Port가 정상적으로 LISTEN 중이기 때문에 Host의 Process 문제가 아니다.

외부 Packet이 EC2까지 도달하지 못하고 있다고 판단할 수 있다.

---

## 6. Security Group 확인

마지막으로 AWS Console에서 EC2의 Security Group Inbound Rule을 확인하였다.

HTTP 80번 규칙이 존재하지 않았다.

최종 원인은 다음과 같다.

> **Security Group의 HTTP 80 Inbound Rule 누락**

이 규칙을 다시 추가하였다.

```text
Type     : HTTP
Protocol : TCP
Port     : 80
Source   : 허용 Source 설정
```

설정한 뒤 다시 외부에서 접속하였다.

이번에는 정상적으로 웹페이지가 나타났다.

---

## 7. 장애 진단 흐름 정리

이번 장애의 전체 진단 과정을 정리하면 다음과 같다.

```text
외부 사이트 접속 실패
  ↓
Timeout
  ↓
EC2 내부 curl
  ↓
200 OK
  ↓
Container 확인
  ↓
모두 Up
  ↓
ss로 80번 확인
  ↓
LISTEN 정상
  ↓
"서버 내부는 정상"
  ↓
외부 Network 확인
  ↓
Security Group 80번 누락
  ↓
원인 확정
```

이 실습에서 가장 크게 배운 것은 "외부에서 접속이 안 된다"고 해서 Application 설정부터 볼 필요가 없다는 점이었다.

내부에서 정상이라는 것을 먼저 증명하면 확인해야 할 범위가 자연스럽게 AWS Network 앞단으로 좁혀진다.

즉 장애 대응에서 중요한 것은 많은 설정을 한꺼번에 확인하는 것이 아니라,

> **"마지막으로 정상인 지점이 어디인가?"**

를 찾는 것이라고 느꼈다.
