# 피킹(Picking) 프로세스 구현 정리

## 개요

피킹은 **출고 주문에 대해 창고 로케이션에서 지시된 상품을 꺼내는 작업**이다.
PDA 앱에서 피킹 리스트 바코드 스캔 → 상품 바코드 스캔 → 수량 입력 순서로 진행하며,
모든 품목이 완료되면 피킹 완료 버튼으로 일괄 확정한다.

---

## 전체 흐름 다이어그램

```
[출고 주문 생성 → 피킹 리스트 생성]
         │
         ▼
  ob_pick_hdr: pick_stts_cd = '01' (대기)
  ob_pick_dtl: cmpl_yn = 'N', pick_qty = 0
         │
         ▼
[PDA 피킹 화면 오픈]
    ─ task == null 상태: 리스트 스캔 섹션 표시 ─
         │
         ▼
[Step 1: 피킹 리스트 바코드 스캔]
         │
         ├── 로컬 ob_pick_hdr 조회 (getPickingList)
         └── 없으면 온라인 시 GET /outbound/pda/picking/{pickListNo} 폴백
         │
         ▼
    ─ task != null 상태: 상품 스캔 섹션 + 아이템 목록 표시 ─
         │
         ▼
[Step 2: 상품 바코드 스캔]
         │
         ├── items에서 barcode 매칭
         ├── 이미 완료(isCompleted)이면 오류
         └── activeSeq 설정 → 해당 카드로 자동 스크롤
         │
         ▼
[Step 3: 수량 입력 (PickingItemCard의 수량 입력 위젯)]
         │
         ▼
    updatePickedQty(seq, qty)
         │
         ├── 상태 즉시 업데이트 (낙관적)
         └── executePickingItem() 비동기 호출
               │
           [온라인]──────────────────────[오프라인]
               │                              │
               ▼                              ▼
    POST /outbound/pda/picking/item    아웃박스에 저장
               │                       (eventTypeCd='PICKING_ITEM')
               ▼                              │
    @Transactional                     로컬 pick_qty/cmpl_yn 업데이트
    selectPickingDetailForUpdate (FOR UPDATE)
    updatePickingDetailPicked
    updatePickingHeaderInProgress
    updateOrderDetailPickingStatusByPickItem
    updateOrderHeaderObStatusDerived
         │
         ▼
[모든 items.isCompleted → task.allCompleted == true]
         │
         ▼
[피킹 완료 버튼 활성화 → AlertDialog 확인]
         │
         ▼
    completePicking()
         │
         ▼
    POST /outbound/pda/picking/{pickListNo}/complete
         │
         ▼
    @Transactional
    countIncompletePdaPickingDetails → 0이면 진행
    updatePickingHeaderCompleted     (pick_stts_cd='03', cmpl_yn='Y')
    selectOutboundStockHistoryCode   → stk_hist_cd 'O' 코드 조회
    insertStockHistPickingComplete   (st_stk_hist: stk_hist_cd='O')
    updateOrderDetailPickingStatusByPickList ('03')
    updateOrderHeaderObItemQtyAndOutboundWaiting
    updateStockStatusPickingComplete (st_stk: stk_stts_cd='O')
         │
         ▼
    state = PickingState.initial() (화면 초기화)
```

---

## Flutter 레이어별 구현

### 1. UI — `picking_screen.dart`

**화면 전환 구조:**

| `state.task` | 표시 내용 |
|---|---|
| `null` | `PickingScanSection` (리스트 바코드 입력) + `WmsEmptyState` |
| not null | `PickingScanSection` (상품 바코드 입력) + `PickingHeaderCard` + `PickingProgressCard` + `ListView(PickingItemCard)` + `PickingCompleteBar` |

**`ref.listen` 사이드 이펙트:**
- `errorMessage` 변경 → SnackBar 표시
- `infoMessage` 변경 → SnackBar 표시
- `activeSeq` 변경 → `Scrollable.ensureVisible()` 로 해당 아이템 카드 자동 스크롤 (alignment: 0.3)

**수량 입력 콜백 (fire-and-forget):**
```dart
onChangedQty: (qty) {
  notifier.updatePickedQty(item.seq, qty);  // await 없음
},
```
> `updatePickedQty`는 async이지만 화면에서 await하지 않는다. 상태는 먼저 업데이트되고 API는 뒤에서 실행된다.

**피킹 완료 버튼 조건:**
```dart
enabled: task.allCompleted && !state.isCompleting
```

**새로고침:**
```dart
onRefresh: () {
  notifier.reset();
  _listController.clear();
  _productController.clear();
}
```

---

### 2. State — `PickingState` / `PickingNotifier`

**파일:** `picking_state.dart`, `picking_provider.dart`

```dart
class PickingState {
  final bool isLoadingList;       // 리스트 조회 중
  final bool isCompleting;        // 완료 처리 중
  final String listScanInput;     // 리스트 바코드 입력값
  final String productScanInput;  // 상품 바코드 입력값
  final PickingList? task;        // 현재 작업 중인 피킹 리스트
  final int? activeSeq;           // 현재 활성 아이템 seq
  final bool isHeaderCollapsed;   // 헤더 카드 접힘 여부
  final String? errorMessage;
  final String? infoMessage;
}
```

**`copyWith` 특수 파라미터:**
- `clearError: true` → errorMessage = null
- `clearInfo: true` → infoMessage = null
- `clearTask: true` → task = null
- `keepActiveItem: false` → activeSeq = 넘긴 값 그대로 (기본 true면 기존값 유지)

**주요 메서드:**

| 메서드 | 역할 |
|--------|------|
| `updateListScanInput(value)` | 리스트 스캔 입력값 저장 |
| `updateProductScanInput(value)` | 상품 스캔 입력값 저장 |
| `loadPickingList()` | 로컬/서버에서 피킹 리스트 조회, task 설정 |
| `scanProduct()` | barcode로 items 매칭, activeSeq 설정 |
| `updatePickedQty(seq, qty)` | 수량 낙관적 업데이트 + API 비동기 호출 |
| `toggleHeader()` | 헤더 카드 접기/펼치기 |
| `reset()` | 전체 초기화 (PickingState.initial()) |
| `completePicking()` | 피킹 완료 확정 |

**`updatePickedQty` 내부 로직:**
```
1. qty 범위 제한: 0 ≤ qty ≤ ordrQty (clamping)
2. items 목록에서 해당 seq를 찾아 pickQty 교체
3. isCompleted(pickQty >= ordrQty) 여부에 따라 activeSeq 결정
   - 완료: activeSeq = null (포커스 해제)
   - 미완: activeSeq = 해당 seq 유지
4. 상태 즉시 업데이트
5. executePickingItem() await (but 호출자는 await 안 함)
```

---

### 3. Repository — `PickingRepository`

**파일:** `picking_repository.dart`

#### `syncPickingOnStartup()` — 앱 시작/복구 시 동기화

```
GET /outbound/pda/picking/sync → 로컬 full replace
```

로컬 sync 방식: 해당 `center_id`의 `ob_pick_hdr`, `ob_pick_dtl` 전체 DELETE → 서버 응답 INSERT

#### `findPickingList(pickListNo)` — 리스트 조회

```
로컬 ob_pick_hdr 조회 → 있으면 + ob_pick_dtl JOIN하여 반환
없으면 온라인 시 GET /outbound/pda/picking/{pickListNo} 단건 조회 → 로컬 sync → 반환
오프라인이면 null
```

#### `executePickingItem(...)` — 단건 수량 처리

```
온라인: postPickingItem() → 로컬 updatePickedQty()
오프라인: 아웃박스 INSERT (eventTypeCd='PICKING_ITEM') → 로컬 updatePickedQty()
```

아웃박스 payload:
```json
{
  "pickListNo": "PL2024000001",
  "seq": 1,
  "pickQty": 5,
  "ordrQty": 10,
  "centerId": "C001",
  "workerId": "user123"
}
```

#### `completePickingList(pickListNo)` — 완료 확정

```
온라인: postCompletePickingList()
오프라인: 아웃박스 INSERT (eventTypeCd='PICK_COMPLETED')
```

아웃박스 payload:
```json
{
  "pickListNo": "PL2024000001",
  "centerId": "C001",
  "workerId": "user123"
}
```

> 완료 처리 시 로컬 DB 삭제는 하지 않는다. state = initial()로 화면만 초기화.

---

### 4. Remote Datasource — `PickingRemoteDatasource`

**파일:** `picking_remote_datasource.dart`

| 메서드 | HTTP | 경로 |
|--------|------|------|
| `fetchPickingSync()` | GET | `/outbound/pda/picking/sync?centerId=` |
| `fetchPickingListDetail()` | GET | `/outbound/pda/picking/{pickListNo}?centerId=` |
| `postPickingItem()` | POST | `/outbound/pda/picking/item` |
| `postCompletePickingList()` | POST | `/outbound/pda/picking/{pickListNo}/complete?centerId=&workerId=` |

`postPickingItem` 요청 body:
```json
{
  "pickListNo": "PL2024000001",
  "seq": 1,
  "pickQty": 5,
  "centerId": "C001",
  "workerId": "user123"
}
```

`fetchPickingSync` 응답 구조:
```json
{
  "items": [
    {
      "pickListNo": "...",
      "ordrNo": "...",
      "details": [ { "seq": 1, "barcode": "...", ... } ]
    }
  ]
}
```

---

### 5. Local Datasource — `PickingLocalDatasource`

**파일:** `picking_local_datasource.dart`  
**DB 버전:** v17 (app_database.dart)

**`ob_pick_hdr` 테이블:**

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `pick_list_no` | TEXT PK | 피킹 리스트 번호 |
| `center_id` | TEXT | 센터 ID |
| `ordr_no` | TEXT | 출고 주문 번호 |
| `vend_id` | TEXT | 거래처 ID |
| `vend_nm` | TEXT | 거래처명 |
| `keep_type_cd` | TEXT | 보관구분 코드 |
| `pick_stts_cd` | TEXT | 피킹 상태 코드 |
| `item_qty` | INTEGER | 총 품목 수 |
| `cmpl_yn` | TEXT | 완료 여부 |
| `reg_dt` | TEXT | 등록일 |
| `synced_at` | TEXT | 동기화 시각 |

**`ob_pick_dtl` 테이블:**

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `pick_list_no` | TEXT | 피킹 리스트 번호 (PK 일부) |
| `center_id` | TEXT | 센터 ID |
| `seq` | INTEGER | 순번 (PK 일부) |
| `goods_cd` | TEXT | 상품코드 |
| `barcode` | TEXT | 바코드 |
| `goods_nm` | TEXT | 상품명 |
| `goods_unit` | TEXT | 단위 |
| `loc_id` | TEXT | 피킹 로케이션 |
| `exp` | TEXT | 유통기한 |
| `ordr_qty` | INTEGER | 주문수량 |
| `pick_qty` | INTEGER | 피킹수량 |
| `cmpl_yn` | TEXT | 완료 여부 |
| `synced_at` | TEXT | 동기화 시각 |

**`updatePickedQty` — 로컬 수량 갱신:**
```dart
db.update(_dtlTable, {
  'pick_qty': pickQty,
  'cmpl_yn': pickQty >= ordrQty ? 'Y' : 'N',
}, where: 'pick_list_no = ? AND seq = ?', ...);
```

---

## Domain 모델

### `PickingItem`

```dart
class PickingItem {
  final int seq;          // 순번 (PK 일부)
  final String goodsCd;
  final String barcode;
  final String goodsNm;
  final String goodsUnit;
  final String locId;     // 피킹 대상 로케이션
  final String? exp;      // 유통기한
  final int ordrQty;      // 주문수량
  final int pickQty;      // 현재 피킹수량
  final String cmplYn;    // 서버 기준 완료 여부

  bool get isCompleted => pickQty >= ordrQty;  // 클라이언트 판단 기준
}
```

> `isCompleted`는 서버의 `cmplYn`이 아닌 `pickQty >= ordrQty`로 판단한다.

### `PickingListHeader`

```dart
class PickingListHeader {
  final String pickListNo;   // 피킹 리스트 번호
  final String ordrNo;       // 출고 주문 번호
  final String vendId;       // 거래처 ID
  final String vendNm;       // 거래처명
  final String keepTypeCd;   // 보관구분
  final String pickSttsCd;   // 피킹 상태 ('01'대기 '02'진행중 '03'완료)
  final int itemQty;         // 총 품목 수
  final String cmplYn;       // 완료 여부
  final String regDt;        // 등록일
}
```

### `PickingList`

```dart
class PickingList {
  final PickingListHeader header;
  final List<PickingItem> items;

  int get totalCount => items.length;
  int get completedCount => items.where((e) => e.isCompleted).length;
  int get progressPercent => ((completedCount / totalCount) * 100).round();
  bool get allCompleted => items.isNotEmpty && items.every((e) => e.isCompleted);
}
```

---

## 백엔드 레이어

### API 컨트롤러 — `OutboundOrderController` (outbound 모듈)

| 엔드포인트 | 동작 |
|-----------|------|
| `GET /outbound/pda/picking/sync` | 센터 기준 진행 중 피킹 리스트 전체 반환 |
| `GET /outbound/pda/picking/{pickListNo}` | 단건 피킹 리스트 + 상세 반환 |
| `POST /outbound/pda/picking/item` | 단건 품목 수량 처리 |
| `POST /outbound/pda/picking/{pickListNo}/complete` | 피킹 리스트 완료 확정 |

---

### `processPdaPickingItem` — 단건 품목 수량 처리

**파일:** `OutboundOrderServiceImpl.java:1486`

```
@Transactional
1. selectPickingDetailForUpdate(pickListNo, seq)  → FOR UPDATE 락
   ├── row == null → 404
   ├── cmplYn == 'Y' → 409 Already picked
   └── pickQty > ordrQty → 400

2. isComplete = (pickQty >= ordrQty)
   updatePickingDetailPicked(pickListNo, seq, pickQty, cmplYn, cmplDt)
   └── cmpl_yn: isComplete ? 'Y' : 'N'
   └── cmpl_dt: isComplete ? today : null

3. updatePickingHeaderInProgress(pickListNo)
   └── pick_stts_cd = '02' WHERE pick_stts_cd = '01' (최초 피킹 시만)

4. updateOrderDetailPickingStatusByPickItem(pickListNo, seq, '02')
   └── ob_ordr_dtl.ins_stts_cd = '02'

5. updateOrderHeaderObStatusDerived(pickListNo, workerId, today)
   └── ob_ordr_hdr.ob_stts_cd 파생 갱신
       - 전체 '03' → '04' (완료)
       - 일부 '02'/'03' → '03' (진행중)
       - 기타 → '02'
```

---

### `completePdaPickingList` — 피킹 완료 확정

**파일:** `OutboundOrderServiceImpl.java:1530`

```
@Transactional
1. countIncompletePdaPickingDetails(pickListNo)
   └── cmpl_yn != 'Y' 행이 1개라도 있으면 → 409 CONFLICT

2. updatePickingHeaderCompleted(pickListNo, workerId, today)
   └── ob_pick_hdr: pick_stts_cd='03', cmpl_yn='Y'

3. selectOutboundStockHistoryCode()
   └── cm_cd_dtl WHERE cmmn_group_cd='STK_HIST_CD' AND cmmn_cd_nm_en='Outbound'
   └── null이면 requireCode() → IllegalStateException → 500

4. insertStockHistPickingComplete(pickListNo, centerId, outboundCode, workerId, today)
   └── ob_pick_dtl JOIN ob_pick_hdr JOIN ob_ordr_hdr LEFT JOIN st_stk
   └── st_stk_hist: stk_hist_cd='O', mod_qty=-pick_qty (차감)

5. updateOrderDetailPickingStatusByPickList(pickListNo, '03')
   └── ob_ordr_dtl.ins_stts_cd = '03'

6. updateOrderHeaderObItemQtyAndOutboundWaiting(pickListNo, workerId, today)
   └── ob_ordr_hdr.ob_item_qty 누적 + ob_stts_cd 파생 갱신

7. updateStockStatusPickingComplete(pickListNo, centerId, workerId, today)
   └── st_stk.stk_stts_cd = 'O', last_ob_dt = today
```

---

## SQL 상세 — `OutboundOrderMapper.xml`

### `insertStockHistPickingComplete` (INSERT-SELECT)

```sql
INSERT INTO wms.st_stk_hist (
    stk_hist_id, center_id, goods_cd, barcode, loc_id, exp, stk_hist_cd,
    mod_qty, orgn_stk_qty, mod_stk_qty, ref_no, rmrk, reg_id, reg_dt
)
SELECT
    'ST' || LPAD(nextval('wms.st_stk_hist_id_seq')::text, 18, '0'),
    oh.center_id,
    pd.goods_cd,
    pd.barcode,
    pd.loc_id,
    pd.exp,
    #{outboundCode},       -- cm_cd_dtl에서 조회한 'O' 코드값
    -pd.pick_qty,          -- 차감 (음수)
    COALESCE(s.tot_stk, 0),
    COALESCE(s.tot_stk, 0) - pd.pick_qty,
    ph.ordr_no,
    '피킹이 완료된 상태',
    #{workerId},
    #{today}
FROM wms.ob_pick_dtl pd
JOIN wms.ob_pick_hdr ph ON ph.pick_list_no = pd.pick_list_no
JOIN wms.ob_ordr_hdr oh ON oh.ordr_no = ph.ordr_no
LEFT JOIN wms.st_stk s
    ON s.center_id = oh.center_id
   AND s.goods_cd  = pd.goods_cd
   AND s.barcode   = pd.barcode
   AND COALESCE(s.exp, '') = COALESCE(pd.exp, '')
   AND s.loc_id    = pd.loc_id
WHERE pd.pick_list_no = #{pickListNo}
  AND oh.center_id    = #{centerId}
```

> `st_stk`가 LEFT JOIN이므로 재고가 없으면 `orgn_stk_qty = 0`으로 이력이 생성된다.

### `updateStockStatusPickingComplete`

```sql
UPDATE wms.st_stk s
SET stk_stts_cd = 'O',
    last_ob_dt  = #{today},
    upd_id      = #{workerId},
    upd_dt      = #{today}
FROM wms.ob_pick_dtl pd
JOIN wms.ob_pick_hdr ph ON ph.pick_list_no = pd.pick_list_no
JOIN wms.ob_ordr_hdr oh ON oh.ordr_no = ph.ordr_no
WHERE pd.pick_list_no = #{pickListNo}
  AND oh.center_id    = #{centerId}
  AND s.center_id     = oh.center_id
  AND s.goods_cd      = pd.goods_cd
  AND s.barcode       = pd.barcode
  AND COALESCE(s.exp, '') = COALESCE(pd.exp, '')
  AND s.loc_id        = pd.loc_id
```

### `updateOrderHeaderObStatusDerived` — 주문 헤더 상태 파생

```sql
ob_stts_cd = CASE
  WHEN (ob_ordr_dtl에 ins_stts_cd != '03'인 행이 하나도 없음) THEN '04'  -- 전체완료
  WHEN (ob_ordr_dtl에 ins_stts_cd IN ('02','03')인 행이 있음)  THEN '03'  -- 진행중
  ELSE '02'                                                               -- 대기
END
```

---

## 오프라인 처리 — 아웃박스 패턴

### 단건 피킹 (오프라인)

```
eventTypeCd: 'PICKING_ITEM'
payload: { pickListNo, seq, pickQty, ordrQty, centerId, workerId }
로컬: updatePickedQty(pick_qty, cmpl_yn) 즉시 반영
```

### 피킹 완료 (오프라인)

```
eventTypeCd: 'PICK_COMPLETED'
payload: { pickListNo, centerId, workerId }
로컬: 별도 처리 없음
```

### 네트워크 복구 시 플러시 (`sync_on_reconnect.dart`)

```
SyncQueueService.flushPending() — created_dtm ASC 순서

eventTypeCd == 'PICKING_ITEM'
→ pickingRemote.postPickingItem(pickListNo, seq, pickQty, centerId, workerId)

eventTypeCd == 'PICK_COMPLETED'
→ pickingRemote.postCompletePickingList(pickListNo, centerId, workerId)
```

> **순서 보장이 중요하다.** `PICKING_ITEM` 이벤트가 모두 플러시되어 서버의 `cmpl_yn='Y'`가 된 뒤에 `PICK_COMPLETED`가 처리되어야 한다. `created_dtm ASC` 정렬이 이를 보장한다.

---

## 주요 코드값 정리

| 구분 | 코드 | 의미 |
|------|------|------|
| `pick_stts_cd` | `'01'` | 피킹 대기 |
| `pick_stts_cd` | `'02'` | 피킹 진행중 |
| `pick_stts_cd` | `'03'` | 피킹 완료 |
| `ob_stts_cd` | `'02'` | 출고 대기 |
| `ob_stts_cd` | `'03'` | 출고 진행중 |
| `ob_stts_cd` | `'04'` | 출고 완료 |
| `ins_stts_cd` | `'02'` | 피킹 진행중 |
| `ins_stts_cd` | `'03'` | 피킹 완료 |
| `stk_hist_cd` | `'O'` | 출고 이력 (피킹 완료 시 생성) |
| `stk_stts_cd` | `'O'` | 출고 상태 |
| `cmpl_yn` | `'Y'` / `'N'` | 완료 여부 |

---

## 알려진 이슈

### `st_stk_hist` 'O' 이력 미생성 문제

`completePdaPickingList`에서 `insertStockHistPickingComplete`가 실행되지 않는 케이스:

1. **`countIncompletePdaPickingDetails > 0` → 409 응답**
   - 원인: `postPickingItem` 실패 시 서버의 `cmpl_yn`이 'Y'가 되지 않음
   - Flutter 쪽에서 `updatePickedQty`는 await하지 않고, try-catch도 없어 API 실패가 조용히 무시됨
   - UI에는 완료로 보이지만 서버 DB는 갱신 안 된 상태

2. **`selectOutboundStockHistoryCode()` → null → 500 응답**
   - 원인: `cm_cd_dtl` 테이블에 `cmmn_group_cd='STK_HIST_CD'`, `cmmn_cd_nm_en='Outbound'`, `use_yn='Y'` 행 없음

3. **INSERT 0행 (조용한 실패)**
   - 원인: `ob_pick_hdr.ordr_no`에 대응하는 `ob_ordr_hdr` 행 없을 때 JOIN 실패
   - API는 200 반환하지만 실제로 이력이 생성되지 않음

---

## 관련 파일 위치

### Flutter (PDA App)
```
lib/features/picking/
├── data/
│   ├── datasource/
│   │   ├── picking_remote_datasource.dart   # API 호출
│   │   ├── picking_local_datasource.dart    # SQLite CRUD
│   │   ├── picking_local_store.dart         # 인터페이스
│   │   └── web_picking_local_datasource.dart
│   ├── model/
│   │   ├── picking_item.dart                # PickingItem (isCompleted 포함)
│   │   └── picking_list.dart                # PickingList + PickingListHeader
│   └── repository/
│       └── picking_repository.dart          # 온/오프라인 분기
└── presentation/
    ├── provider/
    │   ├── picking_provider.dart            # Provider + PickingNotifier
    │   └── picking_state.dart               # PickingState
    ├── screen/
    │   └── picking_screen.dart              # UI + 스캔 로직
    └── widget/
        ├── picking_header_card.dart
        ├── picking_item_card.dart
        ├── picking_progress_card.dart
        ├── picking_scan_section.dart
        └── picking_complete_bar.dart
```

### Backend (Spring Boot)
```
outbound/
├── controller/
│   └── OutboundOrderController.java
├── service/
│   ├── OutboundOrderService.java
│   └── impl/OutboundOrderServiceImpl.java   # processPdaPickingItem, completePdaPickingList
└── dto/
    ├── PickingPdaItemRequest.java
    └── PickingPdaDetailRow.java

resources/mapper/outbound/
└── OutboundOrderMapper.xml                  # selectPickingDetailForUpdate,
                                             # updatePickingDetailPicked,
                                             # insertStockHistPickingComplete,
                                             # updateStockStatusPickingComplete 등
```
