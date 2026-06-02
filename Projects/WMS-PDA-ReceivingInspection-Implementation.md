# 상품입고검수 기능 구현 상세 정리

> 작성일: 2026-06-02  
> 프로젝트: K-MART WMS PDA App  
> 브랜치: dev_wonjun

---

## 1. 기능 개요

| 항목 | 내용 |
|------|------|
| 화면명 | 상품입고검수 |
| 진입 경로 | 홈 메뉴 → 상품입고검수 |
| 대상 데이터 | `ib_stts_cd = '03'` (입고완료) 상태의 컨테이너만 표시 |
| 검수 흐름 | 컨테이너 목록 → 컨테이너 상세 → 상품별 정상/불량 수량 입력 → 확인(중간저장) / 검수 완료 |
| 오프라인 지원 | 오프라인 시 로컬 DB에 저장 + 아웃박스 큐 적재, 온라인 복구 시 자동 전송 |

---

## 2. DB 스키마 관련

### 2-1. 관련 테이블

#### `wms.ib_list_hdr` (입고 헤더)
| 컬럼 | 설명 |
|------|------|
| `ib_no` | 입고번호 (PK) |
| `ib_stts_cd` | 입고상태 (`01`=예정, `02`=진행중, `03`=입고완료) |
| `tot_po_qty` | 총 발주수량 |
| `tot_ib_qty` | 총 입고수량 |
| `asn_dt` | 입고예정일 (정렬 기준) |
| `center_id` | 센터코드 (`HN`, `HC`) |

#### `wms.ib_list_dtl` (입고 상세)
| 컬럼 | 설명 |
|------|------|
| `ins_stts_cd` | 검수상태 (`01`=대기, `02`=검수중, `03`=검수완료) |
| `ib_qty` | 실제 입고수량 (검수 완료 기준값) |
| `ins_qty` | 검수자가 입력한 정상수량 |
| `rmrk` | 불량수량 저장 포맷: `DEFECT:{n}` |
| `po_qty` | 발주수량 (참고용 표시) |

### 2-2. 로컬 SQLite 마이그레이션 (v9 → v10)

```dart
// app_database.dart
if (oldVersion < 10) await _addInsQtyColumnToIbListDtl(db);

static Future<void> _addInsQtyColumnToIbListDtl(Database db) async {
  if (await _columnExists(db, tableIbListDtl, 'ins_qty')) return;
  await db.execute(
    'ALTER TABLE $tableIbListDtl ADD COLUMN ins_qty INTEGER NOT NULL DEFAULT 0',
  );
}
```

`ib_list_dtl` CREATE TABLE에도 `ins_qty INTEGER NOT NULL DEFAULT 0` 추가.

---

## 3. 백엔드 구현

### 3-1. 신규 API 엔드포인트

```
PATCH /inbound/pda/{ibNo}/inspect
```

- **인증**: Spring Security `permitAll` (`/inbound/pda/**`)
- **요청 Body**: `InboundPdaInspectRequest`
- **응답**: 204 No Content (void)

### 3-2. 신규 DTO

#### `InboundPdaInspectItem.java`
```java
public record InboundPdaInspectItem(
        String goodsCd,
        String barcode,
        String exp,
        Integer insQty,
        Integer defectiveQty,
        String insSttsCd
) {
    public String rmrk() {
        return (defectiveQty != null && defectiveQty > 0) ? "DEFECT:" + defectiveQty : null;
    }
}
```

- `rmrk()`는 MyBatis `#{item.rmrk}` 호출 시 자동으로 사용되는 메서드
- 불량수량이 없으면 `null` 반환 → DB에 NULL 저장

#### `InboundPdaInspectRequest.java`
```java
public record InboundPdaInspectRequest(List<InboundPdaInspectItem> items) {}
```

### 3-3. 기존 DTO 수정

#### `InboundPdaDetailItem.java` — `insQty` 필드 추가
```java
public record InboundPdaDetailItem(
        String ibNo, String goodsCd, String barcode, String goodsNm,
        String exp, Integer poQty, Integer ibQty, Integer insQty,  // insQty 추가
        String insSttsCd, String rmrk
) {}
```

### 3-4. MyBatis Mapper XML 변경 (`InboundMapper.xml`)

#### resultMap에 `ins_qty` 추가
```xml
<resultMap id="inboundPdaDetailItemResultMap" ...>
    <constructor>
        ...
        <arg column="ib_qty"      javaType="java.lang.Integer"/>
        <arg column="ins_qty"     javaType="java.lang.Integer"/>  <!-- 추가 -->
        <arg column="ins_stts_cd" javaType="java.lang.String"/>
        ...
    </constructor>
</resultMap>
```

#### `selectInboundDetailsByIbNos` 쿼리에 컬럼 추가
```xml
COALESCE(d.ib_qty, 0)  AS ib_qty,
COALESCE(d.ins_qty, 0) AS ins_qty,   -- 추가
d.ins_stts_cd,
```

#### 정렬 순서 변경 (입고예정일 오름차순)
```xml
ORDER BY h.asn_dt ASC, h.ib_no ASC
```

#### 신규 UPDATE 쿼리 (`updateInboundDtlForPdaInspect`)
```xml
<update id="updateInboundDtlForPdaInspect">
    <foreach item="item" collection="items" separator=";">
        UPDATE wms.ib_list_dtl
        SET ins_qty     = #{item.insQty},
            rmrk        = #{item.rmrk},
            ins_stts_cd = #{item.insSttsCd}
        WHERE ib_no    = #{ibNo}
          AND goods_cd = #{item.goodsCd}
          AND barcode  = #{item.barcode}
          AND exp      = #{item.exp}
    </foreach>
</update>
```

- `ib_list_hdr.ib_stts_cd`는 **변경하지 않음** (검수는 이미 입고완료된 항목에 대한 후처리)
- `foreach`에 `separator=";"` 사용 → 한 트랜잭션에서 다중 UPDATE

### 3-5. Mapper 인터페이스 추가
```java
void updateInboundDtlForPdaInspect(
    @Param("ibNo") String ibNo,
    @Param("items") List<InboundPdaInspectItem> items
);
```

### 3-6. Service / Controller
```java
// InboundService.java
void savePdaInspection(String ibNo, InboundPdaInspectRequest request);

// InboundServiceImpl.java
@Override
@Transactional
public void savePdaInspection(String ibNo, InboundPdaInspectRequest request) {
    if (request.items() == null || request.items().isEmpty()) return;
    inboundMapper.updateInboundDtlForPdaInspect(ibNo, request.items());
}

// InboundController.java
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

### 4-2. Domain 모델 수정

#### `InspectionItem` — `insQty`, `defectiveQty` getter, `copyWith` 추가

```dart
class InspectionItem {
  final int insQty;       // 검수자 입력 정상수량 (DB: ins_qty)
  // ...

  int get defectiveQty {  // rmrk에서 파싱 (DEFECT:{n})
    if (rmrk == null) return 0;
    final match = RegExp(r'DEFECT:(\d+)').firstMatch(rmrk!);
    return int.tryParse(match?.group(1) ?? '') ?? 0;
  }

  InspectionItem copyWith({int? insQty, int? defectiveQtyOverride, String? insSttsCd}) {
    final newRmrk = defectiveQtyOverride != null
        ? (defectiveQtyOverride > 0 ? 'DEFECT:$defectiveQtyOverride' : null)
        : rmrk;
    return InspectionItem(...);
  }
}
```

- `toMap()` / `fromMap()` / `fromJson()` 모두 `ins_qty` ↔ `insQty` 매핑 추가
- 불량수량은 별도 컬럼 없이 `rmrk` 필드에 `DEFECT:N` 포맷으로 저장/파싱

---

### 4-3. Data Layer

#### `InboundLocalStore` (abstract interface) — 신규 메서드 2개
```dart
Future<void> updateItemInspection({
  required String ibNo,
  required String goodsCd,
  required String barcode,
  required String exp,
  required int insQty,
  required int defectiveQty,
  required String insSttsCd,
});

Future<int> countWaitingInbound(String centerId);
```

#### `InboundLocalDatasource` (SQLite)
- `updateItemInspection`: `UPDATE ib_list_dtl SET ins_qty=?, rmrk=?, ins_stts_cd=? WHERE ...`
- `countWaitingInbound`: `SELECT SUM(tot_po_qty) FROM ib_list_hdr WHERE center_id=? AND ib_stts_cd='01'`
- `getContainersByCenter`: `ib_stts_cd = '03'` 필터, `asn_dt ASC` 정렬

#### `WebInboundLocalDatasource` (Sembast)
- 동일 인터페이스를 Sembast `StoreRef` API로 구현
- `Filter.and([...'03'])` 으로 필터링
- `fold<int>` 타입 명시로 타입 추론 오류 방지

#### `InboundRemoteDatasource` — `patchInspection` 추가
```dart
Future<void> patchInspection({
  required String ibNo,
  required List<Map<String, dynamic>> items,
}) async {
  await _dio.patch('/inbound/pda/$ibNo/inspect', data: {'items': items});
}
```

#### `InboundRepository` — `saveInspection` 추가
```
saveInspection(ibNo, items, complete):
  1. 로컬 DB 즉시 업데이트 (updateItemInspection × items)
  2. 온라인이면 → patchInspection API 호출 → 성공 시 종료
  3. 오프라인 또는 API 실패 → outbox에 INSP_SAVE / INSP_COMPLETE 이벤트 적재
```

- **로컬 우선 저장**: API 실패해도 로컬 DB는 항상 업데이트됨
- `complete=true` → `insSttsCd = '03'`, `false` → `'02'`
- `IB_STTS_CD`(헤더 상태)는 변경하지 않음

---

### 4-4. Sync (아웃박스 패턴)

#### `SyncModels` — 검수 이벤트 타입 추가
```dart
static const String inspSave     = 'INSP_SAVE';
static const String inspComplete = 'INSP_COMPLETE';
```

#### `SyncQueueService.flushPending` — 실제 dispatch로 교체
```dart
Future<void> flushPending(
  Future<void> Function(String eventType, Map<String, dynamic> payload) onDispatch,
) async {
  final pending = await outboxDao.findPendingOutboxes();
  for (final row in pending) {
    try {
      await onDispatch(row['event_type_cd'], jsonDecode(row['payload_json']));
      // → DONE
    } catch (_) {
      // → FAILED, retryCnt + 1
    }
  }
}
```

#### `sync_on_reconnect.dart` — 네트워크 복구 시 아웃박스 플러시
```dart
await SyncQueueService(OutboxDao()).flushPending((eventType, payload) async {
  if (eventType == SyncModels.inspSave || eventType == SyncModels.inspComplete) {
    final ibNo = payload['ibNo'] as String;
    final items = (payload['items'] as List).map(...).toList();
    await remote.patchInspection(ibNo: ibNo, items: items);
  }
});
```

---

### 4-5. Presentation Layer

#### `InboundNotifier` — `saveInspection` 추가
```dart
Future<void> saveInspection({String ibNo, List<InspectionItem> items, bool complete}) async {
  state = state.copyWith(isLoading: true);
  await repository.saveInspection(ibNo: ibNo, items: items, complete: complete);
  final containers = await repository.getContainers(); // 목록 갱신
  state = state.copyWith(isLoading: false, containers: containers);
}
```

#### `ReceivingInspectionDetailScreen` — `ConsumerStatefulWidget`으로 변환

| 버튼 | 조건 | 동작 |
|------|------|------|
| 확인 | 항상 활성 | `saveInspection(complete: false)` → `ins_stts_cd = '02'` |
| 검수 완료 | 모든 항목 `정상+불량 == ib_qty` 일 때 활성화 | `saveInspection(complete: true)` → `ins_stts_cd = '03'` → pop |

- 저장 중 `_isSaving = true` → 버튼 비활성화 + 스피너 표시
- 완료 후 자동으로 상위 목록 화면으로 복귀

#### 수량 입력 UX 개선
- `onTap`: 필드 탭 시 전체 선택 → 기존 `0` 즉시 대체 입력 가능
- `inputFormatters: [FilteringTextInputFormatter.digitsOnly]`: 숫자만 입력 허용

#### `InspectionItemCard` — 수치 기준 수정
| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 우측 N (`검수 X / N`) | `orderedQty` (= `poQty`) | `ibQty` (실제 입고수량) |
| 초과 판정 기준 | `inspectedTotal > poQty` | `inspectedTotal > ibQty` |
| helper 텍스트 | — | `발주 수량: {poQty}개` |

#### `InspectionContainerCard` — 수치 기준 수정
| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 우측 N (`검수 X / N`) | items 합계 `poQty` | `container.totIbQty` |
| 뱃지 추가 | — | `발주 {totPoQty}` 뱃지 |

#### `ReceivingInspectionScreen`
- `_isContainerCompleted`: `ibQty` 기준으로 변경
- `_ensureControllers`: 신규 항목은 `item.insQty` / `item.defectiveQty`로 초기화 (재진입 시 기존 값 유지)
- 컨테이너 정렬: `asn_dt ASC` (입고예정일 빠른 순)

---

### 4-6. 홈화면 입고 대기 카운트

#### 문제
`HomeState.initial()`에 `waitingInboundCount: 15` 하드코딩 → 실제 DB와 무관하게 고정 표시

#### 해결
```dart
// HomeNotifier
Future<void> loadUnsyncedCount() async {
  final centerId = _centerId(); // authSessionProvider에서 읽음
  final waitingInboundCount = centerId != null
      ? await ref.read(inboundLocalStoreProvider).countWaitingInbound(centerId)
      : 0;
  state = state.copyWith(waitingInboundCount: waitingInboundCount, ...);
}
```

- `countWaitingInbound`: `ib_stts_cd = '01'`인 헤더들의 `tot_po_qty` 합계
- 로그인 후 `loadUnsyncedCount()` 호출 시 실제 값으로 갱신
- `HomeState.initial()` 모든 더미 숫자 → 0으로 교체

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
              → 로컬 DB upsert
```

- 앱 시작 시 로그인 전에도 동기화: centerId가 없으므로 `['HN', 'HC']`를 직접 순회
- `since` 파라미터가 있으면 증분 동기화, 없으면 최근 30일 전체 동기화
- Spring Security: `/inbound/pda/**` → `permitAll()` (JWT 불필요)

---

## 6. 전체 변경 파일 목록

### 백엔드
| 파일 | 변경 내용 |
|------|-----------|
| `SecurityConfig.java` | `/inbound/pda/**` permitAll 추가 |
| `InboundPdaDetailItem.java` | `insQty` 필드 추가 |
| `InboundPdaInspectItem.java` | 신규 생성 |
| `InboundPdaInspectRequest.java` | 신규 생성 |
| `InboundMapper.java` | `updateInboundDtlForPdaInspect` 추가 |
| `InboundMapper.xml` | resultMap에 `ins_qty` 추가, 동기화 쿼리에 `ins_qty` 추가, UPDATE 쿼리 추가, 정렬 ASC 변경 |
| `InboundService.java` | `savePdaInspection` 인터페이스 추가 |
| `InboundServiceImpl.java` | `savePdaInspection` 구현 |
| `InboundController.java` | `PATCH /pda/{ibNo}/inspect` 엔드포인트 추가 |

### 프론트엔드
| 파일 | 변경 내용 |
|------|-----------|
| `app_database.dart` | v10 마이그레이션: `ib_list_dtl.ins_qty` 컬럼 추가 |
| `inspection_item.dart` | `insQty`, `defectiveQty` getter, `copyWith`, JSON/Map 직렬화 추가 |
| `inbound_local_store.dart` | `updateItemInspection`, `countWaitingInbound` 인터페이스 추가 |
| `inbound_local_datasource.dart` | 두 메서드 구현, `ib_stts_cd='03'` 필터, ASC 정렬 |
| `web_inbound_local_datasource.dart` | 두 메서드 구현, Sembast 필터/정렬 |
| `inbound_remote_datasource.dart` | `patchInspection` 추가 |
| `inbound_repository.dart` | `saveInspection` 추가 (로컬 우선 + 아웃박스 폴백) |
| `sync_models.dart` | `inspSave`, `inspComplete` 상수 추가 |
| `sync_queue_service.dart` | `flushPending(onDispatch)` 실제 구현으로 교체 |
| `sync_on_reconnect.dart` | 네트워크 복구 시 검수 아웃박스 플러시 연결 |
| `inbound_provider.dart` | `inboundLocalStoreProvider` 공개, `InboundNotifier.saveInspection` 추가 |
| `inspection_item_card.dart` | `ibQty` 기준 수치, `발주 수량` helper, 전체선택 onTap, 숫자전용 inputFormatters |
| `inspection_container_card.dart` | `totIbQty` 기준, 발주수량 뱃지 추가 |
| `receiving_inspection_detail_screen.dart` | `ConsumerStatefulWidget`, 확인/검수완료 버튼 연결 |
| `receiving_inspection_screen.dart` | `ibQty` 기준 완료 판정, controller 초기값 기존 `insQty` 반영 |
| `home_provider.dart` | `Ref` 기반으로 변경, 실제 입고 대기 카운트 로드 |
| `home_state.dart` | 더미 초기값 0으로 교체 |
| `main.dart` | `goodsBootstrapProvider`, `inboundBootstrapProvider` 추가 |
