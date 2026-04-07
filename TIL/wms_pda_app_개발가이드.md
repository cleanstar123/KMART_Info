# WMS PDA App 개발 가이드

> 작성일: 2026-04-07  
> 프로젝트: K-Market WMS PDA App (`wms_pda_app`)  
> 위치: `C:\wonjun\P_K_MART_WMS\wms_pda_app`

---

## 1. 개발 환경 세팅

### 1-1. Flutter 설치

| 항목 | 버전 |
|---|---|
| Flutter | 3.41.6 (stable 채널) |
| Dart | 3.11.4 |
| DevTools | 2.54.2 |

```bash
# Flutter 버전 확인
flutter --version

# 환경 전체 진단 (Android SDK, JDK 등 누락 여부 확인)
flutter doctor
```

Flutter 공식 설치 경로: https://docs.flutter.dev/get-started/install/windows

**설치 순서**
1. Flutter SDK 다운로드 후 압축 해제 (예: `C:\flutter`)
2. 시스템 환경변수 PATH에 `C:\flutter\bin` 추가
3. Android Studio 설치
4. Android Studio → SDK Manager → Android SDK Command-line Tools 설치
5. `flutter doctor --android-licenses` 실행 후 라이선스 동의

---

### 1-2. Android 설정

| 항목         | 설정값                                                |
| ---------- | -------------------------------------------------- |
| compileSdk | flutter.compileSdkVersion (Flutter가 권장하는 버전 자동 사용) |
| minSdk     | flutter.minSdkVersion                              |
| targetSdk  | flutter.targetSdkVersion                           |
| Java 버전    | 17 (SOURCE / TARGET 모두)                            |
| JVM Target | 17                                                 |
| Kotlin     | 사용 중                                               |

`android/app/build.gradle.kts` 핵심 설정:

```kotlin
compileOptions {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
kotlinOptions {
    jvmTarget = JavaVersion.VERSION_17.toString()
}
```

> **주의**: JDK 17이 설치되어 있어야 빌드 가능. Android Studio에 번들된 JDK를 사용하거나, 별도로 JDK 17을 설치하고 `JAVA_HOME` 환경변수 설정 필요.

---

### 1-3. pubspec.yaml 패키지 설치

```bash
# 프로젝트 루트에서 실행
flutter pub get
```

#### 운영 의존성 (dependencies)

| 패키지 | 버전 | 용도 |
|---|---|---|
| `flutter_riverpod` | ^2.5.1 | 상태관리 (StateNotifierProvider) |
| `go_router` | ^14.2.0 | 화면 라우팅 |
| `dio` | ^5.7.0 | HTTP 통신 (서버 API 연동) |
| `freezed_annotation` | ^2.4.4 | 불변 모델 코드 생성용 어노테이션 |
| `json_annotation` | ^4.9.0 | JSON 직렬화 어노테이션 |
| `logger` | ^2.4.0 | 디버그 로그 출력 |
| `connectivity_plus` | ^6.0.5 | 네트워크 온/오프라인 감지 |
| `uuid` | ^4.5.1 | 고유 ID 생성 (outbox_id 등) |
| `sqflite` | ^2.3.3+1 | 로컬 SQLite DB (오프라인 데이터 저장) |
| `path` | ^1.9.0 | 파일 경로 처리 |
| `path_provider` | ^2.1.4 | 기기 내 파일 시스템 경로 접근 |
| `cupertino_icons` | ^1.0.8 | iOS 스타일 아이콘 |

#### 개발 의존성 (dev_dependencies)

| 패키지 | 버전 | 용도 |
|---|---|---|
| `build_runner` | ^2.4.12 | 코드 자동 생성 실행 도구 |
| `freezed` | ^2.5.7 | Freezed 모델 코드 자동 생성 |
| `json_serializable` | ^6.8.0 | `fromJson` / `toJson` 코드 자동 생성 |
| `flutter_lints` | ^6.0.0 | 코드 품질 린트 규칙 |

#### 코드 자동 생성 실행 방법

```bash
# 1회성 생성
dart run build_runner build --delete-conflicting-outputs

# 파일 변경 시 자동 재생성 (개발 중 권장)
dart run build_runner watch --delete-conflicting-outputs
```

---

## 2. 프로젝트 폴더 구조

```
wms_pda_app/
├── lib/
│   ├── main.dart                        # 앱 진입점
│   ├── app/                             # 앱 전역 설정
│   │   ├── app.dart                     # MaterialApp.router 루트 위젯 (WmsPdaApp)
│   │   ├── config/
│   │   │   └── env.dart                 # 환경 설정 (API URL 등)
│   │   ├── router/
│   │   │   └── app_router.dart          # GoRouter 라우트 정의
│   │   └── theme/
│   │       └── app_theme.dart           # 앱 전체 색상/테마 상수
│   │
│   ├── core/                            # 앱 전반에 걸쳐 공유되는 핵심 기능
│   │   ├── database/
│   │   │   ├── app_database.dart        # SQLite DB 초기화 및 테이블 생성
│   │   │   ├── outbox_dao.dart          # wms_sync_outbox 테이블 CRUD
│   │   │   └── outbox_table.dart        # 아웃박스 테이블 스키마 정의
│   │   └── sync/
│   │       ├── sync_models.dart         # 동기화 이벤트 타입 상수 정의
│   │       └── sync_queue_service.dart  # PENDING 상태 outbox를 순서대로 서버에 전송
│   │
│   └── features/                        # 기능 단위 모듈 (Feature-first 구조)
│       ├── home/                        # 홈 화면
│       │   └── presentation/
│       │       ├── provider/
│       │       │   ├── home_provider.dart   # homeProvider (StateNotifierProvider)
│       │       │   └── home_state.dart      # HomeState, HomeMenuItem 모델
│       │       ├── screen/
│       │       │   └── home_screen.dart     # 홈 화면 UI (메뉴 그리드, 헤더, 통계카드)
│       │       └── widget/
│       │           ├── home_header.dart         # 상단 헤더 (작업자명, 온라인 상태, 동기화 버튼)
│       │           ├── menu_grid.dart           # 기능 메뉴 그리드 (6개 메뉴)
│       │           ├── quick_stats_card.dart    # 대기 건수 요약 카드
│       │           └── sync_status_banner.dart  # 동기화 상태 배너
│       │
│       ├── picking/                     # 상품 피킹 기능
│       │   ├── data/
│       │   │   └── datasource/
│       │   │       └── picking_local_datasource.dart  # 피킹 데이터 로컬 DB 조회
│       │   └── presentation/
│       │       ├── provider/
│       │       │   ├── picking_provider.dart   # pickingProvider (StateNotifierProvider)
│       │       │   └── picking_state.dart      # PickingState, PickingTask, PickingItem 모델
│       │       ├── screen/
│       │       │   └── picking_screen.dart     # 피킹 화면 UI (바코드 스캔, 피킹 목록)
│       │       └── widget/
│       │           ├── picking_complete_bar.dart    # 하단 피킹 완료 버튼 바
│       │           ├── picking_header_card.dart     # 피킹리스트 헤더 정보 카드
│       │           ├── picking_item_card.dart       # 개별 피킹 아이템 카드
│       │           ├── picking_progress_card.dart   # 진행률 카드 (완료/전체)
│       │           └── picking_scan_section.dart    # 바코드 스캔 입력 섹션
│       │
│       └── product_info/                # 상품 정보 조회 기능
│           └── presentation/
│               ├── provider/
│               │   ├── product_info_provider.dart   # productInfoProvider
│               │   └── product_info_state.dart      # ProductInfoState 모델
│               ├── screen/
│               │   └── product_info_screen.dart     # 상품 정보 조회 화면
│               └── widget/
│                   ├── barcode_scan_section.dart    # 바코드 스캔 입력 위젯
│                   └── product_info_card.dart       # 상품 정보 결과 카드
│
├── android/                             # Android 네이티브 프로젝트
│   └── app/
│       └── build.gradle.kts             # Android 빌드 설정 (SDK 버전, Java 17)
├── test/                                # 테스트 코드
├── pubspec.yaml                         # 패키지 의존성 및 앱 버전 정의
└── analysis_options.yaml                # Dart 린트 규칙 설정
```

---

## 3. 아키텍처 설계 방식

### Feature-first 구조

각 기능(feature)은 독립적인 폴더 안에서 `data / presentation` 레이어로 분리됩니다.

```
features/
  <기능명>/
    data/          → 데이터 소스 (로컬 DB, 서버 API)
    presentation/
      provider/    → 상태 관리 (Riverpod Provider + State)
      screen/      → 화면 단위 Widget (1개 화면 = 1개 파일)
      widget/      → 화면 내 재사용 위젯 조각들
```

### 상태 관리: Riverpod (StateNotifierProvider)

- `StateNotifierProvider`를 사용하여 `Provider`와 `State`를 분리
- `State`는 불변 클래스로 설계하고 `copyWith`로 변경
- Freezed 패키지로 불변 모델 코드 자동 생성 가능

```dart
// 사용 예시
final homeProvider = StateNotifierProvider<HomeNotifier, HomeState>(
  (ref) => HomeNotifier(),
);

// 화면에서 사용
final state = ref.watch(homeProvider);
final notifier = ref.read(homeProvider.notifier);
```

### 라우팅: GoRouter

| 경로 | 이름 | 화면 |
|---|---|---|
| `/` | home | HomeScreen |
| `/product_info` | product_info | ProductInfoScreen |
| `/picking` | picking | PickingScreen |

```dart
// 화면 이동 방법
context.push('/picking');   // 스택에 추가
context.go('/');            // 스택 초기화 후 이동
```

---

## 4. 로컬 데이터베이스 구조 (SQLite)

`sqflite`를 사용하여 기기 내 SQLite DB를 관리합니다.  
DB 파일명: `wms_pda_app.db`

| 테이블명 | 설명 |
|---|---|
| `wms_if_item` | 상품 마스터 (ERP에서 동기화된 상품 정보) |
| `wms_if_location` | 로케이션 마스터 (창고 위치 정보) |
| `wms_pick_task_h` | 피킹 작업 헤더 (피킹 지시 단위) |
| `wms_pick_task_d` | 피킹 작업 상세 (피킹 라인별 상품/수량) |
| `wms_pick_scan_log` | 바코드 스캔 이력 로그 |
| `wms_sync_outbox` | 서버 동기화 대기 큐 (Outbox 패턴) |
| `wms_sync_log` | 서버 동기화 결과 로그 |

### Outbox 패턴 동작 방식

1. PDA에서 피킹 완료 등 이벤트 발생
2. `wms_sync_outbox`에 `sync_status_cd = 'PENDING'`으로 저장
3. 네트워크 연결 시 `SyncQueueService.flushPending()` 호출
4. 성공 시 `DONE`, 실패 시 `FAILED`로 업데이트
5. 이를 통해 오프라인에서도 데이터 유실 없이 작업 가능

---

## 5. 주요 화면 구성

| 화면 | 경로 | 주요 기능 |
|---|---|---|
| 홈 | `/` | 메뉴 그리드, 작업 대기 건수, 동기화 상태, 온/오프라인 표시 |
| 상품정보조회 | `/product_info` | 바코드 스캔으로 상품 정보 조회 |
| 상품피킹 | `/picking` | 피킹리스트 스캔 → 상품 바코드 스캔 → 수량 확인 → 피킹 완료 |

### 홈 화면 메뉴 목록

| 메뉴 | 영문명 | routeKey |
|---|---|---|
| 상품정보조회 | Product Lookup | `product_info` |
| 상품입고검수 | Inbound Inspection | `receiving_inspection` |
| 입고적재 | Put-away | `receiving_putaway` |
| 상품피킹 | Picking | `picking` |
| 로케이션이동 | Location Transfer | `location_move` |
| 재고조사 | Inventory Count | `stock_taking` |

> 현재 구현 완료: 상품정보조회, 상품피킹  
> 미구현 (향후 추가 예정): 나머지 4개 메뉴

---

## 6. 앱 실행 방법

```bash
# Android 에뮬레이터 또는 연결된 기기에서 실행
flutter run

# 특정 기기 지정 실행
flutter run -d <device_id>

# 연결된 기기 목록 확인
flutter devices

# APK 빌드 (테스트용)
flutter build apk --debug

# APK 빌드 (배포용)
flutter build apk --release
```

---

## 7. 개발 시 주의사항

1. **sqflite는 웹/Windows에서 동작하지 않음** → Android/iOS 에뮬레이터 사용 필수
2. **Freezed 사용 시 반드시 `build_runner` 실행** → `.freezed.dart`, `.g.dart` 파일 자동 생성 필요
3. **PDA 기기 특성** → 화면이 좁고 터치보다 바코드 스캔 위주의 UX 설계 필요
4. **오프라인 우선 설계** → 모든 데이터는 로컬 SQLite에 먼저 저장 후 서버 동기화
5. **앱 테마 색상** → 메인 그린 `#006B3F`, 피킹 앰버 `#F59E0B`
