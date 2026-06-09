# 입고적재 Sync & Local DB 구현 정리

> 작성일: 2026-06-09  
> 대상 기능: 입고적재 (Receiving Putaway)  
> 관련 피처: 입고검수 (Receiving Inspection)

---

## 1. 전체 아키텍처 개요

```
[서버 API]
    ↕ PutawayRemoteDatasource (Dio)
[Repository]  PutawayRepository
    ↕
[Local Store] PutawayLocalStore (interface)
    ├── PutawayLocalDatasource     ← Android PDA (SQLite / sqflite)
    └── WebPutawayLocalDatasource  ← 웹 브라우저 (Sembast / IndexedDB)
    ↕
[Provider]    putawayProvider (StateNotifier)
    ↕
[Screen]      ReceivingPutawayScreen
```

플랫폼 분기는 `putaway_provider.dart`의 `_putawayLocalStoreProvider`에서 `kIsWeb`으로 결정한다.

```dart
final _putawayLocalStoreProvider = Provider<PutawayLocalStore>((ref) {
  if (kIsWeb) return WebPutawayLocalDatasource();
  return PutawayLocalDatasource();
});
```

---

## 2. 로컬 DB 테이블 스키마

### 2-1. `putaway_pending` — 적재 대기 상품

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `stk_hist_id` | TEXT PK | 재고 이력 ID (서버 생성) |
| `center_id` | TEXT | 센터 코드 |
| `goods_cd` | TEXT | 상품 코드 |
| `barcode` | TEXT | 바코드 |
| `exp` | TEXT | 유통기한 |
| `goods_nm` | TEXT | 상품명 |
| `default_loc_id` | TEXT | 기본 적재 위치 |
| `keep_type_cd` | TEXT | 보관 유형 (01:냉장 / 02:냉동 / 03:상온) |
| `qty` | INTEGER | 수량 |
| `synced_at` | TEXT | 마지막 동기화 시각 |

### 2-2. `lc_master` — 로케이션 마스터

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `loc_id` | TEXT PK | 로케이션 ID |
| `center_id` | TEXT PK | 센터 코드 |
| `keep_type_cd` | TEXT | 보관 유형 |
| `load_yn` | TEXT | 적재 가능 여부 (Y/N) |
| `use_yn` | TEXT | 사용 여부 (Y/N) |
| `synced_at` | TEXT | 마지막 동기화 시각 |

SQLite(네이티브)는 `_createPutawayTables()`에서 DDL로 생성, Sembast(웹)는 스토어 이름으로 자동 생성된다.

---

## 3. 앱 시작 시 동기화 흐름

### 트리거

`putawayBootstrapProvider`는 `FutureProvider`로, 앱이 처음 마운트될 때 한 번 실행된다.

```dart
final putawayBootstrapProvider = FutureProvider<bool>((ref) async {
  return ref.read(putawayRepositoryProvider).syncPutawayOnStartup();
});
```

### `syncPutawayOnStartup()` 흐름

```
1. centerId 없으면 → false 반환 (로그인 전 접근 방지)

2. [병렬 API 호출] Future.wait([...])
   ├── fetchPutawayPending(centerId)   → GET /inbound/pda/putaway/pending
   └── fetchLocationsForSync(centerId) → GET /inbound/pda/location/sync

3. [순차 로컬 저장] — 락 충돌 방지
   ├── syncPutawayPending(centerId, items)
   └── syncLocations(centerId, locations)

4. 성공 → true, AppException / 기타 예외 → false
```

2단계의 두 API는 완전히 독립적이므로 `Future.wait()`로 병렬 실행하여 네트워크 대기 시간을 절반으로 줄인다. 3단계는 동일한 DB에 쓰므로 순차 실행으로 락 충돌을 방지한다.

---

## 4. 로컬 저장 로직 (syncPutawayPending / syncLocations)

### 전략: Full Replacement (전체 교체)

적재 대기 목록과 로케이션 마스터는 증분 동기화 없이 **항상 전체 교체**한다. 서버가 항상 완전한 목록을 내려주고, 적재가 완료된 항목은 서버에서 이미 제거되어 있다.

### SQLite 구현 (`PutawayLocalDatasource`)

```dart
await db.transaction((txn) async {
  // 1. 기존 데이터 전체 삭제
  await txn.delete(_pendingTable, where: 'center_id = ?', whereArgs: [centerId]);
  // 2. 새 데이터 삽입
  for (final item in items) {
    await txn.insert(_pendingTable, item.toRow(centerId, syncedAt));
  }
});
```

트랜잭션으로 묶어 삭제 후 삽입 사이에 빈 상태가 노출되지 않도록 한다.

### Sembast 구현 (`WebPutawayLocalDatasource`)

```dart
await db.transaction((txn) async {
  // 1. Finder로 해당 센터 레코드 일괄 삭제 (단일 호출)
  await _pendingStore.delete(
    txn,
    finder: Finder(filter: Filter.equals('center_id', centerId)),
  );
  // 2. 새 데이터 upsert
  for (final item in items) {
    await _pendingStore.record(item.stkHistId)
        .put(txn, item.toRow(centerId, syncedAt).cast<String, Object?>());
  }
});
```

이전 구현은 `find()` 후 루프로 레코드를 개별 삭제했다. 현재는 `store.delete(txn, finder: ...)` 단일 호출로 교체하여 N번의 IndexedDB 트랜잭션을 1번으로 줄였다.

---

## 5. 로케이션 유효성 검증 흐름

사용자가 로케이션 바코드를 스캔했을 때의 흐름이다.

```
validateLocation(locId)
  └── [1차] 로컬 DB 조회 (getLocation)
        ├── 있음 → 반환
        └── 없음 + 온라인 → [2차] 서버 API 호출
              GET /inbound/pda/location/{locId}
              └── 성공 → LocationInfo 반환
              └── 실패 → null 반환 (오프라인 폴백)
```

로컬에 캐시된 로케이션이 있으면 서버를 호출하지 않는다. 신규 로케이션이거나 캐시 미스 시에만 API를 호출한다.

검증 결과:
- `null` → 존재하지 않는 로케이션 → 스낵바 오류
- `loc.isLoadable == false` (`load_yn != 'Y'`) → 적재 불가 로케이션 → 스낵바 오류
- 정상 → `scannedLocation` 상태에 저장

---

## 6. 적재 실행 흐름 (executePutawayItem)

```
executePutawayItem(stkHistId, locId)
  ├── [온라인]
  │     ├── POST /inbound/pda/putaway/execute
  │     └── 성공 → removePutawayPendingItem(stkHistId) — 로컬 삭제
  │
  └── [오프라인]
        ├── Outbox에 삽입 (eventTypeCd = 'PUTAWAY_ITEM')
        │     payload: { stkHistId, locId, centerId }
        └── removePutawayPendingItem(stkHistId) — 로컬 삭제 (낙관적 삭제)
```

오프라인 상태에서도 항목은 즉시 로컬 목록에서 제거된다. 실제 서버 전송은 네트워크 복구 시 Outbox 플러시로 처리된다.

---

## 7. Outbox 패턴

오프라인 중 발생한 작업(적재 실행, 검수 저장·완료)은 `wms_sync_outbox` 테이블에 적재된다.

```
wms_sync_outbox
  outbox_id      TEXT PK   (UUID v4)
  site_cd        TEXT      (centerId)
  event_type_cd  TEXT      (PUTAWAY_ITEM / PUTAWAY / INSP_SAVE / INSP_COMPLETE)
  payload_json   TEXT      (JSON 직렬화된 요청 데이터)
  sync_status_cd TEXT      (PENDING → DONE / FAILED)
  retry_cnt      INTEGER
  created_dtm    TEXT
```

`OutboxDao`가 플랫폼에 따라 `SqliteOutboxStore`(네이티브) 또는 `WebOutboxStore`(웹)를 자동 선택한다.

---

## 8. 네트워크 복구 시 동기화 순서

`onNetworkRestored(ref)` — `sync_on_reconnect.dart`에 정의되어 있다.

```
1. [unawaited] 공통코드 동기화 (백그라운드, 비차단)

2. 상품 마스터·위치·재고이력 동기화 (syncGoodsOnStartup)
3. 사용자 마스터 동기화 (syncUsersOnStartup)

4. [Outbox 플러시] SyncQueueService.flushPending(...)
   ├── INSP_SAVE / INSP_COMPLETE → patchInspection API
   ├── PUTAWAY                   → postPutaway API
   └── PUTAWAY_ITEM              → postPutawayItem API
   ※ 로컬 작업을 서버에 먼저 반영한 뒤 Pull해야 stale 덮어쓰기 방지

5. 입고검수 마스터 동기화 (syncInboundOnStartup)
6. 입고적재 마스터 동기화 (syncPutawayOnStartup)

7. 홈 화면 미동기화 카운트 갱신 (loadUnsyncedCount)
```

4번(Outbox 플러시) → 5·6번(Pull) 순서가 핵심이다. 순서가 바뀌면 서버의 오래된 데이터가 로컬 로컬 오프라인 작업을 덮어쓴다.

---

## 9. 입고검수 동기화와의 차이점

| 항목 | 입고검수 (InboundRepository) | 입고적재 (PutawayRepository) |
|------|------------------------------|-------------------------------|
| 동기화 방식 | 증분 동기화 (`since` 파라미터) | 전체 교체 (항상 full) |
| 마지막 동기화 시각 보관 | `wms_app_setting`에 저장 | 없음 |
| API 병렬 호출 | 단일 엔드포인트 | 2개 병렬 (`Future.wait`) |
| 오프라인 쓰기 | Outbox (INSP_SAVE / INSP_COMPLETE) | Outbox (PUTAWAY_ITEM) |
| 로컬 낙관적 업데이트 | `updateItemInspection` 즉시 반영 | `removePutawayPendingItem` 즉시 삭제 |

---

## 10. IbListSembastHelper 배치 삭제 최적화

검수 목록 Full Sync 시 기존 레코드를 삭제하는 로직 (`ib_list_sembast_helper.dart`):

**이전 (N×2 delete 호출):**
```dart
for (final record in existing) {
  final ibNo = record.value['ib_no'] as String;
  await dtlStore.delete(db, finder: Finder(filter: Filter.equals('ib_no', ibNo)));
  await hdrStore.record(record.key).delete(db);
}
```

**현재 (2 delete 호출):**
```dart
if (existing.isNotEmpty) {
  final ibNos = existing.map((r) => r.value['ib_no'] as String? ?? '').toList();
  final dtlFilter = ibNos.length == 1
      ? Filter.equals('ib_no', ibNos.first)
      : Filter.or(ibNos.map((n) => Filter.equals('ib_no', n)).toList());
  await dtlStore.delete(db, finder: Finder(filter: dtlFilter)); // 전체 dtl 1회 삭제
  await hdrStore.delete(db, finder: Finder(filter: _hdrFilter(...))); // 전체 hdr 1회 삭제
}
```

입고건 N개가 있으면 이전 방식은 `2N`번 IndexedDB를 호출하지만, 현재는 항상 `2`번이다.

---

## 11. 관련 파일 경로

```
lib/
├── core/
│   ├── database/
│   │   ├── app_database.dart               # SQLite 초기화·마이그레이션·테이블 상수
│   │   ├── outbox_dao.dart                 # Outbox 진입점 (플랫폼 자동 분기)
│   │   ├── sqlite_outbox_store.dart        # SQLite Outbox 구현
│   │   └── web_outbox_store.dart           # Sembast Outbox 구현
│   ├── local/
│   │   └── ib_list_sembast_helper.dart     # 검수 hdr/dtl Sembast 공통 로직 (배치 삭제)
│   └── sync/
│       ├── sync_on_reconnect.dart          # 네트워크 복구 시 동기화 순서 정의
│       ├── sync_queue_service.dart         # Outbox 플러시 서비스
│       └── sync_models.dart               # 이벤트 타입 상수 (PUTAWAY_ITEM 등)
│
└── features/receiving_putaway/
    ├── domain/model/
    │   ├── putaway_item.dart               # 적재 대기 도메인 모델 (fromJson/fromRow/toRow)
    │   └── location_info.dart             # 로케이션 도메인 모델
    ├── data/
    │   ├── datasource/
    │   │   ├── putaway_local_store.dart        # 로컬 저장소 인터페이스
    │   │   ├── putaway_local_datasource.dart   # SQLite 구현
    │   │   ├── web_putaway_local_datasource.dart # Sembast 구현
    │   │   └── putaway_remote_datasource.dart  # Dio API 클라이언트
    │   └── repository/
    │       └── putaway_repository.dart         # 온·오프라인 분기 + Outbox 처리
    └── presentation/
        └── provider/
            └── putaway_provider.dart           # StateNotifier + Bootstrap FutureProvider
```
