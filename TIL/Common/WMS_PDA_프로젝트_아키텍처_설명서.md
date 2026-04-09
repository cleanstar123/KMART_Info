# WMS PDA 프로젝트 아키텍처 설명서

> 이 문서는 프로젝트의 폴더 구조, 계층 역할, 설계 의도를 설명하는 기준 문서입니다.
> 현재 실제 코드 구조를 기준으로 작성되었습니다.

---

## 목차

1. [아키텍처 개요](#1-아키텍처-개요)
2. [최상위 폴더 구조](#2-최상위-폴더-구조)
3. [계층별 역할과 데이터 흐름](#3-계층별-역할과-데이터-흐름)
4. [app 폴더 상세](#4-app-폴더-상세)
5. [core 폴더 상세](#5-core-폴더-상세)
6. [features 폴더 상세](#6-features-폴더-상세)
7. [shared 폴더 상세](#7-shared-폴더-상세)
8. [파일 배치 원칙](#8-파일-배치-원칙)

---

## 1. 아키텍처 개요

### 한 줄 정의

> **업무 기능별(feature)로 모듈을 나누고, 각 기능 내부에서 `data / presentation` 계층으로 책임을 분리하며, 전역 공통 기능은 `core / shared`로 분리한 Feature-First + Layered Architecture**

### 아키텍처를 구성하는 3가지 관점

**1) Feature-First Architecture**

기능 중심으로 폴더를 나누는 방식이다. WMS처럼 기능이 많은 시스템에서 유리하며, 각 기능이 독립적으로 진화할 수 있고 화면/상태/데이터 관련 코드가 기능 단위로 모인다.

```
features/auth          로그인 및 인증
features/home          메인 홈 화면
features/picking       피킹 작업
features/product_info  상품 정보 조회
```

**2) Layered Architecture**

기능 내부를 역할에 따라 계층으로 분리한다.

```
data          외부 데이터 처리 (API, SQLite)
presentation  화면 / UI / 상태관리
```

**3) Offline-First 설계 수용**

PDA 기기 특성상 네트워크가 불안정한 환경에서도 작업이 끊기지 않아야 한다. 모든 작업은 로컬 DB에 먼저 저장하고, 네트워크가 복구되면 Outbox 패턴으로 서버에 동기화한다.

### 이 구조가 이 프로젝트에 적합한 이유

| 요구사항 | 구조적 대응 |
|---|---|
| 오프라인 우선 (Offline-First) | `core/database` + `core/sync` Outbox 패턴 |
| PDA 스캐너 연동 | `core/scanner` (예정) |
| WMS API + 로컬 캐시 이중화 | `data/datasource` remote + local 분리 |
| 기능별 독립 개발 | `features/` feature 단위 모듈화 |
| 공통 보안 처리 | `core/security` AES-256, SHA-256 |

---

## 2. 최상위 폴더 구조

```
lib/
├── app/        앱 전체 설정 (라우팅, 테마, 환경설정)
├── core/       전역 기술 인프라 (DB, 네트워크, 보안, 동기화)
├── features/   업무 기능 모듈
└── shared/     기능 간 공통 재사용 자원
```

| 폴더 | 한 줄 역할 | 업무 로직 포함 여부 |
|---|---|---|
| `app/` | 앱 구성과 실행 설정 | 없음 |
| `core/` | 기술 기반 인프라 | 없음 (WMS 용어 금지) |
| `features/` | 각 업무 기능의 전체 구현 | 있음 (기능 전용) |
| `shared/` | 여러 기능이 공유하는 표현 자원 | 있음 (기능 공통) |

---

## 3. 계층별 역할과 데이터 흐름

### 전체 흐름

```
┌─────────────────────────────────────────┐
│           Screen / Widget               │
│   사용자 입력 수신 · 상태 구독 · UI 렌더링   │
└───────────────────┬─────────────────────┘
                    │ watch / read
┌───────────────────▼─────────────────────┐
│          Provider / Notifier            │
│   UI 상태 관리 · 비즈니스 흐름 조율        │
└───────────────────┬─────────────────────┘
                    │ 호출
┌───────────────────▼─────────────────────┐
│             Repository                  │
│   온/오프라인 전략 결정                   │
│   API 우선 → 실패 시 로컬 폴백            │
└──────────┬──────────────────┬───────────┘
           │                  │
┌──────────▼───────┐  ┌───────▼──────────┐
│  Remote          │  │  Local           │
│  Datasource      │  │  Datasource      │
│  WMS API (Dio)   │  │  SQLite          │
└──────────────────┘  └───────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Outbox / Sync     │
                    │   오프라인 큐 관리    │
                    └─────────────────────┘
```

### 계층별 책임 요약

| 계층 | 책임 | 코드 위치 |
|---|---|---|
| Screen / Widget | 화면 렌더링, 사용자 이벤트 수신 | `presentation/screen/`, `presentation/widget/` |
| Provider / State | UI 상태 보관, Notifier 로직 | `presentation/provider/` |
| Repository | 온/오프라인 전략, 로그 기록 | `data/repository/` |
| Datasource | API 호출 또는 DB 조회/저장 | `data/datasource/` |
| Model | API 응답/DB Row → Dart 객체 변환 | `data/model/` |
| Core | DB, 네트워크, 보안, Sync 인프라 | `core/` |

---

## 4. app 폴더 상세

```
app/
├── config/    env.dart
├── router/    app_router.dart
└── theme/     app_theme.dart
```

> **원칙:** 앱 구성 선언만 둔다. 업무 로직, DB 호출, API 호출은 두지 않는다.

### app/config
환경 설정을 관리한다.

- WMS API baseUrl
- connectTimeout, receiveTimeout
- 앱 실행 환경 (dev / prod)

현재 파일: `env.dart`

### app/router
화면 이동 정책을 관리한다.

- go_router 라우팅 테이블
- 인증 여부에 따른 redirect (비로그인 → 로그인 화면)
- 화면 경로 상수

현재 파일: `app_router.dart`

### app/theme
앱 전반의 UI 스타일 기준을 관리한다.

- 색상 팔레트 (primaryGreen, gray계열 등)
- 공통 TextStyle
- 상태 컬러 (성공 / 경고 / 오류)

현재 파일: `app_theme.dart`

---

## 5. core 폴더 상세

```
core/
├── database/    app_database.dart, outbox_dao.dart, outbox_table.dart
├── error/       app_exception.dart, api_error_handler.dart
├── network/     dio_client.dart
├── security/    field_encryptor.dart, password_hasher.dart
└── sync/        sync_queue_service.dart, sync_models.dart
```

> **원칙:** WMS 업무 용어(피킹, 입고, 재고 등)가 코드에 나오면 `core`가 아니다.
> 기능에 종속되지 않는 기술 처리만 둔다.

### core/database
로컬 SQLite의 모든 기반을 담당한다.

- `app_database.dart` — DB 싱글톤, 전체 테이블 생성(`CREATE TABLE IF NOT EXISTS`), 시드 데이터
- `outbox_dao.dart` — Outbox 큐 삽입/조회/상태 갱신
- `outbox_table.dart` — `wms_sync_outbox` 테이블 스키마 정의

**관리되는 테이블 목록**

| 테이블명 | 역할 |
|---|---|
| `wms_user` | 로그인 계정 정보 (개인정보 AES 암호화) |
| `wms_login_log` | 로그인 시도 이력 |
| `wms_if_item` | 상품 마스터 |
| `wms_if_location` | 위치 마스터 |
| `wms_pick_task_h` | 피킹 작업 헤더 |
| `wms_pick_task_d` | 피킹 작업 상세 |
| `wms_pick_scan_log` | 피킹 스캔 이력 |
| `wms_sync_outbox` | 서버 동기화 대기 이벤트 큐 |
| `wms_sync_log` | 동기화 결과 이력 |

### core/error
에러 처리를 표준화한다.

- `app_exception.dart` — sealed class 기반 예외 타입 정의

| 예외 클래스 | 발생 조건 |
|---|---|
| `NetworkException` | 연결 불가 / 타임아웃 |
| `AuthException` | 401 / 403 인증 거부 |
| `ServerException` | 5xx 서버 오류 |
| `OfflineFallbackException` | 오프라인 + 로컬 캐시 없음 |

- `api_error_handler.dart` — `DioException` → `AppException` 변환

### core/network
API 통신 기반을 담당한다.

- `dio_client.dart` — Dio 싱글톤 인스턴스, baseUrl / timeout / Content-Type 설정

모든 remote datasource는 `DioClient.instance`를 사용한다.

### core/security
보안 관련 공통 기능을 담당한다.

- `field_encryptor.dart` — AES-256 CBC 양방향 암호화 (개인정보 필드용)
  - 저장 형식: `"base64(IV):base64(암호문)"`
  - 매 암호화마다 랜덤 IV 생성 → 같은 값도 매번 다른 암호문
- `password_hasher.dart` — SHA-256 단방향 해싱 (비밀번호 저장용)

### core/sync
오프라인 동기화 인프라를 담당한다.

- `sync_queue_service.dart` — Outbox 큐 처리, 서버 전송, retry 정책
- `sync_models.dart` — 동기화 관련 모델 정의

### 예정 폴더

| 폴더 | 용도 |
|---|---|
| `core/scanner/` | PDA 스캐너 추상화, Zebra/Honeywell 벤더별 구현 |
| `core/utils/` | 날짜 포맷, 바코드 검증, 숫자 포맷 |
| `core/constants/` | API path 상수, DB 테이블명 상수 |

---

## 6. features 폴더 상세

```
features/
├── auth/
│   ├── data/
│   │   ├── datasource/
│   │   ├── model/
│   │   └── repository/
│   └── presentation/
│       ├── provider/
│       ├── screen/
│       └── widget/
│
├── home/
│   └── presentation/
│       ├── provider/
│       ├── screen/
│       └── widget/
│
├── picking/
│   └── presentation/
│       ├── provider/
│       ├── screen/
│       └── widget/
│
└── product_info/
    └── presentation/
        ├── provider/
        ├── screen/
        └── widget/
```

> **원칙:** `data/` 폴더는 datasource 파일이 실제로 생길 때 추가한다.
> UI 개발 단계에서는 `presentation/`만 유지한다.

---

### 6.1 auth — 인증 (data 계층 완성)

현재 가장 구조가 완성된 feature. 다른 feature 개발 시 참고 기준으로 삼는다.

#### data 계층

**datasource/**

| 파일 | 역할 |
|---|---|
| `auth_remote_datasource.dart` | WMS API 로그인 (`POST /api/auth/login`), 유저 목록 조회 |
| `auth_local_datasource.dart` | SQLite 로컬 로그인, 유저 캐싱(upsert), 유저 동기화 |
| `login_log_datasource.dart` | `wms_login_log` 테이블에 로그인 이력 기록 |

**model/**

| 파일 | 역할 |
|---|---|
| `user_model.dart` | WMS API JSON → Dart 객체, SQLite Row → Dart 객체 변환 |

**repository/**

| 파일 | 역할 |
|---|---|
| `auth_repository.dart` | 하이브리드 로그인 전략 (API 우선 → NetworkException 시 로컬 폴백) |

**로그인 전략 흐름**

```
AuthRepository.login(id, pw)
    │
    ├── WMS API 시도
    │       ├── 성공  →  로컬 캐싱 → 로그 기록 → UserModel 반환
    │       │
    │       └── NetworkException
    │               ├── 로컬 캐시 있음  →  로그 기록 → UserModel 반환
    │               └── 로컬 캐시 없음  →  OfflineFallbackException
    │
    └── AuthException / ServerException
            →  로그 기록 → 즉시 rethrow (폴백 없음)
```

#### presentation 계층

**provider/**

| 파일 | 역할 |
|---|---|
| `login_state.dart` | 로그인 화면 UI 상태 정의 (idInput, pwInput, isLoading, errorMessage, isSuccess) |
| `login_provider.dart` | LoginNotifier — 로그인 흐름 조율, `autoDispose` 적용 |
| `auth_session_provider.dart` | `StateProvider<UserModel?>` — 앱 전역 로그인 세션 보관 |

**screen/**
- `login_screen.dart` — 로그인 화면. `ref.listen`으로 `isSuccess` 감지 → `context.go('/')`

**widget/**
- `login_header` / `login_id_field` / `login_pw_field` / `login_button` / `login_error_banner` / `login_offline_notice`

---

### 6.2 home — 메인 홈 화면 (presentation만 유지)

대시보드 성격의 화면. 메뉴 진입점, 작업 요약, 동기화 상태 표시 역할.

현재 서버 데이터 연동 없이 `HomeState.initial()`의 하드코딩 데이터로 표시된다.

**provider/**

| 파일 | 역할 |
|---|---|
| `home_state.dart` | 홈 화면 UI 상태 (`workerName`, `centerName`, `isOnline`, `menus` 등) + `HomeMenuItem` 모델 |
| `home_provider.dart` | HomeNotifier — `refresh()` / `toggleNetworkStatus()` / `markSynced()` |

> `refresh()` 는 현재 TODO 스텁. 추후 SyncQueueService 연동 예정.

**screen/**
- `home_screen.dart` — 메뉴 그리드, 통계 카드, 헤더, 풀-투-리프레시

**widget/**
- `home_header` — 작업자 이름, 센터명, 온/오프라인 상태, 동기화 버튼
- `menu_grid` — 기능 메뉴 격자 표시
- `quick_stats_card` — 입고/피킹/재고조사 대기 건수 요약
- `sync_status_banner` — 미동기화 건수 알림 배너

---

### 6.3 picking — 피킹 작업 (data 계층 개발 예정)

WMS PDA 핵심 기능. 피킹리스트 스캔 → 상품 스캔 → 수량 입력 → 완료 처리 흐름.

**provider/**

| 파일 | 역할 |
|---|---|
| `picking_state.dart` | 피킹 UI 상태 + `PickingTaskModel` / `PickingItemModel` / `PickingTaskHeader` 모델 |
| `picking_provider.dart` | PickingNotifier — UI 상태 관리. `loadPickingList()` / `updatePickedQty()`는 TODO 스텁 |

**TODO 스텁 메서드 (데이터 연동 개발 시 구현)**

| 메서드 | 구현 예정 내용 |
|---|---|
| `loadPickingList()` | `wms_pick_task_d` 조회 → `PickingTaskModel` 생성 |
| `updatePickedQty()` | SQLite 업데이트 + `wms_sync_outbox` 큐잉 |

**screen/**
- `picking_screen.dart` — 피킹리스트 스캔 → 상품 스캔 → 아이템별 수량 입력 → 완료 처리

**widget/**
- `picking_scan_section` — 바코드 스캔 입력 영역
- `picking_header_card` — 피킹 작업 헤더 정보 (접기/펼치기)
- `picking_progress_card` — 진행률 표시
- `picking_item_card` — 개별 상품 카드 + 수량 입력
- `picking_complete_bar` — 전체 완료 시 활성화되는 완료 버튼 바

**향후 data 계층 추가 계획**

```
picking/data/
├── datasource/
│   ├── picking_local_datasource.dart     SQLite 피킹 조회/저장
│   └── picking_remote_datasource.dart    WMS API 피킹 다운로드/확정
├── model/
│   ├── picking_task_model.dart
│   └── picking_item_model.dart
└── repository/
    └── picking_repository.dart
```

---

### 6.4 product_info — 상품 정보 조회 (data 계층 개발 예정)

상품 바코드 스캔 → 상품 기본 정보 / 재고 / 로케이션 조회 화면.

**provider/**

| 파일 | 역할 |
|---|---|
| `product_info_state.dart` | 조회 UI 상태 + `ProductInfoModel` (상품 정보 표시용 모델) |
| `product_info_provider.dart` | ProductInfoNotifier — `searchProduct()`는 TODO 스텁 |

**screen/**
- `product_info_screen.dart` — 바코드 입력 → 상품 정보 카드 표시

**widget/**
- `barcode_scan_section` — 바코드 입력 필드 + 검색/초기화 버튼
- `product_info_card` — 상품 기본정보, 재고, 로케이션, 입출고 이력 표시

**향후 추가 계획**

```
product_info/data/
├── datasource/
│   ├── product_local_datasource.dart     wms_if_item SQLite 조회
│   └── product_remote_datasource.dart    WMS API 상품 조회
├── model/
│   └── product_model.dart
└── repository/
    └── product_repository.dart
```

---

### 6.5 향후 신설 예정 feature

현재 `home` 메뉴에 연결된 미개발 기능들은 각각 독립 feature로 신설한다.

| Feature 폴더 | 기능 |
|---|---|
| `features/receiving_inspection/` | 입고검수 — ASN 조회, 바코드 검수, 수량 입력 |
| `features/receiving_putaway/` | 입고적재 — 검수완료 건 조회, 적치 위치 스캔, 적재 완료 |
| `features/location_move/` | 로케이션이동 — 출발/도착 위치 스캔, 이동 수량 입력 |
| `features/stock_taking/` | 재고조사 — 실사 대상 조회, 실사 수량 입력, 차이 기록 |

---

## 7. shared 폴더 상세

```
shared/
├── widget/    app_footer.dart
├── enum/      (예정)
├── model/     (예정)
└── provider/  (예정)
```

> **core vs shared 구분 기준**
>
> | | core | shared |
> |---|---|---|
> | 성격 | 기술 인프라 | 업무 표현 자원 |
> | WMS 업무 용어 | 없음 | 있음 |
> | 예시 | `DioClient`, `AppDatabase` | `AppFooter`, `StatusBadge` |

### shared/widget
feature에 관계없이 여러 화면에서 재사용하는 UI 컴포넌트.

현재: `app_footer.dart`

예정: `status_badge.dart`, `scan_input_field.dart`, `info_row.dart`, `common_card.dart`

### 예정 폴더

| 폴더 | 관리 대상 |
|---|---|
| `shared/enum/` | 작업상태, 입출고유형, 스캔타입 등 업무 공통 enum |
| `shared/model/` | 여러 feature가 공유하는 단순 공통 모델 |
| `shared/provider/` | 네트워크 연결 상태, 현재 센터 선택 등 앱 전역 상태 |

---

## 8. 파일 배치 원칙

### 어느 폴더에 둘지 판단하는 기준

```
Q1. 업무 용어(피킹, 입고, 재고 등)가 코드에 나오는가?
     │
     ├── NO  →  core/
     │
     └── YES →  Q2. 특정 feature에만 쓰이는가?
                     │
                     ├── YES →  features/해당feature/
                     └── NO  →  shared/
```

### feature 내부 파일 위치 기준

| 코드 성격 | 위치 |
|---|---|
| 화면 레이아웃, 위젯 조합 | `presentation/screen/`, `presentation/widget/` |
| UI 상태 데이터 | `presentation/provider/*_state.dart` |
| UI 상태 관리 로직 | `presentation/provider/*_provider.dart` |
| API 호출, DB 조회/저장 | `data/datasource/` |
| API 응답 / DB Row 변환 모델 | `data/model/` |
| 온/오프라인 전략 분기 | `data/repository/` |

### 네이밍 규칙

| 접미사 | 위치 | 의미 |
|---|---|---|
| `*Model` | `data/model/` | DB Row / API JSON 직접 변환 객체 |
| `*State` | `presentation/provider/` | UI 상태 객체 (불변, 직렬화 없음) |
| `*Datasource` | `data/datasource/` | 단일 데이터 소스 접근 클래스 |
| `*Repository` | `data/repository/` | 온/오프라인 전략 통합 클래스 |
| `*Notifier` | `presentation/provider/` | 상태 변경 로직 클래스 |
| `*Provider` | `presentation/provider/` | Riverpod Provider 선언 |

### 파일 배치 예시

| 파일 | 위치 | 이유 |
|---|---|---|
| `dio_client.dart` | `core/network/` | 전역 기술 인프라, 업무 무관 |
| `field_encryptor.dart` | `core/security/` | 전역 기술 인프라, 업무 무관 |
| `user_model.dart` | `features/auth/data/model/` | 인증 기능 전용 데이터 모델 |
| `picking_provider.dart` | `features/picking/presentation/provider/` | 피킹 화면 전용 상태관리 |
| `app_footer.dart` | `shared/widget/` | 여러 화면 공통 UI |
| `app_router.dart` | `app/router/` | 앱 전체 화면 이동 설정 |
