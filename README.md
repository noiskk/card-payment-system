# MSA 기반 카드 결제 시스템

> POS부터 VAN, 카드사, 은행까지 실제 카드 결제 프로세스를 구현하고,<br>
> 카드사 내부를 MSA로 분리하여 분산 환경에서 발생하는 데이터 정합성 문제를 해결한 프로젝트입니다.

우리FISA 팀 프로젝트를 기반으로 개인 프로젝트로 재설계했습니다.<br>
카드사 내부를 **Gateway · Payment · Ledger · FDS** 서비스로 분리하고,<br>
**STAN 기반 멱등성, 망취소 대사, 보상 트랜잭션**을 적용해 분산 환경에서도 일관된 거래 처리를 구현했습니다.

<br>

---

# 프로젝트 개요

### 구현 범위

- POS → VAN → 카드사 → 은행까지 카드 결제 프로세스 구현
- 카드사 내부 MSA 아키텍처 설계
- ISO 8583 기반 전문 처리
- Spring Cloud Gateway · Eureka · OpenFeign 기반 서비스 연동

### 중점적으로 고민한 내용

- STAN 기반 멱등성으로 중복 결제 방지
- 망취소 대사를 통한 데이터 정합성 확보
- 원장 분리와 보상 트랜잭션
- BIN 기반 카드사 라우팅
- 점수 기반 이상거래 탐지(FDS)

<br>

---

# 아키텍처

<img width="700" alt="01_시스템구성도" src="https://github.com/user-attachments/assets/2e3c0fce-dccd-448b-8c7c-9e52f14a3ba8" />
<br>
시스템 구성도

<br><br>

<img width="700" alt="02_승인처리흐름" src="https://github.com/user-attachments/assets/0463684d-c42a-425d-b6eb-04bd6ebaec42" />
<br>
승인 처리 흐름

<br><br>

<img width="700" alt="03_시퀀스다이어그램" src="https://github.com/user-attachments/assets/6bbda2fc-a44f-45af-81a6-2542ed541e06" />
<br>
시퀀스 다이어그램

<br>

---

# 서비스 구성

| 서비스 | 포트 | 소속 | 역할 | 저장소 |
|---|---|---|---|---|
| pos-client | 6060 | 가맹점 | POS 단말 에뮬레이터 (ISO 8583 전문 생성·TCP 전송) | [noiskk/pos-client](https://github.com/noiskk/pos-client) |
| VAN-service | 7070 / TCP 7777 | VAN사 | 전문 수신·검증·카드사 라우팅 | [noiskk/van-service](https://github.com/noiskk/van-service) |
| card-gateway | 9000 | 카드사 | 내부 단일 진입점 (Spring Cloud Gateway) | [noiskk/card-service](https://github.com/noiskk/card-service) |
| eureka-server | 8761 | 카드사 | 내부 서비스 레지스트리 | [noiskk/card-service](https://github.com/noiskk/card-service) |
| card-payment-service | 9091 | 카드사 | 승인 오케스트레이션 · 멱등성 · 보상 · 대사 배치 | [noiskk/card-service](https://github.com/noiskk/card-service) |
| card-fds-service | 9090 | 카드사 | 이상거래 판정 | [noiskk/card-service](https://github.com/noiskk/card-service) |
| ledger-service | 9094 | 카드사 | 승인 원장 · 정산 배치 | [noiskk/card-service](https://github.com/noiskk/card-service) |
| bank-service | 8080 | 은행 | 계좌 · 출금 · 취소 · 동시성 제어 | [noiskk/bank-service](https://github.com/noiskk/bank-service) |

<br>

---

# 핵심 설계

<br>

## STAN 기반 멱등성

단말이 생성하는 STAN을 멱등키로 사용해 동일한 결제가 여러 번 처리되지 않도록 구현했습니다.
UNIQUE 제약을 활용해 데이터베이스 수준에서 중복 결제를 방지했습니다.

<br>

## 망취소 대사

은행 호출이 타임아웃되면 거래를 실패로 처리하지 않고 불확실 거래(PENDING)로 저장합니다.
Spring Batch를 이용한 대사를 통해 거래를 최종 확정하거나 취소하며 장부와 실제 계좌 잔액의 정합성을 유지했습니다.

> **"모른다를 실패로 기록하지 않는다."**

<br>

## 원장 분리와 보상 트랜잭션

원장을 독립 서비스로 분리하면서
"출금 성공 + 원장 기록 실패"라는 분산 시스템 문제를 명시적으로 다루게 되었습니다.

원장 기록 실패 시 보상 트랜잭션을 수행해 서비스 간 데이터 일관성을 유지했습니다.

<br>

## BIN 기반 카드사 라우팅

카드번호 앞 6자리(BIN)를 기반으로 발급 카드사를 판별하도록 구현했습니다.
라우팅 정보는 설정으로 관리하고 최장 일치 방식을 적용해 카드사를 선택하도록 설계했습니다.

<br>

---

# 트러블슈팅

| 문제 | 해결 |
|------|------|
| 실패 기록이 롤백됨 | `REQUIRES_NEW`로 독립 트랜잭션 적용 |
| Spring Batch가 일부 데이터를 건너뜀 | Keyset Paging 적용 |
| Reader 상태가 실행 간 유지됨 | `@StepScope` 적용 |
| Feign이 정상 거절을 예외 처리 | 비즈니스 오류를 응답코드로 통일 |

<br>

---

# 기술 스택

- Java 17
- Spring Boot
- Spring Cloud Gateway
- Netflix Eureka
- OpenFeign
- Spring Data JPA
- Spring Batch
- MySQL (H2 Profile)
- ISO 8583
- Thymeleaf

<br>

---

# 참고 자료

- https://toss.tech/series/payments-legacy
