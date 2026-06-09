# WMS PDA 앱 — 오프라인 동기화 & 로컬 DB 통합 아키텍처

> 작성일: 2026-06-09

---

## 목차

1. [전체 설계 철학](#1-전체-설계-철학)
2. [온라인 상태 감지](#2-온라인-상태-감지)
3. [플랫폼별 로컬 DB](#3-플랫폼별-로컬-db)
4. [로컬 DB 테이블 전체 목록](#4-로컬-db-테이블-전체-목록)
5. [동기화 트리거 3가지](#5-동기화-트리거-3가지)
6. [피처별 동기화 전략](#6-피처별-동기화-전략)
7. [증분 동기화 공통 패턴](#7-증분-동기화-공통-패턴)
8. [Outbox 패턴 (오프라인 쓰기)](#8-outbox-패턴-오프라인-쓰기)
9. [네트워크 복구 시 재동기화 순서](#9-네트워크-복구-시-재동기화-순서)
10. [읽기 요청의 온·오프라인 분기](#10-읽기-요청의-온오프라인-분기)
11. [홈 화면 미동기화 카운트 갱신](#11-홈-화면-미동기화-카운트-갱신)
12. [핵심 구현 패턴 모음](#12-핵심-구현-패턴-모음)
13. [관련 파일 경로 전체 목록](#13-관련-파일-경로-전체-목록)

---

## 1. 전체 설계 철학

```
[서버 API] ←→ Remote Datasource
                      ↕
             Repository (온·오프라인 분기)
                      ↕
[로컬 DB]  ←→ Local Store (interface)
               ├── SQLite 구현  ← Android PDA (sqflite)
               └── Sembast 구현 ← 웹 브라우저 (IndexedDB)
                      ↕
             StateNotifier (Provider)
                      ↕
             Screen (UI)
```

**핵심 원칙 세 가지**

| 원칙 | 내용 |
|------|------|
| Local-first | 항상 로컬 DB에서 화면을 그린다. 서버 응답은 로컬을 갱신한 뒤 반영 |
| Outbox 패턴 | 오프라인 중 발생한 쓰기는 `wms_sync_outbox`에 적재, 복구 시 플러시 |
| 낙관적 업데이트 | 적재 실행·검수 저장은 서버 응답을 기다리지 않고 로컬 상태를 즉시 반영 |

---

## 2. 온라인 상태 감지

온라인 여부는 **두 조건이 모두 충족**될 때만 `true`다.

```
wmsOnlineProvider (bool)
  ├── networkStatusProvider   ← connectivity_plus 패키지
  │     Wi-Fi·모바일 등 기기 네트워크 연결 여부 (StreamProvider)
  │     none / bluetooth → false
  │
  └── serverReachableProvider ← WMS 서버 생존 확인
        GET /auth/users 에 5초 타임아웃으로 헬스체크
        ├── 네트워크 변경 이벤트 발생 시 즉시 체크
        └── 20초 주기 폴링 타이머 유지
```

`app.dart`에서 `wmsOnlineProvider`가 `false → true`로 바뀔 때 `onNetworkRestored(ref)`를 호출한다.

```dart
// app.dart
ref.listen(wmsOnlineProvider, (previous, next) {
  if (next != true) return;
  if (previous == true) return;      // 이미 온라인이면 무시
  onNetworkRestored(ref);
});
```

---

## 3. 플랫폼별 로컬 DB

```dart
// 각 feature provider에서 kIsWeb으로 분기
final _putawayLocalStoreProvider = Provider<PutawayLocalStore>((ref) {
  if (kIsWeb) return WebPutawayLocalDatasource();  // Sembast / IndexedDB
  return PutawayLocalDatasource();                  // SQLite / sqflite
});
```

| 항목 | Android PDA | 웹 브라우저 |
|------|-------------|------------|
| 엔진 | sqflite (SQLite) | sembast_web (IndexedDB) |
| DB 파일 | `wms_pda_app.db` | `wms_pda_web.db` |
| 트랜잭션 | `db.transaction((txn) {...})` | `db.transaction((txn) {...})` |
| 레코드 삭제 | `txn.delete(table, where: ...)` | `store.delete(txn, finder: ...)` |
| 레코드 삽입 | `txn.insert(table, row)` | `store.record(key).put(txn, map)` |
| 마지막 동기화 시각 | `wms_app_setting` 테이블 (`setting_key` PK) | `wms_app_setting` Sembast 스토어 |

Sembast는 키-맵 구조(`StoreRef<String, Map<String,Object?>>`)이며 SQL 스키마가 없다. SQLite는 `AppDatabase._createTables()`에서 DDL로 스키마를 정의하고 버전별 마이그레이션(`onUpgrade`)을 관리한다.

---

## 4. 로컬 DB 테이블 전체 목록

| 테이블 상수 | 실제 이름 | 용도 |
|-------------|-----------|------|
| `tableCmUser` | `cm_user` | 사용자 마스터 (오프라인 로그인용) |
| `tableWmsLoginLog` | `wms_login_log` | 로그인 이력 |
| `tableWmsAppSetting` | `wms_app_setting` | 앱 설정 / **마지막 동기화 시각** |
| `tableWmsI18n` | `wms_i18n` | 다국어 메시지 |
| `tableWmsSyncOutbox` | `wms_sync_outbox` | **오프라인 쓰기 아웃박스** |
| `tableMdGoods` | `md_goods` | 상품 마스터 |
| `tableMdGoodsLocation` | `md_goods_location` | 상품별 로케이션·재고 |
| `tableWmsStockHist` | `wms_stock_hist` | 재고 변동 이력 |
| `tableIbListHdr` | `ib_list_hdr` | 입고 헤더 (입고검수) |
| `tableIbListDtl` | `ib_list_dtl` | 입고 상세 (입고검수) |
| `tableCmCdDtl` | `cm_cd_dtl` | 공통코드 |
| `tablePutawayPending` | `putaway_pending` | **입고적재 대기 목록** |
| `tableLcMaster` | `lc_master` | **로케이션 마스터** |

`wms_app_setting`은 `setting_key`를 PK로 쓰는 KV 저장소다. 동기화 시각 키는 `SyncSettingKeys`에 상수로 정의되어 있다.

```dart
class SyncSettingKeys {
  static const ibList       = 'ib_list_sync_time';
  static const mdGoods      = 'md_goods_sync_time';
  static const mdGoodsLocation = 'md_goods_location_sync_time';
  static const wmsStockHist = 'wms_stock_hist_sync_time';
  static const cmUser       = 'cm_user_sync_time';
}
```

---

## 5. 동기화 트리거 3가지

### 트리거 A — 로그인 성공 후 (fire-and-forget)

`login_provider.dart`에서 로그인 성공 직후 3개 sync를 백그라운드로 실행한다. UI를 블락하지 않는다.

```dart
// login_provider.dart
unawaited(_ref.read(goodsRepositoryProvider).syncGoodsOnStartup());
unawaited(_ref.read(inboundRepositoryProvider).syncInboundOnStartup());
unawaited(_ref.read(putawayRepositoryProvider).syncPutawayOnStartup());
```

### 트리거 B — 네트워크 복구 시 (순서 보장)

`app.dart`에서 `wmsOnlineProvider`가 `false → true`로 전환될 때 `onNetworkRestored(ref)`를 호출한다. 순서가 중요하므로 `await`로 직렬 실행한다 (→ [9장](#9-네트워크-복구-시-재동기화-순서) 참조).

### 트리거 C — 화면 진입 시 (실시간 갱신)

각 화면의 `initState`나 `load()`에서 온라인이면 서버를 먼저 조회하고 로컬에 캐시한 뒤 반환한다.

```dart
// putaway_provider.dart
Future<void> load() async {
  final items = await _ref.read(putawayRepositoryProvider).getPutawayItems();
  // getPutawayItems(): 온라인이면 서버 조회 → 로컬 동기화 → 반환
  //                   오프라인이면 로컬 DB에서 반환
}
```

---

## 6. 피처별 동기화 전략

### 6-1. 사용자 마스터 (Auth)

| 항목 | 내용 |
|------|------|
| 전략 | 증분 동기화 (`since` 파라미터) |
| API | `GET /auth/users?since=...` |
| 결과 DTO | `UserSyncResult { users, syncedAt, isIncremental }` |
| Full sync 시 | `use_yn='N'`으로 비활성화 (물리 삭제 아님) |
| 동기화 시각 키 | `cm_user_sync_time` |
| 특이사항 | 오프라인 로그인용 비밀번호 해시도 로컬에 보관 |

**오프라인 로그인 흐름:**
```
login(id, pw)
  ├── [온라인] API 인증 → 성공 시 비밀번호 해시를 로컬에 upsert
  └── [오프라인 / NetworkException]
        ├── 로컬 해시 일치 → 로그인 성공
        └── 불일치 → OfflineNeedOnlineFirstException
              (온라인으로 1회 로그인한 이력이 없으면 오프라인 불가)
```

---

### 6-2. 공통코드 (CommonCode)

| 항목 | 내용 |
|------|------|
| 전략 | 전체 교체 (항상 full, `since` 없음) |
| API | `GET /common/codes/all` |
| 실패 시 | 로컬 캐시 반환 (예외를 삼키고 fallback) |
| 특이사항 | 네트워크 복구 시 `unawaited`로 백그라운드 실행 (비차단) |

---

### 6-3. 상품 마스터·위치·재고이력 (Goods)

| 항목 | 내용 |
|------|------|
| 전략 | 증분 동기화 (`since` 파라미터) |
| API 3개 병렬 호출 | `GET /common/goods/pda-sync` `GET /common/goods/locations` `GET /common/goods/stock-history` |
| 결과 DTO | `GoodsSyncResult`, `LocationSyncResult`, `StockHistSyncResult` |
| 동기화 시각 키 | `md_goods_sync_time`, `md_goods_location_sync_time`, `wms_stock_hist_sync_time` |

**`syncGoods()` 3단계 흐름:**

```
1단계 — 마지막 동기화 시각 3개 병렬 조회 (로컬 DB KV)
  Future.wait([
    _local.getLastSyncTime(),
    _locationLocal.getLastSyncTime(),
    _stockHistLocal.getLastSyncTime(),
  ])

2단계 — 서버 API 3개 병렬 호출 (네트워크 I/O 병목 제거)
  Future.wait([
    remote.fetchGoodsForStartupSync(since: goodsSince),
    remote.fetchGoodsLocationsForStartupSync(since: locSince),
    remote.fetchStockHistForStartupSync(since: histSince),
  ])

3단계 — 로컬 DB 저장 순차 실행 (SQLite 락 충돌 방지)
  await _local.syncGoods(...)
  await _locationLocal.syncLocations(...)
  await _stockHistLocal.syncStockHist(...)
```

바코드 개별 조회 시에는 온라인이면 서버 API를 먼저 호출하고 결과를 로컬에 `upsert`한다. API 실패 시 로컬 폴백.

---

### 6-4. 입고검수 (InboundInspection)

| 항목 | 내용 |
|------|------|
| 전략 | 증분 동기화 (`since` 파라미터) |
| API | `GET /inbound/pda/sync?centerId=...&since=...` |
| 로컬 구조 | `ib_list_hdr` (1) + `ib_list_dtl` (N) — 1:N 관계 |
| 동기화 시각 키 | `ib_list_sync_time` |
| 공통 Sembast 로직 | `IbListSembastHelper` (hdr/dtl 동시 처리) |

**Full sync 시 배치 삭제 (`IbListSembastHelper`):**

이전: 입고건 N개에 대해 dtl N번 + hdr N번 = `2N`회 IndexedDB 호출  
현재: `Filter.or([ib_no = A, ib_no = B, ...])` 1회 + hdr 필터 삭제 1회 = 항상 `2`회

```dart
// 현재 구현
final ibNos = existing.map((r) => r.value['ib_no'] as String).toList();
final dtlFilter = ibNos.length == 1
    ? Filter.equals('ib_no', ibNos.first)
    : Filter.or(ibNos.map((n) => Filter.equals('ib_no', n)).toList());
await dtlStore.delete(db, finder: Finder(filter: dtlFilter));
await hdrStore.delete(db, finder: Finder(filter: _hdrFilter(...)));
```

**검수 저장·완료 오프라인 경로:**

```
saveInspection(ibNo, items, complete)
  ├── 로컬 DB 즉시 반영 (updateItemInspection + updateHdrStatus) ← 낙관적 업데이트
  │
  ├── [온라인] PATCH /inbound/pda/{ibNo}/inspect → 성공 시 return
  │
  └── [오프라인 또는 API 실패]
        Outbox 삽입: INSP_SAVE 또는 INSP_COMPLETE
        payload: { ibNo, items: [{goodsCd, barcode, exp, insQty, defectiveQty, insSttsCd}] }
```

---

### 6-5. 입고적재 (Putaway)

| 항목 | 내용 |
|------|------|
| 전략 | 전체 교체 (Full Replacement, `since` 없음) |
| API 2개 병렬 호출 | `GET /inbound/pda/putaway/pending` `GET /inbound/pda/location/sync` |
| 특이사항 | 적재 완료된 항목은 서버에서 이미 제거되므로 전체 교체가 안전 |

**`syncPutawayOnStartup()` 흐름:**

```
1. Future.wait([fetchPutawayPending, fetchLocationsForSync])  ← 병렬
2. await syncPutawayPending(centerId, items)                  ← 순차
3. await syncLocations(centerId, locations)                   ← 순차
```

**로케이션 유효성 검증 (`validateLocation`):**

```
사용자가 로케이션 바코드 스캔
  └── [1차] 로컬 lc_master 조회
        ├── 있음 → 반환 (API 미호출)
        └── 없음 + 온라인
              GET /inbound/pda/location/{locId}
              └── 성공 → LocationInfo 반환

검증 결과
  ├── null → "로케이션을 찾을 수 없습니다"
  ├── load_yn != 'Y' → "적재 불가 로케이션입니다"
  └── 정상 → scannedLocation 상태에 저장
```

**적재 실행 오프라인 경로:**

```
executePutawayItem(stkHistId, locId)
  ├── [온라인]
  │     POST /inbound/pda/putaway/execute
  │     성공 → removePutawayPendingItem(stkHistId) 로컬 삭제
  │
  └── [오프라인]
        Outbox 삽입: PUTAWAY_ITEM
        payload: { stkHistId, locId, centerId }
        이후 → removePutawayPendingItem(stkHistId) 낙관적 삭제
```

---

### 피처별 동기화 전략 비교표

| 피처 | 전략 | 병렬 API | 오프라인 쓰기 | 동기화 시각 |
|------|------|----------|--------------|------------|
| 사용자 | 증분 | 1개 | 없음 | O |
| 공통코드 | 전체 | 1개 | 없음 | X |
| 상품 마스터·위치·이력 | 증분 | 3개 동시 | 없음 (읽기 전용) | O (3개) |
| 입고검수 | 증분 | 1개 | Outbox (INSP_SAVE/COMPLETE) | O |
| 입고적재 | 전체 교체 | 2개 동시 | Outbox (PUTAWAY_ITEM) | X |

---

## 7. 증분 동기화 공통 패턴

서버는 `since`(ISO 8601) 파라미터를 받아 해당 시각 이후 변경된 항목만 반환한다.

**SyncResult DTO 패턴:**

모든 증분 동기화 API의 응답은 동일한 구조를 가진다.

```dart
class GoodsSyncResult {
  final List<GoodsModel> goods;
  final String syncedAt;      // 서버 기준 동기화 완료 시각
  final bool isIncremental;   // true: 변경분만 / false: 전체
}
// LocationSyncResult, StockHistSyncResult, UserSyncResult 동일 구조
```

**서버 응답 파싱:**

```dart
final isIncremental =
    (body['isIncremental'] ?? body['incremental']) as bool? ?? (since != null);
```

`since`를 보냈는데 서버가 플래그를 안 보내면 `since != null`로 fallback.

**로컬 저장 시 분기:**

```dart
// isIncremental = true → upsert만 (기존 항목 유지)
// isIncremental = false → 전체 삭제 후 재삽입
await _local.syncGoods(goods, isIncremental: result.isIncremental);
await _local.setLastSyncTime(result.syncedAt);   // 다음 증분 기준점
```

---

## 8. Outbox 패턴 (오프라인 쓰기)

오프라인 중 발생한 쓰기 작업은 `wms_sync_outbox` 테이블에 PENDING 상태로 적재한다. 네트워크 복구 시 `SyncQueueService.flushPending()`이 PENDING 항목을 순차적으로 서버에 전송한다.

### 테이블 스키마

```
wms_sync_outbox
  outbox_id      TEXT PK   — UUID v4
  site_cd        TEXT      — centerId
  event_type_cd  TEXT      — 이벤트 종류 (아래 참조)
  payload_json   TEXT      — JSON 직렬화된 요청 데이터
  sync_status_cd TEXT      — PENDING → DONE / FAILED
  retry_cnt      INTEGER   — 실패 시 1씩 증가
  created_dtm    TEXT      — 생성 시각 (ISO 8601)
```

### 이벤트 타입 (`SyncModels`)

| 상수 | 값 | 발생 시점 |
|------|----|-----------|
| `inspSave` | `INSP_SAVE` | 검수 저장 (미완료) |
| `inspComplete` | `INSP_COMPLETE` | 검수 완료 |
| `putaway` | `PUTAWAY` | 입고적재 일괄 처리 |
| `putawayItem` | `PUTAWAY_ITEM` | 입고적재 단건 실행 |

### OutboxDao 플랫폼 분기

```dart
class OutboxDao {
  OutboxDao({OutboxStore? store})
    : _store = store ?? (kIsWeb ? WebOutboxStore() : SqliteOutboxStore());
}
```

### Flush 로직 (`SyncQueueService.flushPending`)

```
PENDING 항목 전체 조회
  └── for each pending:
        payload JSON 파싱
        → onDispatch(eventType, payload) 콜백 호출
              ├── INSP_SAVE / INSP_COMPLETE → PATCH /inbound/pda/{ibNo}/inspect
              ├── PUTAWAY → POST /inbound/pda/{ibNo}/putaway
              └── PUTAWAY_ITEM → POST /inbound/pda/putaway/execute
        성공 → sync_status_cd = 'DONE'
        실패 → sync_status_cd = 'FAILED', retry_cnt++
```

---

## 9. 네트워크 복구 시 재동기화 순서

`sync_on_reconnect.dart` → `onNetworkRestored(ref)`

```
1. [unawaited] 공통코드 동기화 ← 비차단, 백그라운드

2. await 상품 마스터·위치·재고이력 동기화 (syncGoodsOnStartup)
3. await 사용자 동기화 (syncUsersOnStartup)

4. await [Outbox 플러시]                     ← ★ 핵심 순서
     INSP_SAVE/COMPLETE → patchInspection
     PUTAWAY → postPutaway
     PUTAWAY_ITEM → postPutawayItem

5. await 입고검수 마스터 동기화 (syncInboundOnStartup)
6. await 입고적재 동기화 (syncPutawayOnStartup)

7. await 홈 화면 미동기화 카운트 갱신 (loadUnsyncedCount)
```

**4번 → 5·6번 순서가 필수인 이유:**

오프라인 중 검수 저장/완료를 했다면 서버 상태가 아직 구식이다. Outbox를 먼저 플러시해야 서버가 최신 상태가 되고, 그 이후 Pull해야 올바른 데이터를 받아온다. 순서가 바뀌면 서버의 stale 데이터가 로컬 오프라인 작업을 덮어쓴다.

---

## 10. 읽기 요청의 온·오프라인 분기

Repository의 읽기 메서드는 모두 동일한 패턴을 따른다.

```dart
// 예시: GoodsRepository.findByBarcode
Future<GoodsModel?> findByBarcode(String barcode) async {
  final isOnline = _ref.read(wmsOnlineProvider);

  if (isOnline && centerCode != null) {
    try {
      final remote = await _remote.fetchGoodsDetailByBarcode(...);
      if (remote == null) return null;
      await _local.upsertGoods(remote);   // 로컬 캐시 갱신
      return remote;
    } on AppException { /* 로컬 폴백 */ }
    on FormatException { /* 로컬 폴백 */ }
  }

  return _local.findByBarcode(barcode, centerCode: centerCode); // 로컬 폴백
}
```

**공통 패턴:**
- 온라인 → 서버 우선 → 성공 시 로컬 캐시 갱신 → 반환
- 오프라인 또는 API 실패 → 로컬 DB에서 반환
- `AppException`과 `FormatException`만 잡아 폴백, 나머지 예외는 상위로 전파

---

## 11. 홈 화면 미동기화 카운트 갱신

`HomeNotifier.loadUnsyncedCount()`는 두 가지를 합산해 표시한다.

```dart
Future<void> loadUnsyncedCount() async {
  final unsyncedCount = await _outboxDao.countPending();          // Outbox PENDING 건수
  final waitingInboundCount =
      await inboundLocalStore.countWaitingInbound(centerId);      // ib_stts_cd='01' tot_po_qty 합계
  state = state.copyWith(
    unsyncedCount: unsyncedCount,
    waitingInboundCount: waitingInboundCount,
  );
}
```

**갱신 시점:**
- 앱 시작 시 (`homeProvider` 생성 → `loadUnsyncedCount` 자동 호출)
- 네트워크 복구 후 (재동기화 완료 후 마지막 단계)
- 홈 화면 당겨서 새로고침 / 헤더 새로고침 버튼

로컬 DB 작업(검수 저장, 적재 실행)은 완료 즉시 `loadUnsyncedCount()`를 호출하지 않는다. 홈 화면으로 돌아올 때 provider가 재마운트되거나 갱신되면서 반영된다.

---

## 12. 핵심 구현 패턴 모음

### 패턴 1 — 병렬 API 호출 + 이기종 결과 캡처

```dart
late final GoodsSyncResult goodsResult;
late final LocationSyncResult locResult;
await Future.wait([
  _remote.fetchGoodsForStartupSync(...).then((r) => goodsResult = r),
  _remote.fetchGoodsLocationsForStartupSync(...).then((r) => locResult = r),
]);
```

반환 타입이 다른 Future들을 `Future.wait<void>` + `late final` + `.then()`으로 병렬 실행하면서 각각 캡처한다.

### 패턴 2 — SQLite 트랜잭션 내 Full Replacement

```dart
await db.transaction((txn) async {
  await txn.delete(table, where: 'center_id = ?', whereArgs: [centerId]);
  for (final item in items) {
    await txn.insert(table, item.toRow(centerId, syncedAt));
  }
});
```

삭제와 삽입을 하나의 트랜잭션으로 묶어 빈 상태가 UI에 노출되지 않도록 한다.

### 패턴 3 — Sembast 배치 삭제

```dart
// 개별 삭제 루프 대신 store.delete(finder) 단일 호출
await _pendingStore.delete(
  txn,
  finder: Finder(filter: Filter.equals('center_id', centerId)),
);
```

N건을 N번 delete하는 루프를 IndexedDB 1번 호출로 대체한다.

### 패턴 4 — Sembast Filter.or 배치 삭제 (IbListSembastHelper)

```dart
final dtlFilter = ibNos.length == 1
    ? Filter.equals('ib_no', ibNos.first)
    : Filter.or(ibNos.map((n) => Filter.equals('ib_no', n)).toList());
await dtlStore.delete(db, finder: Finder(filter: dtlFilter));
```

입고건이 N개일 때 dtl 삭제를 N번에서 1번으로 줄인다.

### 패턴 5 — fire-and-forget (unawaited)

```dart
unawaited(_ref.read(goodsRepositoryProvider).syncGoodsOnStartup());
```

로그인 직후 UI 전환을 블락하지 않고 백그라운드에서 동기화를 시작한다. 실패해도 로컬 데이터를 유지하므로 무시해도 안전하다.

### 패턴 6 — SyncResult DTO로 isIncremental 전달

```dart
// Remote에서 서버 플래그를 파싱해 DTO에 담아 반환
return GoodsSyncResult(
  goods: result,
  syncedAt: syncedAt,
  isIncremental: isIncremental,  // 로컬 저장 방식 결정에 사용
);

// Repository에서 DTO를 소비
await _local.syncGoods(goodsResult.goods, isIncremental: goodsResult.isIncremental);
```

Remote와 Local 레이어가 직접 결합되지 않고 DTO를 통해 의사소통한다.

---

## 13. 관련 파일 경로 전체 목록

```
lib/
├── app/
│   └── app.dart                                   ← 온라인 복구 감지, onNetworkRestored 호출
│
├── core/
│   ├── database/
│   │   ├── app_database.dart                      ← SQLite 초기화·마이그레이션·테이블 상수
│   │   ├── outbox_dao.dart                        ← Outbox 진입점 (플랫폼 자동 분기)
│   │   ├── sqlite_outbox_store.dart               ← SQLite Outbox
│   │   └── web_outbox_store.dart                  ← Sembast Outbox
│   ├── local/
│   │   ├── ib_list_sembast_helper.dart            ← 검수 hdr/dtl Sembast 공통 (배치 삭제)
│   │   ├── ib_list_status_scope.dart              ← 검수 상태 필터 범위 정의
│   │   └── sembast_last_sync_time_store.dart      ← wms_app_setting KV 동기화 시각 헬퍼
│   ├── network/
│   │   ├── wms_online_provider.dart               ← 온라인 상태 최종 판단
│   │   ├── network_status_provider.dart           ← 기기 네트워크 감지 (connectivity_plus)
│   │   └── server_reachability_holder.dart        ← WMS 서버 헬스체크 + 20s 폴링
│   └── sync/
│       ├── sync_on_reconnect.dart                 ← ★ 네트워크 복구 재동기화 순서 정의
│       ├── sync_queue_service.dart                ← Outbox 플러시 실행
│       ├── sync_models.dart                       ← 이벤트 타입 상수 (PUTAWAY_ITEM 등)
│       └── sync_setting_keys.dart                 ← 동기화 시각 설정키 상수
│
└── features/
    ├── auth/
    │   ├── data/
    │   │   ├── datasource/auth_local_store.dart   ← 사용자 로컬 저장소 인터페이스
    │   │   ├── datasource/auth_local_datasource.dart  ← SQLite 구현
    │   │   ├── datasource/web_auth_local_datasource.dart ← Sembast 구현
    │   │   └── repository/auth_repository.dart    ← 로그인 + 사용자 동기화
    │   └── presentation/provider/
    │       └── login_provider.dart                ← 로그인 후 sync fire-and-forget
    │
    ├── common_code/
    │   └── data/repository/common_code_repository.dart ← 공통코드 full sync
    │
    ├── product_info/
    │   └── data/
    │       ├── datasource/goods_remote_datasource.dart ← 상품 API (3개 엔드포인트)
    │       ├── datasource/goods_sync_result.dart
    │       ├── datasource/location_sync_result.dart
    │       ├── datasource/stock_hist_sync_result.dart
    │       └── repository/goods_repository.dart   ← 3단계 병렬 sync 구현
    │
    ├── receiving_inspection/
    │   └── data/
    │       ├── datasource/inbound_local_store.dart ← 검수 로컬 저장소 인터페이스
    │       ├── datasource/inbound_local_datasource.dart ← SQLite 구현
    │       ├── datasource/web_inbound_local_datasource.dart ← Sembast 구현
    │       ├── datasource/inbound_remote_datasource.dart
    │       └── repository/inbound_repository.dart ← 검수 sync + Outbox 쓰기
    │
    ├── receiving_putaway/
    │   └── data/
    │       ├── datasource/putaway_local_store.dart ← 적재 로컬 저장소 인터페이스
    │       ├── datasource/putaway_local_datasource.dart ← SQLite 구현
    │       ├── datasource/web_putaway_local_datasource.dart ← Sembast 구현
    │       ├── datasource/putaway_remote_datasource.dart
    │       └── repository/putaway_repository.dart ← 적재 sync + Outbox 쓰기
    │
    └── home/
        └── presentation/provider/
            └── home_provider.dart                 ← Outbox 건수 + 입고대기 건수 갱신
```
