---
title: "1편. IP와 Port는 무엇이 다를까? — 웹 요청이 서버까지 도달하는 과정 이해하기"
date: 2026-11-12 10:00:00 +0900
categories: [Network, 백호 튜터링]
tags: [ip, port, cidr, nat, aws, vpc, tcp]
---

백호 튜터링 첫 주에는 웹서비스가 동작하기 위해 필요한 기본적인 네트워크 구조를 공부하였다.

나는 그중 IPv4와 IP, Port, Public IP와 Private IP, CIDR, NAT 등의 내용을 맡아 발표하였다.

처음에는 IP와 Port를 모두 단순히 "주소" 정도로 생각하기 쉬웠지만, 실제로는 서로 역할이 다르다.

## 1. IP와 Port의 차이

IP는 네트워크에서 어떤 Host를 찾아갈 것인지를 나타낸다.

반면 Port는 그 Host 안에서 어떤 Process와 통신할 것인지를 구분한다.

예를 들어 다음과 같은 주소가 있다고 가정할 수 있다.

```text
192.168.0.10:4000
```

여기에서

- `192.168.0.10` → 통신할 서버
- `4000` → 그 서버에서 통신할 Application

을 의미한다.

이를 단순하게 나타내면 다음과 같다.

```text
Client
  ↓
IP 주소로 서버를 찾음
  ↓
Port 번호로 Application을 찾음
```

따라서 서버가 살아 있다고 해서 특정 API까지 정상적으로 접근할 수 있다는 의미는 아니다.

서버는 실행 중이지만 해당 Port에 Process가 없을 수도 있다.

---

## 2. Public IP와 Private IP

AWS 환경을 이해하려면 Public IP와 Private IP의 차이도 중요하다.

Public IP는 인터넷에서 직접 접근할 수 있는 주소이고, Private IP는 VPC와 같은 내부 Network에서 사용하는 주소이다.

예를 들어 EC2는 내부적으로 Private IP를 가지고 있고, 외부에서 직접 접근할 필요가 있다면 Public IP를 추가로 사용할 수 있다.

| 구분 | 용도 |
| --- | --- |
| Public IP | 인터넷을 통한 외부 접근 |
| Private IP | VPC 내부 통신 |

실제 서비스에서는 모든 서버를 인터넷에 직접 노출하지 않는다.

예를 들어 Database는 외부에 공개하지 않고 Private Subnet에 배치하여 Application Server만 접근할 수 있도록 구성하는 것이 일반적이다.

---

## 3. CIDR과 Subnet

IP 주소를 하나씩 관리하는 것이 아니라 범위 단위로 관리하기 위해 CIDR을 사용한다.

예를 들어 다음과 같은 형태이다.

```text
10.0.0.0/16
```

AWS에서는 하나의 VPC 안에서도 여러 Subnet으로 Network를 나눌 수 있다.

전체적인 구조를 단순하게 정리하면 다음과 같다.

```text
AWS Region
  ↓
VPC
  ↓
Subnet
  ↓
EC2, RDS 등의 Resource
```

Public Subnet에는 외부 통신이 필요한 Resource를 배치하고, Private Subnet에는 외부에서 직접 접근할 필요가 없는 Resource를 배치할 수 있다.

---

## 4. NAT는 왜 필요한가?

Private IP는 인터넷에서 직접 사용할 수 없다.

하지만 Private Subnet의 서버도 Package 설치나 외부 API 호출처럼 인터넷으로 나가야 하는 경우가 있다.

이때 NAT를 이용할 수 있다.

```text
Private Server
  ↓
NAT
  ↓
Public Network
```

NAT는 내부의 Private IP를 외부 통신이 가능한 주소로 변환하는 역할을 한다.

이 과정을 공부하면서 "외부에서 들어오는 통신"과 "내부 서버가 외부로 나가는 통신"을 구분해서 생각할 수 있게 되었다.

---

## 5. TCP 연결까지 생각하기

웹서비스의 요청 흐름을 단순하게 표현하면 다음과 같다.

```text
Browser
  ↓
DNS
  ↓
IP 확인
  ↓
TCP 연결
  ↓
Port
  ↓
Application
```

따라서 장애가 발생했을 때도 한 번에 "서버가 안 된다"고 판단하기보다 어느 단계까지 정상인지 확인해야 한다.

예를 들어 다음과 같은 결과가 있다.

```text
Connection refused
```

이 경우 목적지 Host까지는 도달했지만 해당 Port에서 연결을 받아주는 Process가 없을 가능성이 있다.

반면

```text
Connection timed out
```

은 일정 시간 동안 응답 자체를 받지 못한 상황이므로 중간 Network, Security Group, Firewall 등을 확인해야 할 수 있다.

---

## 6. 실제 장애 대응과 연결하기

이후 튜터링에서 AWS와 Docker 기반 서비스를 직접 배포하고 장애를 만들어 보면서 1주차에 공부한 IP와 Port 개념을 계속 사용하였다.

예를 들어 서버 내부에서 다음 명령어를 사용할 수 있다.

```bash
ss -lntp
```

이 명령어를 사용하면 현재 서버에서 어떤 Port가 실제로 LISTEN 중인지 확인할 수 있다.

다른 서버의 Port까지 연결할 수 있는지는 다음과 같이 확인하였다.

```bash
nc -zv HOST PORT
```

처음에는 IP, Port, CIDR, NAT를 각각 따로 공부했지만 실제 웹서비스를 구축하고 장애를 해결해 보면서 이 개념들이 하나의 요청 흐름 안에서 모두 연결되어 있다는 것을 알 수 있었다.

특히 장애 대응에서는 "어떤 설정이 틀렸는가?"를 바로 찾기보다

> **"요청이 어느 구간까지 정상적으로 도달했는가?"**

를 확인하는 것이 중요하다는 점을 배울 수 있었다.
