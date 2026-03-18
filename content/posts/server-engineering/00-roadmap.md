---
title: "서버 엔지니어링 로드맵 — 전체 학습 가이드"
date: 2026-03-18T09:00:00+09:00
draft: false
tags: ["로드맵", "서버 엔지니어링", "학습 가이드"]
series: ["서버 엔지니어링 로드맵"]
summary: "14개 카테고리, 75편 이상의 글로 구성된 서버 엔지니어링 완전 학습 가이드. 카테고리별 독립 구성이라 관심 분야부터 시작할 수 있다."
weight: 1
---

이 블로그는 **서버 엔지니어링의 핵심 영역을 14개 카테고리**로 나누어 다룬다. 각 카테고리는 독립적으로 구성되어 있어, **순서 없이 관심 있는 분야부터 읽으면 된다.**

단, 카테고리 **안의 글은 순서대로 읽는 것을 권장한다.** 앞 글의 개념이 뒤 글의 전제가 되는 경우가 많기 때문이다.

> 목표: **읽으면 해당 분야의 전문가가 된다.**

---

## 카테고리 한눈에 보기

| # | 카테고리 | 핵심 주제 | 편수 |
|---|---------|-----------|------|
| 1 | [OS와 네트워크](#1-os와-네트워크) | 리눅스, 프로세스, TCP/IP, TLS | 8편 |
| 2 | [서버 애플리케이션 설계](#2-서버-애플리케이션-설계) | SOLID, 디자인 패턴, API, 아키텍처 | 9편 |
| 3 | [데이터베이스](#3-데이터베이스) | 인덱스, 트랜잭션, NoSQL, 스케일링 | 9편 |
| 4 | [인증과 보안](#4-인증과-보안) | JWT, OAuth, XSS/CSRF, 암호화 | 7편 |
| 5 | [비동기 처리와 이벤트 드리븐](#5-비동기-처리와-이벤트-드리븐) | Kafka, 메시지 큐, CQRS, WebSocket | 7편 |
| 6 | [배포와 CI/CD](#6-배포와-cicd) | Git 전략, 파이프라인, 무중단 배포 | 7편 |
| 7 | [로그, 모니터링, 관측 가능성](#7-로그-모니터링-관측-가능성) | ELK, Prometheus, 분산 트레이싱 | 5편 |
| 8 | [부하, 성능, 캐싱](#8-부하-성능-캐싱) | JVM 튜닝, Redis, CDN, 부하 테스트 | 6편 |
| 9 | [분산 시스템과 마이크로서비스](#9-분산-시스템과-마이크로서비스) | 합의 알고리즘, MSA, 서비스 메시 | 7편 |
| 10 | [테스트 전략](#10-테스트-전략) | 단위/통합/E2E, TDD, 커버리지 | 4편 |
| 11 | Docker & 컨테이너 | 컨테이너, K8s, 오케스트레이션 | *준비 중* |
| 12 | AWS / 클라우드 | EC2, Lambda, VPC, IAM | *준비 중* |
| 13 | AI / ML | 모델 서빙, 벡터 DB, MLOps | *준비 중* |
| 14 | Frontend (React 등) | 컴포넌트, 상태 관리, SSR | *준비 중* |

---

## 어디서부터 읽을까?

**서버 개발 입문자라면:**
> [1] OS와 네트워크 → [2] 서버 애플리케이션 설계 → [3] 데이터베이스

**현업 백엔드 개발자라면:**
> 약한 부분부터 골라 읽자. 각 카테고리는 독립적이다.

**DevOps/인프라에 관심 있다면:**
> [6] 배포와 CI/CD → [7] 모니터링 → [11] Docker → [12] 클라우드

---

## 1. OS와 네트워크

서버의 기반이 되는 운영체제와 네트워크 원리. 프로세스, 메모리, 파일시스템, TCP/IP, TLS까지.

| 편 | 제목 |
|----|------|
| 1편 | [리눅스와 프로세스 — 서버 안에서 무슨 일이 일어나는가](/blog/posts/server-engineering/01-linux-process/) |
| 2편 | [CPU 스케줄링과 시스템 콜 — OS가 프로세스를 다루는 법](/blog/posts/server-engineering/02-cpu-scheduling-syscall/) |
| 3편 | [메모리 구조와 관리 — 서버가 메모리를 다루는 방법](/blog/posts/server-engineering/03-memory-management/) |
| 4편 | [파일 시스템과 I/O — 디스크와 서버의 대화](/blog/posts/server-engineering/04-filesystem-io/) |
| 5편 | [네트워크와 커넥션 — 서버가 요청을 받고 응답을 돌려주기까지](/blog/posts/server-engineering/05-network-connection/) |
| 6편 | [TCP 심화 — 신뢰성의 원리](/blog/posts/server-engineering/06-tcp-deep-dive/) |
| 7편 | [웹 서버와 리버스 프록시 — WAS 앞에 왜 한 계층이 더 필요한가](/blog/posts/server-engineering/07-webserver-reverse-proxy/) |
| 8편 | [TLS/HTTPS 심화 — 암호화 통신의 원리](/blog/posts/server-engineering/08-tls-https/) |

📚 추천 도서: 「리눅스 커널 이해」, 「TCP/IP Illustrated Vol.1」, 「HTTP 완벽 가이드」

---

## 2. 서버 애플리케이션 설계

좋은 서버 코드를 만드는 설계 원칙. 객체지향, 패턴, API, 아키텍처, 에러 처리, 도메인 모델링.

| 편 | 제목 |
|----|------|
| 1편 | [객체지향 설계 원칙 — SOLID가 실제로 의미하는 것](/blog/posts/server-engineering/09-solid-principles/) |
| 2편 | [디자인 패턴 실전 — 서버 코드에서 자주 만나는 패턴들](/blog/posts/server-engineering/10-design-patterns/) |
| 3편 | [동시성과 I/O 모델 — 서버가 수만 요청을 처리하는 방법](/blog/posts/server-engineering/11-concurrency-io-model/) |
| 4편 | [API 설계 — 좋은 서버 인터페이스의 조건](/blog/posts/server-engineering/12-api-design/) |
| 5편 | [API 설계 심화 — 현실의 복잡한 요구사항 다루기](/blog/posts/server-engineering/13-api-design-advanced/) |
| 6편 | [계층 구조와 의존성 — 코드를 어떻게 나누는가](/blog/posts/server-engineering/14-layered-architecture/) |
| 7편 | [모듈 시스템 설계 — 큰 코드베이스를 다루는 법](/blog/posts/server-engineering/15-module-system/) |
| 8편 | [에러 처리와 예외 전략 — 실패를 우아하게](/blog/posts/server-engineering/16-error-handling/) |
| 9편 | [도메인 모델링 — 비즈니스를 코드로 표현하는 법](/blog/posts/server-engineering/17-domain-modeling/) |

📚 추천 도서: 「Clean Architecture」, 「도메인 주도 설계」, 「Head First Design Patterns」, 「Effective Java」

---

## 3. 데이터베이스

인덱스부터 트랜잭션, 쿼리 최적화, NoSQL, 샤딩, 운영까지. 데이터를 다루는 모든 것.

| 편 | 제목 |
|----|------|
| 1편 | [인덱스 — B-Tree가 실제로 하는 일](/blog/posts/server-engineering/18-index-btree/) |
| 2편 | [트랜잭션과 락 — 동시성 제어의 깊이](/blog/posts/server-engineering/19-transaction-lock/) |
| 3편 | [쿼리 최적화 — 느린 쿼리를 빠르게](/blog/posts/server-engineering/20-query-optimization/) |
| 4편 | [데이터 모델링 — 테이블 설계의 원칙](/blog/posts/server-engineering/21-data-modeling/) |
| 5편 | [동시성과 데이터 정합성 — 실전 시나리오](/blog/posts/server-engineering/22-concurrency-consistency/) |
| 6편 | [NoSQL — 언제, 왜, 어떻게](/blog/posts/server-engineering/23-nosql/) |
| 7편 | [특수 목적 DB — 검색, 시계열, 벡터](/blog/posts/server-engineering/24-special-purpose-db/) |
| 8편 | [DB 스케일링 — 데이터베이스 확장의 현실](/blog/posts/server-engineering/25-db-scaling/) |
| 9편 | [DB 운영 — 프로덕션 DB 관리](/blog/posts/server-engineering/26-db-operations/) |

📚 추천 도서: 「데이터 중심 애플리케이션 설계(DDIA)」, 「High Performance MySQL」, 「Real MySQL 8.0」

---

## 4. 인증과 보안

세션, JWT, OAuth부터 웹 공격 방어, 암호화, 보안 감사까지.

| 편 | 제목 |
|----|------|
| 1편 | [인증 기초 — 세션, 토큰, 그 선택의 기준](/blog/posts/server-engineering/27-auth-basics/) |
| 2편 | [OAuth 2.0과 OpenID Connect — 위임 인증의 원리](/blog/posts/server-engineering/28-oauth-oidc/) |
| 3편 | [권한 관리 — 인가(Authorization) 설계](/blog/posts/server-engineering/29-authorization/) |
| 4편 | [웹 보안 공격과 방어 — 실전 위협 모델](/blog/posts/server-engineering/30-web-security/) |
| 5편 | [Rate Limiting과 DDoS 방어 — 과부하 공격 대응](/blog/posts/server-engineering/31-rate-limiting/) |
| 6편 | [데이터 보안 — 저장과 전송의 암호화](/blog/posts/server-engineering/32-data-security/) |
| 7편 | [보안 감사와 취약점 관리 — 지속적인 보안](/blog/posts/server-engineering/33-security-audit/) |

📚 추천 도서: 「OAuth 2 in Action」, 「웹 애플리케이션 보안」, OWASP Cheat Sheet Series

---

## 5. 비동기 처리와 이벤트 드리븐

메시지 큐, Kafka, 배치 처리, 분산 락, CQRS, 실시간 통신.

| 편 | 제목 |
|----|------|
| 1편 | 메시지 큐 기초 — 요청을 기다리지 않는 서버 *(작성 예정)* |
| 2편 | Kafka 심화 — 대용량 이벤트 스트리밍 *(작성 예정)* |
| 3편 | 비동기 패턴과 신뢰성 — 실패해도 괜찮은 시스템 *(작성 예정)* |
| 4편 | 스케줄러와 배치 처리 — 시간이 트리거인 작업들 *(작성 예정)* |
| 5편 | 분산 락 실전 — 여러 서버가 하나의 자원을 다투는 법 *(작성 예정)* |
| 6편 | 이벤트 소싱과 CQRS — 상태 대신 이벤트를 저장하는 설계 *(작성 예정)* |
| 7편 | 실시간 통신 — 서버에서 클라이언트로 데이터 밀기 *(작성 예정)* |

📚 추천 도서: 「Kafka: The Definitive Guide」, 「Enterprise Integration Patterns」

---

## 6. 배포와 CI/CD

Git 전략, 빌드, 아티팩트, 설정 관리, 파이프라인, 무중단 배포, IaC.

| 편 | 제목 |
|----|------|
| 1편 | Git 브랜치 전략 — 팀이 코드를 합치는 규칙 *(작성 예정)* |
| 2편 | 빌드와 패키징 — 코드가 실행 가능한 산출물이 되기까지 *(작성 예정)* |
| 3편 | 아티팩트 관리 — 빌드 산출물의 생명주기 *(작성 예정)* |
| 4편 | 설정 관리와 환경 분리 — 코드와 설정을 분리하는 법 *(작성 예정)* |
| 5편 | CI/CD 파이프라인 — 자동화된 품질 게이트 *(작성 예정)* |
| 6편 | 배포 전략 — 무중단 배포의 기술 *(작성 예정)* |
| 7편 | 환경 관리와 IaC — 인프라를 코드로 *(작성 예정)* |

📚 추천 도서: 「Continuous Delivery」, 「Infrastructure as Code」, 「The Phoenix Project」

---

## 7. 로그, 모니터링, 관측 가능성

로그 설계, ELK, 메트릭, 분산 트레이싱, 장애 진단 플로우.

| 편 | 제목 |
|----|------|
| 1편 | 로그 설계 — 추적 가능한 로그 만들기 *(작성 예정)* |
| 2편 | 로그 수집과 저장 — 분산 환경의 로그 파이프라인 *(작성 예정)* |
| 3편 | 메트릭과 모니터링 — 숫자로 서버 상태 읽기 *(작성 예정)* |
| 4편 | 분산 트레이싱 — 마이크로서비스 요청 추적 *(작성 예정)* |
| 5편 | 관측 가능성 실전 — 장애를 빠르게 찾는 법 *(작성 예정)* |

📚 추천 도서: 「Observability Engineering」, 「Site Reliability Engineering(SRE)」

---

## 8. 부하, 성능, 캐싱

JVM 튜닝, 캐싱 전략, CDN, 부하 테스트, 병목 분석, Graceful Shutdown.

| 편 | 제목 |
|----|------|
| 1편 | JVM과 성능 분석 — 서버가 느릴 때 *(작성 예정)* |
| 2편 | 캐싱 전략 — 빠른 서비스의 비밀 *(작성 예정)* |
| 3편 | HTTP 캐싱과 CDN — 네트워크 레벨의 성능 *(작성 예정)* |
| 4편 | 부하 테스트 — 서버의 한계를 미리 아는 법 *(작성 예정)* |
| 5편 | 병목 분석과 장애 대응 — 서버는 어떻게 죽고, 어떻게 버티는가 *(작성 예정)* |
| 6편 | Graceful Shutdown과 무중단 운영 *(작성 예정)* |

📚 추천 도서: 「자바 최적화」, 「Systems Performance」, 「Release It!」

---

## 9. 분산 시스템과 마이크로서비스

분산 시스템 이론, 합의 알고리즘, MSA, 서비스 메시, API Gateway, Saga 패턴.

| 편 | 제목 |
|----|------|
| 1편 | 분산 시스템의 기초 — 네트워크는 믿을 수 없다 *(작성 예정)* |
| 2편 | 분산 코디네이션과 리더 선출 — 조율의 기술 *(작성 예정)* |
| 3편 | 스케일링 전략 — 서버 한 대로는 부족할 때 *(작성 예정)* |
| 4편 | 마이크로서비스 — 분리의 이점과 대가 *(작성 예정)* |
| 5편 | 서비스 메시와 인프라 — MSA 운영의 현실 *(작성 예정)* |
| 6편 | API Gateway와 서비스 디스커버리 *(작성 예정)* |
| 7편 | 분산 트랜잭션과 멱등성 — 여러 서비스에 걸친 일관성 *(작성 예정)* |

📚 추천 도서: 「데이터 중심 애플리케이션 설계(DDIA)」, 「마이크로서비스 패턴」, 「Building Microservices」

---

## 10. 테스트 전략

단위 테스트, 통합 테스트, E2E, 계약 테스트, TDD, 테스트 문화.

| 편 | 제목 |
|----|------|
| 1편 | 단위 테스트 — 신뢰할 수 있는 코드 *(작성 예정)* |
| 2편 | 통합 테스트 — 조각을 합치면 *(작성 예정)* |
| 3편 | E2E 테스트와 계약 테스트 *(작성 예정)* |
| 4편 | 테스트 문화와 전략 — 팀으로 테스트하기 *(작성 예정)* |

📚 추천 도서: 「단위 테스트(Vladimir Khorikov)」, 「테스트 주도 개발(Kent Beck)」

---

## 11~14. 준비 중

| # | 카테고리 | 상태 |
|---|---------|------|
| 11 | Docker & 컨테이너 | 준비 중 |
| 12 | AWS / 클라우드 | 준비 중 |
| 13 | AI / ML | 준비 중 |
| 14 | Frontend (React 등) | 준비 중 |

---

*이 페이지는 새 글이 추가될 때마다 업데이트됩니다.*
