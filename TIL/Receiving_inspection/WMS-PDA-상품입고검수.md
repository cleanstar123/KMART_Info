# 상품입고검수 기능 구현 상세 정리

> 최초 작성일: 2026-06-02  
> 최종 수정일: 2026-06-04  
> 프로젝트: K-MART WMS PDA App  
> 브랜치: dev_wonjun

---

## 1. 기능 개요

| 항목 | 내용 |
|------|------|
| 화면명 | 상품입고검수 |
| 진입 경로 | 홈 메뉴 → 상품입고검수 |
| 대상 데이터 | `ib_stts_cd IN ('01', '02')` — 검수전·검수중 컨테이너만 표시 |
| 검수 흐름 | 컨테이너 목록 → 컨테이너 상세 → 상품별 정상/불량 수량 입력 → 저장(중간저장) / 검수 완료 |
| 오프라인 지원 | 오프라인 시 로컬 DB에 저장 + 아웃박스 큐 적재, 온라인 복구 시 자동 전송 |

---

## 2. DB 스키마 관련

### 2-1. 관련 테이블

#### `wms.ib_list_hdr` (입고 헤더)
| 컬럼 | 설명 |
|------|------|
| `ib_no` | 입고번호 (PK) |
| `ib_stts_cd` | 입고상태 (`01`=검수전, `02`=검수중, `03`=검수완료) |
| `ib_type_cd` | 입고유형 (`01`=해외수입, `02`=현지매입, `03`=반품) |
| `tot_po_qty` | 총 발주수량 |
| `tot_ib_qty` | 총 입고수량 |
| `asn_dt` | 입고예정일 (정렬 기준) |
| `ib_dt` | 실제 입고일 (검수완료 시 자동 세팅) |
| `center_id` | 센터코드 (`HN`, `HC`) |

#### `wms.ib_list_dtl` (입고 상세)
| 컬럼 | 설명 |
|------|------|
| `ins_stts_cd` | 검수상태 (`01`=대기, `02`=검수중, `03`=검수완료) |
| `ib_qty` | 실제 입고수량 (현지매입·반품은 검수 후 `ins_qty + defect_qty`로 갱신) |
| `ins_qty` | 검수자가 입력한 정상수량 |
| `defect_qty` | 검수자가 입력한 불량수량 (신규 컬럼) |
| `po_qty` | 발주수량 (참고용 표시) |
| `rmrk` | 비고 (향후 활용을 위해 유지, DEFECT 패턴 방식은 제거됨) |

### 2-2. ib_stts_cd 상태 전이

```
01 (검수전)
  └─▶ [저장 버튼] ─▶ 02 (검수중)   : dtl ins_stts_cd='02', hdr ib_stts_cd='02' (01→02만 전이)
                                        현지매입·반품: dtl.ib_qty, hdr.tot_ib_qty 갱신
  └─▶ [검수완료 버튼] ─▶ 03 (검수완료) : dtl ins_stts_cd='03', hdr ib_stts_cd='03', hdr.ib_dt 세팅
                                          현지매입·반품: dtl.ib_qty, hdr.tot_ib_qty 갱신
02 (검수중)
  └─▶ [저장 버튼] ─▶ 02 유지 (조건: ib_stts_cd='01'일 때만 02로 전이, 이미 02면 유지)
  └─▶ [검수완료 버튼] ─▶ 03 (검수완료)

03 (검수완료) → 목록에서 제외 (조회 조건: IN ('01','02'))
```

### 2-3. ib_type_cd 분기 처리 (현지매입·반품)

`ib_type_cd IN ('02', '03')` 인 경우:
- 현지매입·반품은 `po_qty = ib_qty`로 초기 세팅되어 있으나 검수 후 수량이 다를 수 있음
- **저장/검수완료 모두**: `dtl.ib_qty = ins_qty + defect_qty` 로 갱신
- **저장 시**: `hdr.tot_ib_qty = SUM(dtl.ib_qty)` 별도 갱신
- **검수완료 시**: `updateInboundAsnConfirmed`가 `ib_stts_cd='03'`, `ib_dt`, `tot_ib_qty` 일괄 처리

### 2-4. 로컬 SQLite 마이그레이션

```dart
// app_database.dart — v11 (defect_qty 컬럼 추가)
static const int _dbVersion = 11;

// onUpgrade
if (oldVersion < 10) await _addInsQtyColumnToIbListDtl(db);
if (oldVersion < 11) await _addDefectQtyColumnToIbListDtl(db);

static Future<void> _addDefectQtyColumnToIbListDtl(Database db) async {
  if (await _columnExists(db, tableIbListDtl, 'defect_qty')) return;
  await db.execute(
    'ALTER TABLE $tableIbListDtl ADD COLUMN defect_qty INTEGER NOT NULL DEFAULT 0',
  );
}
```

`ib_list_dtl` CREATE TABLE에도 `defect_qty INTEGER NOT NULL DEFAULT 0` 추가 (신규 설치 기준).

---

## 3. 백엔드 구현

### 3-1. API 엔드포인트

```
GET  /inbound/pda/sync?centerId={id}&since={datetime}   → 입고 목록 동기화
PATCH /inbound/pda/{ibNo}/inspect                        → 검수 저장/완료
```

- **인증**: Spring Security `permitAll` (`/inbound/pda/**`)
- **PATCH 응답**: 204 No Content (void)

### 3-2. DTO

#### `InboundPdaInspectItem.java`
```java
public record InboundPdaInspectItem(
        String goodsCd,
        String barcode,
        String exp,
        Integer insQty,
        Integer defectiveQty,
        String insSttsCd
) {}
// rmrk() 메서드 제거 — defect_qty 컬럼 직접 저장으로 변경
```

#### `InboundPdaInspectRequest.java`
```java
public record InboundPdaInspectRequest(
        List<InboundPdaInspectItem> items,
        boolean complete   // true=검수완료, false=저장(중간)
) {}
```

#### `InboundPdaDetailItem.java`
```java
public record InboundPdaDetailItem(
        String ibNo, String goodsCd, String barcode, String goodsNm,
        String exp, Integer poQty, Integer ibQty, Integer insQty,
        String insSttsCd, Integer defectQty, String rmrk
) {}
```

### 3-3. MyBatis Mapper XML (`InboundMapper.xml`)

#### resultMap — `defect_qty` 추가
```xml
<resultMap id="inboundPdaDetailItemResultMap" ...>
    <constructor>
        ...
        <arg column="ins_qty"     javaType="java.lang.Integer"/>
        <arg column="ins_stts_cd" javaType="java.lang.String"/>
        <arg column="defect_qty"  javaType="java.lang.Integer"/>  <!-- 추가 -->
        <arg column="rmrk"        javaType="java.lang.String"/>
    </constructor>
</resultMap>
```

#### `selectInboundListForPdaSync` — 조회 조건 추가
```xml
WHERE h.center_id = #{centerId}
  AND h.asn_dt >= #{startDt}
  AND h.ib_stts_cd IN ('01', '02')   -- 검수전·검수중만 동기화
ORDER BY h.asn_dt ASC, h.ib_no ASC
```

#### `selectInboundDetailsByIbNos` — `defect_qty` 추가
```xml
COALESCE(d.ins_qty, 0)    AS ins_qty,
d.ins_stts_cd,
COALESCE(d.defect_qty, 0) AS defect_qty,   -- 추가
d.rmrk
```

#### `updateInboundDtlForPdaInspect` — `rmrk` → `defect_qty` 교체
```xml
<update id="updateInboundDtlForPdaInspect">
    <foreach item="item" collection="items" separator=";">
        UPDATE wms.ib_list_dtl
        SET ins_qty     = #{item.insQty},
            defect_qty  = #{item.defectiveQty},   -- rmrk 방식 제거
            ins_stts_cd = #{item.insSttsCd}
        WHERE ib_no    = #{ibNo}
          AND goods_cd = #{item.goodsCd}
          AND barcode  = #{item.barcode}
          AND exp      = #{item.exp}
    </foreach>
</update>
```

#### 신규 쿼리 3개

```xml
<!-- 현지매입·반품: dtl.ib_qty = ins_qty + defect_qty -->
<update id="updateIbQtyForLocalPurchase">
    UPDATE wms.ib_list_dtl
    SET ib_qty = ins_qty + defect_qty
    WHERE ib_no = #{ibNo}
      AND EXISTS (
          SELECT 1 FROM wms.ib_list_hdr
          WHERE ib_no = #{ibNo} AND ib_type_cd IN ('02', '03')
      )
</update>

<!-- 저장 시 hdr.tot_ib_qty 갱신 (현지매입·반품) -->
<update id="updateHdrTotIbQtyForLocalPurchase">
    UPDATE wms.ib_list_hdr
    SET tot_ib_qty = (SELECT COALESCE(SUM(ib_qty), 0) FROM wms.ib_list_dtl WHERE ib_no = #{ibNo})
    WHERE ib_no = #{ibNo} AND ib_type_cd IN ('02', '03')
</update>

<!-- 저장 시 hdr.ib_stts_cd = '02' (검수전→검수중 전이만) -->
<update id="updateHdrSttsForPdaInspect">
    UPDATE wms.ib_list_hdr
    SET ib_stts_cd = #{sttsCd}
    WHERE ib_no = #{ibNo}
    <if test="sttsCd == '02'">
        AND ib_stts_cd = '01'
    </if>
</update>
```

- **검수완료** 시 hdr 상태 처리는 기존 `updateInboundAsnConfirmed` 재사용  
  (`ib_stts_cd='03'`, `ib_dt=today`, `tot_ib_qty=SUM(dtl.ib_qty)` 일괄 처리)

### 3-4. Mapper 인터페이스 (`InboundMapper.java`)

```java
// 기존
void updateInboundDtlForPdaInspect(@Param("ibNo") String ibNo,
        @Param("items") List<InboundPdaInspectItem> items);

// 신규
void updateIbQtyForLocalPurchase(@Param("ibNo") String ibNo);
void updateHdrTotIbQtyForLocalPurchase(@Param("ibNo") String ibNo);
void updateHdrSttsForPdaInspect(@Param("ibNo") String ibNo, @Param("sttsCd") String sttsCd);
```

### 3-5. Service

```java
// InboundServiceImpl.java
@Override
@Transactional
public void savePdaInspection(String ibNo, InboundPdaInspectRequest request) {
    if (request.items() == null || request.items().isEmpty()) return;

    // 1. dtl ins_qty, defect_qty, ins_stts_cd 업데이트
    inboundMapper.updateInboundDtlForPdaInspect(ibNo, request.items());

    // 2. 현지매입·반품: dtl.ib_qty = ins_qty + defect_qty
    inboundMapper.updateIbQtyForLocalPurchase(ibNo);

    String today = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE);

    if (request.complete()) {
        // 검수완료: hdr ib_stts_cd='03', ib_dt, tot_ib_qty 일괄 처리
        inboundMapper.updateInboundAsnConfirmed(ibNo, today);
    } else {
        // 저장: 현지매입·반품 tot_ib_qty 갱신 후 hdr 검수중으로 전환
        inboundMapper.updateHdrTotIbQtyForLocalPurchase(ibNo);
        inboundMapper.updateHdrSttsForPdaInspect(ibNo, "02");
    }
}
```

### 3-6. Controller
```java
// InboundController.java (기존 유지)
@PatchMapping("/pda/{ibNo}/inspect")
public void savePdaInspection(
        @PathVariable String ibNo,
        @RequestBody InboundPdaInspectRequest request) {
    inboundService.savePdaInspection(ibNo, request);
}
```

---

## 4. 프론트엔드 구현

### 4-1. 아키텍처 레이어

```
presentation (UI / Notifier)
    ↓
domain (InspectionItem, InspectionContainer)
    ↓
data
  ├── repository (InboundRepository)
  ├── remote   (InboundRemoteDatasource)
  └── local    (InboundLocalStore ← SQLite / Sembast)
```

---

### 4-2. Domain 모델 (`InspectionItem`)

```dart
class InspectionItem {
  final int insQty;     // 정상수량 (DB: ins_qty)
  final int defectQty;  // 불량수량 (DB: defect_qty) — 신규
  final String? rmrk;   // 향후 활용을 위해 유지
  // ...

  // copyWith — defectiveQtyOverride 제거, defectQty로 교체
  InspectionItem copyWith({int? insQty, int? defectQty, String? insSttsCd}) { ... }
}
```

- `defectiveQty` computed getter (rmrk DEFECT 파싱) 제거 → `defectQty` 직접 필드로 변경
- `fromJson`, `toMap`, `fromMap` 모두 `defect_qty` ↔ `defectQty` 매핑 포함

---

### 4-3. Data Layer

#### `InboundLocalStore` (abstract interface)
```dart
Future<void> updateItemInspection({
  required String ibNo,
  required String goodsCd,
  required String barcode,
  required String exp,
  required int insQty,
  required int defectQty,     // defectiveQty → defectQty로 변경
  required String insSttsCd,
});

Future<void> updateHdrStatus({  // 신규
  required String ibNo,
  required String ibSttsCd,
});

Future<int> countWaitingInbound(String centerId);
```

#### `InboundLocalDatasource` (SQLite)
- `getContainersByCenter`: `ib_stts_cd IN ('01','02')` 필터, `asn_dt ASC` 정렬
- `syncInbound` 전체 동기화 시 삭제 범위: `ib_stts_cd IN ('01','02')`만 삭제 — `'03'`(검수완료)은 삭제하지 않음
- hdr INSERT: `ConflictAlgorithm.ignore` — 기존 로컬 상태를 서버 데이터로 덮어쓰지 않음 (오프라인 완료 보호)
- `updateItemInspection`: `SET ins_qty=?, defect_qty=?, ins_stts_cd=?` (rmrk 제거)
- `updateHdrStatus`: `'02'`는 `ib_stts_cd='01'` 조건부 전이, `'03'`은 무조건 업데이트

#### `WebInboundLocalDatasource` (Sembast)
- `getContainersByCenter`: `Filter.or([Filter.equals('ib_stts_cd','01'), Filter.equals('ib_stts_cd','02')])`
- `syncInbound` 전체 동기화 시 삭제 범위: `ib_stts_cd IN ('01','02')`만 삭제 — `'03'` 보호
- `updateItemInspection`: `defect_qty` 저장 (rmrk 제거)
- `updateHdrStatus`: 레코드 조회 후 `ib_stts_cd` 덮어쓰기

#### `InboundRemoteDatasource`
```dart
Future<void> patchInspection({
  required String ibNo,
  required List<Map<String, dynamic>> items,
  required bool complete,   // 신규 — 백엔드 complete 플래그 전달
}) async {
  await _dio.patch('/inbound/pda/$ibNo/inspect',
      data: {'items': items, 'complete': complete});
}
```

#### `InboundRepository` — `saveInspection`

```
saveInspection(ibNo, items, complete):
  1. 로컬 dtl 업데이트 (updateItemInspection × items)
  2. 로컬 hdr 상태 업데이트 (updateHdrStatus: '02' 또는 '03')
  3. 온라인이면 → patchInspection(ibNo, items, complete) → 성공 시 종료
  4. 오프라인 또는 API 실패 → outbox에 INSP_SAVE / INSP_COMPLETE 적재
```

- **로컬 우선**: API 실패해도 로컬 DB는 항상 반영
- `complete=true` → `insSttsCd='03'`, `false` → `'02'`
- payload의 `defectiveQty` 키: `item.defectQty` 값 사용

---

### 4-4. Sync (아웃박스 패턴)

#### `SyncModels`
```dart
static const String inspSave     = 'INSP_SAVE';
static const String inspComplete = 'INSP_COMPLETE';
```

#### `sync_on_reconnect.dart` — 네트워크 복구 시 실행 순서

> **중요**: 아웃박스 플러시가 반드시 인바운드 sync보다 먼저 실행되어야 한다.  
> 플러시 전에 sync를 실행하면 서버의 stale 데이터(`ib_stts_cd='01'`)가 로컬의 완료 상태(`'03'`)를 덮어쓸 수 있다.

```dart
Future<void> onNetworkRestored(WidgetRef ref) async {
  // 1. 상품·사용자 마스터 동기화
  await ref.read(goodsRepositoryProvider).syncGoodsOnStartup();
  await ref.read(authRepositoryProvider).syncUsersOnStartup();

  // 2. 아웃박스 플러시 (반드시 inbound sync 이전에 실행)
  if (AuthTokenHolder.token != null) {
    await SyncQueueService(OutboxDao()).flushPending((eventType, payload) async {
      if (eventType == SyncModels.inspSave || eventType == SyncModels.inspComplete) {
        final ibNo = payload['ibNo'] as String;
        final items = (payload['items'] as List).map(...).toList();
        await remote.patchInspection(
          ibNo: ibNo,
          items: items,
          complete: eventType == SyncModels.inspComplete,
        );
      }
      // putaway 이벤트 처리 ...
    });
  }

  // 3. 서버 데이터 동기화 (플러시 완료 후)
  await ref.read(inboundRepositoryProvider).syncInboundOnStartup();
  await ref.read(putawayRepositoryProvider).syncPutawayOnStartup();

  // 4. 홈 카운트 갱신
  if (AuthTokenHolder.token == null) return;
  await ref.read(homeProvider.notifier).loadUnsyncedCount();
}
```

---

### 4-5. Presentation Layer

#### `InboundNotifier.saveInspection`
```dart
Future<void> saveInspection({String ibNo, List<InspectionItem> items, bool complete}) async {
  state = state.copyWith(isLoading: true);
  await repository.saveInspection(ibNo: ibNo, items: items, complete: complete);
  final containers = await repository.getContainers();
  state = state.copyWith(isLoading: false, containers: containers);
  // 오프라인 작업도 홈 대시보드에 즉시 반영 (로컬 DB + 아웃박스 기반이므로 온/오프 무관)
  await _ref.read(homeProvider.notifier).loadUnsyncedCount();
}
```

- 검수 저장/완료 후 `homeProvider.notifier.loadUnsyncedCount()` 호출
  - `waitingInboundCount` 감소 (완료된 컨테이너는 `ib_stts_cd='03'`이므로 카운트 제외)
  - `unsyncedCount` 증가 (아웃박스에 대기 이벤트 추가됨)

#### `ReceivingInspectionDetailScreen` 버튼 동작

| 버튼 | 텍스트 | 조건 | 동작 |
|------|--------|------|------|
| 저장 | `저장` (변경: 확인→저장) | 항상 활성 | `saveInspection(complete: false)` → dtl `ins_stts_cd='02'`, hdr `ib_stts_cd='02'` |
| 검수 완료 | `검수 완료` | 모든 항목 `정상+불량 == ibQty` | `saveInspection(complete: true)` → dtl `ins_stts_cd='03'`, hdr `ib_stts_cd='03'`, `ib_dt` 세팅 → pop |

- 검수완료 후 컨테이너가 목록에서 자동으로 제외됨 (ib_stts_cd='03'은 조회 조건 밖)
- `_isAllCompleted()`: `item.ibQty > 0 && (normalQty + defectiveQty) == item.ibQty`

#### 수량 입력 UX
- `onTap`: 필드 탭 시 전체선택 → 기존 `0` 즉시 대체
- `FilteringTextInputFormatter.digitsOnly`: 숫자만 입력 허용
- 초기값: `item.insQty`, `item.defectQty` (DB 저장값 복원)

#### `InspectionItemCard` 수치 기준

| 항목 | 내용 |
|------|------|
| 우측 N (`검수 X / N`) | `ibQty` (실제 입고수량) |
| 초과 판정 | `inspectedTotal > ibQty` |
| helper 텍스트 | `발주 수량: {poQty}개` |

#### `InspectionContainerCard` 수치 기준

| 항목 | 내용 |
|------|------|
| 우측 N (`검수 X / N`) | `container.totIbQty` |
| 뱃지 | `발주 {totPoQty}` 추가 |

---

### 4-6. 홈화면 입고 대기 카운트

```dart
// HomeNotifier
Future<void> loadUnsyncedCount() async {
  final centerId = _centerId();
  final waitingInboundCount = centerId != null
      ? await ref.read(inboundLocalStoreProvider).countWaitingInbound(centerId)
      : 0;
  state = state.copyWith(waitingInboundCount: waitingInboundCount, ...);
}
```

- `countWaitingInbound`: `ib_stts_cd='01'` 헤더의 `tot_po_qty` 합계
- `HomeState.initial()` 더미 숫자 → 모두 0으로 교체

---

## 5. 앱 시작 시 데이터 동기화 흐름

```
main()
  ├── authBootstrapProvider   → cm_user 로컬 동기화
  ├── goodsBootstrapProvider  → md_goods 로컬 동기화
  └── inboundBootstrapProvider
        → InboundRepository.syncInboundOnStartup()
              → ['HN', 'HC'] 순회
              → GET /inbound/pda/sync?centerId={id}&since={lastSyncTime}
              → ib_stts_cd IN ('01','02') 만 서버에서 내려옴
              → 로컬 DB upsert
```

- 로그인 전에도 동기화: `['HN','HC']` 직접 순회 (authSessionProvider 불필요)
- `since` 파라미터 있으면 증분, 없으면 최근 30일 전체 동기화
- 완료된 컨테이너(`ib_stts_cd='03'`)는 서버 응답에서 제외 → 로컬 DB에서도 자연히 사라짐

---

## 6. 전체 변경 파일 목록

### 백엔드

| 파일 | 변경 내용 |
|------|-----------|
| `SecurityConfig.java` | `/inbound/pda/**` permitAll 추가 |
| `InboundPdaDetailItem.java` | `insQty`, `defectQty` 필드 추가, `rmrk` 유지 |
| `InboundPdaInspectItem.java` | 신규 생성 → `rmrk()` 메서드 제거 (defect_qty 컬럼 직접 사용) |
| `InboundPdaInspectRequest.java` | 신규 생성 → `complete` boolean 추가 |
| `InboundMapper.java` | `updateInboundDtlForPdaInspect`, `updateIbQtyForLocalPurchase`, `updateHdrTotIbQtyForLocalPurchase`, `updateHdrSttsForPdaInspect` 추가 |
| `InboundMapper.xml` | resultMap `defect_qty` 추가, sync 쿼리 `ib_stts_cd IN ('01','02')` 필터, UPDATE `defect_qty` 교체, 신규 쿼리 3개 |
| `InboundService.java` | `savePdaInspection` 인터페이스 추가 |
| `InboundServiceImpl.java` | `savePdaInspection` 구현 — ib_type_cd 분기, complete 분기 |
| `InboundController.java` | `PATCH /pda/{ibNo}/inspect` 엔드포인트 추가 |

### 프론트엔드

| 파일 | 변경 내용 |
|------|-----------|
| `app_database.dart` | v11 마이그레이션: `ib_list_dtl.defect_qty` 컬럼 추가 |
| `inspection_item.dart` | `defectQty` 필드 추가, `defectiveQty` getter 및 DEFECT 파싱 제거, `copyWith` 파라미터 교체 |
| `inbound_local_store.dart` | `updateItemInspection(defectQty)`, `updateHdrStatus` 추가 |
| `inbound_local_datasource.dart` | `ib_stts_cd IN ('01','02')` 필터, `defect_qty` 저장, `updateHdrStatus` 구현, full sync 삭제 범위 `('01','02')`로 제한, hdr INSERT `ConflictAlgorithm.ignore` 적용 |
| `web_inbound_local_datasource.dart` | `Filter.or` 방식 IN 조건, `defect_qty` 저장, `updateHdrStatus` 구현, full sync 삭제 범위 `('01','02')`로 제한 |
| `inbound_remote_datasource.dart` | `patchInspection(complete)` 파라미터 추가 |
| `inbound_repository.dart` | `defectQty` 사용, `updateHdrStatus` 호출, `complete` 전달 |
| `sync_models.dart` | `inspSave`, `inspComplete` 상수 추가 |
| `sync_queue_service.dart` | `flushPending(onDispatch)` 실제 구현으로 교체 |
| `sync_on_reconnect.dart` | 아웃박스 플러시를 inbound sync 이전으로 이동 (sync 순서 버그 수정), `complete` 플래그 전달 |
| `inbound_provider.dart` | `inboundLocalStoreProvider` 공개, `InboundNotifier.saveInspection` 추가, 저장 완료 후 `homeProvider.notifier.loadUnsyncedCount()` 호출 추가 |
| `inspection_item_card.dart` | `ibQty` 기준, `발주 수량` helper, 전체선택 onTap, 숫자전용 inputFormatters |
| `inspection_container_card.dart` | `totIbQty` 기준, 발주수량 뱃지 추가 |
| `receiving_inspection_detail_screen.dart` | 버튼 텍스트 "확인"→"저장", `copyWith(defectQty:)` 교체 |
| `receiving_inspection_screen.dart` | `ibQty` 기준 완료 판정, controller 초기값 `defectQty` 반영 |
| `home_provider.dart` | `Ref` 기반으로 변경, 실제 입고 대기 카운트 로드 |
| `home_state.dart` | 더미 초기값 0으로 교체 |
| `main.dart` | `goodsBootstrapProvider`, `inboundBootstrapProvider` 추가 |
