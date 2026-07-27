# Flutter APK 빌드 및 PDA 설치 가이드

> 프로젝트: K-MART WMS PDA App  
> 작성일: 2026-07-27

---

## 프로젝트 정보

| 항목 | 값 |
|------|-----|
| Application ID | `com.umac.wms` |
| Flutter 버전 | 3.41.6 (stable) |
| Dart 버전 | 3.11.4 |
| 빌드 설정 파일 | `android/app/build.gradle.kts` |
| APK 출력 경로 | `build/app/outputs/flutter-apk/` |

---

## 1. 사전 준비

### Flutter 환경 확인

```bash
flutter --version
flutter doctor
```

`flutter doctor` 에서 Android toolchain 항목이 ✓ 이어야 빌드 가능.

### PDA 개발자 옵션 활성화 (ADB 설치 시 필요)

1. PDA **설정 → 기기 정보 → 빌드 번호** 7번 연속 탭
2. **설정 → 개발자 옵션 → USB 디버깅** ON
3. PC와 USB 케이블로 연결 후 PDA에서 "USB 디버깅 허용" 팝업 → 허용

---

## 1-1. flutter doctor 오류 해결

### Android toolchain 오류 발생 시

`flutter doctor` 실행 결과에서 아래 두 가지 오류가 발생할 수 있음.

#### 오류 1: cmdline-tools component is missing

```
[!] Android toolchain
    X cmdline-tools component is missing.
```

**해결:**

1. Android Studio 실행
2. 우측 상단 **Settings (톱니바퀴)** → **SDK Manager**
3. **SDK Tools** 탭 클릭
4. **Android SDK Command-line Tools (latest)** 체크박스 선택
5. **Apply → OK**

#### 오류 2: Android license status unknown / Some Android licenses not accepted

```
[!] Android toolchain
    ! Some Android licenses not accepted.
```

cmdline-tools 설치 완료 후 실행:

```bash
flutter doctor --android-licenses
```

프롬프트가 여러 번 나오면 모두 `y` 입력.

#### 정상 상태 확인

```bash
flutter doctor
```

```
[√] Android toolchain - develop for Android devices (Android SDK version XX.X.X)
```

`[√]` 로 표시되면 빌드 준비 완료.

---

## 2. APK 빌드

프로젝트 루트(`wms_pda_app/`)에서 실행.

### 디버그 빌드 (테스트용)

```bash
flutter build apk --debug
```

- 빠른 빌드
- Flutter DevTools, 로그 출력 가능
- 파일 크기 큼

### 릴리즈 빌드 (실 배포용)

```bash
flutter build apk --release
```

- 코드 최적화, 트리 쉐이킹 적용
- 파일 크기 최소화
- 로그 출력 없음

### 실기기(PDA) 빌드 시 백엔드 URL 지정 (필수)

앱의 기본 API 주소(`lib/app/config/env.dart`)는 Android 에뮬레이터 전용 주소(`10.0.2.2`)를 사용한다.
실제 PDA에서는 이 주소로 서버에 접근할 수 없어 **오프라인으로 잘못 인식**되므로, 빌드 시 반드시 실제 서버 IP를 지정해야 한다.

**1. 백엔드 PC의 Wi-Fi IP 확인 (PowerShell)**

```powershell
ipconfig
# → 무선 LAN 어댑터 Wi-Fi: 항목의 IPv4 주소 확인
# 예: 192.168.110.49
```

> PDA와 백엔드 PC가 **동일한 Wi-Fi 네트워크**에 연결되어 있어야 함.

**2. IP를 포함한 빌드 명령어**

```bash
flutter build apk --release --target-platform android-arm64 --dart-define=WMS_BASE_URL=http://192.168.110.49:7100
```

`192.168.110.49` 부분을 실제 확인한 IP로 교체.

---

### 아키텍처 지정 빌드 (파일 크기 최적화)

PDA CPU 아키텍처에 맞춰 빌드하면 APK 용량을 크게 줄일 수 있음.

```bash
# PDA 아키텍처 확인 (USB 연결 상태에서)
adb shell getprop ro.product.cpu.abi

# ARM64 전용 (최신 PDA 대부분) — 현재 기기: arm64-v8a
flutter build apk --release --target-platform android-arm64

# ARM32 전용 (구형 PDA)
flutter build apk --release --target-platform android-arm
```

### 빌드 완료 후 APK 위치

```
wms_pda_app/
└── build/
    └── app/
        └── outputs/
            └── flutter-apk/
                ├── app-debug.apk        ← 디버그 빌드
                └── app-release.apk      ← 릴리즈 빌드
```

---

## 3. PDA 설치 방법

### 방법 A — ADB 설치 (USB 연결, 가장 빠름)

```bash
# 연결된 기기 목록 확인
adb devices

# 신규 설치
adb install build/app/outputs/flutter-apk/app-release.apk

# 기존 앱 덮어쓰기 (업데이트)
adb install -r build/app/outputs/flutter-apk/app-release.apk

# 설치 후 바로 실행
adb shell am start -n com.umac.wms/.MainActivity
```

### 방법 B — 파일 전송 후 수동 설치

1. APK 파일을 USB 드라이브 또는 공유 폴더로 PDA에 복사
2. PDA **설정 → 보안 → 알 수 없는 소스 허용** ON
3. PDA 파일 관리자에서 APK 파일 선택 → 설치

---

## 4. 빌드 + 설치 원스텝 (USB 연결 상태)

빌드 후 바로 ADB로 설치까지 한번에 실행:

```bash
flutter build apk --release && adb install -r build/app/outputs/flutter-apk/app-release.apk
```

또는 USB 연결된 기기에 직접 실행 (디버그만 가능):

```bash
# 연결된 기기에 바로 설치 및 실행
flutter run --release
```

---

## 5. 로그 확인 (문제 발생 시)

```bash
# Flutter 로그만 필터링
flutter logs

# ADB 전체 로그 (Flutter 태그 필터)
adb logcat | findstr flutter

# 크래시 로그만
adb logcat *:E | findstr com.umac.wms
```

---

## 6. 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| `adb devices` 에 기기 미표시 | USB 디버깅 미허용 | PDA에서 USB 디버깅 허용 팝업 확인 |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | 기존 앱과 서명 불일치 | 기존 앱 삭제 후 재설치 (`adb uninstall com.umac.wms`) |
| `INSTALL_FAILED_OLDER_SDK` | PDA Android 버전이 minSdk 미만 | `build.gradle.kts`에서 `minSdk` 낮추기 |
| 앱 설치 후 바로 크래시 | 백엔드 API 주소가 로컬 PC IP | 실제 서버 IP/도메인으로 변경 후 재빌드 |
| `Gradle build failed` | Gradle 캐시 오류 | `flutter clean && flutter pub get` 후 재빌드 |

---

## 7. 빌드 전 체크리스트

- [ ] `flutter doctor` 이상 없음
- [ ] 백엔드 API URL이 PDA에서 접근 가능한 주소인지 확인
- [ ] `applicationId = "com.umac.wms"` 설정 확인
- [ ] PDA USB 디버깅 활성화 (ADB 방식 사용 시)

---

## 참고 명령어 모음

```bash
# 프로젝트 클린 (빌드 오류 시)
flutter clean && flutter pub get

# 설치된 앱 삭제
adb uninstall com.umac.wms

# APK 정보 확인
adb shell dumpsys package com.umac.wms | findstr versionName
```
