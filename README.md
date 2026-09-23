# sj-lab-apigateway — MSA 단일 진입점

> 8개로 나뉜 sj-lab 서비스 앞단에서 **라우팅·CORS·공통 필터**를 담당하는 Spring Cloud Gateway입니다.
> 프론트엔드는 이 주소 하나만 알면 되고, 뒤쪽 서비스가 몇 개인지·어디 떠 있는지는 Eureka가 해결합니다.

| | |
|---|---|
| **운영** | `https://api.sj-lab.co.kr` |
| **로컬** | `http://localhost:8100` |
| **스택** | Java 17 · Spring Boot 3.3.2 · Spring Cloud 2023.0.3 (Gateway / WebFlux) · Eureka Client · LoadBalancer |

---

## 1. 위치와 라우팅

```
[브라우저] sj-lab.co.kr (허브) · sj-lab.co.kr/map/ (지도)
     │
     ▼
[이 서비스] :8100  ──서비스 조회──▶  [Eureka] :8761
     │  /map/**         → lb://MAPSERVICE-REST     지도·시설물 API
     │  /scheduler/**   → lb://SJ-LAB-SCHEDULER    공공데이터 수집 배치
     │  /auth/**        → lb://SJ-LAB-AUTHSERVER   로그인·JWT
     │  /fast-api-ai/** → lb://FAST-API-AI         FastAPI 서비스
     ▼
[백엔드 서비스들]
```

**경로는 벗기지 않고 그대로 전달합니다.** 각 서비스가 같은 prefix를 context-path로 쓰기 때문에, 게이트웨이에 prefix 제거 규칙을 두지 않아도 되고 서비스 단독 실행 시 경로가 달라지지 않습니다.

---

## 2. 면접에서 봐주셨으면 하는 부분

### ① 라우팅을 코드가 아니라 설정으로

라우트는 `application.yml`의 `spring.cloud.gateway.routes`에 선언합니다. 서비스가 늘어날 때 **Java 코드 변경·재컴파일 없이** 항목만 추가하면 되고, 리뷰에서 변경 범위가 한눈에 보입니다. (`FilterConfig.java`에 `RouteLocatorBuilder` 방식 예시가 주석으로 남아 있지만 비활성입니다 — 두 방식이 섞이면 어디가 진짜인지 헷갈리므로 한쪽으로 고정했습니다.)

### ② Eureka 기반 로드밸런싱

`uri: lb://SERVICE-ID` 형태라 **파드 IP·포트를 알 필요가 없습니다.** 백엔드는 `server.port: 0`(랜덤 포트)으로 떠도 되고, 인스턴스를 여러 개 띄우면 자동으로 분산됩니다. 쿠버네티스 Service가 아니라 Eureka 레지스트리를 쓰는 구조라, 로컬에서도 운영과 동일한 경로로 동작합니다.

### ③ CORS를 게이트웨이 한 곳에서

각 서비스에 CORS 설정을 흩뿌리지 않고 `globalcors`에 모았습니다. 허용 오리진은 `localhost:4000`, `sj-lab.co.kr`, `www.sj-lab.co.kr`이며, 첨부 파일명을 프론트가 읽을 수 있도록 `Content-Disposition`을 노출 헤더에 포함했습니다. `DedupeResponseHeader`로 중복 CORS 헤더도 정리합니다.

> 운영에서 실제로 겪은 것: 프론트를 다른 포트로 띄우면 **403**이 납니다. 새 도메인·포트를 추가할 때 이 목록을 함께 고치는 것을 문서 규칙으로 만들었습니다.

### ④ 프로파일 분리

`application.yml`(공통) + `application-local.yml` / `application-prod.yml`(Eureka 주소만 다름). **로컬 실행 시 `local` 프로파일이 없으면 Eureka 주소가 비어 라우팅이 죽습니다** — 이 함정을 문서와 기동 스크립트에 반영했습니다.

---

## 3. 필터

| 필터 | 적용 | 역할 |
|---|---|---|
| `GlobalFilter` | `default-filters`(전 라우트) | PRE/POST 요청 로깅 |
| `CustomFilter` | 라우트별 `filters:` | 라우트 단위 로깅 |
| `LoggingFilter` | 미사용 | `OrderedGatewayFilter` 예시 |

모두 `AbstractGatewayFilterFactory`를 상속한 `@Component`이며, **클래스명이 곧 설정에서 참조하는 이름**입니다.

---

## 4. 실행

```bash
mvnw.cmd clean package                                      # target/sj-lab-apigateway.jar
mvnw.cmd spring-boot:run -Dspring-boot.run.profiles=local   # 8100
mvnw.cmd test
```

기동 순서는 Eureka(8761) → 백엔드 → 게이트웨이 → 프론트(4000)입니다. 총괄 저장소(`mapservice-rest`)의 `scripts/local-stack.ps1`이 한 번에 띄웁니다.

동작 확인:

```powershell
Invoke-WebRequest -Uri "http://localhost:8100/map/admin-area/sido" -Headers @{Origin="http://localhost:4000"}
```

---

## 5. 배포

```
git push → Jenkins(빌드 → 이미지 push) → sj-lab-k8s-manifests 의 image.tag 자동 커밋
        → ArgoCD 동기화 → Kubernetes 롤아웃 (NodePort 30089 → 8100)
```

Dockerfile은 미리 빌드된 jar를 복사하므로 `package`가 선행되어야 합니다. 산출물명(`sj-lab-apigateway.jar`)은 `pom.xml`의 `<finalName>`과 Dockerfile 두 곳에 고정돼 있습니다.

---

## 6. 현재 한계

- **JWT 검증이 없습니다.** 로그인 서버는 별도로 있지만 게이트웨이에서 토큰을 확인하지 않아 백엔드 API가 열려 있습니다. 전역 필터로 검증 후 사용자·역할을 헤더로 전달하는 것이 다음 과제입니다.
- 레이트 리미팅·서킷 브레이커가 없습니다.
- 테스트는 컨텍스트 로딩 스모크뿐입니다.

## 참고

- 전체 구조·API 계약: 총괄 저장소 `mapservice-rest`의 `docs/system-architecture.md`
- 로컬 포트·CORS: 같은 저장소의 `docs/dev-environment.md`
- 작업 규칙: 이 저장소의 `CLAUDE.md`
