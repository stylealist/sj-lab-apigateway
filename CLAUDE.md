# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Spring Cloud Gateway 기반 API 게이트웨이 서비스(`sj-lab-apigateway`)입니다. Eureka에 등록된 백엔드 마이크로서비스들(`MAPSERVICE-REST`, `SJ-LAB-SCHEDULER`, `FAST-API-AI` 등) 앞단에서 라우팅, 필터링, CORS 처리를 담당합니다. Java 17 + Spring Boot 3.3.2 + Spring Cloud 2023.0.3 (WebFlux 기반 리액티브 스택).

## 자주 사용하는 명령어

Windows 환경이므로 `mvnw.cmd`를 사용합니다.

```
mvnw.cmd clean package          # 빌드 (target/sj-lab-apigateway.jar 생성)
mvnw.cmd test                   # 전체 테스트 실행
mvnw.cmd test -Dtest=ApigatewayServiceApplicationTests   # 단일 테스트 실행
mvnw.cmd spring-boot:run -Dspring-boot.run.profiles=local   # local 프로파일로 로컬 실행 (포트 8100)
```

active profile을 지정하지 않으면 `spring.profiles.active`가 비어 있어 Eureka 등록 주소가 설정되지 않으므로, 로컬 실행 시 반드시 `local` 프로파일을 지정해야 합니다.

Docker 빌드/실행:
```
mvnw.cmd clean package
docker build -t sj-lab-apigateway .
docker run -p 8100:8100 -e SPRING_PROFILES_ACTIVE=prod sj-lab-apigateway
```

## 아키텍처

- **라우팅은 Java 코드가 아닌 `application.yml`에 선언**되어 있습니다 (`src/main/resources/application.yml`의 `spring.cloud.gateway.routes`). `FilterConfig.java`는 Java `RouteLocatorBuilder` 방식의 예시 코드지만 전체가 주석 처리되어 비활성 상태이며, 새 라우트를 추가할 때는 이 파일이 아니라 `application.yml`을 수정합니다.
- 각 라우트는 `uri: lb://SERVICE-ID` 형태로 Eureka 서비스명을 참조하고 (`spring-cloud-starter-loadbalancer`가 로드밸런싱), `Path=/xxx/**` predicate와 `CustomFilter`, `PreserveHostHeader` 필터를 공통으로 사용합니다.
- **커스텀 필터 3종** (`src/main/java/.../filter/`)은 모두 `AbstractGatewayFilterFactory`를 상속한 `@Component`이며, 클래스명이 곧 `application.yml`에서 참조하는 필터 이름입니다.
  - `GlobalFilter` — `default-filters`로 모든 라우트에 전역 적용, PRE/POST 로깅
  - `CustomFilter` — 개별 라우트에서 `filters:`로 명시 지정
  - `LoggingFilter` — `OrderedGatewayFilter` 사용 예시 코드로 구현되어 있으나 현재 어떤 라우트에서도 참조되지 않는 미사용 필터
- CORS는 `spring.cloud.gateway.globalcors`에 전역 설정되어 있고, 허용 오리진이 하드코딩되어 있으므로(`localhost:4000`, `sj-lab.co.kr` 등) 새 프론트엔드 도메인 추가 시 이 목록을 갱신해야 합니다.

## 참고

- **프로파일 구조**: `application.yml`(공통 서버/게이트웨이 설정) + `application-local.yml`(Eureka `localhost:8761`) + `application-prod.yml`(Eureka `eureka.sj-lab.co.kr`). 두 프로파일 파일은 Eureka `defaultZone`만 다르게 정의합니다.
- **포트**: 8100 고정.
- **빌드 산출물명**: `pom.xml`의 `<finalName>` 및 Dockerfile에서 `sj-lab-apigateway.jar`로 고정되어 있으므로 아티팩트 이름을 바꾸려면 두 곳을 함께 수정해야 합니다.
- **테스트**: `ApigatewayServiceApplicationTests`뿐이며 컨텍스트 로딩만 검증하는 스모크 테스트입니다.

## README 유지 규칙

- **이 저장소에 기능·API·화면·실행 방법·설정(환경변수/시크릿)·배포 방식이 추가되거나 바뀌면, 같은 작업에서 `README.md`도 함께 갱신할 것.** 코드만 고치고 README를 그대로 두지 말 것.
- 갱신 대상 예: 새 엔드포인트·화면·모듈, 빌드/실행 명령 변경, 포트·의존 서비스 변경, 환경변수·Secret 추가, 배포 절차 변경, 해결한 이슈·새로 생긴 한계.
- **README는 면접관·처음 보는 사람이 읽는 문서**다(이 프로젝트는 포트폴리오). 사용자·리뷰어 관점의 설명(무엇을·왜·어떻게 확인하는지)은 README에, 에이전트/내부 작업 규칙은 이 문서(CLAUDE.md)에 둔다.
- 문구가 실제 코드와 어긋나지 않는지 확인하고, 구현되지 않은 기능을 적지 말 것. 한계·미구현 항목은 숨기지 말고 "현재 한계"에 적는다.

## 통합 허브

저장소를 넘나드는 작업(DB → 백엔드 → 디스커버리 → 게이트웨이 → 프론트엔드)의 총괄 기준 저장소는 `C:\developer\workspace\mapservice-rest`입니다. 시스템 전체 구조·API 계약은 그 저장소의 `docs/system-architecture.md`, 로컬 포트·기동 순서·CORS는 `docs/dev-environment.md`에 있고, MCP(GitHub/DB)와 로컬 비밀값도 그 저장소에서만 관리합니다.