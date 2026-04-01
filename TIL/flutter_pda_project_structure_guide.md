# Flutter PDA 프로젝트 시작 가이드

## 1. 프로젝트 개요

본 문서는 WMS 연동 PDA 애플리케이션 개발을 위한 **Flutter 프로젝트 폴더 구조**, **권장 개발 툴**, **사전 설치 항목**, **필수 패키지**를 정리한 문서이다.

대상 프로젝트 특성:
- WMS 연동 PDA 앱
- 바코드 스캐너 연동 필요
- 오프라인 우선(Offline-First) 구조
- SQLite 기반 로컬 저장
- Spring Boot 서버 연동

---

## 2. 권장 개발 툴

### 2.1 IDE
권장 개발 툴은 아래 둘 중 하나이다.

- **Android Studio**
- **Visual Studio Code**

### 2.2 추천
실무적으로는 아래처럼 추천한다.

- **Android Studio**
  - Flutter / Android SDK / Emulator 관리가 편함
  - Platform Channel, Android native 연동 작업에 유리
  - PDA 스캐너 연동 시 Android 관련 설정 확인이 쉬움

- **Visual Studio Code**
  - 가볍고 빠름
  - Flutter 코드 작성은 편함
  - 다만 Android 설정이나 native 연동은 Android Studio가 더 편리함

### 2.3 최종 추천
본 프로젝트는 **Flutter + Platform Channel + Android PDA 연동**이 필요하므로  
**주 개발 툴은 Android Studio**를 추천한다.

보조적으로 VS Code를 사용해도 된다.

---

## 3. 사전 설치 항목

프로젝트를 시작하기 전에 아래 항목을 설치해야 한다.

### 3.1 필수 설치
- Flutter SDK
- Dart SDK
- Android Studio
- Android SDK
- JDK 17 이상
- Git

### 3.2 선택 설치
- Visual Studio Code
- Postman 또는 Bruno
- DB Browser for SQLite
- DBeaver
- GitKraken 또는 SourceTree

---

## 4. 설치 후 확인할 것

터미널에서 아래 명령어를 실행한다.

```bash
flutter doctor
```

확인 항목:
- Flutter SDK 정상 설치
- Android toolchain 정상
- Android Studio 정상 인식
- connected device 인식 가능 여부

---

## 5. 프로젝트 생성 명령어

```bash
flutter create wms_pda_app
```

생성 후 프로젝트 폴더로 이동:

```bash
cd wms_pda_app
```

---

## 6. 권장 프로젝트 폴더 구조

본 프로젝트는 기능별(feature-first) + 공통(core) 구조를 추천한다.

```text
lib/
├─ main.dart
├─ app/
│  ├─ app.dart
│  ├─ router/
│  │  └─ app_router.dart
│  ├─ theme/
│  │  └─ app_theme.dart
│  └─ config/
│     └─ env.dart
│
├─ core/
│  ├─ constants/
│  │  ├─ api_constants.dart
│  │  ├─ db_constants.dart
│  │  └─ app_constants.dart
│  ├─ utils/
│  │  ├─ logger.dart
│  │  ├─ date_util.dart
│  │  └─ barcode_util.dart
│  ├─ errors/
│  │  ├─ exceptions.dart
│  │  └─ failures.dart
│  ├─ network/
│  │  ├─ dio_client.dart
│  │  ├─ dio_interceptor.dart
│  │  └─ network_checker.dart
│  ├─ database/
│  │  ├─ app_database.dart
│  │  ├─ tables/
│  │  └─ daos/
│  ├─ scanner/
│  │  ├─ scanner_service.dart
│  │  ├─ scanner_channel.dart
│  │  └─ scanner_models.dart
│  ├─ sync/
│  │  ├─ sync_service.dart
│  │  ├─ sync_queue_service.dart
│  │  ├─ sync_worker.dart
│  │  └─ sync_models.dart
│  └─ widgets/
│     ├─ common_app_bar.dart
│     ├─ common_button.dart
│     └─ loading_overlay.dart
│
├─ features/
│  ├─ auth/
│  │  ├─ data/
│  │  │  ├─ datasource/
│  │  │  ├─ dto/
│  │  │  ├─ repository/
│  │  │  └─ mapper/
│  │  ├─ domain/
│  │  │  ├─ model/
│  │  │  ├─ repository/
│  │  │  └─ usecase/
│  │  └─ presentation/
│  │     ├─ provider/
│  │     ├─ screen/
│  │     └─ widget/
│  │
│  ├─ product_info/
│  ├─ receiving_inspection/
│  ├─ receiving_putaway/
│  ├─ picking/
│  ├─ location_move/
│  └─ stock_taking/
│
└─ shared/
   ├─ model/
   ├─ enum/
   └─ provider/
```

---

## 7. 기능 폴더 내부 권장 구조

예: `features/picking`

```text
features/picking/
├─ data/
│  ├─ datasource/
│  │  ├─ picking_local_datasource.dart
│  │  └─ picking_remote_datasource.dart
│  ├─ dto/
│  │  ├─ picking_task_dto.dart
│  │  └─ picking_item_dto.dart
│  ├─ mapper/
│  │  └─ picking_mapper.dart
│  └─ repository/
│     └─ picking_repository_impl.dart
│
├─ domain/
│  ├─ model/
│  │  ├─ picking_task.dart
│  │  └─ picking_item.dart
│  ├─ repository/
│  │  └─ picking_repository.dart
│  └─ usecase/
│     ├─ get_picking_task_usecase.dart
│     ├─ scan_picking_item_usecase.dart
│     └─ complete_picking_usecase.dart
│
└─ presentation/
   ├─ provider/
   │  ├─ picking_provider.dart
   │  └─ picking_state.dart
   ├─ screen/
   │  ├─ picking_list_screen.dart
   │  └─ picking_detail_screen.dart
   └─ widget/
      ├─ picking_item_card.dart
      ├─ picking_progress_bar.dart
      └─ scan_input_field.dart
```

---

## 8. 폴더 구조 설계 이유

### 8.1 core
프로젝트 전체에서 공통으로 쓰는 요소를 모아두는 영역이다.

예:
- Dio 설정
- DB 설정
- Logger
- Scanner 공통 서비스
- Sync 공통 서비스

### 8.2 features
업무 기능별 모듈 영역이다.

예:
- 상품조회
- 입고검수
- 입고적재
- 피킹
- 로케이션 이동
- 재고조사

### 8.3 shared
기능 간 공통으로 재사용되는 모델/enum/provider를 둔다.

---

## 9. 필수 패키지 정리

아래 패키지들을 `pubspec.yaml`에 추가한다.

### 9.1 기본 패키지

```yaml
dependencies:
  flutter:
    sdk: flutter

  flutter_riverpod: ^2.5.1
  go_router: ^14.2.0
  dio: ^5.7.0
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0
  connectivity_plus: ^6.0.5
  uuid: ^4.5.1
  logger: ^2.4.0
  workmanager: ^0.5.2
  sqflite: ^2.3.3+1
  path: ^1.9.0
  path_provider: ^2.1.4
```

### 9.2 코드 생성 패키지

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter

  build_runner: ^2.4.12
  freezed: ^2.5.7
  json_serializable: ^6.8.0
```

### 9.3 선택 패키지
프로젝트 상황에 따라 아래도 고려 가능하다.

```yaml
dependencies:
  flutter_dotenv: ^5.1.0
  intl: ^0.19.0
  permission_handler: ^11.3.1
  flutter_secure_storage: ^9.2.2
```

---

## 10. 패키지 설치 명령어

```bash
flutter pub get
```

---

## 11. 코드 생성 명령어

freezed / json_serializable 사용 시 아래 명령어를 사용한다.

```bash
dart run build_runner build --delete-conflicting-outputs
```

자동 감시 모드:

```bash
dart run build_runner watch --delete-conflicting-outputs
```

---

## 12. Android 연동 시 추가 고려사항

본 프로젝트는 PDA 스캐너 연동이 필요하므로 Android 폴더에 대한 이해가 필요하다.

관련 위치:
- `android/app/src/main/`
- `android/app/src/main/kotlin/...`

주요 작업:
- Platform Channel 설정
- 벤더별 Scanner Adapter 구현
- 권한 설정
- Broadcast / Intent 처리
- DataWedge 또는 벤더 SDK 연동

---

## 13. 추천 보조 툴

### 13.1 API 테스트
- Postman
- Bruno

### 13.2 DB 확인
- DB Browser for SQLite
- DBeaver

### 13.3 문서/설계
- draw.io
- Figma
- FigJam
- Notion

### 13.4 로그 확인
- Android Studio Logcat

---

## 14. 개발 시작 순서 추천

### 1단계
환경 세팅
- Flutter 설치
- Android Studio 설치
- SDK 설치
- JDK 설치
- flutter doctor 확인

### 2단계
기본 프로젝트 생성
- Flutter 프로젝트 생성
- 패키지 설치
- 폴더 구조 생성

### 3단계
공통 기반 개발
- Router
- Dio Client
- Logger
- SQLite 초기 설정
- Connectivity 감지
- UUID 생성 유틸

### 4단계
핵심 공통 기능 개발
- Scanner Adapter
- Sync Queue
- Local DB Repository
- Base Response 처리

### 5단계
업무 기능 개발
- 로그인
- 상품조회
- 피킹
- 입고검수
- 입고적재
- 로케이션 이동
- 재고조사

---

## 15. 권장 추가 사항

### 15.1 SQLite 구현 방식
문서상 로컬 DB는 SQLite로 정의하되,
실제 구현은 아래 중 하나를 선택한다.

- sqflite 직접 사용
- Drift 사용

오프라인 동기화와 복잡한 쿼리가 중요하면 **Drift 사용**을 권장한다.

### 15.2 네트워크 판정 방식
`connectivity_plus`는 네트워크 종류 감지만 가능하므로,
실제 온라인 여부는 서버 health check API와 함께 판단하는 것이 좋다.

### 15.3 workmanager 사용 범위
중요한 작업은 아래 순서를 권장한다.

1. 로컬 DB에 즉시 저장
2. 앱 활성 상태에서 즉시 동기화 시도
3. 실패 시 Sync Queue 저장
4. 백그라운드 재시도

---

## 16. 최종 요약

### 개발 툴
- 주 개발 툴: Android Studio
- 보조 툴: Visual Studio Code

### 필수 설치
- Flutter SDK
- Android Studio
- Android SDK
- JDK 17 이상
- Git

### 핵심 패키지
- flutter_riverpod
- go_router
- dio
- freezed_annotation
- json_annotation
- connectivity_plus
- uuid
- logger
- workmanager
- sqflite
- path_provider

### 권장 구조
- `core`: 공통 기능
- `features`: 업무 기능별 모듈
- `shared`: 공통 모델/enum/provider

### 핵심 설계 포인트
- Offline-First
- SQLite 기반 로컬 저장
- Sync Queue 구조
- Platform Channel 기반 Scanner Adapter
- 기능별 폴더 분리
