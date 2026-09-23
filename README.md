# sj-lab-apigateway — Spring Cloud 기반 MSA API 게이트웨이

`sj-lab-apigateway`는 sj-lab 분산 마이크로서비스 생태계의 단일 진입점(Single Point of Entry) 역할을 수행하는 Spring Cloud Gateway 서비스입니다. 클라이언트 요청의 지능적 라우팅, 로드밸런싱, 중앙 집중식 CORS 제어 및 공통 요청/응답 필터링을 담당합니다.

---

## 1. 서비스 역할 및 핵심 책임

- **단일 진입점 및 지능형 라우팅**: 모든 외부 클라이언트(웹 프론트, 모바일 등)의 API 요청을 단일 호스트(`api.sj-lab.co.kr` / `:8100`)로 수신하여 목적지 마이크로서비스로 전달합니다.
- **동적 서비스 디스커버리 연동**: Spring Cloud Netflix Eureka와 연동하여 개별 서비스의 물리적 IP와 포트를 하드코딩하지 않고 서비스 ID(`lb://SERVICE-ID`) 기반 클라이언트 사이드 로드밸런싱을 수행합니다.
- **중앙 집중식 CORS 정책 관리**: 분산된 개별 백엔드 서비스 대신 게이트웨이 계층에서 공통 CORS 정책을 일괄 처리하여 도메인 간 리소스 공유 보안과 헤더 일관성을 보장합니다.
- **공통 트래픽 로깅 및 필터 파이프라인**: 전역 필터(`GlobalFilter`) 및 라우트별 커스텀 필터를 통해 인입되는 HTTP 트래픽의 모니터링, 처리 시간 측정, 진단 로그를 남깁니다.

---

## 2. 기술 스택

- **언어 및 프레임워크**: Java 17, Spring Boot 3.3.2, Spring Cloud 2023.0.3 (Spring Cloud Gateway, Spring WebFlux)
- **서비스 디스커버리 & 로드밸런싱**: Spring Cloud Netflix Eureka Client, Spring Cloud LoadBalancer
- **네트워크 런타임**: Project Reactor Netty (Non-blocking Reactive I/O)
- **배포 환경**: Docker, Kubernetes (NodePort 30089), Helm, Jenkins CI, ArgoCD (GitOps)

---

## 3. 라우팅 구조 및 트래픽 처리 프로세스

### 3.1 요청 흐름도

```
[클라이언트 브라우저] (sj-lab.co.kr / :4000)
       │
       ▼ HTTPS / HTTP 요청
[sj-lab-apigateway] (:8100)
  ├── 1. Global Pre Filter (요청 수신 및 추적 로깅)
  ├── 2. Global CORS Filter (Origin 검증 및 헤더 주입)
  ├── 3. Route Matching & Eureka Service Resolution
  │      ├─ /map/**         ──> lb://MAPSERVICE-REST     (지도/시설물 GeoJSON API)
  │      ├─ /auth/**        ──> lb://SJ-LAB-AUTHSERVER   (인증/JWT 발급)
  │      ├─ /scheduler/**   ──> lb://SJ-LAB-SCHEDULER    (공공데이터 수집 배치)
  │      └─ /fast-api-ai/** ──> lb://FAST-API-AI         (FastAPI AI/RAG 서비스)
  └── 4. Global Post Filter (응답 코드 로깅 및 DedupeResponseHeader 정리)
       │
       ▼ 로드밸런싱 포워딩
[해당 마이크로서비스 Pod]
```

### 3.2 투명 경로 전달(Transparent Path Forwarding)
- 게이트웨이는 URL prefix를 벗겨내지 않고 수신된 context-path를 그대로 백엔드에 전달합니다.
- 모든 백엔드 서비스는 자신의 도메인 접두어(`/map`, `/auth` 등)를 애플리케이션 context-path로 내장하고 있어, 게이트웨이 경유 여부와 무관하게 로컬 단독 테스트 및 통합 테스트 시 동일한 API 경로를 유지할 수 있습니다.

---

## 4. 핵심 엔지니어링 구현 상세

### 4.1 선언적 라우팅 구성 (Configuration-Driven Routing)
라우팅 룰을 자바 코드가 아닌 `application.yml`의 `spring.cloud.gateway.routes` 선언으로 표준화하여 관리합니다. 새 서비스 추가나 라우트 조건 변경 시 코드 재컴파일 없이 설정 갱신만으로 배포할 수 있습니다.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: mapservice-rest
          uri: lb://MAPSERVICE-REST
          predicates:
            - Path=/map/**
        - id: sj-lab-authserver
          uri: lb://SJ-LAB-AUTHSERVER
          predicates:
            - Path=/auth/**
        - id: sj-lab-scheduler
          uri: lb://SJ-LAB-SCHEDULER
          predicates:
            - Path=/scheduler/**
        - id: fast-api-ai
          uri: lb://FAST-API-AI
          predicates:
            - Path=/fast-api-ai/**
```

### 4.2 중앙 집중식 글로벌 CORS 제어 및 중복 방지
프론트엔드(`localhost:4000`, `sj-lab.co.kr`, `www.sj-lab.co.kr`)의 교차 출처 리소스 요청을 게이트웨이 계층에서 안전하게 처리합니다.
- 파일 다운로드 지원을 위해 `Content-Disposition` 헤더를 `exposed-headers`에 명시.
- 백엔드 서비스와 게이트웨이 간 중복 발생할 수 있는 CORS 헤더를 `default-filters`의 `DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_FIRST`로 정리하여 브라우저의 다중 헤더 거부 오류를 방지.

### 4.3 Spring WebFlux 기반의 논블로킹(Non-blocking) 파이프라인
Netty 기반의 이벤트 루프 모델을 사용하여 스레드 블로킹 없이 대규모 동시 연결을 최소한의 시스템 리소스로 처리합니다.

---

## 5. 실행 및 개발 환경

### 로컬 빌드 및 실행
```powershell
# Maven 빌드
mvnw.cmd clean package

# 로컬 프로파일 실행 (Eureka: localhost:8761 연동, 포트 8100)
mvnw.cmd spring-boot:run -Dspring-boot.run.profiles=local
```

> **주의**: 로컬 환경 구동 시 반드시 `-Dspring-boot.run.profiles=local`을 지정해야 `localhost:8761`의 Eureka 서버와 정상 통신합니다.

### 라우팅 및 CORS 동작 검증
```powershell
Invoke-WebRequest -Uri "http://localhost:8100/map/admin-area/sido" -Headers @{Origin="http://localhost:4000"}
```
