# WMS PDA 아키텍처 최종 규칙

> **구조 한 줄 요약**
> 업무 기능별(feature)로 모듈을 나누고, 각 기능 내부에서 `data / presentation` 계층으로 책임을 분리하며, 전역 공통 기능은 `core / shared`로 분리한 **Feature-First + Layered Architecture**

---

## 목차

1. [현재 폴더 구조](#1-현재-폴더-구조)
2. [데이터 흐름](#2-데이터-흐름)
3. [영역별 배치 기준](#3-영역별-배치-기준)
4. [개발 시 판단 기준](#4-개발-시-판단-기준)
5. [신규 기능 개발 순서](#5-신규-기능-개발-순서)
6. [오프라인 우선 원칙](#6-오프라인-우선-원칙)
7. [현재 미개발 영역](#7-현재-미개발-영역)

---

## 1. 현재 폴더 구조

```
lib/
│
├── app/                              # 앱 전체 설정
│   ├── config/    env.dart           # baseUrl, timeout 등
│   ├── router/    app_router.dart    # 화면 경로, 인증 redirect
│   └── theme/     app_theme.dart     # 색상, 폰트, 공통 스타일
│
├── core/                             # 기술 인프라 (업무 로직 없음)
│   ├── database/  app_database.dart, outbox_dao.dart, outbox_table.dart
│   ├── error/     app_exception.dart, api_error_handler.dart
│   ├── network/   dio_client.dart
│   ├── security/  field_encryptor.dart, password_hasher.dart
│   └── sync/      sync_queue_service.dart, sync_models.dart
│
├── features/                         # 업무 기능 모듈
│   │
│   ├── auth/                         ← data 계층 완성
│   │   ├── data/
│   │   │   ├── datasource/           auth_remote_datasource, auth_local_datasource, login_log_datasource
│   │   │   ├── model/                user_model.dart
│   │   │   └── repository/           auth_repository.dart
│   │   └── presentation/
│   │       ├── provider/             login_provider, login_state, auth_session_provider
│   │       ├── screen/               login_screen.dart
│   │       └── widget/               login_header, login_id_field, login_pw_field,
│   │                                 login_button, login_error_banner, login_offline_notice
│   │
│   ├── home/                         ← 서버 연동 불필요 단계
│   │   └── presentation/
│   │       ├── provider/             home_provider, home_state
│   │       ├── screen/               home_screen.dart
│   │       └── widget/               home_header, menu_grid, quick_stats_card, sync_status_banner
│   │
│   ├── picking/                      ← data 계층 개발 예정
│   │   └── presentation/
│   │       ├── provider/             picking_provider, picking_state
│   │       ├── screen/               picking_screen.dart
│   │       └── widget/               picking_scan_section, picking_header_card,
│   │                                 picking_progress_card, picking_item_card, picking_complete_bar
│   │
│   └── product_info/                 ← data 계층 개발 예정
│       └── presentation/
│           ├── provider/             product_info_provider, product_info_state
│           ├── screen/               product_info_screen.dart
│           └── widget/               barcode_scan_section, product_info_card
│
└── shared/                           # 업무 공통 재사용 자원
    └── widget/    app_footer.dart
```

---

## 2. 데이터 흐름

```
┌─────────────────────────────────────────┐
│           Screen / Widget               │  사용자 입력 수신, 상태 구독, UI 렌더링
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│          Provider / Notifier            │  UI 상태 관리, 비즈니스 흐름 조율
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│             Repository                  │  온/오프라인 전략 결정
│         (API 우선 → 로컬 폴백)           │
└──────────┬──────────────────┬───────────┘
           │                  │
┌──────────▼───────┐  ┌───────▼──────────┐
│  Remote          │  │  Local           │
│  Datasource      │  │  Datasource      │
│  (WMS API)       │  │  (SQLite)        │
└──────────────────┘  └───────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Outbox / Sync     │  오프라인 큐 → 네트워크 복구 시 전송
                    └─────────────────────┘
```

---

## 3. 영역별 배치 기준

### `app/` — 앱 설정 전용

> **원칙:** 앱 구성 선언만 둔다. 업무 로직, DB 호출, API 호출은 절대 두지 않는다.

| 폴더 | 관리 대상 |
|---|---|
| `app/config/` | baseUrl, timeout, 환경변수 (dev / prod) |
| `app/router/` | 화면 경로 상수, 인증 redirect, go_router 설정 |
| `app/theme/` | 색상 팔레트, 폰트, 버튼/입력 공통 스타일 |

---

### `core/` — 기술 인프라

> **원칙:** 특정 업무 기능에 종속되지 않는 기술 처리만 둔다.
> WMS 업무 용어(피킹, 입고, 재고 등)가 코드에 나오면 `core`가 아니다.

| 폴더 | 관리 대상 |
|---|---|
| `core/database/` | SQLite 초기화, 테이블 생성, Outbox DAO |
| `core/network/` | Dio 인스턴스, interceptor, timeout |
| `core/error/` | AppException 타입 정의, DioException → AppException 변환 |
| `core/security/` | AES-256 암호화, SHA-256 해싱 |
| `core/sync/` | Outbox 패턴, 동기화 큐 인프라, retry 정책 |
| `core/scanner/` | *(예정)* PDA 스캐너 추상화, 벤더별 구현 |
| `core/utils/` | *(예정)* 날짜 포맷, 바코드 검증, 숫자 포맷 유틸 |

---

### `features/` — 업무 기능 모듈

> **원칙:** 기능별로 폴더를 나누고, `data / presentation` 2계층으로 책임을 분리한다.

**data 폴더 추가 시점**

```
UI 개발 단계     →  presentation/ 만 생성
데이터 연동 단계  →  data/ 추가 (datasource → model → repository 순서로)
```

**feature 현황**

| Feature | data 계층 | 비고 |
|---|---|---|
| `auth` | 완성 | datasource + model + repository 모두 구현 |
| `picking` | 미개발 | presentation만 유지, TODO 스텁 상태 |
| `product_info` | 미개발 | presentation만 유지, TODO 스텁 상태 |
| `home` | 불필요 | 서버 데이터 연동 없는 단계 |

**향후 feature 분리 계획**

현재 `home`의 메뉴에 연결된 기능들은 각각 독립 feature로 신설한다.

```
features/
├── receiving_inspection/   입고검수
├── receiving_putaway/      입고적재
├── location_move/          로케이션이동
└── stock_taking/           재고조사
```

---

### `shared/` — 업무 공통 재사용 자원

> **원칙:** 여러 feature가 공통으로 쓰는 업무 표현 자원을 둔다.
> 기술 인프라는 `core`에, feature 전용 코드는 해당 `feature`에 둔다.

| 폴더 | 관리 대상 |
|---|---|
| `shared/widget/` | feature를 가리지 않는 재사용 UI (AppFooter, StatusBadge 등) |
| `shared/enum/` | *(예정)* 작업상태, 입출고유형, 스캔타입 등 업무 공통 enum |
| `shared/model/` | *(예정)* 여러 feature가 공유하는 단순 공통 모델 |
| `shared/provider/` | *(예정)* 네트워크 연결상태, 현재 센터 선택 등 앱 전역 상태 |

---

## 4. 개발 시 판단 기준

### 파일 위치 결정 플로우

```
Q1. 이 코드에 WMS 업무 용어가 나오는가?
     │
     ├── NO  →  core/  (순수 기술 인프라)
     │
     └── YES →  Q2. 특정 feature에만 쓰이는가?
                     │
                     ├── YES →  features/해당feature/
                     │
                     └── NO  →  shared/
```

### feature 내부 파일 위치

| 코드 성격 | 위치 |
|---|---|
| 화면 레이아웃, 위젯 조합 | `presentation/screen/`, `presentation/widget/` |
| UI 상태 데이터 (State 클래스) | `presentation/provider/*_state.dart` |
| UI 상태 관리 로직 (Notifier) | `presentation/provider/*_provider.dart` |
| API 호출, DB 조회/저장 | `data/datasource/` |
| API 응답 / DB Row 변환 모델 | `data/model/` |
| 온/오프라인 전략 분기 | `data/repository/` |

### 네이밍 규칙

| 접미사 | 위치 | 의미 |
|---|---|---|
| `*Model` | `data/model/` | DB Row / API JSON 직접 변환 객체 |
| `*State` | `presentation/provider/` | UI 상태 객체 (직렬화 없음, 불변) |
| `*Datasource` | `data/datasource/` | 단일 데이터 소스 접근 |
| `*Repository` | `data/repository/` | 온/오프라인 전략 통합 |
| `*Notifier` | `presentation/provider/` | 상태 변경 로직 담당 |

---

## 5. 신규 기능 개발 순서

```
Step 1  UI 골격 작성
        features/<기능명>/presentation/
        ├── provider/*_state.dart      상태 필드 정의
        ├── provider/*_provider.dart   Notifier + TODO 스텁 메서드
        ├── screen/*_screen.dart       화면 레이아웃
        └── widget/                    하위 위젯들

Step 2  라우트 등록
        app/router/app_router.dart 에 경로 추가
        home_screen.dart 메뉴 연결 확인

Step 3  데이터 계층 추가 (연동 개발 시점)
        features/<기능명>/data/
        ├── datasource/*_local_datasource.dart    SQLite
        ├── datasource/*_remote_datasource.dart   WMS API
        ├── model/*_model.dart
        └── repository/*_repository.dart

Step 4  Notifier 연결
        TODO 스텁 → 실제 repository 호출로 교체
```

---

## 6. 오프라인 우선 원칙

> **이 앱의 핵심 설계 원칙: 모든 작업은 로컬 DB에 먼저 저장한다.**
> 네트워크가 없어도 PDA 작업은 끊기지 않는다.

```
작업 발생 (피킹, 입고 등)
    │
    ▼
로컬 SQLite 저장         ← 네트워크 상태 무관, 즉시 저장
    │
    ▼
wms_sync_outbox 큐잉     ← 서버 전송 대기 이벤트 기록
    │
    ▼
네트워크 복구 감지
    │
    ▼
SyncQueueService 실행
    ├── 성공  →  outbox 상태: PENDING → DONE
    └── 실패  →  retry_cnt 증가 → 재시도 (정책에 따라)
```

**로그인은 예외**

```
WMS API 로그인 시도
    ├── 성공   →  로컬 캐시 갱신 후 진입
    └── 실패
          ├── NetworkException  →  로컬 캐시 폴백 (오프라인 로그인)
          └── AuthException     →  에러 표시 (폴백 없음, 인증 거부)
```

---

## 7. 현재 미개발 영역

| 영역 | 위치 | 상태 |
|---|---|---|
| 피킹 리스트 조회 | `picking/presentation/provider/picking_provider.dart` | TODO 스텁 |
| 피킹 수량 저장 | `picking/presentation/provider/picking_provider.dart` | TODO 스텁 |
| 상품 바코드 조회 | `product_info/presentation/provider/product_info_provider.dart` | TODO 스텁 |
| 홈 새로고침 | `home/presentation/provider/home_provider.dart` | TODO 스텁 |
| PDA 스캐너 연동 | `core/scanner/` | 폴더 미생성 |
| 동기화 실행 | `core/sync/sync_queue_service.dart` | 구조만 존재 |
| 입고검수 | `features/receiving_inspection/` | feature 미생성 |
| 입고적재 | `features/receiving_putaway/` | feature 미생성 |
| 로케이션이동 | `features/location_move/` | feature 미생성 |
| 재고조사 | `features/stock_taking/` | feature 미생성 |
