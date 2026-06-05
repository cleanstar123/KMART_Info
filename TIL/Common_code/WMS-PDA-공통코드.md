# 공통코드 오프라인 동기화 구현 상세 정리

> 최초 작성일: 2026-06-05  
> 최종 수정일: 2026-06-05  
> 프로젝트: K-MART WMS PDA App  
> 브랜치: dev_wonjun

---

## 1. 기능 개요

| 항목 | 내용 |
|------|------|
| 목적 | 앱 전반에서 하드코딩된 코드값·코드명을 DB 기반 공통코드로 전환 |
| 오프라인 지원 | 앱 시작 시 전체 공통코드를 로컬 DB에 적재, 오프라인에서도 사용 가능 |
| 메모리 캐시 | 로컬 DB 재조회 없이 메모리에서 동기 접근 (`getCodeName()`, `getCodeList()`) |
| 동기화 시점 | 앱 시작 (bootstrap), Wi-Fi 복구 시 비동기 재동기화 |

---

## 2. DB 스키마

### 2-1. 백엔드 원본 테이블

#### `wms.cm_cd_hdr` (공통코드 그룹)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| `cmmn_group_cd` | varchar(20) PK | 그룹 코드 (예: `IB_STTS_CD`) |
| `cmmn_group_cd_nm_ko` | varchar(50) | 그룹명 (한국어) |
| `use_yn` | varchar(1) | 사용여부 (`Y`/`N`) |
| `seq` | int4 | 정렬순서 |

#### `wms.cm_cd_dtl` (공통코드 상세)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| `cmmn_group_cd` | varchar(20) PK | 그룹 코드 |
| `cmmn_cd` | varchar(20) PK | 코드값 (예: `01`) |
| `cmmn_cd_nm_ko` | varchar(50) | 코드명 — 한국어 |
| `cmmn_cd_nm_en` | varchar(50) | 코드명 — 영어 |
| `cmmn_cd_nm_vi` | varchar(50) | 코드명 — 베트남어 |
| `use_yn` | varchar(1) | 사용여부 |
| `seq` | int4 | 그룹 내 정렬순서 |

### 2-2. PDA 로컬 테이블 (`cm_cd_dtl`)

```sql
CREATE TABLE IF NOT EXISTS cm_cd_dtl (
  cmmn_group_cd TEXT NOT NULL,
  cmmn_cd       TEXT NOT NULL,
  cmmn_cd_nm_ko TEXT NOT NULL DEFAULT '',
  cmmn_cd_nm_en TEXT NOT NULL DEFAULT '',
  cmmn_cd_nm_vi TEXT NOT NULL DEFAULT '',
  seq           INTEGER NOT NULL DEFAULT 0,
  synced_at     TEXT NOT NULL,
  PRIMARY KEY (cmmn_group_cd, cmmn_cd)
)
```

- SQLite v12 마이그레이션으로 추가 (`AppDatabase._dbVersion = 12`)
- Sembast(웹)에서는 동일 이름의 스토어 사용

---

## 3. PDA에서 사용하는 주요 공통코드 그룹

| 그룹 코드 | 그룹명 | 코드값 예시 |
|---|---|---|
| `IB_STTS_CD` | 입고상태 | `01`=검수전, `02`=검수중, `03`=검수완료 |
| `INS_STTS_CD` | 검수상태 | `01`=검수대기, `02`=검수중, `03`=검수완료 |
| `IB_TYPE_CD` | 입고유형 | `01`=수입, `02`=현지매입, `03`=반품 |
| `KEEP_TYPE_CD` | 보관유형 | `01`=냉장, `02`=냉동, `03`=상온 |
| `LOC_TYPE_CD` | 로케이션유형 | `Z`=Zone, `R`=Rack, `L`=Line, `C`=Cell |
| `STOCK_STTS_CD` | 재고상태 | `01`=정상, `02`=주의, `03`=경고, `04`=위험 |
| `MOV_HIST_CD` | 이동이력구분 | `01`=적치, `02`=위치이동, `03`=폐기, `04`=재고조정 |
| `GOODS_UNIT` | 상품단위 | `01`=BX, `02`=CS, `03`=EA |
| `STK_HIST_CD` | 재고이력구분 | ⚠️ TODO: 명세(I/O/R/D/S/L) vs 실제코드('11'/'31') 불일치 확인 필요 |

---

## 4. 백엔드 구현

### 4-1. PDA 전용 엔드포인트

```
GET /common/pda/common-codes
```

- 인증 필요 (JWT)
- 응답 형식: `GoodsSyncResponse<CommonCodePdaItem>`

```json
{
  "items": [
    {
      "groupCd": "IB_STTS_CD",
      "code": "01",
      "codeNameKo": "검수전",
      "codeNameEn": "Waiting",
      "codeNameVi": "Chờ kiểm tra",
      "seq": 1
    }
  ],
  "syncedAt": "2026-06-05T10:00:00",
  "isIncremental": false
}
```

### 4-2. SQL (`CommonMapper.xml`)

```xml
<select id="selectCommonCodesForPda" resultType="...CommonCodePdaItem">
    SELECT
        d.CMMN_GROUP_CD AS groupCd,
        d.CMMN_CD       AS code,
        d.CMMN_CD_NM_KO AS codeNameKo,
        d.CMMN_CD_NM_EN AS codeNameEn,
        d.CMMN_CD_NM_VI AS codeNameVi,
        d.SEQ           AS seq
    FROM WMS.CM_CD_DTL d
    JOIN WMS.CM_CD_HDR h ON h.CMMN_GROUP_CD = d.CMMN_GROUP_CD
    WHERE d.USE_YN = 'Y'
      AND h.USE_YN = 'Y'
    ORDER BY d.CMMN_GROUP_CD, d.SEQ ASC
</select>
```

> `cm_cd_hdr`를 JOIN하는 이유: 그룹 자체가 비활성(`use_yn='N'`)인 경우에도 코드를 제외하기 위함.

### 4-3. 관련 파일

| 파일 | 역할 |
|------|------|
| `common/dto/CommonCodePdaItem.java` | 응답 DTO |
| `common/mapper/CommonMapper.java` | `selectCommonCodesForPda()` 인터페이스 |
| `common/mapper/CommonMapper.xml` | SQL 쿼리 |
| `common/controller/CommonController.java` | `GET /common/pda/common-codes` 엔드포인트 |

---

## 5. 프론트엔드 구현

### 5-1. 아키텍처 레이어 구조

```
lib/features/common_code/
├── domain/
│   └── model/
│       └── common_code_item.dart         ← 도메인 모델
├── data/
│   └── datasource/
│       ├── common_code_local_store.dart  ← 추상 인터페이스
│       ├── common_code_local_datasource.dart      ← SQLite 구현
│       ├── web_common_code_local_datasource.dart  ← Sembast 구현
│       └── common_code_remote_datasource.dart     ← API 호출
│   └── repository/
│       └── common_code_repository.dart   ← sync + fallback 로직
└── presentation/
    └── provider/
        └── common_code_provider.dart     ← Riverpod StateNotifier + 메모리 캐시

lib/core/constants/
└── common_code_constants.dart            ← 코드값 상수 클래스
```

### 5-2. 도메인 모델 (`CommonCodeItem`)

```dart
class CommonCodeItem {
  final String groupCd;
  final String code;
  final String codeNameKo;
  final String codeNameEn;
  final String codeNameVi;
  final int seq;
}
```

- `fromJson()`: API 응답 파싱 (camelCase 키)
- `fromMap()`: 로컬 DB 파싱 (snake_case 키)
- `toMap()`: 로컬 DB 저장용 변환

### 5-3. 로컬 스토어 추상화

```dart
abstract interface class CommonCodeLocalStore {
  Future<void> syncCodes(List<CommonCodeItem> items);
  Future<List<CommonCodeItem>> getAllCodes();
}
```

- **SQLite** (`CommonCodeLocalDatasource`): 트랜잭션으로 전체 삭제 후 재삽입 (full replace)
- **Sembast** (`WebCommonCodeLocalDatasource`): 트랜잭션으로 전체 삭제 후 재삽입

### 5-4. Repository 동기화 흐름

```dart
Future<List<CommonCodeItem>> syncOnStartup() async {
  try {
    final items = await _remote.fetchAllCodes();  // API 호출
    if (items.isNotEmpty) {
      await _local.syncCodes(items);              // 로컬 DB 저장
    }
    return items;
  } catch (_) {
    return _local.getAllCodes();                  // 오프라인 → 로컬 DB fallback
  }
}
```

### 5-5. Provider (`CommonCodeState`) — 메모리 캐시

```dart
class CommonCodeState {
  final Map<String, List<CommonCodeItem>> _byGroup;  // 그룹별 인덱싱

  // 코드명 동기 조회 (메모리)
  String getCodeName(String groupCd, String code, {String? fallback}) { ... }

  // 그룹 전체 목록 조회 (seq 오름차순)
  List<CommonCodeItem> getCodeList(String groupCd) { ... }
}
```

위젯에서 사용:
```dart
final commonCode = ref.watch(commonCodeProvider);

// 코드명 표시
Text(commonCode.getCodeName('IB_STTS_CD', container.ibSttsCd))

// 드롭다운 목록
commonCode.getCodeList('KEEP_TYPE_CD')
```

### 5-6. 앱 시작 순서 (`main.dart`)

```
i18nProvider.initialize()           ← 다국어 메시지 로드
    ↓
commonCodeBootstrapProvider         ← 공통코드 로드 (API → 로컬 DB → 메모리)
    ↓
authBootstrapProvider               ← 사용자 동기화
    ↓
goodsBootstrapProvider              ← 상품 마스터 동기화
    ↓
inboundBootstrapProvider            ← 입고 데이터 동기화
    ↓
serverReachableProvider.check()     ← 네트워크 상태 확인
```

> 공통코드를 i18n 직후 로드하는 이유: 이후 bootstrap 과정에서 표시명이 필요할 수 있기 때문.

### 5-7. Wi-Fi 복구 시 재동기화 (`sync_on_reconnect.dart`)

```dart
// 공통코드는 메인 흐름을 블로킹하지 않도록 unawaited 처리
unawaited(
  ref.read(commonCodeRepositoryProvider).syncOnStartup().then((items) {
    ref.read(commonCodeProvider.notifier).load(items);
  }),
);
// 이후 상품, 사용자, 아웃박스, 입고 동기화 순서대로 진행
```

---

## 6. 코드값 상수 관리 (`common_code_constants.dart`)

```dart
abstract final class IbSttsCd {
  static const String waiting    = '01';  // 검수전
  static const String inspecting = '02';  // 검수중
  static const String completed  = '03';  // 검수완료
}

abstract final class InsSttsCd {
  static const String waiting    = '01';
  static const String inspecting = '02';
  static const String completed  = '03';
}

abstract final class KeepTypeCd {
  static const String refrigerated = '01';  // 냉장
  static const String frozen       = '02';  // 냉동
  static const String ambient      = '03';  // 상온
}

// 그 외: IbTypeCd, LocTypeCd, StockSttsCd, MovHistCd, GoodsUnit
```

**상수 vs 동적 조회 사용 기준:**

| 용도 | 방법 | 이유 |
|------|------|------|
| 코드값 비교 (`if`, `switch`, SQL 필터) | **상수** | `cmmn_cd`는 PK — 절대 변경 불가 |
| 화면 표시 텍스트 | **`getCodeName()`** | 표시명은 관리자가 변경 가능 |
| 드롭다운 목록 | **`getCodeList()`** | 코드 추가·삭제 반영 |

---

## 7. 하드코딩 제거 현황

### 프론트엔드 — 변경 완료

| 파일 | 변경 전 | 변경 후 |
|------|------|------|
| `ib_list_status_scope.dart` | `['01', '02']` | `[IbSttsCd.waiting, IbSttsCd.inspecting]` |
| `ib_list_sembast_helper.dart` | `contains('01')` | `contains(IbSttsCd.waiting)` |
| `web_inbound_local_datasource.dart` | `'01'`, `'02'` 직접 비교 | `IbSttsCd.*` 상수 |
| `inbound_repository.dart` | `complete ? '03' : '02'` | `InsSttsCd.completed / .inspecting` |
| `receiving_inspection_detail_screen.dart` | `complete ? '03' : '02'` | `InsSttsCd.completed / .inspecting` |
| `inspection_container.dart` | `?? '01'` 기본값 | `?? IbSttsCd.waiting` |
| `product_info_card.dart` | 한국어 `'상온'`, `'냉장'` 색상 매핑 | `KeepTypeCd.*` 코드값 기반 색상 + `getCodeName()` |
| `goods_model.dart` | `_storageTypeText()` 하드코딩 | `keepTypeCd` raw 코드 전달 |

### 프론트엔드 — TODO 처리

| 파일 | 내용 | 사유 |
|------|------|------|
| `stock_hist_item.dart` | `stkHistCd == '11'` | DB 명세(`I/O/R/D/S/L`)와 실제 코드(`'11'/'31'`) 불일치 확인 필요 |

### 백엔드 — 현재 상태

PDA 관련 파일에서 하드코딩 발견:

| 파일 | 내용 | 처리 방침 |
|------|------|------|
| `InboundMapper.xml:709` | `ib_stts_cd = '03'` (UPDATE) | 웹과 공유 로직 — 현재 유지 |
| `InboundMapper.xml:729-731` | `'01'/'02'/'03'` CASE WHEN | 웹과 공유 로직 — 현재 유지 |
| `InboundMapper.xml:245` | `ib_type_cd = '03'` (반품 판단) | 웹과 공유 로직 — 현재 유지 |

> 백엔드 하드코딩은 웹 프론트와 공유되는 비즈니스 로직이므로 별도 작업으로 분리.

---

## 8. 코드값 변경과 동기화에 관한 설계 원칙

### 관리자가 변경 가능한 항목과 영향

| 변경 유형 | 오프라인 PDA 영향 | 처리 |
|---|---|---|
| 표시명 변경 (`cmmn_cd_nm_ko`) | 재연결 전까지 이전 표시명 노출 | ✅ 재연결 시 자동 동기화 |
| 코드 비활성화 (`use_yn='N'`) | 오프라인 중 해당 코드 계속 사용 가능 | ✅ 서버 수신 후 정상 처리됨 |
| 새 코드값 추가 | 앱이 새 코드 인식 못함 | ✅ 앱 업데이트 + 재배포로 처리 |
| **코드값 변경** (`'01'`→`'10'`) | **시스템 전체 장애** | ❌ 운영 규칙으로 금지 |

### 운영 규칙

> `cmmn_cd` (코드값)은 최초 등록 후 **변경 금지**.  
> 표시명(`cmmn_cd_nm_*`)과 `use_yn`만 변경 가능.  
> 새 코드가 필요하면 기존 코드를 비활성화하고 새 코드를 추가한다.

---

## 9. 관련 파일 목록

### 백엔드

```
wms_backend/src/main/java/com/kmarket/wms/common/
├── dto/CommonCodePdaItem.java
├── mapper/CommonMapper.java
└── controller/CommonController.java

wms_backend/src/main/resources/mapper/common/
└── CommonMapper.xml
```

### 프론트엔드

```
lib/core/constants/
└── common_code_constants.dart

lib/core/database/
└── app_database.dart              (v12, tableCmCdDtl 추가)

lib/core/local/
├── ib_list_status_scope.dart      (IbSttsCd 상수 적용)
└── ib_list_sembast_helper.dart    (IbSttsCd 상수 적용)

lib/core/sync/
└── sync_on_reconnect.dart         (공통코드 재동기화 추가)

lib/main.dart                      (commonCodeBootstrapProvider 추가)

lib/features/common_code/
├── domain/model/common_code_item.dart
├── data/datasource/
│   ├── common_code_local_store.dart
│   ├── common_code_local_datasource.dart
│   ├── web_common_code_local_datasource.dart
│   └── common_code_remote_datasource.dart
├── data/repository/common_code_repository.dart
└── presentation/provider/common_code_provider.dart
```
