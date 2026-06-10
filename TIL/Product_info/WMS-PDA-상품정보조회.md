# 상품정보조회 구현 정리

## 개요

바코드를 입력받아 상품 정보, 위치별 재고, 재고 변동 이력을 조회하는 기능.
Flutter Clean Architecture + Spring Boot MVC + MyBatis 구조로 구현되어 있으며,
온라인/오프라인 상태에 따라 서버 우선 or 로컬 SQLite 폴백 전략을 사용한다.

---

## 전체 구조

```
Flutter (PDA App)
├── Presentation
│   ├── Screen       → ProductInfoScreen
│   ├── Provider     → ProductInfoNotifier (StateNotifier)
│   └── Widget       → ProductInfoCard, GoodsLocationsBanner, StockHistBanner
├── Data
│   ├── Repository   → GoodsRepository
│   ├── Remote DS    → GoodsRemoteDatasource
│   ├── Local DS     → GoodsLocalDatasource, GoodsLocationLocalDatasource, StockHistLocalDatasource
│   └── Model        → GoodsModel, GoodsLocationItem, StockHistItem

Spring Boot (Backend)
├── Controller  → CommonController
├── Mapper      → CommonMapper (interface)
└── XML         → CommonMapper.xml (MyBatis SQL)
```

---

## 1. Flutter - Presentation Layer

### 1-1. ProductInfoScreen

**파일:** `lib/features/product_info/presentation/screen/product_info_screen.dart`

`ConsumerStatefulWidget`. 상품 검색 UI와 두 개의 슬라이드 패널(위치, 이력)을 관리한다.

**로컬 상태 (StatefulWidget 내부):**

```dart
bool _locationPanelVisible = false;
bool _isLoadingLocations = false;
List<GoodsLocationItem> _locations = [];

bool _historyPanelVisible = false;
bool _isLoadingHistory = false;
List<StockHistItem> _history = [];
```

**패널 표시 메서드:**

```dart
Future<void> _showLocationPanel() async {
  setState(() => _isLoadingLocations = true);
  final barcode = ref.read(productInfoProvider).product?.barcode;
  final list = await ref.read(goodsRepositoryProvider).fetchLocationsByBarcode(barcode!);
  setState(() {
    _locations = list;
    _locationPanelVisible = true;
    _isLoadingLocations = false;
  });
}

Future<void> _showHistoryPanel() async {
  // 동일 패턴
}
```

### 1-2. ProductInfoNotifier

**파일:** `lib/features/product_info/presentation/provider/product_info_provider.dart`

**상태:**

```dart
class ProductInfoState {
  final bool isLoading;
  final String barcodeInput;
  final ProductInfoModel? product;  // null이면 미조회 상태
  final String? errorMessage;
}
```

**searchProduct() 흐름:**

1. 바코드 trim 후 공백이면 오류 메시지 설정
2. `GoodsRepository.findByBarcode()` 호출
3. 결과 null → "상품 없음" 메시지
4. 결과 있으면 `GoodsModel.toProductInfoModel()`로 변환 후 상태 업데이트

**UI 모델 변환 주요 로직:**

```dart
// GoodsModel → ProductInfoModel
String _storageTypeText() {
  // keepTypeCd 코드 → 한글 텍스트 (공통코드 lookup)
}

String _packQtyText() {
  // unitQty + goodsUnit 조합 (e.g., "10 EA")
}
```

### 1-3. ProductInfoCard 그리드 항목 (2x4 레이아웃)

| 위치  | 항목                   |
| --- | -------------------- |
| 1   | 입수 수량 (Pack Qty)     |
| 2   | 보관 유형 (칩 스타일, 색상 구분) |
| 3   | 가용 재고 (녹색)           |
| 4   | 할당 재고 (황색)           |
| 5   | 안전 재고                |
| 6   | 주 로케이션 (전체 너비)       |
| 7   | 최종 입고일               |
| 8   | 최종 출고일               |

---

## 2. Flutter - Data Layer

### 2-1. GoodsRepository

**파일:** `lib/features/product_info/data/repository/goods_repository.dart`

#### findByBarcode() - 온/오프라인 전환 전략

```
온라인 + 센터코드 유효
  → GoodsRemoteDatasource.fetchGoodsDetailByBarcode()
      → 성공 → GoodsLocalStore.upsertGoods() 후 반환
      → 실패(404/4xx) → 로컬 DB 폴백
오프라인 또는 센터코드 없음
  → GoodsLocalStore.findByBarcode()
```

#### fetchLocationsByBarcode()

```
온라인 → GET /common/goods/barcode/{barcode}/locations
          → GoodsLocationLocalStore.replaceLocationsForBarcode()
오프라인 → GoodsLocationLocalStore.findByBarcode()
```

#### syncGoods() - 앱 시작 시 동기화

```
[병렬] 마지막 동기화 시각 3개 조회
  ├── GoodsLocalStore.getLastSyncTime()
  ├── GoodsLocationLocalStore.getLastSyncTime()
  └── StockHistLocalStore.getLastSyncTime()

[병렬] 서버 API 3개 호출
  ├── GET /common/goods/pda-sync?since=...
  ├── GET /common/goods/locations?since=...
  └── GET /common/goods/stock-history?since=...

[순차] 로컬 DB 저장  ← SQLite 잠금 충돌 방지
  ├── GoodsLocalStore.syncGoods()
  ├── GoodsLocationLocalStore.syncLocations()
  └── StockHistLocalStore.syncStockHist()
```

**증분 동기화:** `since` 파라미터에 마지막 동기화 시각을 넘기면 변경분만 가져온다.

### 2-2. GoodsRemoteDatasource

**파일:** `lib/features/product_info/data/datasource/goods_remote_datasource.dart`

| 메서드 | 엔드포인트 | 설명 |
|--------|-----------|------|
| `fetchGoodsForStartupSync` | `GET /common/goods/pda-sync` | 전체/증분 상품 동기화 |
| `fetchGoodsDetailByBarcode` | `GET /common/goods/barcode/{barcode}` | 바코드 상세 조회 |
| `fetchGoodsLocationsForStartupSync` | `GET /common/goods/locations` | 전체/증분 위치 동기화 |
| `fetchGoodsLocationsByBarcode` | `GET /common/goods/barcode/{barcode}/locations` | 바코드별 위치 조회 |
| `fetchStockHistForStartupSync` | `GET /common/goods/stock-history` | 전체/증분 이력 동기화 |
| `fetchStockHistByBarcode` | `GET /common/goods/barcode/{barcode}/stock-history` | 바코드별 이력 조회 |

- barcode는 `Uri.encodeComponent(barcode)`로 URI 인코딩 처리
- 응답 공통 래퍼: `GoodsSyncResponse<T> { items, syncedAt, isIncremental }`

### 2-3. 로컬 데이터소스 (SQLite)

**테이블 매핑:**

| 로컬 스토어 | SQLite 테이블 |
|------------|--------------|
| `GoodsLocalDatasource` | `AppDatabase.tableMdGoods` |
| `GoodsLocationLocalDatasource` | `AppDatabase.tableMdGoodsLocation` |
| `StockHistLocalDatasource` | `AppDatabase.tableWmsStockHist` |
| (공통) | `AppDatabase.tableWmsAppSetting` - 동기화 타임스탬프 |

**GoodsLocalDatasource 주요 메서드:**

```dart
// 전체 또는 증분 동기화 저장
Future<void> syncGoods(List<GoodsModel> goods, {bool isIncremental = false})

// 단일 항목 upsert (바코드 검색 후 갱신용)
Future<void> upsertGoods(GoodsModel goods)

// 바코드 또는 상품코드로 조회
Future<GoodsModel?> findByBarcode(String barcode, {String? centerCode})

// 동기화 시각 관리
Future<String?> getLastSyncTime()
Future<void> setLastSyncTime(String time)
```

**GoodsLocationLocalDatasource:**

```dart
// isIncremental=false이면 전체 DELETE 후 INSERT, true이면 upsert
Future<void> syncLocations(List<GoodsLocationItem> list, {bool isIncremental})

// 특정 바코드의 위치 레코드만 교체 (바코드 검색 후 갱신용)
Future<void> replaceLocationsForBarcode(String barcode, List<GoodsLocationItem> list)

// 가용 재고 내림차순 정렬
Future<List<GoodsLocationItem>> findByBarcode({String? centerCode, required String barcode})
```

### 2-4. 데이터 모델

#### GoodsModel

```dart
factory GoodsModel.fromListJson(Map<String, dynamic> json)   // 동기화 목록 파싱
factory GoodsModel.fromDetailJson(Map<String, dynamic> json) // 상세 조회 파싱
factory GoodsModel.fromMap(Map<String, dynamic> map)         // SQLite 레코드 파싱

Map<String, dynamic> toMap()                // SQLite 저장용
ProductInfoModel toProductInfoModel()       // UI 모델 변환
```

#### StockHistItem 계산 속성

```dart
bool get isInbound => modQty > 0;
String get formattedDate => regDt를 yyyy-MM-dd 형식으로 포맷
```

### 2-5. 의존성 주입 (Provider)

**파일:** `lib/features/product_info/presentation/provider/goods_repository_provider.dart`

```dart
// 플랫폼별 로컬 스토어 (Web / Mobile 분기)
final goodsLocalStoreProvider = Provider<GoodsLocalStore>((ref) {
  if (kIsWeb) return WebGoodsLocalDatasource();
  return GoodsLocalDatasource();
});

// 리포지토리
final goodsRepositoryProvider = Provider<GoodsRepository>((ref) {
  return GoodsRepository(
    remote: GoodsRemoteDatasource(),
    local: ref.watch(goodsLocalStoreProvider),
    locationLocal: ref.watch(goodsLocationLocalStoreProvider),
    stockHistLocal: ref.watch(stockHistLocalStoreProvider),
    ref: ref,
  );
});

// 앱 시작 시 동기화 트리거
final goodsBootstrapProvider = FutureProvider<bool>((ref) async {
  return ref.read(goodsRepositoryProvider).syncGoodsOnStartup();
});
```

---

## 3. Spring Boot - Backend

### 3-1. CommonController

**파일:** `src/main/java/com/kmarket/wms/common/controller/CommonController.java`

```java
// 앱 시작 동기화
@GetMapping("/goods/pda-sync")
public GoodsSyncResponse<GoodsSearchItem> getGoodsDetailForPdaSync(
    @RequestParam String centerCode,
    @RequestParam(required = false) String since)

// 바코드 상세 조회 - 없으면 404
@GetMapping("/goods/barcode/{barcode}")
public ResponseEntity<GoodsDetailItem> getGoodsDetailByBarcode(
    @PathVariable String barcode,
    @RequestParam String centerCode)

// 위치별 재고 (전체 동기화)
@GetMapping("/goods/locations")
public GoodsSyncResponse<GoodsLocationItem> getGoodsLocationForPdaSync(...)

// 위치별 재고 (바코드 조회)
@GetMapping("/goods/barcode/{barcode}/locations")
public List<GoodsLocationItem> getGoodsLocationsByBarcode(...)

// 재고 이력 (전체 동기화)
@GetMapping("/goods/stock-history")
public GoodsSyncResponse<StockHistItem> getStockHistForPdaSync(...)

// 재고 이력 (바코드 조회)
@GetMapping("/goods/barcode/{barcode}/stock-history")
public List<StockHistItem> getStockHistByBarcode(...)
```

### 3-2. CommonMapper.xml - 핵심 SQL

#### selectGoodsForPdaSync - 상품 동기화

```sql
SELECT g.goods_cd, gu.cmmn_cd AS goods_unit, ...
FROM wms.md_goods g
LEFT JOIN (
  -- 로케이션별 재고를 상품 단위로 집계
  SELECT center_id, goods_cd, barcode,
    SUM(tot_stk)  AS current_stock,
    SUM(avl_stk)  AS available_stock,
    SUM(alot_stk) AS allocated_stock,
    MAX(ib_dt)    AS last_inbound_dt,
    MAX(last_ob_dt) AS last_outbound_dt,
    -- 가용 재고 기준 대표 로케이션 선정
    (ARRAY_AGG(loc_id ORDER BY avl_stk DESC, loc_id ASC))[1] AS main_location
  FROM wms.st_stk
  GROUP BY center_id, goods_cd, barcode
) s ON s.center_id = g.center_id AND s.goods_cd = g.goods_cd AND s.barcode = g.barcode
LEFT JOIN wms.cm_cd_dtl gu ON gu.cmmn_group_cd = 'GOODS_UNIT' AND gu.cmmn_cd = g.goods_unit
LEFT JOIN wms.cm_cd_dtl kt ON kt.cmmn_group_cd = 'KEEP_TYPE_CD' AND kt.cmmn_cd = g.keep_type_cd
WHERE g.center_id = #{centerCode}
  AND g.use_yn = 'Y'
  <if test="since != null">
    AND g.upd_dt >= TO_CHAR(#{since}::timestamp, 'YYYYMMDD')
  </if>
```

**핵심 포인트:**
- `ARRAY_AGG(...)[1]` - 가용 재고가 가장 많은 로케이션을 주 로케이션으로 선택
- `since` 없으면 전체, 있으면 증분 (upd_dt 기준)

#### selectGoodsDetailByBarcode - 바코드 상세 조회

```sql
WHERE g.use_yn = 'Y'
  AND g.center_id = #{centerCode}
  AND (g.barcode = #{barcode} OR g.goods_cd = #{barcode})  -- 바코드 또는 상품코드 허용
LIMIT 1
```

#### selectGoodsLocationsByBarcode - 위치별 재고

```sql
SELECT
  center_id, barcode, loc_id,
  SUM(tot_stk)  AS current_stock,
  SUM(avl_stk)  AS available_stock,
  SUM(alot_stk) AS allocated_stock
FROM wms.st_stk
WHERE center_id = #{centerCode} AND barcode = #{barcode}
GROUP BY center_id, barcode, loc_id
ORDER BY SUM(avl_stk) DESC, loc_id ASC  -- 가용 재고 많은 순
```

#### selectStockHistByBarcode - 재고 이력

```sql
SELECT stk_hist_id, center_id, goods_cd, barcode, loc_id, stk_hist_cd,
       mod_qty, orgn_stk_qty, mod_stk_qty, ref_no, rmrk, reg_id, reg_dt
FROM wms.st_stk_hist
WHERE center_id = #{centerCode} AND barcode = #{barcode}
ORDER BY reg_dt DESC, stk_hist_id DESC
LIMIT 50  -- 최근 50건
```

#### selectStockHistForStartupSync - 이력 전체 동기화

```sql
WHERE center_id = #{centerCode}
  <if test="since != null">
    AND reg_dt >= TO_CHAR(#{since}::timestamp, 'YYYYMMDD')
  </if>
  <if test="since == null">
    AND reg_dt >= TO_CHAR(NOW() - INTERVAL '90 days', 'YYYYMMDD')  -- 최근 90일
  </if>
ORDER BY reg_dt DESC, stk_hist_id DESC
LIMIT 2000
```

### 3-3. DTO

**파일:** `src/main/java/com/kmarket/wms/common/dto/`

```java
// 바코드 상세 조회 응답
record GoodsDetailItem(
    String centerCode, String goodsCode, String goodsBarcode,
    String goodsName, String goodsSpec, String goodsUnit, Integer unitQty,
    String keepingType, Integer safeStockQty, String mainLocation,
    Integer currentStock, Integer availableStock, Integer allocatedStock,
    String lastInboundDate, String lastOutboundDate
) {}

// 위치별 재고
record GoodsLocationItem(
    String centerCode, String goodsBarcode, String locationCode,
    Integer currentStock, Integer availableStock, Integer allocatedStock
) {}

// 재고 이력
record StockHistItem(
    String stkHistId, String centerCode, String goodsCd, String barcode,
    String locId, String stkHistCd, Integer modQty,
    Integer orgnStkQty, Integer modStkQty,
    String refNo, String rmrk, String regId, String regDt
) {}

// 동기화 응답 래퍼 (제네릭)
record GoodsSyncResponse<T>(
    List<T> items,
    String syncedAt,      // LocalDateTime.now() 기록
    boolean isIncremental
) {}
```

---

## 4. DB 테이블 구조

### wms.md_goods - 상품 마스터

| 컬럼 | 설명 |
|------|------|
| center_id | 센터 ID (PK) |
| goods_cd | 상품 코드 (PK) |
| barcode | 바코드 (PK) |
| goods_nm_ko / en / vi | 상품명 다국어 |
| goods_spec | 규격 |
| goods_unit | 단위 (공통코드: GOODS_UNIT) |
| unit_qty | 입수 수량 |
| keep_type_cd | 보관 유형 (공통코드: KEEP_TYPE_CD) |
| safe_stk_qty | 안전 재고 |
| loc_id | 주 로케이션 |
| use_yn | 사용 여부 |
| upd_dt | 수정일 (증분 동기화 기준) |

### wms.st_stk - 재고 현황

| 컬럼 | 설명 |
|------|------|
| center_id, goods_cd, barcode, loc_id | PK |
| tot_stk | 전체 재고 |
| avl_stk | 가용 재고 |
| alot_stk | 할당 재고 |
| ib_dt | 최종 입고일 |
| last_ob_dt | 최종 출고일 |

### wms.st_stk_hist - 재고 변동 이력

| 컬럼 | 설명 |
|------|------|
| stk_hist_id | PK (`'ST' || LPAD(nextval, 18, '0')`) |
| stk_hist_cd | 변동 유형 (I=입고, L=입고적재, O=출고 등) |
| mod_qty | 변동량 (+입고, -출고) |
| orgn_stk_qty | 변동 전 재고 |
| mod_stk_qty | 변동 후 재고 |
| ref_no | 참조 번호 (입고번호, 주문번호 등) |
| reg_dt | 등록일 (YYYYMMDD) |

---

## 5. 전체 흐름도

### 바코드 검색 흐름

```
[사용자] 바코드 입력
    ↓
ProductInfoNotifier.searchProduct()
    ↓ trim, 공백 검사
GoodsRepository.findByBarcode()
    ├─ 온라인? → GoodsRemoteDatasource → GET /common/goods/barcode/{barcode}
    │               ├─ 200 → upsertGoods() → 반환
    │               └─ 404/오류 → 로컬 폴백
    └─ 오프라인? → GoodsLocalDatasource.findByBarcode()
    ↓
GoodsModel.toProductInfoModel()
    ↓
ProductInfoState 업데이트 → ProductInfoCard 렌더링
```

### 위치/이력 패널 흐름

```
[사용자] 위치 버튼 클릭
    ↓
ProductInfoScreen._showLocationPanel()
    ↓
GoodsRepository.fetchLocationsByBarcode()
    ├─ 온라인 → GET /common/goods/barcode/{barcode}/locations
    │            → replaceLocationsForBarcode() (로컬 갱신)
    └─ 오프라인 → GoodsLocationLocalDatasource.findByBarcode()
    ↓
setState() → GoodsLocationsBanner 슬라이드 표시
```

### 앱 시작 동기화 흐름

```
goodsBootstrapProvider (FutureProvider)
    ↓
GoodsRepository.syncGoodsOnStartup()
    ↓ 예외 시 false 반환 (기존 데이터 유지)
GoodsRepository.syncGoods()
    ↓
[병렬] 마지막 동기화 시각 3개 조회 (AppSetting 테이블)
    ↓
[병렬] API 3개 동시 호출 (since 파라미터로 증분)
    ├─ GET /common/goods/pda-sync
    ├─ GET /common/goods/locations
    └─ GET /common/goods/stock-history
    ↓
[순차] SQLite 저장 (잠금 충돌 방지)
    ├─ syncGoods()
    ├─ syncLocations()
    └─ syncStockHist()
```

---

## 6. 설계 포인트 정리

### 온/오프라인 전환
`wmsOnlineProvider`로 네트워크 상태를 감지하여 Repository에서 분기 처리. 항상 서버 우선 + 로컬 폴백 패턴을 따른다.

### 병렬 API 호출 + 순차 DB 저장
동기화 시 3개 API를 `Future.wait`로 동시 호출하여 속도를 높이고, DB 저장은 순차로 진행하여 SQLite 잠금 충돌을 방지한다.

### 주 로케이션 선정
SQL의 `ARRAY_AGG(loc_id ORDER BY avl_stk DESC)[1]`을 사용하여 가용 재고가 가장 많은 로케이션을 주 로케이션으로 동적 계산한다. md_goods의 loc_id 컬럼과 별개로 현재 재고 상태 기준으로 결정된다. (loc_id 컬럼 값을 주로케이션으로 해야할지 의문?)

### 증분 동기화
마지막 동기화 시각(`syncedAt`)을 로컬 AppSetting 테이블에 저장하고, 다음 동기화 시 `since` 파라미터로 넘긴다. 서버에서 `upd_dt >= since`(상품) 또는 `reg_dt >= since`(이력) 조건으로 변경분만 반환한다.

### 이력 테이블 INSERT 전략
`st_stk_hist`는 순수 이력 테이블로 UPDATE 없이 항상 새 레코드를 INSERT한다. PK는 `'ST' || LPAD(nextval('wms.st_stk_hist_id_seq')::text, 18, '0')` 형식으로 시퀀스를 사용한다.

### 플랫폼별 로컬 스토어 분리
`kIsWeb` 조건으로 Web(`WebGoodsLocalDatasource`)과 Mobile(`GoodsLocalDatasource`)을 다른 구현체로 제공한다. Repository 코드는 `GoodsLocalStore` 인터페이스만 바라본다.
