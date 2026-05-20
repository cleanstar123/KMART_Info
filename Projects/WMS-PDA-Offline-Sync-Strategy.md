# WMS PDA — Offline 전략 & Sync 구조 설계

#kmarket #wms #pda #flutter #offline #sync #architecture

---

## 문서 개요

| 항목        | 내용                                   |
| --------- | ------------------------------------ |
| **대상**    | K-Market WMS PDA App (Flutter)       |
| **목적**    | Offline-First 아키텍처 설계 및 서버 동기화 전략 수립 |
| **작성 기준** | 현재 코드베이스 분석 기반 (`wms_pda_app`)       |
| **작성일**   | 2026-05-07                           |

---

## 1. 현황 분석 (As-Is)

### 1-1. 기술 스택 (현재 확정)

| 레이어     | 라이브러리             | 버전              | 용도                |
| ------- | ----------------- | --------------- | ----------------- |
| 상태관리    | flutter_riverpod  | ^2.5.1          | StateNotifier 패턴  |
| 라우팅     | go_router         | ^14.2.0         | 선언형 네비게이션         |
| HTTP    | dio               | ^5.7.0          | REST API 통신       |
| 로컬 DB   | sqflite           | ^2.3.3+1        | SQLite (모바일)      |
| 보조 DB   | sembast           | ^3.7.1          | NoSQL (선택적)       |
| 네트워크 감지 | connectivity_plus | ^6.0.5          | 온/오프라인 상태 감지      |
| 암호화     | crypto + encrypt  | ^3.0.6 / ^5.0.3 | SHA-256 / AES-256 |
| ID 생성   | uuid              | ^4.5.1          | Outbox ID 등       |

### 1-2. 이미 구현된 Offline 지원

```
✅ Auth (인증)
   ├── 로컬 사용자 캐시 (SQLite: wms_user)
   ├── SHA-256 비밀번호 해싱
   ├── 개인정보 AES-256 암호화 (주소, 전화번호, 이메일)
   └── 오프라인 폴백 로그인 (API 실패 → 캐시 로그인)

✅ i18n (다국어)
   ├── 앱 번들 JSON 폴백 (assets/i18n/{ko,en,vi}.json)
   ├── 로컬 DB 캐시 (wms_i18n)
   ├── 서버 동기화 후 오버레이 적용
   └── 로케일 설정 영속화 (wms_app_setting)

✅ Outbox 기반 구조 (골격만 존재)
   ├── wms_sync_outbox 테이블 정의됨
   ├── OutboxDao (insertOutbox, findPendingOutboxes)
   └── SyncQueueService (flushPending - 미완성)
```

### 1-3. 미구현 부분 (Gap 분석)

```
❌ Picking — Repository/Datasource 없음, Mock 데이터만 존재
❌ 입고검수 — Provider/Repository 없음, Mock 데이터만 존재
❌ 입고적재 — Provider/Repository 없음, Mock 데이터만 존재
❌ 로케이션이동 — Provider/Repository 없음, Mock 데이터만 존재
❌ 상품정보 — Repository 없음, searchProduct() TODO 상태
❌ SyncEngine — flushPending()이 실제 API 호출 없이 DONE 처리
❌ 네트워크 감지 — connectivity_plus import만 됨, 미연결
❌ 충돌 해결 — 전략 없음
❌ 델타 동기화 — lastSyncedAt 미추적
❌ 백그라운드 동기화 — workmanager 주석 처리됨
```

---

## 2. Offline 전략 설계 (To-Be)

### 2-1. 핵심 원칙

```
┌──────────────────────────────────────────────────────────────────┐
│                    Offline-First 핵심 원칙                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. LOCAL-FIRST                                                  │
│     모든 쓰기 작업은 로컬 DB에 먼저 저장 → 이후 서버 동기화              │
│                                                                  │
│  2. OPTIMISTIC UI                                                │
│     사용자 액션 즉시 UI 반영 (서버 응답 대기 없음)                      │
│                                                                  │
│  3. OUTBOX PATTERN                                               │
│     모든 변경사항을 큐에 적재 → 신뢰성 있는 순서 보장 전송              │
│                                                                  │
│  4. PULL-ON-OPEN                                                 │
│     앱 기동 시 / 온라인 복귀 시 서버에서 최신 데이터 Pull               │
│                                                                  │
│  5. SERVER-WINS (마스터 데이터)                                    │
│     상품, 로케이션 마스터는 서버 데이터가 항상 우선                      │
│                                                                  │
│  6. LAST-WRITE-WINS (작업 수량)                                   │
│     피킹/적재 수량 등 사용자 입력은 최신 타임스탬프 우선                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 2-2. 기능별 Offline 전략

#### 상품 마스터 / 로케이션 (Master Data)

| 항목 | 전략 |
|------|------|
| **방향** | Pull-Only (서버 → 로컬) |
| **트리거** | 앱 기동 시, 수동 동기화 버튼 |
| **충돌** | Server-Wins (항상 서버 데이터 덮어쓰기) |
| **주기** | 로그인 직후 1회 + 수동 |
| **기존 테이블** | `wms_if_item`, `wms_if_location` (이미 스키마 존재) |
| **오프라인 동작** | 마지막 동기화된 마스터 데이터로 조회 |

#### 피킹 (Picking)

| 항목 | 전략 |
|------|------|
| **Pull** | 담당 피킹 리스트 → 로컬 저장 |
| **Push** | 수량 업데이트, 완료 처리 → Outbox 큐 |
| **충돌** | 서버 배정 수량 Server-Wins / 실피킹 수량 Last-Write-Wins |
| **오프라인 동작** | 로컬 저장된 태스크로 완전한 작업 가능 |
| **기존 테이블** | `wms_pick_task_h`, `wms_pick_task_d`, `wms_pick_scan_log` (스키마 존재) |

#### 입고검수 (Receiving Inspection)

| 항목 | 전략 |
|------|------|
| **Pull** | 검수 대상 컨테이너/상품 리스트 |
| **Push** | 정상수량, 불량수량 입력 → Outbox 큐 |
| **충돌** | Last-Write-Wins (현장 검수 결과 우선) |
| **오프라인 동작** | 전체 검수 작업 오프라인 가능, 온라인 복귀 시 일괄 전송 |

#### 입고적재 (Receiving Putaway)

| 항목 | 전략 |
|------|------|
| **Pull** | 적재 대기 상품 리스트 |
| **Push** | 로케이션 배정 완료 → Outbox 큐 |
| **충돌** | Last-Write-Wins |
| **오프라인 동작** | 로컬 저장 상품 기준 스캔 작업 가능 |

#### 로케이션이동 (Location Move)

| 항목 | 전략 |
|------|------|
| **Pull** | 이동 지시 리스트 |
| **Push** | 이동 완료 (FROM → TO) → Outbox 큐 |
| **충돌** | Last-Write-Wins |
| **오프라인 동작** | 지시된 이동 오프라인 처리 가능 |

#### 상품정보 조회 (Product Info)

| 항목 | 전략 |
|------|------|
| **방향** | 주로 Pull (Read-Only) |
| **오프라인** | `wms_if_item` 로컬 캐시에서 바코드 검색 |
| **온라인** | 서버 실시간 조회 (재고수량 등 최신값 필요) |
| **Fallback** | 서버 응답 없으면 로컬 캐시 결과 표시 + 안내 메시지 |

---

## 3. Sync 구조 설계

### 3-1. 전체 아키텍처

```
┌────────────────────────────────────────────────────────────────────┐
│                        PDA App (Flutter)                           │
│                                                                    │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────┐    │
│  │   UI Layer  │◄──►│  Providers   │◄──►│   Repositories     │    │
│  │ (Screens)   │    │ (Riverpod)   │    │ (Impl)             │    │
│  └─────────────┘    └──────────────┘    └─────────┬──────────┘    │
│                                                    │               │
│                              ┌─────────────────────┴──────────┐   │
│                              │                                │   │
│                     ┌────────▼────────┐         ┌────────────▼─┐  │
│                     │  LocalDatasource│         │RemoteDatasrc │  │
│                     │   (SQLite)      │         │   (Dio)      │  │
│                     └────────┬────────┘         └──────────────┘  │
│                              │                                     │
│                     ┌────────▼──────────────────────────────────┐  │
│                     │              SyncEngine                   │  │
│                     │  ┌───────────┐    ┌──────────────────┐   │  │
│                     │  │  Outbox   │    │   Pull Sync      │   │  │
│                     │  │  (Push)   │    │  (Delta Pull)    │   │  │
│                     │  └───────────┘    └──────────────────┘   │  │
│                     └───────────────────────────────────────────┘  │
│                                                                    │
└──────────────────────────────────────────┬─────────────────────────┘
                                           │ HTTPS
                                ┌──────────▼──────────┐
                                │    WMS API Server    │
                                │  (Spring Boot, etc.) │
                                └─────────────────────┘
```

### 3-2. Outbox Pattern (Push Sync)

#### 흐름도

```
사용자 액션 (예: 피킹 수량 입력)
        │
        ▼
[1] UI 즉시 업데이트 (Optimistic)
        │
        ▼
[2] 로컬 DB 저장 (wms_pick_task_d)
        │
        ▼
[3] Outbox 큐에 적재 (wms_sync_outbox)
    {
      outbox_id: uuid,
      event_type_cd: "PICK_QTY_UPDATED",
      sync_status_cd: "PENDING",
      payload_json: { pick_no, pick_seq, picked_qty, timestamp }
    }
        │
        ├─── 온라인? ──► [4] 즉시 flushPending() 시도
        │                         │
        │                    성공 ──► sync_status: DONE
        │                    실패 ──► retry_cnt++, 큐 유지
        │
        └─── 오프라인? ─► 큐 유지, 온라인 복귀 시 자동 flush
```

#### Outbox 이벤트 타입 (eventTypeCd)

| 이벤트 | 설명 | API 엔드포인트 |
|--------|------|---------------|
| `PICK_QTY_UPDATED` | 피킹 수량 변경 | PUT /api/picking/{pick_no}/lines/{seq} |
| `PICK_COMPLETED` | 피킹 완료 | POST /api/picking/{pick_no}/complete |
| `INSPECT_QTY_UPDATED` | 검수 수량 입력 | PUT /api/inspection/{insp_no}/items/{seq} |
| `INSPECT_COMPLETED` | 검수 완료 | POST /api/inspection/{insp_no}/complete |
| `PUTAWAY_COMPLETED` | 적재 완료 | POST /api/putaway/{putaway_no}/items/{seq}/complete |
| `LOCATION_MOVE_DONE` | 로케이션이동 완료 | POST /api/location-move/{move_no}/items/{seq}/complete |

#### Retry 전략 (Exponential Backoff)

```dart
// SyncQueueService 구현 방향
int _backoffSeconds(int retryCnt) {
  // 1회: 즉시, 2회: 30초, 3회: 2분, 4회: 10분, 5회+: 30분
  final delays = [0, 30, 120, 600, 1800];
  return delays[retryCnt.clamp(0, delays.length - 1)];
}

const int maxRetry = 5; // 5회 초과 시 FAILED 처리 + 알림
```

### 3-3. Pull Sync (Delta Pull)

#### 흐름도

```
앱 기동 / 온라인 복귀 / 수동 새로고침
        │
        ▼
[1] 각 테이블별 lastSyncedAt 조회
    (wms_app_setting: 'last_sync_{feature}')
        │
        ▼
[2] GET /api/{feature}/changes?since={lastSyncedAt}
        │
        ▼
[3] 변경 항목 Upsert (로컬 DB)
        │
        ▼
[4] lastSyncedAt 갱신
        │
        ▼
[5] Provider 상태 갱신 → UI 자동 반영
```

#### Delta Pull API 규약 (서버와 합의 필요)

```
GET /api/picking/tasks?since=2026-05-01T09:00:00Z&site_cd=HAN01
→ Response:
{
  "updated": [...],   // 변경/신규 태스크
  "deleted": [...]    // 삭제된 태스크 ID 목록
}

GET /api/master/items?since=2026-05-01T09:00:00Z
GET /api/master/locations?since=2026-05-01T09:00:00Z
```

### 3-4. SyncEngine 설계

```dart
// lib/core/sync/sync_engine.dart

class SyncEngine {
  // ── Push (Outbox Flush) ─────────────────────────────────────────
  Future<SyncResult> pushPending() async {
    final pending = await _outboxDao.findPendingOutboxes();
    for (final entry in pending) {
      try {
        await _dispatchToApi(entry);                       // 실제 API 호출
        await _outboxDao.updateSyncStatus(entry['outbox_id'], 'DONE', 0);
      } on NetworkException {
        // 네트워크 에러: retry_cnt 유지, 나중에 재시도
        break;
      } on AppException catch (e) {
        // 비즈니스 에러: FAILED 처리
        await _outboxDao.updateSyncStatus(
          entry['outbox_id'], 'FAILED', entry['retry_cnt'] + 1
        );
      }
    }
    return SyncResult(...);
  }

  // ── Pull (Delta Download) ───────────────────────────────────────
  Future<void> pullAll() async {
    await Future.wait([
      _pullMasterData(),
      _pullPickingTasks(),
      _pullInspectionTasks(),
      _pullPutawayTasks(),
      _pullLocationMoveTasks(),
    ]);
  }

  // ── Full Sync ───────────────────────────────────────────────────
  Future<SyncResult> sync() async {
    final push = await pushPending();  // 먼저 로컬 변경사항 전송
    await pullAll();                   // 서버 최신 데이터 수신
    await _updateSyncTimestamp();
    return push;
  }

  // ── API 디스패치 (이벤트 타입별 라우팅) ────────────────────────
  Future<void> _dispatchToApi(Map<String, dynamic> entry) async {
    final payload = jsonDecode(entry['payload_json'] as String);
    switch (entry['event_type_cd']) {
      case 'PICK_QTY_UPDATED':
        await _pickingRemote.updateLineQty(payload);
      case 'PICK_COMPLETED':
        await _pickingRemote.completeTask(payload);
      // ... 기타 이벤트
    }
  }
}
```

### 3-5. 충돌 해결 전략 (Conflict Resolution)

```
┌───────────────────────────────────────────────────────┐
│                  충돌 해결 매트릭스                       │
├────────────────┬─────────────────┬────────────────────┤
│   데이터 유형   │     전략         │      이유           │
├────────────────┼─────────────────┼────────────────────┤
│ 상품 마스터     │ Server-Wins     │ ERP 원본 신뢰        │
│ 로케이션 마스터 │ Server-Wins     │ ERP 원본 신뢰        │
│ 피킹 배정 수량  │ Server-Wins     │ 서버가 배정 권한 보유 │
│ 피킹 실피킹 수량│ Last-Write-Wins │ 작업자 현장 측정값    │
│ 검수 수량      │ Last-Write-Wins │ 작업자 현장 측정값    │
│ 적재 로케이션  │ Last-Write-Wins │ 작업자 현장 결정값    │
│ 이동 완료 상태 │ Last-Write-Wins │ 작업자 현장 결정값    │
│ 앱 설정        │ Local-Wins      │ 기기별 개인 설정      │
└────────────────┴─────────────────┴────────────────────┘
```

---

## 4. DB 스키마 확장 계획

### 4-1. 기존 테이블 활용 (변경 없음)

| 테이블 | 목적 | 상태 |
|--------|------|------|
| `wms_user` | 사용자 캐시 | ✅ 완성 |
| `wms_i18n` | 다국어 메시지 | ✅ 완성 |
| `wms_app_setting` | 앱 설정/메타 | ✅ 완성 (lastSyncedAt 키 추가 예정) |
| `wms_sync_outbox` | Outbox 큐 | ✅ 스키마 완성, 로직 미완 |
| `wms_sync_log` | 동기화 이력 | ✅ 완성 |
| `wms_pick_task_h` | 피킹 헤더 | ✅ 스키마 완성, 데이터 연결 미완 |
| `wms_pick_task_d` | 피킹 상세 | ✅ 스키마 완성, 데이터 연결 미완 |
| `wms_if_item` | 상품 마스터 | ✅ 스키마 완성, Pull 로직 미완 |
| `wms_if_location` | 로케이션 마스터 | ✅ 스키마 완성, Pull 로직 미완 |

### 4-2. 신규 테이블 (추가 필요)

```sql
-- 입고검수 헤더
CREATE TABLE IF NOT EXISTS wms_insp_task_h (
  insp_no       TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  cntr_cd       TEXT,              -- 컨테이너 코드
  status_cd     TEXT DEFAULT 'NEW',  -- NEW / IN_PROGRESS / COMPLETED
  sync_status   TEXT DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT
);

-- 입고검수 상세
CREATE TABLE IF NOT EXISTS wms_insp_task_d (
  insp_no       TEXT NOT NULL,
  insp_seq      INTEGER NOT NULL,
  item_cd       TEXT NOT NULL,
  order_qty     INTEGER DEFAULT 0,
  normal_qty    INTEGER DEFAULT 0,
  defective_qty INTEGER DEFAULT 0,
  sync_status   TEXT DEFAULT 'SYNCED',
  local_upd_dt  TEXT,
  PRIMARY KEY (insp_no, insp_seq)
);

-- 입고적재 태스크
CREATE TABLE IF NOT EXISTS wms_putaway_task (
  putaway_no    TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  status_cd     TEXT DEFAULT 'NEW',
  sync_status   TEXT DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT
);

-- 입고적재 상세
CREATE TABLE IF NOT EXISTS wms_putaway_task_d (
  putaway_no      TEXT NOT NULL,
  putaway_seq     INTEGER NOT NULL,
  item_cd         TEXT NOT NULL,
  quantity        INTEGER DEFAULT 0,
  staging_loc     TEXT,
  target_loc      TEXT,
  status_cd       TEXT DEFAULT 'PENDING',  -- PENDING / COMPLETED
  sync_status     TEXT DEFAULT 'SYNCED',
  local_upd_dt    TEXT,
  PRIMARY KEY (putaway_no, putaway_seq)
);

-- 로케이션 이동 태스크
CREATE TABLE IF NOT EXISTS wms_loc_move_task (
  move_no       TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  status_cd     TEXT DEFAULT 'NEW',
  sync_status   TEXT DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT
);

-- 로케이션 이동 상세
CREATE TABLE IF NOT EXISTS wms_loc_move_task_d (
  move_no       TEXT NOT NULL,
  move_seq      INTEGER NOT NULL,
  item_cd       TEXT NOT NULL,
  quantity      INTEGER DEFAULT 0,
  from_loc      TEXT NOT NULL,
  to_loc        TEXT,
  status_cd     TEXT DEFAULT 'PENDING',  -- PENDING / COMPLETED
  sync_status   TEXT DEFAULT 'SYNCED',
  local_upd_dt  TEXT,
  PRIMARY KEY (move_no, move_seq)
);
```

### 4-3. wms_app_setting 키 추가 (lastSyncedAt 관리)

```
기존 테이블에 아래 키들 insert/upsert:

  key: 'last_sync_master'      value: ISO-8601 timestamp
  key: 'last_sync_picking'     value: ISO-8601 timestamp
  key: 'last_sync_inspection'  value: ISO-8601 timestamp
  key: 'last_sync_putaway'     value: ISO-8601 timestamp
  key: 'last_sync_location'    value: ISO-8601 timestamp
```

### 4-4. sync_status_cd 표준

| 코드 | 의미 |
|------|------|
| `SYNCED` | 서버와 동기화 완료 |
| `PENDING` | 아직 서버에 전송 안 됨 |
| `FAILED` | 전송 실패 (maxRetry 초과) |
| `CONFLICT` | 충돌 감지, 수동 해결 필요 |

---

## 5. 코드 구조 (Directory Structure)

```
lib/
├── core/
│   ├── database/
│   │   ├── app_database.dart        ← 기존 (신규 테이블 추가)
│   │   └── outbox_dao.dart          ← 기존 (완성)
│   ├── sync/
│   │   ├── sync_engine.dart         ← 신규 (핵심 구현)
│   │   ├── sync_queue_service.dart  ← 기존 (재구현)
│   │   ├── sync_models.dart         ← 기존 (이벤트 상수 확장)
│   │   └── sync_state.dart          ← 신규 (동기화 상태 모델)
│   ├── network/
│   │   ├── dio_client.dart          ← 기존
│   │   └── network_state.dart       ← 신규 (connectivity_plus 연결)
│   ├── error/
│   │   ├── app_exception.dart       ← 기존
│   │   └── api_error_handler.dart   ← 기존
│   └── security/
│       ├── password_hasher.dart     ← 기존
│       └── field_encryptor.dart     ← 기존 (키 관리 개선 필요)
│
├── features/
│   ├── picking/
│   │   └── data/
│   │       ├── repository/picking_repository_impl.dart  ← 신규
│   │       └── datasource/
│   │           ├── picking_local_datasource.dart        ← 신규
│   │           └── picking_remote_datasource.dart       ← 신규
│   ├── receiving_inspection/
│   │   └── data/
│   │       ├── repository/inspection_repository_impl.dart
│   │       └── datasource/{local,remote}_datasource.dart
│   ├── receiving_putaway/
│   │   └── data/
│   │       ├── repository/putaway_repository_impl.dart
│   │       └── datasource/{local,remote}_datasource.dart
│   └── location_move/
│       └── data/
│           ├── repository/location_move_repository_impl.dart
│           └── datasource/{local,remote}_datasource.dart
```

---

## 6. 네트워크 상태 관리

### 6-1. NetworkState Provider (신규 구현)

```dart
// lib/core/network/network_state.dart

final networkStateProvider = StreamProvider<bool>((ref) {
  return Connectivity()
    .onConnectivityChanged
    .map((results) => results.any((r) => r != ConnectivityResult.none));
});

// HomeProvider에서 연결:
ref.listen(networkStateProvider, (_, next) {
  next.whenData((isOnline) {
    state = state.copyWith(isOnline: isOnline);
    if (isOnline) {
      // 온라인 복귀 → 자동 동기화 트리거
      ref.read(syncEngineProvider).sync();
    }
  });
});
```

### 6-2. 동기화 타이밍

| 트리거 | Pull | Push |
|--------|------|------|
| 앱 기동 | ✅ | ✅ (Pending 있으면) |
| 온라인 복귀 | ✅ | ✅ |
| 수동 새로고침 (pull-to-refresh) | ✅ | ✅ |
| 헤더 Sync 버튼 | ✅ | ✅ |
| 작업 완료 직후 | ❌ | ✅ (완료 이벤트만) |

---

## 7. Provider 연동 패턴

### 7-1. Repository 패턴 (Picking 예시)

```dart
// Riverpod Provider 구성
final pickingRepositoryProvider = Provider<PickingRepository>((ref) {
  return PickingRepositoryImpl(
    local: ref.read(pickingLocalDatasourceProvider),
    remote: ref.read(pickingRemoteDatasourceProvider),
    outbox: ref.read(outboxDaoProvider),
  );
});

// PickingNotifier에서 사용
class PickingNotifier extends StateNotifier<PickingState> {
  final PickingRepository _repo;
  final SyncEngine _syncEngine;

  Future<void> updatePickedQty(int seq, int qty) async {
    // 1. Optimistic UI
    state = state.copyWith(/* 수량 반영 */);

    // 2. 로컬 저장 + Outbox 적재
    await _repo.updateLineQtyLocal(
      pickNo: state.task!.header.pickNo,
      seq: seq,
      qty: qty,
    );

    // 3. Push 시도 (실패해도 Outbox에 남음)
    await _syncEngine.pushPending();
  }
}
```

### 7-2. 홈 화면 통계 연동

```dart
// HomeState의 waitingInboundCount, waitingPickingCount 실데이터 연결

class HomeNotifier extends StateNotifier<HomeState> {
  Future<void> refresh() async {
    // 로컬 DB에서 카운트 조회
    final pickingCount = await _pickingLocal.getPendingCount();
    final inboundCount = await _inspectionLocal.getPendingCount();
    state = state.copyWith(
      waitingPickingCount: pickingCount,
      waitingInboundCount: inboundCount,
      unsyncedCount: await _outboxDao.getPendingCount(),
    );
  }
}
```

---

## 8. 구현 로드맵

### Phase 1 — Sync 인프라 완성 (우선순위: 높음)

```
[ ] SyncEngine.pushPending() 실제 API 호출 구현
[ ] NetworkState Provider 연결 (connectivity_plus 활용)
[ ] HomeProvider ↔ NetworkState 연동
[ ] SyncQueueService Retry 로직 (Exponential Backoff)
[ ] wms_app_setting에 lastSyncedAt 키 관리
```

### Phase 2 — Picking 데이터 레이어 (우선순위: 높음)

```
[ ] PickingLocalDatasource (wms_pick_task_h/d CRUD)
[ ] PickingRemoteDatasource (GET /api/picking/tasks)
[ ] PickingRepositoryImpl (Local-First + Outbox)
[ ] PickingNotifier Outbox 연동 (updatePickedQty, completePicking)
[ ] Pull Sync — picking tasks delta download
```

### Phase 3 — 나머지 기능 데이터 레이어 (우선순위: 중간)

```
[ ] 신규 DB 테이블 추가 (app_database.dart 마이그레이션)
[ ] InspectionRepository + Datasources
[ ] PutawayRepository + Datasources
[ ] LocationMoveRepository + Datasources
[ ] 각 Provider Outbox 연동
```

### Phase 4 — 마스터 데이터 Pull (우선순위: 중간)

```
[ ] ProductInfo LocalDatasource (wms_if_item 검색)
[ ] ProductInfo RemoteDatasource (서버 바코드 조회)
[ ] 마스터 데이터 Pull Sync (상품, 로케이션)
[ ] HomeScreen 통계 실데이터 연동
```

### Phase 5 — 고도화 (우선순위: 낮음)

```
[ ] 백그라운드 동기화 (workmanager 주석 해제 후 구현)
[ ] FAILED 항목 사용자 알림
[ ] 충돌 상세 로그 UI
[ ] 암호화 키 → Android Keystore / iOS Keychain 이관
[ ] SSL 인증서 피닝 (Dio interceptor)
```

---

## 9. 보안 고려사항

| 항목 | 현황 | 권고사항 |
|------|------|---------|
| 암호화 키 | FieldEncryptor에 하드코딩 | Android Keystore / iOS Keychain으로 이관 |
| 비밀번호 저장 | SHA-256 해싱 (적절) | 유지 |
| 개인정보 | AES-256 암호화 (적절) | 키 관리만 개선 |
| API 인증 | 미구현 (authCd/authPwd 직접 전송) | JWT Token + Refresh Token으로 전환 |
| 네트워크 | HTTPS 가정, 인증서 검증 없음 | 프로덕션 전 SSL Pinning 추가 |
| Outbox 데이터 | 평문 JSON | 민감 데이터 포함 시 암호화 검토 |

---

## 10. API 명세 요구사항 (서버 팀 공유용)

### 필요한 API 엔드포인트

```
[인증]
POST /api/auth/login
GET  /api/auth/users

[마스터 데이터 - Delta Pull]
GET  /api/master/items?since={datetime}&site_cd={code}
GET  /api/master/locations?since={datetime}&site_cd={code}

[피킹]
GET  /api/picking/tasks?since={datetime}&emp_no={no}
PUT  /api/picking/{pick_no}/lines/{seq}
POST /api/picking/{pick_no}/complete

[입고검수]
GET  /api/inspection/tasks?since={datetime}&emp_no={no}
PUT  /api/inspection/{insp_no}/items/{seq}
POST /api/inspection/{insp_no}/complete

[입고적재]
GET  /api/putaway/tasks?since={datetime}&emp_no={no}
POST /api/putaway/{putaway_no}/items/{seq}/complete

[로케이션이동]
GET  /api/location-move/tasks?since={datetime}&emp_no={no}
POST /api/location-move/{move_no}/items/{seq}/complete

[다국어]
GET  /api/i18n/messages?targetType=PDA&include_common=true

[동기화 상태 확인]
GET  /api/sync/status?site_cd={code}
```

### Delta Pull 응답 규약

```json
{
  "updated": [...],
  "deleted": ["id1", "id2"],
  "syncedAt": "2026-05-07T09:00:00Z"
}
```

---

## 정리

```
현재 상태: Auth + i18n만 Offline 지원, 나머지는 Mock 데이터

목표 상태: 모든 현장 작업 기능의 완전한 Offline-First 지원

핵심 전략:
  Local-First → Outbox Push → Delta Pull → Conflict Resolution

구현 우선순위:
  1. SyncEngine 완성 (Outbox Flush + API 실호출)
  2. 네트워크 상태 연동
  3. Picking 데이터 레이어 (가장 핵심 업무)
  4. 나머지 기능 순차 구현
  5. 마스터 데이터 Pull Sync
```
