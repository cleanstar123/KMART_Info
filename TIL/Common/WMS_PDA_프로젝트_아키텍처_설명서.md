# WMS PDA 프로젝트 아키텍처 설명서

#wms #pda #flutter #architecture #kmart #guide

---

## 목차

1. [프로젝트 아키텍처 개요](#1-프로젝트-아키텍처-개요)
2. [아키텍처 분류와 설계 의도](#2-아키텍처-분류와-설계-의도)
3. [최상위 폴더 구조 설명](#3-최상위-폴더-구조-설명)
4. [계층별 책임과 데이터 흐름](#4-계층별-책임과-데이터-흐름)
5. [폴더별 상세 관리 기준](#5-폴더별-상세-관리-기준)
6. [기능 모듈별 권장 구조](#6-기능-모듈별-권장-구조)
7. [현재 구조에 대한 해석과 보완 권장사항](#7-현재-구조에-대한-해석과-보완-권장사항)
8. [파일 배치 원칙](#8-파일-배치-원칙)
9. [최종 정리](#9-최종-정리)

---

# 1. 프로젝트 아키텍처 개요

본 프로젝트의 `lib/` 폴더 구조는 Flutter 기반 PDA 애플리케이션에서 많이 사용하는
**Feature-First + Layered Architecture + 공통 Core 분리 구조**로 해석할 수 있다.

즉, 아키텍처를 한 문장으로 정리하면 아래와 같다.

> **업무 기능별(feature)로 모듈을 나누고, 각 기능 내부에서는 data / domain / presentation 계층으로 책임을 분리하며, 프로젝트 전역 공통 기능은 core와 shared로 분리한 구조**

현재 구조는 WMS PDA 프로젝트에서 필요한 아래 요구사항에 잘 맞는다.

- 오프라인 우선(Offline-First)
- 로컬 DB 활용
- 네트워크/API 통신 분리
- 스캐너 연동
- 동기화(sync) 구조
- 기능별 독립 개발
- 유지보수성과 확장성 확보

이 구조는 Flutter PDA 프로젝트 시작 가이드에서 제안한 `app / core / features / shared` 중심 구조와 같은 방향성을 가진다.

---

# 2. 아키텍처 분류와 설계 의도

## 2.1 아키텍처 분류

현재 프로젝트는 다음 3가지 관점이 결합된 구조이다.

### 1) Feature-First Architecture
기능 중심으로 폴더를 나누는 방식이다.

예:
- `features/auth`
- `features/home`
- `features/picking`
- `features/product_info`

이 방식은 WMS처럼 기능이 많은 시스템에서 매우 유리하다.
각 기능이 독립적으로 진화할 수 있고, 화면/상태/API 관련 코드가 기능 단위로 모이기 때문이다.

### 2) Layered Architecture
기능 내부를 역할에 따라 계층으로 분리하는 방식이다.

예:
- `data`: 외부 데이터 처리
- `domain`: 비즈니스 규칙, 모델, 추상화
- `presentation`: 화면/UI/상태관리

이 구조는 인증, 피킹, 입고검수, 재고조사처럼 비즈니스 흐름이 있는 업무에 적합하다.

### 3) Clean Architecture 지향 구조
완전한 엄격한 Clean Architecture라고 단정하기보다는,
**Clean Architecture를 지향하는 실무형 Flutter 구조**에 가깝다.

그 이유는 아래와 같다.

- `presentation` 분리 존재
- `data` 분리 존재
- 일부 기능(`auth`)은 `domain` 계층 존재
- 공통 인프라(`core`)와 재사용 요소(`shared`) 분리

다만 현재는 모든 기능에 `domain`이 동일하게 들어가 있지는 않으므로,
엄격한 클린 아키텍처 완성형이라기보다는 **점진적으로 확장 중인 구조**로 보는 것이 가장 정확하다.

## 2.2 설계 의도

이 구조의 핵심 의도는 아래와 같다.

1. **업무 기능별 독립성 확보**
2. **UI와 데이터 처리 분리**
3. **공통 기능 재사용성 확보**
4. **오프라인/온라인 동기화 구조 수용**
5. **스캐너/보안/네트워크 같은 인프라 영역 분리**
6. **추후 기능 추가 시 영향 범위 최소화**

---

# 3. 최상위 폴더 구조 설명

현재 `lib/` 구조는 아래와 같다.

```text
lib/
├─ app
├─ core
├─ features
└─ shared
```

각 영역의 의미는 다음과 같다.

| 폴더 | 역할 | 설명 |
|------|------|------|
| `app` | 앱 진입/구성 | 앱 전체 설정, 라우팅, 테마, 환경설정 관리 |
| `core` | 전역 공통 인프라 | 네트워크, DB, 에러, 스캐너, 동기화 등 프로젝트 기반 기술 요소 |
| `features` | 업무 기능 모듈 | 로그인, 홈, 피킹, 상품/입고/재고 관련 기능 |
| `shared` | 범용 재사용 자원 | 여러 feature에서 공통으로 쓰는 enum, model, provider, widget |

이 분리는 Flutter 프로젝트 구조 가이드에서 제안한 `app / core / features / shared` 패턴과 동일한 계열이다.

---

# 4. 계층별 책임과 데이터 흐름

## 4.1 전체 흐름

PDA 앱에서 일반적인 데이터 흐름은 아래와 같다.

```text
[Screen / Widget]
      ↓
[Provider / State]
      ↓
[UseCase 또는 Repository]
      ↓
[Datasource]
  ↓           ↓
[Local DB]   [Remote API]
      ↓
[Sync / Outbox]
```

## 4.2 계층별 책임

### Presentation 계층
사용자와 직접 만나는 영역이다.

- 화면(Screen)
- UI 위젯(Widget)
- 상태관리(Provider)
- 사용자 입력 처리
- 로딩/에러/성공 상태 표시

### Domain 계층
업무 규칙과 기능의 중심이 되는 영역이다.

- 엔티티/도메인 모델
- Repository 인터페이스
- UseCase
- 비즈니스 규칙

### Data 계층
실제 데이터를 가져오고 저장하는 영역이다.

- API 호출
- SQLite 조회/저장
- DTO/Model 변환
- Repository 구현체

### Core 계층
앱 전체가 의존하는 기술 기반 영역이다.

- DB 연결
- Dio 설정
- 에러 공통 처리
- 스캐너 연동
- 보안 처리
- Sync 인프라

### Shared 계층
기능을 가리지 않고 재사용하는 범용 자산이다.

- 공용 enum
- 공통 모델
- 공통 Provider
- 공통 Widget

---

# 5. 폴더별 상세 관리 기준

# 5.1 app

```text
app/
├─ config
├─ router
└─ theme
```

`app` 폴더는 **앱 전체 구성을 선언하는 영역**이다.
개별 업무 로직은 넣지 않고, 앱 차원의 설정만 둔다.

## app/config

환경설정 파일을 관리한다.

관리 대상 예시:
- baseUrl
- 앱 실행 모드(dev/stage/prod)
- API timeout
- feature flag
- 앱 버전 정책
- 환경별 설정 클래스

예시 파일:
- `env.dart`
- `app_config.dart`
- `flavor_config.dart`

## app/router

화면 이동 정책과 라우팅 테이블을 관리한다.

관리 대상 예시:
- go_router 설정
- 화면 경로 상수
- 인증 여부에 따른 redirect
- 홈/로그인/피킹 상세 화면 이동 규칙

예시 파일:
- `app_router.dart`
- `route_names.dart`
- `route_guards.dart`

## app/theme

앱 전반의 UI 스타일 기준을 관리한다.

관리 대상 예시:
- ColorScheme
- TextTheme
- InputDecorationTheme
- ButtonTheme
- 공통 spacing 기준
- 상태 컬러(성공/경고/오류)

예시 파일:
- `app_theme.dart`
- `app_colors.dart`
- `app_text_styles.dart`

---

# 5.2 core

```text
core/
├─ constants
├─ database
├─ error
├─ network
├─ scanner
├─ security
├─ sync
├─ utils
└─ widgets
```

`core`는 프로젝트 전역에서 공통으로 사용하는 **기술 기반 인프라 영역**이다.
업무 기능에 종속적인 코드는 두지 않는 것이 원칙이다.

## core/constants

앱 전역 상수 정의를 관리한다.

관리 대상 예시:
- API path 상수
- DB 테이블명 상수
- 스캔 타입 상수
- sync 상태값 상수
- 앱 공통 문자열 키

예시 파일:
- `api_constants.dart`
- `db_constants.dart`
- `app_constants.dart`

## core/database

로컬 DB와 관련된 설정 및 기본 구조를 관리한다.

WMS PDA는 Offline-First 구조가 핵심이므로 매우 중요한 영역이다. PDA 기술 스택 정의서에서도 SQLite 기반 로컬 저장과 sync queue 구조를 핵심으로 설명한다. fileciteturn0file4

관리 대상 예시:
- SQLite 초기화
- DB open/close
- 테이블 생성 스크립트
- 공통 DAO 베이스
- 트랜잭션 처리
- migration

예시 파일:
- `app_database.dart`
- `database_helper.dart`
- `tables/*.dart`
- `daos/*.dart`

## core/error

에러 처리 표준화를 위한 영역이다.

관리 대상 예시:
- AppException
- Failure 클래스
- API 에러 파싱
- 네트워크 예외 처리
- DB 예외 처리
- 사용자 표시용 에러 메시지 변환

예시 파일:
- `exceptions.dart`
- `failures.dart`
- `api_error_handler.dart`
- `error_messages.dart`

## core/network

API 통신 기반을 담당한다.

관리 대상 예시:
- Dio 인스턴스
- interceptor
- 인증 토큰 첨부
- timeout 설정
- request/response logging
- 네트워크 연결 체크
- health check

예시 파일:
- `dio_client.dart`
- `dio_interceptor.dart`
- `network_checker.dart`

## core/scanner

PDA 스캐너 연동 공통 계층이다.

PDA 기술 스택 정의서에서는 Scanner Adapter, ScannerService 인터페이스, 벤더별 구현 분리를 핵심 요소로 설명한다.

관리 대상 예시:
- ScannerService 인터페이스
- Zebra/Honeywell/카메라 fallback 추상화
- 스캔 이벤트 모델
- 스캔 결과 스트림
- 바코드 포맷 처리
- 스캔 성공/실패 콜백

예시 파일:
- `scanner_service.dart`
- `scanner_channel.dart`
- `scanner_models.dart`
- `scanner_adapter.dart`

## core/security

보안 관련 공통 기능을 관리한다.

관리 대상 예시:
- 토큰 저장
- secure storage
- 세션 관리
- 생체 인증 연동 가능 구조
- 권한 체크 유틸
- 로그아웃 시 민감 데이터 삭제

예시 파일:
- `token_storage.dart`
- `secure_storage_service.dart`
- `auth_guard.dart`
- `session_manager.dart`

## core/sync

오프라인 작업과 서버 동기화의 기반을 담당한다.

이 프로젝트에서 가장 중요한 인프라 영역 중 하나다.
PDA 기술 스택 문서에서는 모든 작업을 로컬 DB에 먼저 저장하고, Outbox Pattern과 Sync Queue로 서버와 동기화하는 구조를 핵심 전략으로 제시한다. fileciteturn0file4

관리 대상 예시:
- sync service
- outbox queue
- retry 정책
- background sync
- sync 상태 기록
- 충돌 해결 정책

예시 파일:
- `sync_service.dart`
- `sync_queue_service.dart`
- `sync_worker.dart`
- `sync_models.dart`
- `sync_policy.dart`

## core/utils

공통 유틸리티를 관리한다.

관리 대상 예시:
- 날짜 포맷
- 바코드 검증
- logger
- debounce
- 문자열 가공
- 숫자/수량 포맷

예시 파일:
- `logger.dart`
- `date_util.dart`
- `barcode_util.dart`
- `number_util.dart`

## core/widgets

전역 공통 위젯을 관리한다.

관리 대상 예시:
- 공통 AppBar
- 공통 Button
- 로딩 오버레이
- 빈 화면 UI
- 에러 화면 UI
- 스캔 입력 필드 공통 컴포넌트

예시 파일:
- `common_app_bar.dart`
- `common_button.dart`
- `loading_overlay.dart`
- `empty_view.dart`
- `error_view.dart`

---

# 5.3 features

```text
features/
├─ auth
├─ home
├─ picking
└─ product_info
```

`features`는 업무 기능별 모듈 영역이다.
각 폴더는 하나의 업무 기능 또는 화면군을 중심으로 관리한다.

## 5.3.1 auth

```text
auth/
├─ data
│  ├─ datasource
│  ├─ model
│  └─ repository
├─ domain
└─ presentation
   ├─ provider
   ├─ screen
   └─ widget
```

`auth`는 현재 가장 구조가 잘 잡혀 있는 기능 모듈이다.
`data / domain / presentation`이 분리되어 있어, 이 구조를 다른 feature의 기준으로 삼아도 좋다.

### auth/data
외부 인증 데이터 처리 영역이다.

관리 대상 예시:
- 로그인 API 호출
- 사용자 정보 조회
- 토큰 refresh 호출
- local session 저장
- repository 구현체

#### auth/data/datasource
- remote datasource
- local datasource

예시 파일:
- `auth_remote_datasource.dart`
- `auth_local_datasource.dart`

#### auth/data/model
API 응답용 모델, DTO, JSON 파싱 모델을 둔다.

예시 파일:
- `user_model.dart`
- `login_request.dart`
- `login_response.dart`

#### auth/data/repository
repository 구현체를 둔다.

예시 파일:
- `auth_repository_impl.dart`

### auth/domain
비즈니스 규칙과 추상화를 둔다.

관리 대상 예시:
- User 엔티티
- AuthRepository 인터페이스
- LoginUseCase
- LogoutUseCase
- GetCurrentUserUseCase

예시 파일:
- `auth_repository.dart`
- `login_usecase.dart`
- `user.dart`

### auth/presentation
UI와 상태관리를 담당한다.

#### provider
- 로그인 상태
- 토큰 유효성
- 현재 사용자 상태

#### screen
- 로그인 화면
- 자동 로그인 화면

#### widget
- 로그인 입력폼
- 인증 버튼
- 사용자 정보 표시 위젯

## 5.3.2 home

```text
home/
└─ presentation
   ├─ provider
   ├─ screen
   └─ widget
```

`home`은 현재 presentation 중심의 단순 기능으로 보인다.
대시보드 성격이거나 메뉴 진입 화면일 가능성이 높다.

관리 대상 예시:
- 메인 홈 화면
- 사용자별 메뉴 목록
- 오늘 작업 요약
- 최근 작업 이력
- 동기화 상태 표시

권장 파일 예시:
- `home_screen.dart`
- `home_provider.dart`
- `menu_grid_widget.dart`
- `sync_status_widget.dart`

현재는 복잡한 비즈니스 규칙이 적다면 presentation만 있어도 무방하다.
다만 홈에서 서버 데이터, 로컬 데이터, 작업 상태 요약이 많아지면 `data` 계층을 추가하는 것이 좋다.

## 5.3.3 picking

```text
picking/
├─ data
│  └─ datasource
└─ presentation
   ├─ provider
   ├─ screen
   └─ widget
```

`picking`은 WMS PDA의 핵심 기능 중 하나이다.
제안서에서도 피킹은 작업 지시 다운로드, 오프라인 피킹, 로컬 큐 적재, 네트워크 복구 시 일괄 확정의 흐름으로 정의되어 있다. fileciteturn0file1

현재는 `data/datasource`와 `presentation`만 있으므로,
앞으로 `domain`과 `repository`를 추가하면 구조가 더 안정적이다.

### picking/data/datasource
관리 대상 예시:
- 피킹 작업 목록 조회 API
- 피킹 상세 조회 API
- 로컬 피킹 테이블 조회
- 피킹 결과 저장
- 가출고/부분완료 저장

권장 파일:
- `picking_remote_datasource.dart`
- `picking_local_datasource.dart`

### picking/presentation/provider
관리 대상 예시:
- 피킹 리스트 상태
- 피킹 상세 상태
- 스캔 진행률
- 완료 처리 상태
- 오프라인 저장 상태

### picking/presentation/screen
관리 대상 예시:
- 피킹 작업 목록 화면
- 피킹 상세 화면
- 스캔/수량 입력 화면
- 완료 확인 화면

### picking/presentation/widget
관리 대상 예시:
- 피킹 아이템 카드
- 진행률 바
- 스캔 입력 박스
- 위치 정보 위젯
- 수량 증감 위젯

권장 보완:
- `picking/domain`
- `picking/data/repository`
- `picking/data/model`

## 5.3.4 product_info

```text
product_info/
├─ location_move
├─ presentation
│  ├─ provider
│  ├─ screen
│  └─ widget
├─ receiving_inspection
├─ receiving_putaway
└─ stock_taking
```

이 구조는 해석이 조금 필요하다.
현재 `product_info` 아래에 실제로는 여러 업무 기능이 함께 들어가 있다.
즉 이름은 `product_info`이지만 실질적으로는 **작업 기능 묶음 폴더**처럼 사용되고 있다.

이 안에 있는 하위 폴더를 보면 아래 기능이 섞여 있다.

- 상품 조회/정보
- 로케이션 이동
- 입고 검수
- 입고 적치
- 재고 조사

이는 향후 기능이 커질수록 관리가 어려워질 수 있다.
실무적으로는 아래 둘 중 하나가 더 적합하다.

### 방식 A: product_info를 진짜 상품조회 기능으로 축소
- `features/product_info`
- `features/location_move`
- `features/receiving_inspection`
- `features/receiving_putaway`
- `features/stock_taking`

### 방식 B: operations 같은 상위 업무군으로 명확히 변경
- `features/operations/product_info`
- `features/operations/location_move`
- `features/operations/receiving_inspection`
- `features/operations/receiving_putaway`
- `features/operations/stock_taking`

현재 상태 기준으로 각 하위 폴더의 역할은 다음처럼 정의할 수 있다.

### product_info/location_move
로케이션 간 재고 이동 기능 관련 파일을 관리한다.

관리 대상 예시:
- 이동 대상 조회
- 출발 로케이션/도착 로케이션 스캔
- 이동 수량 입력
- 이동 결과 저장
- 이동 내역 동기화

### product_info/receiving_inspection
입고 검수 기능 관련 파일을 관리한다.

관리 대상 예시:
- ASN/입고예정 조회
- 바코드 검수
- 수량/유통기한 입력
- 오입고/과입고 예외 처리
- 로컬 검수 저장

입고 검수와 가입고/입고대기 개념은 제안서의 입고 관리 프로세스에도 정의되어 있다. fileciteturn0file1

### product_info/receiving_putaway
입고 적치 기능 관련 파일을 관리한다.

관리 대상 예시:
- 검수 완료 건 조회
- 적치 대상 위치 추천
- 적치 위치 스캔
- 적치 완료 저장
- 적치 sync

### product_info/stock_taking
재고 조사 기능 관련 파일을 관리한다.

관리 대상 예시:
- 재고조사 대상 조회
- 위치별 재고 스캔
- 실사 수량 입력
- 차이 기록
- 재고조사 결과 동기화

### product_info/presentation
현재 `presentation`은 product_info 상위 공통 UI일 가능성이 있다.
만약 실제로 상품조회 화면만 담당한다면 아래를 둔다.

관리 대상 예시:
- 상품 조회 화면
- 상품 상세 화면
- 바코드 검색 화면
- 상품정보 카드

하지만 하위 기능들이 많아질 경우,
각 기능별로 자체 `presentation`을 가지는 것이 더 바람직하다.

---

# 5.4 shared

```text
shared/
├─ enum
├─ model
├─ provider
└─ widget
```

`shared`는 여러 feature가 함께 쓰는 범용 자원을 두는 영역이다.
`core`와 헷갈릴 수 있는데, 구분 기준은 아래와 같다.

- `core`: 기술 인프라 공통
- `shared`: 업무/표현 레벨 재사용 자원

## shared/enum

관리 대상 예시:
- 작업 상태 enum
- 입고 상태 enum
- 출고 상태 enum
- sync 상태 enum
- 온도대 enum
- 스캔 타입 enum

예시 파일:
- `work_status.dart`
- `temperature_type.dart`
- `sync_status.dart`

## shared/model

관리 대상 예시:
- 공통 페이지 모델
- 코드/명칭 모델
- 바코드 정보 모델
- 로케이션 요약 모델
- 사용자 요약 모델

예시 파일:
- `common_code.dart`
- `paged_result.dart`
- `location_summary.dart`

## shared/provider

관리 대상 예시:
- 공통 로딩 상태
- 앱 전체 사용자 상태
- 연결 상태 provider
- 현재 센터/창고 선택 상태
- 다국어/설정 상태

예시 파일:
- `app_state_provider.dart`
- `network_status_provider.dart`

## shared/widget

관리 대상 예시:
- 공통 카드 UI
- 공통 상태 배지
- 공통 검색바
- 공통 정보 row
- 수량 표시 컴포넌트

예시 파일:
- `status_badge.dart`
- `info_row.dart`
- `common_card.dart`

---

# 6. 기능 모듈별 권장 구조

현재 기준에서 가장 권장되는 방향은 각 feature를 아래처럼 통일하는 것이다.

```text
features/<feature_name>/
├─ data/
│  ├─ datasource/
│  ├─ model/
│  └─ repository/
├─ domain/
│  ├─ model/
│  ├─ repository/
│  └─ usecase/
└─ presentation/
   ├─ provider/
   ├─ screen/
   └─ widget/
```

예를 들어 `picking`은 아래처럼 확장하는 것이 좋다.

```text
features/picking/
├─ data/
│  ├─ datasource/
│  │  ├─ picking_local_datasource.dart
│  │  └─ picking_remote_datasource.dart
│  ├─ model/
│  │  ├─ picking_task_model.dart
│  │  └─ picking_item_model.dart
│  └─ repository/
│     └─ picking_repository_impl.dart
├─ domain/
│  ├─ model/
│  │  ├─ picking_task.dart
│  │  └─ picking_item.dart
│  ├─ repository/
│  │  └─ picking_repository.dart
│  └─ usecase/
│     ├─ get_picking_tasks_usecase.dart
│     ├─ scan_picking_item_usecase.dart
│     └─ complete_picking_usecase.dart
└─ presentation/
   ├─ provider/
   ├─ screen/
   └─ widget/
```

이 구조는 Flutter PDA 프로젝트 시작 가이드에서 제안한 feature 내부 구성과도 일치한다. fileciteturn0file6

---

# 7. 현재 구조에 대한 해석과 보완 권장사항

## 7.1 잘 잡힌 부분

현재 구조에서 좋은 점은 아래와 같다.

- `app / core / features / shared`의 큰 축이 이미 있음
- `auth`는 계층 분리가 비교적 잘 되어 있음
- `core`에 네트워크, DB, scanner, sync가 분리되어 있음
- PDA 프로젝트에서 중요한 인프라 요소가 이미 폴더 차원에서 확보되어 있음

## 7.2 보완이 필요한 부분

### 1) feature 구조 통일성 부족
- `auth`는 `domain`이 있음
- `picking`은 `domain`이 없음
- `home`은 `presentation`만 있음

기능 규모 차이로 그럴 수 있지만,
중장기적으로는 최소한 `data / presentation` 또는 `data / domain / presentation` 패턴을 일관되게 맞추는 것이 좋다.

### 2) product_info 하위 구조가 모호함
현재 `product_info`는 이름에 비해 하위 기능 범위가 너무 넓다.
기능이 커지면 유지보수 혼란이 생길 수 있다.

### 3) shared와 core의 경계 규칙 문서화 필요
예를 들어 공통 widget을 `core/widgets`에 둘지 `shared/widget`에 둘지 기준이 없으면 폴더가 금방 흐려진다.

권장 기준:
- `core/widgets`: 앱 전역 인프라성 UI
- `shared/widget`: 업무 표현 재사용 UI

### 4) model 용어 기준 통일 필요
- `data/model`은 DTO/API/DB 모델
- `domain/model`은 비즈니스 모델
- `shared/model`은 기능 공통 단순 모델

이 기준을 팀 내에서 합의해야 한다.

---

# 8. 파일 배치 원칙

## 8.1 app에 두면 안 되는 것

아래는 `app`에 두지 않는 것이 좋다.

- 피킹 비즈니스 로직
- 로그인 API 호출
- DB 조회 로직
- 스캐너 처리 로직

`app`은 앱 설정만 둔다.

## 8.2 core에 두면 좋은 것

- 특정 기능에 종속되지 않는 기술 처리
- 모든 기능이 공통으로 쓰는 infra
- DB, 네트워크, 보안, sync, scanner

## 8.3 feature에 두면 좋은 것

- 특정 업무 기능에만 쓰는 화면
- 특정 기능의 provider
- 특정 기능의 datasource/repository/model
- 특정 기능 전용 widget

## 8.4 shared에 두면 좋은 것

- 여러 feature가 공통으로 재사용하는 표현 요소
- 업무 공통 enum/model/widget/provider

## 8.5 예시 기준

| 파일 | 추천 위치 | 이유 |
|------|----------|------|
| `dio_client.dart` | `core/network` | 전역 공통 네트워크 인프라 |
| `sync_service.dart` | `core/sync` | 전역 공통 동기화 인프라 |
| `user_model.dart` | `features/auth/data/model` | 인증 기능 전용 데이터 모델 |
| `picking_provider.dart` | `features/picking/presentation/provider` | 피킹 화면 전용 상태관리 |
| `status_badge.dart` | `shared/widget` | 여러 기능에서 재사용 가능 |
| `temperature_type.dart` | `shared/enum` | 여러 기능이 공유하는 업무 enum |

---

# 9. 최종 정리

현재 프로젝트 구조는 다음과 같이 정의할 수 있다.

> **Flutter 기반 WMS PDA 앱을 위한 Feature-First + Layered Architecture이며, Clean Architecture를 실무적으로 지향하는 형태의 구조**

핵심 특징은 아래와 같다.

- `app`: 앱 구성과 실행 설정
- `core`: 기술 인프라 공통 모듈
- `features`: 업무 기능 모듈
- `shared`: 기능 간 재사용 자원

또한 이 구조는 PDA 프로젝트의 핵심 요구사항인
**오프라인 처리, 로컬 DB, sync queue, scanner abstraction**을 담기 적합한 구조이다. fileciteturn0file4 fileciteturn0file6

앞으로 구조를 더 안정적으로 만들려면 아래 3가지를 우선 추천한다.

1. `auth` 기준으로 feature 구조를 점진적으로 통일
2. `product_info` 하위 기능을 독립 feature로 재정리
3. `core`와 `shared`의 배치 기준을 팀 문서로 확정

이 문서를 기준으로 팀 내 폴더 배치 규칙까지 같이 정하면,
이후 기능 추가와 유지보수가 훨씬 쉬워진다.
