# 입고적재(Putaway) 프로세스 구현 정리

## 개요

입고적재는 **검수 완료 후 가로케이션(`WT0000`)에 임시 보관된 재고를 실제 창고 로케이션으로 옮기는 작업**이다.
PDA 앱에서 로케이션 바코드 → 상품 바코드 순서로 스캔하면 서버에서 재고/이력 테이블을 원자적으로 업데이트한다.

---

## 전체 흐름 다이어그램

```
[입고검수 완료]
        │
        ▼
  st_stk: loc_id = 'WT0000', stk_stts_cd = '01'
  st_stk_hist: stk_hist_cd = '03' (검수완료)
        │
        ▼
[PDA 적재 화면 오픈]
        │
        ├── GET /inbound/pda/putaway/pending  → pending items 목록
        └── GET /inbound/pda/location/sync   → 로케이션 마스터
        │
        ▼
[Step 1: 로케이션 바코드 스캔]
        │
        ├── 로컬 lc_master 조회 → isLoadable(load_yn='Y') 검증
        └── 없으면 온라인 시 GET /inbound/pda/location/{locId} 폴백
        │
        ▼
[Step 2: 상품 바코드 or 상품코드 스캔]
        │
        ├── pending items에서 barcode 또는 goodsCd 매칭
        └── keepTypeCd(보관구분) 불일치 시 오류 반환
        │
        ▼
[PutawayConfirmDialog 표시]
        │
        ▼
[Step 3: 적재 실행]
        │
    [온라인]───────────────────────────[오프라인]
        │                                  │
        ▼                                  ▼
POST /inbound/pda/putaway/execute    아웃박스에 저장
        │                            (eventTypeCd='PUTAWAY_ITEM')
        ▼                                  │
  @Transactional                    로컬 pending에서 즉시 제거
  insertStkHistForPutaway()               │
  updateStkLocForPutaway()           [네트워크 복구 시]
  acquireLocMovHistLock()                 │
  insertLocMovHist()                 flushPending() → postPutawayItem()
        │
        ▼
 로컬 pending에서 제거 + UI 갱신
```

---

## Flutter 레이어별 구현

### 1. UI — `receiving_putaway_screen.dart`

**상태 초기화:**
```
initState → Future.microtask → putawayProvider.notifier.load()
```

**3단계 입력 흐름:**

| 단계 | 트리거 | 처리 메서드 | 성공 결과 |
|------|--------|------------|---------|
| 1 | 로케이션 스캔 | `_onLocationSubmit()` → `validateLocation()` | `state.scannedLocation` 설정 |
| 2 | 상품 스캔 | `_onProductSubmit()` | 매칭 아이템 선택 |
| 3 | 확인 다이얼로그 | `_executePutaway()` → `PutawayConfirmDialog` | pending에서 제거 |

**keep type 불일치 검증 (양방향):**
- 로케이션 스캔 시 → 이미 선택된 상품이 있으면 keepTypeCd 비교
- 상품 스캔 시 → 이미 스캔된 로케이션이 있으면 keepTypeCd 비교
- 불일치 시 오류 메시지 반환, 스캔 초기화 없음 (사용자가 재시도 가능)

---

### 2. State — `PutawayState` / `PutawayNotifier`

**파일:** `putaway_provider.dart`

```dart
class PutawayState {
  final bool isLoading;
  final List<PutawayItem> items;       // 적재 대기 목록
  final String? errorMessage;
  final bool isOffline;
  final LocationInfo? scannedLocation; // 현재 스캔된 로케이션
  final bool isValidatingLocation;     // 로케이션 조회 중
}
```

**주요 메서드:**

| 메서드 | 역할 |
|--------|------|
| `load()` | 적재 대기 목록 로드 (온라인: 서버 → 로컬 sync, 오프라인: 로컬 폴백) |
| `validateLocation(locId)` | 로케이션 유효성 검증, `scannedLocation` 업데이트 |
| `executePutawayItem({item, locId})` | 적재 실행, 성공 시 items 목록에서 해당 항목 제거 |
| `clearLocation()` | 스캔된 로케이션 초기화 |

---

### 3. Repository — `PutawayRepository`

**파일:** `putaway_repository.dart`

#### `syncPutawayOnStartup()` — 앱 시작/복구 시 동기화

```dart
// 두 API를 병렬 호출
await Future.wait([
  _remote.fetchPutawayPending(centerId)  → _local.syncPutawayPending(),
  _remote.fetchLocationsForSync(centerId) → _local.syncLocations(),
]);
```

로컬 sync 방식: 해당 `center_id`의 전체 데이터를 DELETE → 서버 응답 전체 INSERT (full replace)

#### `getPutawayItems()` — 목록 조회

```
온라인: 서버 호출 → 로컬 sync → 로컬 반환
오프라인: 로컬 반환
```

#### `validateLocation(locId)` — 로케이션 검증

```
로컬 lc_master 조회 → 있으면 반환
없으면 온라인 시 서버 호출
오프라인이면 null 반환 (적재 불가)
```

#### `executePutawayItem({stkHistId, locId})` — 적재 실행

```
온라인: postPutawayItem() → 로컬 pending 제거
오프라인: 아웃박스 INSERT → 로컬 pending 제거 (낙관적 업데이트)
```

---

### 4. Remote Datasource — `PutawayRemoteDatasource`

**파일:** `putaway_remote_datasource.dart`

| 메서드 | HTTP | 경로 |
|--------|------|------|
| `fetchPutawayPending()` | GET | `/inbound/pda/putaway/pending?centerId=` |
| `fetchLocationsForSync()` | GET | `/inbound/pda/location/sync?centerId=` |
| `fetchLocationInfo()` | GET | `/inbound/pda/location/{locId}?centerId=` |
| `postPutawayItem()` | POST | `/inbound/pda/putaway/execute` |
| `postPutaway()` | POST | `/inbound/pda/{ibNo}/putaway` |

`postPutawayItem` 요청 body:
```json
{
  "stkHistId": "ST000000000000000001",
  "locId": "A01-01-01",
  "centerId": "C001",
  "workerId": "user123"
}
```

---

### 5. Local Datasource — `PutawayLocalDatasource`

**파일:** `putaway_local_datasource.dart`  
**DB 버전:** v13 (app_database.dart)

**`putaway_pending` 테이블:**

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `stk_hist_id` | TEXT PK | 재고이력 ID |
| `center_id` | TEXT | 센터 ID |
| `goods_cd` | TEXT | 상품코드 |
| `barcode` | TEXT | 바코드 |
| `exp` | TEXT | 유통기한 |
| `goods_nm` | TEXT | 상품명 |
| `default_loc_id` | TEXT | 기본 로케이션 |
| `keep_type_cd` | TEXT | 보관구분 코드 |
| `qty` | INTEGER | 수량 |
| `synced_at` | TEXT | 동기화 시각 |

**`lc_master` 테이블:**

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `loc_id` | TEXT | 로케이션 ID (PK 일부) |
| `center_id` | TEXT | 센터 ID (PK 일부) |
| `keep_type_cd` | TEXT | 보관구분 코드 |
| `load_yn` | TEXT | 적재 가능 여부 |
| `use_yn` | TEXT | 사용 여부 |
| `synced_at` | TEXT | 동기화 시각 |

---

## Domain 모델

### `PutawayItem`

```dart
class PutawayItem {
  final String stkHistId;      // 재고이력 ID (= id)
  final String goodsCd;        // 상품코드
  final String? goodsNm;       // 상품명
  final String barcode;        // 바코드
  final int qty;               // 수량
  final String? exp;           // 유통기한
  final String? keepTypeCd;    // 보관구분 ('01'냉장 '02'냉동 '03'상온)
  final String? defaultLocId;  // 기본 로케이션
  final PutawayStatus status;  // pending / completed
  final String? targetLocId;   // 적재 대상 로케이션
}
```

### `LocationInfo`

```dart
class LocationInfo {
  final String locId;
  final String centerCode;
  final String? keepTypeCd;
  final String loadYn;    // 'Y': 적재 가능
  final String useYn;     // 'Y': 사용 중

  bool get isLoadable => loadYn == 'Y';
}
```

---

## 백엔드 레이어

### API 컨트롤러 — `InboundController` (inbound 모듈)

| 엔드포인트 | 동작 |
|-----------|------|
| `GET /inbound/pda/putaway/pending` | 센터 기준 적재 대기 목록 반환 |
| `GET /inbound/pda/location/sync` | 센터 전체 로케이션 마스터 반환 |
| `GET /inbound/pda/location/{locId}` | 단건 로케이션 정보 반환 |
| `POST /inbound/pda/putaway/execute` | 단건 적재 실행 (WT0000 → 실로케이션) |
| `POST /inbound/pda/{ibNo}/putaway` | 배치 적재 실행 (ibNo 기준 일괄) |

### 서비스 — `PutawayServiceImpl` (stock 모듈)

**파일:** `stock/service/impl/PutawayServiceImpl.java`

#### `executePutawayItem(PutawayExecuteDTO dto)` — 단건 적재

```
@Transactional
1. insertStkHistForPutaway(stkHistId, newLocId, today, workerId)
   └── st_stk_hist에서 기존 행 복사, loc_id=newLocId, stk_hist_cd='I' 로 INSERT
2. updateStkLocForPutaway(centerId, goodsCd, barcode, exp, newLocId, today)
   └── st_stk의 loc_id = 'WT0000' → newLocId, stk_stts_cd = 'I'
3. acquireLocMovHistLock(datePrefix)
   └── pg_advisory_xact_lock — lc_mov_hist_id 중복 방지
4. insertLocMovHist(LocMovHistDTO)
   └── lc_mov_hist에 이동이력 삽입 (mov_hist_cd='01')
```

#### `executePutaway(ibNo, centerId, items)` — 배치 적재

```
@Transactional
for each item:
  1. selectCurrentStockQty(centerId, goodsCd, exp, locId)
  2. upsertStock(StockDTO) — st_stk ON CONFLICT DO UPDATE
  3. insertStockHistory(StockHistoryDTO) — stk_hist_cd='L'
```

---

## SQL 상세 — `StockMapper.xml`

### `insertStkHistForPutaway` (INSERT-SELECT 패턴)

```sql
INSERT INTO wms.st_stk_hist (
    stk_hist_id, center_id, goods_cd, barcode, loc_id, exp, stk_hist_cd,
    mod_qty, orgn_stk_qty, mod_stk_qty, ref_no, rmrk, reg_id, reg_dt
)
SELECT
    'ST' || LPAD(nextval('wms.st_stk_hist_id_seq')::text, 18, '0'),
    center_id, goods_cd, barcode, #{newLocId}, exp, 'I',
    mod_qty, orgn_stk_qty, mod_stk_qty, ref_no, '입고로 인한 재고 변경 구분',
    #{workerId}, #{today}
FROM wms.st_stk_hist
WHERE stk_hist_id = #{stkHistId}
```

기존 검수 이력(`stk_hist_id`)을 복사해 새 이력을 만든다. `loc_id`를 실로케이션으로, `stk_hist_cd`를 `'I'`로 교체.

### `updateStkLocForPutaway`

```sql
UPDATE wms.st_stk
SET loc_id      = #{newLocId},
    stk_stts_cd = 'I',
    upd_id      = 'system',
    upd_dt      = #{today}
WHERE center_id = #{centerId}
  AND goods_cd  = #{goodsCd}
  AND barcode   = #{barcode}
  AND exp       = #{exp}
  AND loc_id    = 'WT0000'
```

> **주의:** 대상 로케이션에 이미 동일 (center_id, goods_cd, exp, loc_id) PK가 존재하면 충돌 발생. 현재는 최초 적재 케이스만 처리.

---

## 오프라인 처리 — 아웃박스 패턴

### 아웃박스 저장 (오프라인 시)

```dart
await _outboxDao.insertOutbox(
  outboxId:    _uuid.v4(),
  siteCd:      centerId,
  eventTypeCd: SyncModels.putawayItem,   // 'PUTAWAY_ITEM'
  payloadJson: jsonEncode({
    'stkHistId': stkHistId,
    'locId':     locId,
    'centerId':  centerId,
    'workerId':  workerId,
  }),
  syncStatusCd: 'PENDING',
  retryCnt:     0,
  createdDtm:  DateTime.now().toIso8601String(),
);
await _local.removePutawayPendingItem(stkHistId); // 낙관적 업데이트
```

### 네트워크 복구 시 플러시 (`onNetworkRestored` in `sync_on_reconnect.dart`)

```
SyncQueueService(OutboxDao()).flushPending(...)
  └── eventTypeCd == 'PUTAWAY_ITEM'
      → putawayRemote.postPutawayItem(stkHistId, locId, centerId, workerId)
  └── eventTypeCd == 'PUTAWAY'
      → putawayRemote.postPutaway(ibNo, centerId, items)
```

처리 순서: `created_dtm ASC` (먼저 저장된 작업부터 처리)

---

## 주요 코드값 정리

| 구분 | 코드 | 의미 |
|------|------|------|
| `stk_hist_cd` | `'03'` | 검수완료 |
| `stk_hist_cd` | `'I'` | 입고적재 완료 |
| `stk_hist_cd` | `'L'` | 적재 (배치 경로) |
| `stk_hist_cd` | `'O'` | 출고 |
| `stk_stts_cd` | `'01'` | 초기 (적재 전) |
| `stk_stts_cd` | `'I'` | 적재완료 |
| `stk_stts_cd` | `'O'` | 출고 |
| `loc_id` | `'WT0000'` | 가로케이션 (입고 후 임시 위치) |
| `keepTypeCd` | `'01'` | 냉장 |
| `keepTypeCd` | `'02'` | 냉동 |
| `keepTypeCd` | `'03'` | 상온 |
| `mov_hist_cd` | `'01'` | 입고 이동 |

---

## 관련 파일 위치

### Flutter (PDA App)
```
lib/features/receiving_putaway/
├── data/
│   ├── datasource/
│   │   ├── putaway_remote_datasource.dart   # API 호출
│   │   ├── putaway_local_datasource.dart    # SQLite CRUD
│   │   ├── putaway_local_store.dart         # 인터페이스
│   │   └── web_putaway_local_datasource.dart
│   └── repository/
│       └── putaway_repository.dart          # 온/오프라인 분기
├── domain/
│   └── model/
│       ├── putaway_item.dart
│       └── location_info.dart
└── presentation/
    ├── provider/
    │   └── putaway_provider.dart            # StateNotifier + State
    └── screen/
        └── receiving_putaway_screen.dart    # UI + 스캔 로직
```

### Backend (Spring Boot)
```
stock/
├── service/
│   ├── PutawayService.java
│   └── impl/PutawayServiceImpl.java
├── dto/
│   ├── PutawayExecuteDTO.java
│   ├── PutawayItemDTO.java
│   ├── LocMovHistDTO.java
│   └── StockDTO.java
└── mapper/
    └── StockMapper.java

resources/mapper/stock/
└── StockMapper.xml   # SQL: insertStkHistForPutaway, updateStkLocForPutaway, insertLocMovHist
```
