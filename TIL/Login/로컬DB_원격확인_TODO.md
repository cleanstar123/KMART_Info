# PDA 로컬 DB 원격 확인 방법 및 TODO

> 작성일: 2026-04-08  
> 프로젝트: K-Market WMS PDA App (`wms_pda_app`)

---

## 배경

- PDA 기기는 베트남 현장에서 사용
- 한국에서 로그인 이력, 작업 기록 등 PDA 로컬 DB 데이터를 원격으로 확인해야 함
- USB 직접 연결(ADB)은 거리상 불가 → 아래 5가지 방법으로 해결 가능

---

## 방법 A — WMS 서버로 로그 전송

> WMS API 개발 완료 후 적용 가능한 정식 방법

### 흐름

```
PDA 로컬 DB (베트남 현장)
  ↓
wms_sync_outbox 테이블에 전송할 데이터 적재
  ↓
백그라운드 동기화 → WMS API (POST)
  ↓
WMS DB (서버)
  ↓
한국 PC에서 DB 툴 또는 관리자 웹으로 조회
```

### 현재 상태

- `wms_sync_outbox`, `wms_sync_log` 테이블은 이미 설계 완료 (`app_database.dart`)
- 데이터를 서버로 전송하는 API 및 동기화 로직은 미개발

### TODO 항목

- [ ] WMS API — 로그 수신 엔드포인트 개발 (`POST /api/sync/outbox`)
- [ ] `SyncOutboxDatasource` 구현 — `wms_sync_outbox`에 이벤트 적재
- [ ] 백그라운드 동기화 서비스 구현
  - 온라인 상태 감지 후 outbox 데이터 서버로 전송
  - `workmanager` 패키지 사용 예정 (pubspec에 주석 처리됨)
- [ ] `wms_login_log` 서버 전송 연동
  - 한국에서 누가 언제 로그인했는지 원격 모니터링 가능
- [ ] WMS 서버 DB에서 조회할 관리자 웹 또는 DB 툴 접근 방법 정립

---

## 방법 B — 앱 내 관리자 뷰어 화면

> 방법 A 이후 보조 수단 / 현장 담당자용

### 흐름

```
관리자 계정(menu_level 높음)으로 로그인
  ↓
앱 내 DB 뷰어 화면 진입
  ↓
wms_login_log, wms_user 등 테이블 조회
  ↓
현장 담당자가 화상통화로 화면 공유 또는 스크린샷 전달
```

### 현재 상태

- 관리자 뷰어 화면 미개발
- `menu_level` 필드는 이미 `wms_user`에 존재 → 권한 분기 기준으로 활용 가능

### TODO 항목

- [ ] 관리자 전용 DB 뷰어 화면 구현
  - `menu_level` 기준으로 접근 제한
  - 조회 대상: `wms_login_log`, `wms_user`, `wms_sync_outbox` 등
- [ ] 조회 결과 CSV/텍스트 내보내기 기능 (선택)
  - 현장 담당자가 파일로 한국에 전달 가능

---

## 방법 C — ADB over WiFi

> WMS API 없이 즉시 사용 가능 / 개발·테스트 단계에 유용

### 조건

- PDA와 한국 PC가 같은 네트워크에 있어야 함
- 사내 VPN으로 베트남 현장 네트워크에 접근 가능한 경우 사용 가능
- PDA에서 무선 디버깅 허용 설정 필요

### 사용 방법

```bash
# PDA에서 무선 디버깅 활성화 후 IP 확인 (설정 → 개발자 옵션 → 무선 디버깅)
adb connect 192.168.x.x:5555

# DB 파일 PC로 복사
adb pull /data/data/com.example.wms_pda_app/databases/wms_pda_app.db

# DB Browser for SQLite로 파일 열어서 확인
```

### TODO 항목

- [ ] 베트남 현장 네트워크 → 한국 VPN 접근 가능 여부 확인
- [ ] 운영용 PDA 기기에서 무선 디버깅 허용 정책 결정
  - 보안상 운영 단계에서는 비활성화 권장

---

## 방법 D — 파일 내보내기 + 클라우드/이메일 전송

> WMS API 없이 구현 가능 / 현장 담당자가 직접 전달

### 흐름

```
앱 내 "로그 내보내기" 버튼 (관리자 계정)
  ↓
wms_login_log 등을 CSV로 변환
  ↓
Google Drive / 이메일로 전송
  ↓
한국 PC에서 수신 후 확인
```

### TODO 항목

- [ ] CSV 내보내기 기능 구현
  - 조회 대상: `wms_login_log`, `wms_sync_outbox` 등
- [ ] 공유 방법 결정 (Google Drive, 이메일, 카카오톡 등)
- [ ] 내보내기 접근 권한 `menu_level` 기준으로 제한

---

## 방법 E — 클라우드 로깅 서비스 (Firebase / Sentry)

> WMS API 독립적으로 구현 가능 / 에러 추적 겸용

### 흐름

```
로그인 이력, 에러 발생 시
  ↓
Firebase Firestore 또는 Sentry에 직접 전송
  ↓
한국 PC에서 웹 대시보드로 실시간 확인
```

### 용도 구분

| 서비스 | 주 용도 |
|---|---|
| Firebase Firestore | 로그인 이력, 작업 기록 실시간 조회 |
| Sentry | 앱 에러/크래시 추적 — 운영 배포 전 도입 권장 |

### TODO 항목

- [ ] Firebase 또는 Sentry 도입 여부 결정
- [ ] 도입 시 `wms_login_log` 이벤트를 클라우드로도 전송하도록 연동

---

## 우선순위 정리

| 방법 | 난이도 | WMS API 의존 | 권장 시점 |
|---|---|---|---|
| C. ADB over WiFi | 낮음 | 없음 | 개발/테스트 중 즉시 |
| D. 파일 내보내기 | 낮음 | 없음 | 초기 운영 임시방편 |
| A. WMS 서버 동기화 | 높음 | 있음 | WMS API 완성 후 (정식) |
| E. 클라우드 로깅 | 중간 | 없음 | 운영 배포 전 |
| B. 앱 내 관리자 뷰어 | 중간 | 없음 | 방법 A 이후 보조 수단 |

---

## SQLite 데이터 영속성

앱을 껐다 켜도 로컬 DB 데이터는 사라지지 않음

| 상황 | 데이터 유지 여부 |
|---|---|
| 앱 종료 후 재실행 | ✅ 유지 |
| PDA 재부팅 | ✅ 유지 |
| 앱 업데이트 (일반) | ✅ 유지 |
| 앱 삭제 후 재설치 | ❌ 삭제 |
| 설정 → 앱 → 데이터 초기화 | ❌ 삭제 |
| `adb shell pm clear` (개발용) | ❌ 삭제 |

오프라인 폴백 로그인이 가능한 이유:
```
최초 온라인 로그인 → upsertUser()로 로컬 저장
  ↓
이후 네트워크 없이 앱 껐다 켜도 로컬 캐시로 로그인 가능
```

---

## 참고 — USB 직접 연결 시 (개발 중만 가능)

```bash
# PDA가 USB로 개발 PC에 연결된 경우에만 사용 가능
adb pull /data/data/com.example.wms_pda_app/databases/wms_pda_app.db

# DB Browser for SQLite로 파일 열어서 확인
# 또는 앱 데이터 전체 초기화 (개발 중 DB 리셋)
adb shell pm clear com.example.wms_pda_app
```
