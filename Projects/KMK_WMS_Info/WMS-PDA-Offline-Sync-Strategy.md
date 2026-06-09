# WMS PDA — Offline 전략 & Sync 구조 설계

#kmarket #wms #pda #flutter #offline #sync #architecture

---

## 문서 개요

| 항목        | 내용                                       |
| --------- | ---------------------------------------- |
| **대상**    | K-Market WMS PDA App (Flutter)           |
| **목적**    | Offline-First 아키텍처 설계 및 서버 동기화 전략 수립    |
| **작성 기준** | 실제 코드베이스 직접 분석 (`wms_pda_app`, 2026-05-20 기준) |
| **최초 작성** | 2026-05-07                               |
| **최종 갱신** | 2026-05-20                               |

---

## 1. 현황 분석 (As-Is)

### 1-1. 기술 스택 (확정)

| 레이어     | 라이브러리             | 버전              | 용도                |
| ------- | ----------------- | --------------- | ----------------- |
| 상태관리    | flutter_riverpod  | ^2.5.1          | StateNotifier 패턴  |
| 라우팅     | go_router         | ^14.2.0         | 선언형 네비게이션         |
| HTTP    | dio               | ^5.7.0          | REST API 통신       |
| 로컬 DB   | sqflite           | ^2.3.3+1        | SQLite (모바일)      |
| 보조 DB   | sembast           | ^3.7.1          | NoSQL (Web 플랫폼용)  |
| 네트워크 감지 | connectivity_plus | ^6.0.5          | 온/오프라인 상태 감지      |
| 암호화     | crypto + encrypt  | ^3.0.6 / ^5.0.3 | SHA-256 / AES-256 |
| ID 생성   | uuid              | ^4.5.1          | Outbox ID 등       |

### 1-2. 현재 SQLite 스키마 (app_database.dart 실제 코드 기준)

```sql
-- 사용자 캐시
wms_user (emp_no PK, emp_nm, auth_cd, auth_pwd, cntr_cd, menu_level, use_yn, synced_at, emp_addr, tel_no, cel_no, e_mail)

-- 로그인 로그
wms_login_log (log_id PK, emp_no, auth_cd, login_type_cd, result_cd, fail_reason, login_dtm)

-- 상품 마스터 (ERP 인터페이스)
wms_if_item (site_cd+goods_cd+barcode_no PK, goods_unit, unit_inqty, goods_nm, goods_spec, maker_cd, storage_type_cd, safe_stock_qty, use_yn, erp_sync_dtm)

-- 로케이션 마스터 (ERP 인터페이스)
wms_if_location (site_cd+loc_cd PK, loc_nm, zone_cd, block_cd, stair_cd, cell_cd, loc_level_cd, load_loc_cd, goods_save_cd, pick_priority, use_yn)

-- 피킹 헤더
wms_pick_task_h (site_cd+pick_no PK, order_no, sale_no, delivery_ymd, cust_cd, pick_status_cd, delivery_type_cd, progress_rate, worker_emp_no)

-- 피킹 상세
wms_pick_task_d (site_cd+pick_no+pick_seq PK, goods_cd, barcode_no, pick_loc_cd, pick_path_seq, alloc_qty, picked_qty, short_qty, line_status_cd, exception_cd)

-- 피킹 스캔 로그
wms_pick_scan_log (scan_log_id PK, site_cd, pick_no, pick_seq, scan_type_cd, scan_value, scan_result_cd, device_id, worker_emp_no, scan_dtm)

-- Outbox 동기화 큐
wms_sync_outbox (outbox_id PK, site_cd, device_id, event_type_cd, ref_table_nm, ref_key_json, payload_json, sync_status_cd, retry_cnt, created_dtm)

-- 동기화 로그
wms_sync_log (sync_log_id PK, site_cd, device_id, event_type_cd, request_json, response_json, result_cd, error_msg, logged_dtm)

-- 다국어 메시지
wms_i18n (target_type+msg_key PK, voca_ko, voca_en, voca_vi, use_yn, reg_dt, upd_dt, synced_at)

-- 앱 설정
wms_app_setting (setting_key PK, setting_value, updated_at)
```

> **현재 DB 버전: 1** — 아래 신규 테이블은 아직 추가되지 않음

### 1-3. 이미 구현된 Offline 지원

```
✅ Auth (인증)
   ├── 로컬 사용자 캐시 (wms_user) — API 동기화 전 seed 데이터 사전 삽입
   ├── SHA-256 비밀번호 해싱 (PasswordHasher)
   ├── AES-256 개인정보 암호화 (FieldEncryptor) — 주소, 전화번호, 이메일
   └── 오프라인 폴백 로그인 — DioException → AuthLocalDatasource 분기

✅ i18n (다국어)
   ├── 앱 번들 JSON 폴백 (assets/i18n/{ko,en,vi}.json) — 137개 키
   ├── 로컬 DB 캐시 (wms_i18n)
   ├── 서버 동기화 후 오버레이 (I18nRemoteDatasource)
   └── 로케일 설정 영속화 (wms_app_setting)

✅ DB 인프라 (골격 완성)
   ├── wms_sync_outbox — 스키마 완성 (outbox_id, event_type_cd, payload_json, sync_status_cd, retry_cnt)
   ├── OutboxDao — insertOutbox(), findPendingOutboxes(), updateSyncStatus()
   └── SyncModels — PICK_QTY_UPDATED, PICK_COMPLETED 이벤트 상수 정의

✅ Feature 도메인 모델 (골격)
   ├── Picking — PickingTaskModel, PickingTaskHeader, PickingItemModel (presentation/provider)
   ├── StockTaking — StockTakingInstruction, StockTakingLine, StockTakingLocation (domain/model)
   ├── ReceivingInspection — InspectionContainer, InspectionItem (domain/model)
   ├── ReceivingPutaway — PutawayItem (domain/model)
   └── LocationMove — LocationMoveItem (domain/model)
```

### 1-4. 미구현 부분 — Gap 분석 (2026-05-20 기준)

```
❌ SyncQueueService.flushPending() — 실제 API 호출 없음
   현재 코드: Future.delayed(300ms) → 무조건 DONE 처리 (가짜 플러시)

❌ SyncEngine — 존재하지 않음
   pushPending() / pullAll() / sync() 설계됨, 파일 없음

❌ NetworkState — connectivity_plus import만 됨, Provider 미연결

❌ SyncModels — 2개 이벤트만 정의 (나머지 6개 미정의)
   현재: PICK_QTY_UPDATED, PICK_COMPLETED
   필요: INSPECT_*, PUTAWAY_*, LOCATION_MOVE_*, STOCK_TAKE_*

❌ Picking 데이터 레이어
   lib/features/picking/data/ 디렉토리는 생성됨
   → repository/, datasource/ 비어 있음, Mock 데이터만 사용 중

❌ ProductInfo 데이터 레이어
   lib/features/product_info/data/ 디렉토리는 생성됨
   → datasource/, repository/ 비어 있음

❌ StockTaking 데이터 레이어
   도메인 모델은 완성, data/ 레이어 없음, Provider 없음 (LocalState 사용)

❌ ReceivingInspection / Putaway / LocationMove 데이터 레이어
   Provider 없음, Repository 없음, Datasource 없음

❌ DB 신규 테이블 미추가
   wms_insp_task_h/d, wms_putaway_task/d, wms_loc_move_task/d, wms_stock_take_h/d

❌ Delta Pull — lastSyncedAt 추적 없음 (wms_app_setting에 키 없음)

❌ 충돌 해결 로직 없음

❌ 암호화 키 하드코딩 (FieldEncryptor) — Keystore 미이관
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
│     앱 기동 시 / 온라인 복귀 시 서버에서 최신 데이터 Delta Pull         │
│                                                                  │
│  5. SERVER-WINS (마스터 데이터)                                    │
│     상품(wms_if_item), 로케이션(wms_if_location)은 서버 데이터 우선   │
│                                                                  │
│  6. LAST-WRITE-WINS (현장 작업 데이터)                             │
│     피킹/검수/적재/재고조사 수량은 최신 타임스탬프 우선                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 2-2. 기능별 Offline 전략

#### 상품 마스터 / 로케이션 (Master Data)

| 항목 | 전략 |
|------|------|
| **방향** | Pull-Only (서버 → 로컬) |
| **트리거** | 로그인 직후 1회 + 수동 동기화 |
| **충돌** | Server-Wins (항상 덮어쓰기) |
| **로컬 테이블** | `wms_if_item`, `wms_if_location` (스키마 완성) |
| **오프라인 동작** | 마지막 동기화된 마스터로 바코드 검색 / 로케이션 조회 |

#### 피킹 (Picking)

| 항목 | 전략 |
|------|------|
| **Pull** | 담당자 배정 피킹리스트 → wms_pick_task_h/d 로컬 저장 |
| **Push** | 수량 변경(PICK_QTY_UPDATED), 완료(PICK_COMPLETED) → Outbox 큐 |
| **스캔 로그** | wms_pick_scan_log에 로컬 기록 → 별도 Push 이벤트 |
| **충돌** | 배정 수량(alloc_qty) Server-Wins / 실피킹(picked_qty) Last-Write-Wins |
| **오프라인 동작** | 로컬 저장된 태스크로 완전한 피킹 작업 가능 |

#### 재고조사 (Stock Taking)

| 항목 | 전략 |
|------|------|
| **Pull** | 조사 지시(StockTakingInstruction) 목록 → wms_stock_take_h 저장 |
| **Push** | 로케이션별 실사 수량 입력 → STOCK_TAKE_QTY_SUBMITTED Outbox |
| **충돌** | Last-Write-Wins (현장 실사 결과 우선) |
| **오프라인 동작** | 조사 지시 사전 다운로드 → 완전 오프라인 작업 가능 |
| **도메인 모델** | StockTakingInstruction / StockTakingLine / StockTakingLocation 완성 |

#### 입고검수 (Receiving Inspection)

| 항목 | 전략 |
|------|------|
| **Pull** | 검수 대상 컨테이너/상품 리스트 → wms_insp_task_h/d 저장 |
| **Push** | 정상/불량 수량 입력 → INSPECT_QTY_UPDATED, INSPECT_COMPLETED |
| **충돌** | Last-Write-Wins (현장 검수 결과 우선) |
| **오프라인 동작** | 전체 검수 작업 오프라인 가능, 온라인 복귀 시 일괄 전송 |

#### 입고적재 (Receiving Putaway)

| 항목 | 전략 |
|------|------|
| **Pull** | 적재 대기 상품 리스트 → wms_putaway_task/d 저장 |
| **Push** | 로케이션 배정 완료 → PUTAWAY_COMPLETED Outbox |
| **충돌** | Last-Write-Wins |
| **오프라인 동작** | 로컬 저장 상품 기준 스캔 작업 가능 |

#### 로케이션이동 (Location Move)

| 항목 | 전략 |
|------|------|
| **Pull** | 이동 지시 리스트 → wms_loc_move_task/d 저장 |
| **Push** | 이동 완료 (FROM→TO) → LOCATION_MOVE_DONE Outbox |
| **충돌** | Last-Write-Wins |
| **오프라인 동작** | 지시된 이동 완전 오프라인 처리 가능 |

#### 상품정보 조회 (Product Info)

| 항목 | 전략 |
|------|------|
| **방향** | 주로 Pull (Read-Only) |
| **오프라인** | wms_if_item에서 barcode_no 검색 |
| **온라인** | 서버 실시간 조회 (재고수량, 최신 입출고 이력) |
| **Fallback** | 서버 응답 없으면 로컬 캐시 결과 + 오프라인 안내 메시지 |

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
│                     │  │  Outbox   │    │   Delta Pull     │   │  │
│                     │  │  (Push)   │    │   (Pull Sync)    │   │  │
│                     │  └───────────┘    └──────────────────┘   │  │
│                     └─────────────────────┬─────────────────────┘  │
│                                           │                         │
│                     ┌─────────────────────▼──────────────────────┐ │
│                     │           NetworkState (Provider)           │ │
│                     │     connectivity_plus → StreamProvider      │ │
│                     └────────────────────────────────────────────┘ │
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
[2] 로컬 DB 저장 (예: wms_pick_task_d.picked_qty)
        │
        ▼
[3] Outbox 큐에 적재 (wms_sync_outbox)
    {
      outbox_id: uuid(),
      site_cd: 'HAN01',
      event_type_cd: 'PICK_QTY_UPDATED',
      ref_table_nm: 'wms_pick_task_d',
      ref_key_json: '{"pick_no":"PL-001","pick_seq":1}',
      payload_json: '{"picked_qty":5,"local_upd_dt":"2026-05-20T10:00:00"}',
      sync_status_cd: 'PENDING',
      retry_cnt: 0,
      created_dtm: now
    }
        │
        ├─── 온라인? ──► [4] SyncEngine.pushPending() 즉시 호출
        │                         │
        │                    성공 ──► sync_status_cd: 'DONE'
        │                    실패 ──► retry_cnt++, 큐 유지
        │
        └─── 오프라인? ─► 큐 유지, 온라인 복귀 시 자동 flush
```

#### Outbox 이벤트 타입 (SyncModels 확장 계획)

| 이벤트 상수 | 설명 | API 엔드포인트 |
|------------|------|---------------|
| `PICK_QTY_UPDATED` | 피킹 수량 변경 | PUT /api/picking/{pick_no}/lines/{seq} |
| `PICK_COMPLETED` | 피킹 완료 | POST /api/picking/{pick_no}/complete |
| `INSPECT_QTY_UPDATED` | 검수 수량 입력 | PUT /api/inspection/{insp_no}/items/{seq} |
| `INSPECT_COMPLETED` | 검수 완료 | POST /api/inspection/{insp_no}/complete |
| `PUTAWAY_COMPLETED` | 적재 완료 | POST /api/putaway/{putaway_no}/items/{seq}/complete |
| `LOCATION_MOVE_DONE` | 로케이션이동 완료 | POST /api/location-move/{move_no}/items/{seq}/complete |
| `STOCK_TAKE_QTY_SUBMITTED` | 재고조사 실사 수량 제출 | PUT /api/stock-taking/{take_no}/lines/{seq} |
| `STOCK_TAKE_COMPLETED` | 재고조사 완료 | POST /api/stock-taking/{take_no}/complete |

> `SyncModels`에 현재 PICK_QTY_UPDATED, PICK_COMPLETED 2개만 정의됨. 나머지 추가 필요.

#### Retry 전략 (Exponential Backoff)

```dart
// lib/core/sync/sync_engine.dart

int _backoffSeconds(int retryCnt) {
  // 0회(최초): 즉시, 1회: 30초, 2회: 2분, 3회: 10분, 4회+: 30분
  const delays = [0, 30, 120, 600, 1800];
  return delays[retryCnt.clamp(0, delays.length - 1)];
}

const int maxRetry = 5; // 5회 초과 → sync_status_cd: 'FAILED' + 사용자 알림
```

### 3-3. Delta Pull (Pull Sync)

#### 흐름도

```
앱 기동 / 온라인 복귀 / 수동 새로고침
        │
        ▼
[1] wms_app_setting에서 lastSyncedAt 조회
    setting_key: 'last_sync_{feature}'  (예: 'last_sync_picking')
        │
        ▼
[2] GET /api/{feature}/changes?since={lastSyncedAt}&emp_no={empNo}
        │
        ▼
[3] 변경 항목 Upsert (로컬 DB) + 삭제 항목 처리
        │
        ▼
[4] wms_app_setting의 lastSyncedAt 갱신
        │
        ▼
[5] Provider 상태 갱신 → UI 자동 반영
```

#### wms_app_setting 키 목록

```
setting_key                 | 설명
'last_sync_master'          | 상품/로케이션 마스터 마지막 동기화
'last_sync_picking'         | 피킹 태스크 마지막 동기화
'last_sync_inspection'      | 입고검수 마지막 동기화
'last_sync_putaway'         | 입고적재 마지막 동기화
'last_sync_location_move'   | 로케이션이동 마지막 동기화
'last_sync_stock_taking'    | 재고조사 마지막 동기화
'locale'                    | 현재 언어 설정 (기존)
```

#### Delta Pull API 응답 규약 (서버 합의 필요)

```json
GET /api/picking/tasks?since=2026-05-01T09:00:00Z&emp_no=00029

{
  "updated": [...],         // 신규/변경된 태스크
  "deleted": ["id1", "id2"], // 삭제된 태스크 ID
  "syncedAt": "2026-05-20T10:00:00Z"
}
```

### 3-4. SyncEngine 설계

```dart
// lib/core/sync/sync_engine.dart  ← 신규 생성

class SyncEngine {
  final OutboxDao _outboxDao;
  final DioClient _dio;
  // 각 feature remote datasource 주입

  // ── Push (Outbox Flush) ─────────────────────────────────────────
  Future<SyncResult> pushPending() async {
    final pending = await _outboxDao.findPendingOutboxes();

    for (final entry in pending) {
      final retryCnt = entry['retry_cnt'] as int;
      if (!_isRetryReady(retryCnt, entry['created_dtm'] as String)) continue;

      try {
        await _dispatchToApi(entry);
        await _outboxDao.updateSyncStatus(
          outboxId: entry['outbox_id'] as String,
          syncStatusCd: 'DONE',
          retryCnt: retryCnt,
        );
      } on NetworkException {
        break; // 네트워크 없음 → 전체 중단, 나중에 재시도
      } on AppException {
        final nextRetry = retryCnt + 1;
        await _outboxDao.updateSyncStatus(
          outboxId: entry['outbox_id'] as String,
          syncStatusCd: nextRetry >= maxRetry ? 'FAILED' : 'PENDING',
          retryCnt: nextRetry,
        );
      }
    }
    return SyncResult(/* pending count, failed count */);
  }

  // ── Pull (Delta Download) ───────────────────────────────────────
  Future<void> pullAll(String empNo) async {
    await Future.wait([
      _pullMasterData(),
      _pullPickingTasks(empNo),
      _pullInspectionTasks(empNo),
      _pullPutawayTasks(empNo),
      _pullLocationMoveTasks(empNo),
      _pullStockTakingTasks(empNo),
    ]);
  }

  // ── Full Sync ───────────────────────────────────────────────────
  Future<SyncResult> sync(String empNo) async {
    final push = await pushPending();   // 로컬 변경 먼저 전송
    await pullAll(empNo);               // 서버 최신 데이터 수신
    return push;
  }

  // ── API 디스패치 (이벤트 타입별 라우팅) ────────────────────────
  Future<void> _dispatchToApi(Map<String, dynamic> entry) async {
    final payload = jsonDecode(entry['payload_json'] as String);
    switch (entry['event_type_cd']) {
      case SyncModels.pickQtyUpdated:
        await _pickingRemote.updateLineQty(payload);
      case SyncModels.pickCompleted:
        await _pickingRemote.completeTask(payload);
      case SyncModels.inspectQtyUpdated:
        await _inspectionRemote.updateItemQty(payload);
      case SyncModels.inspectCompleted:
        await _inspectionRemote.completeTask(payload);
      case SyncModels.putawayCompleted:
        await _putawayRemote.completeItem(payload);
      case SyncModels.locationMoveDone:
        await _locationMoveRemote.completeItem(payload);
      case SyncModels.stockTakeQtySubmitted:
        await _stockTakingRemote.submitQty(payload);
      case SyncModels.stockTakeCompleted:
        await _stockTakingRemote.completeTask(payload);
    }
  }

  bool _isRetryReady(int retryCnt, String createdDtm) {
    if (retryCnt == 0) return true;
    final created = DateTime.parse(createdDtm);
    final elapsed = DateTime.now().difference(created).inSeconds;
    return elapsed >= _backoffSeconds(retryCnt);
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
│ 재고조사 수량  │ Last-Write-Wins │ 작업자 현장 실사값    │
│ 적재 로케이션  │ Last-Write-Wins │ 작업자 현장 결정값    │
│ 이동 완료 상태 │ Last-Write-Wins │ 작업자 현장 결정값    │
│ 앱 설정        │ Local-Wins      │ 기기별 개인 설정      │
└────────────────┴─────────────────┴────────────────────┘
```

---

## 4. DB 스키마 확장 계획

### 4-1. 기존 테이블 — 상태 정리

| 테이블 | 목적 | 스키마 | 로직 |
|--------|------|--------|------|
| `wms_user` | 사용자 캐시 | ✅ | ✅ 완성 |
| `wms_login_log` | 로그인 이력 | ✅ | ✅ 완성 |
| `wms_i18n` | 다국어 메시지 | ✅ | ✅ 완성 |
| `wms_app_setting` | 앱 설정/lastSyncedAt | ✅ | ⚠️ lastSyncedAt 키 미삽입 |
| `wms_sync_outbox` | Outbox 큐 | ✅ | ❌ flushPending() 가짜 구현 |
| `wms_sync_log` | 동기화 이력 | ✅ | ❌ 미사용 |
| `wms_if_item` | 상품 마스터 | ✅ | ❌ Pull 로직 미구현 |
| `wms_if_location` | 로케이션 마스터 | ✅ | ❌ Pull 로직 미구현 |
| `wms_pick_task_h` | 피킹 헤더 | ✅ | ❌ 데이터 연결 미구현 |
| `wms_pick_task_d` | 피킹 상세 | ✅ | ❌ 데이터 연결 미구현 |
| `wms_pick_scan_log` | 피킹 스캔 로그 | ✅ | ❌ 미사용 |

### 4-2. 신규 테이블 (DB 버전 2로 마이그레이션)

```sql
-- ── 재고조사 ──────────────────────────────────────────────────────

CREATE TABLE IF NOT EXISTS wms_stock_take_h (
  take_no       TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  take_nm       TEXT,                         -- 조사 지시명
  status_cd     TEXT NOT NULL DEFAULT 'NEW',  -- NEW / IN_PROGRESS / COMPLETED
  target_dt     TEXT,                         -- 실사 기준 일자
  worker_emp_no TEXT,
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS wms_stock_take_d (
  take_no       TEXT NOT NULL,
  take_seq      INTEGER NOT NULL,
  loc_cd        TEXT NOT NULL,               -- 대상 로케이션
  goods_cd      TEXT NOT NULL,
  barcode_no    TEXT,
  system_qty    REAL DEFAULT 0,              -- 시스템 재고 (Pull 시 서버값)
  actual_qty    REAL,                        -- 실사 수량 (작업자 입력)
  variance_qty  REAL,                        -- 차이 (actual - system)
  status_cd     TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING / COUNTED
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  local_upd_dt  TEXT,
  PRIMARY KEY (take_no, take_seq)
);

-- ── 입고검수 ──────────────────────────────────────────────────────

CREATE TABLE IF NOT EXISTS wms_insp_task_h (
  insp_no       TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  cntr_cd       TEXT,                        -- 컨테이너/입고 번호
  status_cd     TEXT NOT NULL DEFAULT 'NEW',
  insp_dt       TEXT,                        -- 검수 일자
  worker_emp_no TEXT,
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS wms_insp_task_d (
  insp_no       TEXT NOT NULL,
  insp_seq      INTEGER NOT NULL,
  goods_cd      TEXT NOT NULL,
  barcode_no    TEXT,
  order_qty     REAL DEFAULT 0,
  normal_qty    REAL DEFAULT 0,
  defective_qty REAL DEFAULT 0,
  status_cd     TEXT NOT NULL DEFAULT 'PENDING',
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  local_upd_dt  TEXT,
  PRIMARY KEY (insp_no, insp_seq)
);

-- ── 입고적재 ──────────────────────────────────────────────────────

CREATE TABLE IF NOT EXISTS wms_putaway_task (
  putaway_no    TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  insp_no       TEXT,                        -- 연계 검수 번호
  status_cd     TEXT NOT NULL DEFAULT 'NEW',
  worker_emp_no TEXT,
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS wms_putaway_task_d (
  putaway_no    TEXT NOT NULL,
  putaway_seq   INTEGER NOT NULL,
  goods_cd      TEXT NOT NULL,
  barcode_no    TEXT,
  quantity      REAL DEFAULT 0,
  staging_loc   TEXT,                        -- 임시 보관 로케이션
  target_loc    TEXT,                        -- 적재 목표 로케이션
  status_cd     TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING / COMPLETED
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  local_upd_dt  TEXT,
  PRIMARY KEY (putaway_no, putaway_seq)
);

-- ── 로케이션이동 ───────────────────────────────────────────────────

CREATE TABLE IF NOT EXISTS wms_loc_move_task (
  move_no       TEXT PRIMARY KEY,
  site_cd       TEXT NOT NULL,
  status_cd     TEXT NOT NULL DEFAULT 'NEW',
  worker_emp_no TEXT,
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  server_upd_dt TEXT,
  local_upd_dt  TEXT,
  created_dtm   TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS wms_loc_move_task_d (
  move_no       TEXT NOT NULL,
  move_seq      INTEGER NOT NULL,
  goods_cd      TEXT NOT NULL,
  barcode_no    TEXT,
  quantity      REAL DEFAULT 0,
  from_loc      TEXT NOT NULL,
  to_loc        TEXT,
  status_cd     TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING / COMPLETED
  sync_status   TEXT NOT NULL DEFAULT 'SYNCED',
  local_upd_dt  TEXT,
  PRIMARY KEY (move_no, move_seq)
);
```

### 4-3. DB 버전 마이그레이션 전략

```dart
// app_database.dart에서 version: 1 → 2로 변경

return openDatabase(
  path,
  version: 2,                // 1 → 2
  onCreate: (db, version) async {
    await _createTables(db);
    await _seedUsers(db);
  },
  onUpgrade: (db, oldVersion, newVersion) async {
    if (oldVersion < 2) {
      await _addNewFeatureTables(db);  // 신규 테이블만 추가
    }
  },
  onOpen: (db) async {
    await _createTables(db);           // IF NOT EXISTS 보장
  },
);
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

### 현재 상태 → 목표 상태

```
lib/
├── core/
│   ├── database/
│   │   ├── app_database.dart         ✅ 기존 (버전 2 마이그레이션 추가)
│   │   └── outbox_dao.dart           ✅ 기존 (완성)
│   ├── sync/
│   │   ├── sync_engine.dart          ❌ 신규 생성 (핵심)
│   │   ├── sync_queue_service.dart   ✅ 기존 → SyncEngine으로 대체
│   │   ├── sync_models.dart          ✅ 기존 → 이벤트 상수 6개 추가
│   │   └── sync_state.dart           ❌ 신규 생성 (SyncResult 모델)
│   ├── network/
│   │   ├── dio_client.dart           ✅ 기존
│   │   └── network_state.dart        ❌ 신규 생성 (connectivity_plus 연결)
│   ├── error/                        ✅ 기존 유지
│   └── security/                     ✅ 기존 유지
│
├── features/
│   ├── picking/
│   │   └── data/
│   │       ├── repository/
│   │       │   └── picking_repository_impl.dart   ❌ 신규
│   │       └── datasource/
│   │           ├── picking_local_datasource.dart  ❌ 신규
│   │           └── picking_remote_datasource.dart ❌ 신규
│   │
│   ├── stock_taking/
│   │   └── data/
│   │       ├── repository/
│   │       │   └── stock_taking_repository_impl.dart ❌ 신규
│   │       └── datasource/
│   │           ├── stock_taking_local_datasource.dart  ❌ 신규
│   │           └── stock_taking_remote_datasource.dart ❌ 신규
│   │
│   ├── receiving_inspection/
│   │   └── data/
│   │       ├── repository/inspection_repository_impl.dart  ❌ 신규
│   │       └── datasource/{local,remote}_datasource.dart   ❌ 신규
│   │
│   ├── receiving_putaway/
│   │   └── data/
│   │       ├── repository/putaway_repository_impl.dart     ❌ 신규
│   │       └── datasource/{local,remote}_datasource.dart   ❌ 신규
│   │
│   ├── location_move/
│   │   └── data/
│   │       ├── repository/location_move_repository_impl.dart ❌ 신규
│   │       └── datasource/{local,remote}_datasource.dart     ❌ 신규
│   │
│   └── product_info/
│       └── data/
│           ├── repository/product_info_repository_impl.dart ❌ 신규
│           └── datasource/
│               ├── product_info_local_datasource.dart  ❌ 신규
│               └── product_info_remote_datasource.dart ❌ 신규
```

---

## 6. 네트워크 상태 관리

### 6-1. NetworkState Provider (신규)

```dart
// lib/core/network/network_state.dart

final networkStateProvider = StreamProvider<bool>((ref) {
  return Connectivity()
    .onConnectivityChanged
    .map((results) => results.any((r) => r != ConnectivityResult.none));
});
```

### 6-2. HomeProvider 연동

```dart
// home_provider.dart에서 네트워크 상태 구독

@override
void build() {
  ref.listen(networkStateProvider, (_, next) {
    next.whenData((isOnline) {
      state = state.copyWith(isOnline: isOnline);
      if (isOnline) {
        // 온라인 복귀 → 자동 동기화 트리거
        final empNo = ref.read(authSessionProvider)?.authCd ?? '';
        ref.read(syncEngineProvider).sync(empNo);
      }
    });
  });
}
```

### 6-3. 동기화 트리거 타이밍

| 트리거 | Pull | Push |
|--------|------|------|
| 앱 기동 | ✅ | ✅ (Pending 있으면) |
| 온라인 복귀 | ✅ | ✅ |
| 수동 새로고침 | ✅ | ✅ |
| 작업 완료 직후 | ❌ | ✅ (해당 이벤트만) |

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

// PickingNotifier — Optimistic Update + Outbox
class PickingNotifier extends StateNotifier<PickingState> {
  final PickingRepository _repo;
  final SyncEngine _syncEngine;

  Future<void> updatePickedQty(int seq, int qty) async {
    // 1. Optimistic UI 즉시 반영
    state = state.copyWith(/* picked_qty 업데이트 */);

    // 2. 로컬 DB 저장 + Outbox 적재
    await _repo.updateLineQtyLocal(
      pickNo: state.task!.header.number,
      seq: seq,
      qty: qty,
    );

    // 3. Push 시도 (실패해도 Outbox에 남아 나중에 재시도)
    await _syncEngine.pushPending();
  }

  Future<void> completePicking() async {
    state = state.copyWith(/* 완료 상태 반영 */);
    await _repo.completeLocal(pickNo: state.task!.header.number);
    await _syncEngine.pushPending();
  }
}
```

### 7-2. StockTaking 패턴 (도메인 모델 활용)

```dart
// StockTakingInstruction / StockTakingLine / StockTakingLocation 도메인 모델 활용

final stockTakingRepositoryProvider = Provider<StockTakingRepository>((ref) {
  return StockTakingRepositoryImpl(
    local: ref.read(stockTakingLocalDatasourceProvider),
    remote: ref.read(stockTakingRemoteDatasourceProvider),
    outbox: ref.read(outboxDaoProvider),
  );
});

class StockTakingNotifier extends StateNotifier<StockTakingState> {
  Future<void> submitActualQty(String takeNo, int seq, double actualQty) async {
    // Optimistic: 로컬 상태 업데이트
    // 로컬 DB: wms_stock_take_d.actual_qty, sync_status: PENDING
    // Outbox: STOCK_TAKE_QTY_SUBMITTED 이벤트
    // Push 시도
  }
}
```

### 7-3. 홈 화면 통계 실데이터 연동

```dart
class HomeNotifier extends StateNotifier<HomeState> {
  Future<void> refresh() async {
    final pickingCount    = await _pickingLocal.getPendingCount();
    final inboundCount    = await _inspectionLocal.getPendingCount();
    final stockTakeCount  = await _stockTakingLocal.getPendingCount();
    final unsyncedCount   = await _outboxDao.getPendingCount();
    state = state.copyWith(
      waitingPickingCount:  pickingCount,
      waitingInboundCount:  inboundCount,
      waitingStockTake:     stockTakeCount,
      unsyncedCount:        unsyncedCount,
    );
  }
}
```

---

## 8. 구현 로드맵

### Phase 1 — Sync 인프라 완성 (우선순위: 최고)

```
[ ] SyncEngine 신규 생성 (lib/core/sync/sync_engine.dart)
    - pushPending() — 실제 API 호출 (현재 가짜 구현 대체)
    - pullAll() — Delta Pull 뼈대
    - sync() — Push + Pull 통합
    - Exponential Backoff 로직

[ ] SyncModels 확장
    - 현재 2개 → 8개 이벤트 상수 추가

[ ] NetworkState Provider 연결
    - lib/core/network/network_state.dart 생성
    - HomeProvider와 연동

[ ] wms_app_setting lastSyncedAt 키 초기화
    - 앱 기동 시 키가 없으면 epoch time으로 seed
```

### Phase 2 — DB 마이그레이션 (우선순위: 높음)

```
[ ] app_database.dart version 1 → 2
[ ] onUpgrade 콜백에 신규 테이블 추가
    - wms_stock_take_h / wms_stock_take_d
    - wms_insp_task_h / wms_insp_task_d
    - wms_putaway_task / wms_putaway_task_d
    - wms_loc_move_task / wms_loc_move_task_d
```

### Phase 3 — Picking 데이터 레이어 (우선순위: 높음)

```
[ ] PickingLocalDatasource (wms_pick_task_h/d CRUD)
[ ] PickingRemoteDatasource (GET /api/picking/tasks, PUT 수량, POST 완료)
[ ] PickingRepositoryImpl (Local-First + Outbox)
[ ] PickingNotifier → Mock 제거, Repository 연동
[ ] Delta Pull — picking tasks
[ ] wms_pick_scan_log 연동 (스캔 이력 저장)
```

### Phase 4 — StockTaking 데이터 레이어 (우선순위: 높음)

```
[ ] StockTakingLocalDatasource (wms_stock_take_h/d CRUD)
[ ] StockTakingRemoteDatasource (GET, PUT, POST)
[ ] StockTakingRepositoryImpl
[ ] StockTakingNotifier 신규 생성 (현재 LocalState → Provider 전환)
[ ] Delta Pull — stock taking tasks
[ ] 기존 StockTakingListScreen / DetailScreen Provider 연동
```

### Phase 5 — 나머지 기능 데이터 레이어 (우선순위: 중간)

```
[ ] InspectionRepository + Datasources + Provider
[ ] PutawayRepository + Datasources + Provider
[ ] LocationMoveRepository + Datasources + Provider
[ ] 각 Provider Outbox 연동
```

### Phase 6 — 마스터 데이터 & ProductInfo (우선순위: 중간)

```
[ ] ProductInfoLocalDatasource (wms_if_item barcode 검색)
[ ] ProductInfoRemoteDatasource (서버 바코드 조회)
[ ] 마스터 데이터 Pull Sync (wms_if_item, wms_if_location)
[ ] HomeScreen 통계 실데이터 연동
```

### Phase 7 — 고도화 (우선순위: 낮음)

```
[ ] FAILED 항목 사용자 알림 (sync_status_banner 연동)
[ ] 백그라운드 동기화 (workmanager — 현재 주석처리됨)
[ ] 충돌 상세 로그 UI
[ ] 암호화 키 → Android Keystore / iOS Keychain 이관
[ ] SSL 인증서 피닝 (Dio interceptor)
[ ] JWT Token + Refresh Token 전환 (현재 authCd/authPwd 직접 전송)
```

---

## 9. 보안 고려사항

| 항목 | 현황 | 권고사항 |
|------|------|---------|
| 암호화 키 | FieldEncryptor 내 하드코딩 | Android Keystore / iOS Keychain 이관 |
| 비밀번호 저장 | SHA-256 해싱 | 유지 |
| 개인정보 | AES-256 암호화 | 키 관리 개선 후 유지 |
| API 인증 | 미구현 (authCd/authPwd 직전송) | JWT Token + Refresh Token |
| 네트워크 | HTTPS 가정, 인증서 검증 없음 | 프로덕션 전 SSL Pinning |
| Outbox 데이터 | 평문 JSON (wms_sync_outbox) | 민감 payload 포함 시 암호화 검토 |
| Seed 계정 | 테스트 계정 9개 앱에 내장 | 프로덕션 빌드 시 제거 필요 |

---

## 10. API 명세 요구사항 (서버 팀 공유용)

### 필요한 API 엔드포인트

```
[인증]
POST /api/auth/login
GET  /api/auth/users

[마스터 데이터 — Delta Pull]
GET  /api/master/items?since={datetime}&site_cd={code}
GET  /api/master/locations?since={datetime}&site_cd={code}

[피킹]
GET  /api/picking/tasks?since={datetime}&emp_no={no}
PUT  /api/picking/{pick_no}/lines/{seq}          body: { picked_qty, local_upd_dt }
POST /api/picking/{pick_no}/complete

[재고조사]
GET  /api/stock-taking/tasks?since={datetime}&emp_no={no}
PUT  /api/stock-taking/{take_no}/lines/{seq}     body: { actual_qty, local_upd_dt }
POST /api/stock-taking/{take_no}/complete

[입고검수]
GET  /api/inspection/tasks?since={datetime}&emp_no={no}
PUT  /api/inspection/{insp_no}/items/{seq}       body: { normal_qty, defective_qty, local_upd_dt }
POST /api/inspection/{insp_no}/complete

[입고적재]
GET  /api/putaway/tasks?since={datetime}&emp_no={no}
POST /api/putaway/{putaway_no}/items/{seq}/complete  body: { target_loc, local_upd_dt }

[로케이션이동]
GET  /api/location-move/tasks?since={datetime}&emp_no={no}
POST /api/location-move/{move_no}/items/{seq}/complete  body: { to_loc, local_upd_dt }

[상품정보 조회]
GET  /api/products/{barcode_no}?site_cd={code}

[다국어]
GET  /api/i18n/messages?targetType=PDA&include_common=true

[동기화 상태]
GET  /api/sync/status?site_cd={code}
```

### Delta Pull 응답 규약

```json
{
  "updated": [...],
  "deleted": ["id1", "id2"],
  "syncedAt": "2026-05-20T10:00:00Z"
}
```

---

## 정리

```
현재 상태 (2026-05-20):
  ✅ Auth + i18n — 완전한 Offline 지원
  ✅ DB 스키마 인프라 — Picking, 마스터 테이블 완성
  ✅ Feature 도메인 모델 — 전 기능 완성
  ✅ Outbox 테이블/DAO — 스키마 완성
  ❌ SyncEngine — 미존재 (SyncQueueService는 가짜 구현)
  ❌ 모든 Business Feature 데이터 레이어 — Mock 상태

목표 상태:
  모든 현장 작업 기능의 완전한 Offline-First 지원

핵심 전략:
  LOCAL-FIRST → Optimistic UI → Outbox Push → Delta Pull → Conflict Resolution

구현 우선순위:
  1. SyncEngine 완성 (Outbox Flush 실 구현, Backoff)
  2. 네트워크 상태 연동 (connectivity_plus)
  3. DB 버전 2 마이그레이션 (신규 테이블)
  4. Picking 데이터 레이어 (가장 핵심 업무)
  5. StockTaking 데이터 레이어 (도메인 모델 완성 활용)
  6. 나머지 기능 순차 구현
```
