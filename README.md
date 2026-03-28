# AWS 기반 멀티 레이어 헬스체크 시스템

인프라·네트워크·애플리케이션·의존성 4개 레이어를 독립적으로 점검하고
통합 헬스 스코어를 산출하는 AWS 네이티브 경량 헬스체크 도구입니다.

## 서비스 개요

기존 모니터링 도구(Pingdom, Datadog Synthetics 등)는 **외부에서 HTTP 응답 코드와 레이턴시**를 확인합니다.
하지만 실제 운영 환경에서는 응답 코드가 정상이어도 서비스가 느리거나, 일부 기능만 망가져 있거나, 특정 의존성만 죽어 있는 **회색 장애**가 발생합니다.

이 시스템은 AWS VPC 내부에서 **4개 레이어를 독립적으로 점검**하고, 각 레이어에 가중치를 부여해 **통합 헬스 스코어**를 실시간으로 산출합니다.
또한 **k6 부하 테스트**와 연동하여 트래픽이 증가할 때 어느 레이어가 먼저 점수 하락을 보이는지 **병목 원인을 레이어 단위로 추적**합니다.


### 대상 시스템 구성

```
Client → ALB → App Server (ECS/EC2) → RDS
```

## 해결하는 문제

기존 모니터링의 한계

대부분의 헬스체크는 이 한 가지 질문만 합니다.

>"지금 서버가 살아 있나요?"

하지만 실제 장애는 훨씬 교묘합니다. 서버는 살아 있고, 응답 코드는 200 OK이며, 외부 모니터링은 모두 초록불입니다. 그런데 사용자는 느리다고 느낍니다. 특정 기능은 조용히 실패합니다. 결제가 안 되는데 에러 페이지도 뜨지 않습니다.
이런 상태를 **회색 장애(Gray Failure)** 라고 부릅니다. 완전히 죽은 것도 아니고, 완전히 살아 있는 것도 아닌 시스템은 정상이라고 보고하지만 실제로는 망가진 상태입니다.

저희는 이 회색 장애를 탐지하여 시스템 문제를 해결하고자 합니다.

## 외부 도구가 탐지하지 못하는 이유

외부 모니터링 도구는 **AWS VPC 내부 리소스에 구조적으로 접근이 불가**합니다.

- Private VPC 내 RDS, ElastiCache, 내부 MSA 직접 접근 불가
- VPC 서브넷 간 연결 단절, Security Group 묵시적 차단 확인 불가
- 응답 본문의 오염(빈 JSON, 스키마 불일치) 탐지 불가
- 인증이 필요한 내부 API 엔드포인트 smoke test 불가

이 시스템은 **Lambda를 인프라 내부에 배치**하여 위 예시와 같은 항목을 직접 점검합니다.

## 레이어 구조

각 레이어는 **독립적으로 점수를 산출**하며, 가중 평균으로 통합 헬스 스코어를 계산합니다.

주요 점검 항목은 추후 확정 예정

| Layer | 이름 | 가중치 | 주요 점검 항목 | 구현 방식 |
|-------|------|--------|----------------|-----------|
| **Layer 1** | Infra | - | EC2 Status Check, CPU/메모리 사용률, CPU Steal | CloudWatch Metrics |
| **Layer 2** | Network | - | DNS 응답, TCP 연결, TLS 인증서, 내부 레이턴시 | Lambda Probe + CloudWatch |
| **Layer 3** | Application | - | `/health` 응답, P99 레이턴시, 5xx 에러율, 응답 스키마 검증 | Lambda Probe + CloudWatch |
| **Layer 4** | Dependency | - | RDS 커넥션 풀, 쿼리 레이턴시, Replica Lag, RDS CPU | Lambda Probe + CloudWatch |

### Checker 유형

- **Metric Checker**: CloudWatch 등 AWS API를 호출하여 기존 메트릭을 조회
- **Probe Checker**: Lambda에서 직접 HTTP 요청, TCP 연결, SQL 쿼리 등을 수행

### 점수 산출 공식

```
통합 헬스 스코어 = 계산중
```

| 점수 | 판정 | 알림 |
|------|------|------|
| 80 ~ 100 | ✅ Healthy | 없음 |
| 60 ~ 79 | 🟡 Degraded | CloudWatch 메트릭 발행 |
| 40 ~ 59 | 🟠 Warning | SNS → Slack 알림 |
| 0 ~ 39 | 🔴 Critical | SNS → Slack 즉시 알림 |

> **Critical Override**: 어느 Checker라도 critical 발생 시 해당 레이어 최대 30점으로 제한.
> Layer 4가 20점 미만이면 통합 점수 최대 40점으로 제한.

## 기술 스택

| 영역 | 서비스 |
|------|--------|
| 스케줄링 | Amazon EventBridge |
| 오케스트레이션 | AWS Step Functions |
| Checker 실행 | AWS Lambda  |
| 메트릭 조회 | Amazon CloudWatch |
| 결과 저장 | Amazon DynamoDB |
| 알림 | Amazon SNS + Slack Webhook |
| 대시보드 | Amazon CloudWatch Dashboard |
| 부하 테스트 | k6 |

## 역할 분담

### 공통 역할 

| 항목 | 담당 |
|------|------|
| 대상 시스템 최소 범위 정의 | 전체 |
| Step Functions → Lambda 입력 형식 | 예은 |
| Checker 공통 응답 형식 | 선우 & 윤호 |
| 상태값 규약 | 선우 |
| 점수 계산 기준 및 가중치 | 윤호 |

### 역할 1 — 대상 시스템

> **ALB → App → RDS** 구성 및 Checker probe 구현

- 대상 앱 구축 (ECS / EC2)
- `/health` 엔드포인트 구현
- `/health/deep` 엔드포인트 구현
- RDS 연결 및 샘플 데이터베이스 구성
- 장애 유도 로직 
- App Checker probe 일부 (`/health` 호출)
- Dependency Checker probe 일부 

### 역할 2 — 헬스체크 엔진

> **Step Functions 기반 오케스트레이션**

- Amazon EventBridge 스케줄 설정
- AWS Step Functions 워크플로우 설계 및 구현
- Lambda 공통 프로젝트 구조 (폴더 구조, 공통 모듈)
- Checker 실행 방식 및 입출력 contract enforcement

### 역할 3 — 점수화 / 저장 / 리포트

> **점수 산출, 결과 저장, 알림, 대시보드 구축**

- Score Calculator Lambda 구현 (가중 평균 + Override 규칙)
- DynamoDB 테이블 설계 및 저장 로직
  - `hc-run-scores`: 런별 통합/레이어 점수
  - `hc-checker-results`: 개별 Checker 결과
- SNS Topic + Slack Webhook 알림 파이프라인
- CloudWatch Custom Metrics 발행
- CloudWatch Dashboard 구성
- 추세 분석 기초 데이터 구조 (k6 상관관계 분석용)

## 기여자

| 이름 | 역할 |
|------|------|
| [**서예은**](https://github.com/michelle259) | 대상 시스템 (ALB · App · RDS) 및 App/Dependency Checker probe |
| [**최선우**](https://github.com/seonwoochoi24) | 헬스체크 엔진 (Step Functions · EventBridge) |
| [**최윤호**](https://github.com/yunhoch0i)| 점수화 · 저장 · 알림 · 대시보드 (Score Calculator · DynamoDB · SNS) |

### 시스템 아키텍처
