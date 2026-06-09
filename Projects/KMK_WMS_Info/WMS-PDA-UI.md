# WMS PDA — UI 컴포넌트 정의서 (Shared)

#project #kmarket #wms #pda #ui #flutter

> **대상 코드**: `wms_pda_app/lib/shared/`  
> **관련 문서**: [WMS-PDA-UX.md](WMS-PDA-UX.md), [WMS-PDA-StockTaking-Publishing.md](./WMS-PDA-StockTaking-Publishing.md)  
> **기준**: K-Market WMS PDA UI 가이드 (`AppTheme` = `K_market_system_UI_guide_WMS_시안.pdf` 반영)

---

## 1. 레이어 구조

PDA 앱 UI는 **3단 토큰 + 공통 위젯 + 기능별 스타일/위젯**으로 분리합니다.

```
lib/
├── app/theme/app_theme.dart          # 색상·그림자·시맨틱 컬러 (디자인 토큰)
├── shared/
│   ├── style/common_styles.dart      # 간격·라운드·타이포 (레이아웃 토큰)
│   └── widget/*.dart                 # 재사용 UI 컴포넌트 (본 문서)
└── features/{feature}/
    ├── presentation/style/           # 기능 액센트 색·간격 오버라이드
    ├── presentation/widget/          # 기능 전용 조합 위젯 (ScanSection 등)
    ├── presentation/screen/          # 화면 조립
    └── data/mock_*_data.dart         # 퍼블리싱·데모용 더미 데이터
```

### 1.1 분리 원칙

| 구분           | 위치                               | 책임                       |
| ------------ | -------------------------------- | ------------------------ |
| **디자인 토큰**   | `AppTheme`, `CommonStyles`       | 색·간격·폰트 크기 등 변경 시 한곳 수정  |
| **공통 컴포넌트**  | `shared/widget`                  | 2개 이상 기능에서 쓰이는 UI        |
| **기능 조합 위젯** | `features/*/presentation/widget` | 스캔 섹션·카드 등 도메인 문맥이 있는 조립 |
| **기능 스타일**   | `features/*/presentation/style`  | 액센트 컬러·기능별 padding 매핑    |

---

## 2. 디자인 토큰 요약

### 2.1 색상 (`AppTheme`)

| 토큰 | HEX | 용도 |
|------|-----|------|
| `primaryGreen` | `#006B3F` | 브랜드 메인, 입고검수·상품조회 액센트 |
| `successGreen` | `#4CAF50` | 완료·일치 상태 |
| `warningOrange` | `#FF9800` | 경고·피킹 액센트 |
| `errorRed` | `#F44336` | 오류·재고조사 액센트 |
| `infoBlue` | `#2196F3` | 입고적재 액센트 |
| `gray100` | `#F5F5F5` | 페이지 배경 |
| `gray300` | `#E0E0E0` | 테두리·비활성 |
| `gray600` | `#757575` | 보조 텍스트 |
| `gray900` | `#212121` | 본문·타이틀 |
| `errorBackground` | rgba(244,67,54,0.08) | 차이·오류 배지 배경 |
| `primaryBg` | rgba(0,107,63,0.05) | 아이콘 박스·배지 배경 |

### 2.2 레이아웃·타이포 (`CommonStyles`)

| 토큰 | 값 | 용도 |
|------|-----|------|
| `screenPaddingBody` | 16,18,16,28 | 목록·상세 본문 패딩 |
| `sectionCardPadding20` | all 20 | 스캔 섹션 카드 내부 |
| `itemCardPadding16` | all 16 | 아이템·진행 카드 |
| `bottomActionPadding` | 16,12,16,16 | 하단 저장/완료 바 |
| `cardRadius16` | 16 | 섹션·진행 카드 |
| `itemCardRadius14` | 14 | 라인·지시 카드 |
| `gap8` ~ `gap16` | 8~16 | 섹션 간격 |
| `iconBoxSize48` | 48 | 상품/지시 아이콘 박스 |
| `appBarTitle` | 16/w600 | 앱바 제목 |
| `itemName15Bold` | 15/w700 | 상품명·지시 제목 |
| `listHeader` | 14/w700 gray600 | 리스트 섹션 헤더 |

---

## 3. 공통 컴포넌트 카탈로그

### 3.1 페이지 프레임

#### `PdaPageScaffold`

| 항목           | 내용                                                                                                                      |
| ------------ | ----------------------------------------------------------------------------------------------------------------------- |
| **파일**       | `shared/widget/pda_page_scaffold.dart`                                                                                  |
| **역할**       | 모든 작업 화면의 공통 Shell (AppBar + Body + FAB + BottomBar)                                                                    |
| **주요 Props** | `title`, `body`, `backgroundColor`, `appBarBackgroundColor`, `onRefresh`, `floatingActionButton`, `bottomNavigationBar` |
| **사용 기능**    | 피킹, 입고검수, 재고조사, 로케이션이동, 상품조회, 설정 등                                                                                      |

**패턴**: 기능별 `*Styles.accent`를 `appBarBackgroundColor`에 전달.

---

#### `AppFooter`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/app_footer.dart` |
| **역할** | 앱 버전·빌드 정보 하단 표시 |
| **사용** | 홈, 입고검수 목록, 재고조사 목록 등 |

---

### 3.2 리스트·진행·상태

#### `WmsProgressCard` ★ 재고조사 퍼블리싱 시 추출

| 항목           | 내용                                              |
| ------------ | ----------------------------------------------- |
| **파일**       | `shared/widget/wms_progress_card.dart`          |
| **역할**       | 작업 진행률 요약 (라벨 + completed/total + ProgressBar)  |
| **주요 Props** | `label`, `completed`, `total`, `accentColor`    |
| **시각**       | 흰 카드, 6px 라운드 프로그레스 바, 완료 수 accent 색 강조         |
| **사용**       | 재고조사 목록 (`stock_taking_list_screen`)            |
| **유사 구현**    | 입고검수 `_buildProgressCard` (기능 내 인라인 → 추후 통합 가능) |

---

#### `WmsListHeader`

| 항목           | 내용                                                                         |
| ------------ | -------------------------------------------------------------------------- |
| **파일**       | `shared/widget/wms_list_header.dart`                                       |
| **역목**       | 섹션 제목 + 선택적 카운트 배지                                                         |
| **주요 Props** | `title`, `count`, `countLabel`, `alignEnd`, `badgeColor`, `badgeTextColor` |
| **사용**       | 재고조사, 입고검수, 로케이션이동, 입고적재, 설정                                               |

---

#### `WmsEmptyState`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_empty_state.dart` |
| **역할** | 스캔 전·데이터 없음 안내 |
| **주요 Props** | `message`, `icon`, `color`, `iconSize`, `padding` |
| **사용** | 피킹, 상품조회, 재고조사(로케이션 미스캔·품목 없음) |

---

#### `WmsSummaryBar`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_summary_bar.dart` |
| **역할** | 하단 고정 — 진행 카운터 + 프로그레스 + 완료 FilledButton |
| **사용** | 입고적재, 로케이션이동 |

---

#### `WmsStatusBanner`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_status_banner.dart` |
| **역할** | 오류·오프라인 등 상단/인라인 알림 배너 |
| **사용** | 로그인 |

---

### 3.3 바코드 스캔 계열

컴포넌트 계층:

```
WmsScanSection / WmsDualScanSection
  └── ScanRow
        ├── BarcodeScanField  (미스캔)
        └── ScannedBarcodeTag (스캔 완료)
```

#### `WmsScanSection`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_scan_section.dart` |
| **역할** | 단일 스캔 블록 카드 (아이콘+제목+child+footer) |
| **사용** | 피킹, 상품조회, **재고조사 로케이션 스캔** (`StockTakingLocationScanSection`) |

#### `WmsDualScanSection`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_dual_scan_section.dart` |
| **역할** | 2단 스캔 + 구분 화살표 + 실행 버튼 + 도움말 |
| **사용** | 로케이션이동, 입고적재 |

#### `ScanRow`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/scan_row.dart` |
| **역할** | 라벨 + (입력 필드 \| 스캔 완료 태그) |
| **주요 Props** | `controller`, `scannedText`, `subText`, `accentColor`, `onSubmit`, `onClear` |

#### `BarcodeScanField`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/barcode_scan_field.dart` |
| **역할** | PDA 바코드 입력·Enter 제출 |

#### `ScannedBarcodeTag`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/scanned_barcode_tag.dart` |
| **역할** | 스캔 결과 칩 + 클리어 |

---

### 3.4 상품·로케이션 카드

#### `WmsItemIconBox`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_item_icon_box.dart` |
| **역할** | 48×48 라운드 아이콘 영역 |
| **사용** | 입고검수 카드, 재고조사 지시/라인 카드 |

#### `WmsItemMetaRow` / `WmsLocationBadge` / `WmsLocationFlowRow`

| 파일 | 용도 |
|------|------|
| `wms_item_meta_row.dart` | 아이콘+텍스트 메타 한 줄 |
| `wms_location_badge.dart` | 출발/목적 로케이션 뱃지 |
| `wms_location_flow_row.dart` | From → To 흐름 표시 |

#### `SurfaceCard`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/surface_card.dart` |
| **역할** | 범용 흰색 카드 컨테이너 |

---

### 3.5 수량·재고조사 전용 ★ 신규

#### `WmsQtyInputBox`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_qty_input_box.dart` |
| **역할** | 라벨 + 숫자 중앙 정렬 입력 (회색 박스 + 흰 필드) |
| **주요 Props** | `label`, `controller`, `accentColor`, `onChanged`, `readOnly` |
| **시각** | 값 22px / w800, accent 색 적용 |
| **재사용** | `WmsStockCountLineCard`, (향후 입고검수 `InspectionItemCard` 통합 가능) |

#### `WmsStockCountLineCard`

| 항목 | 내용 |
|------|------|
| **파일** | `shared/widget/wms_stock_count_line_card.dart` |
| **역할** | 재고조사 라인 카드 — 상품 정보 + 시스템/실물 수량 + 차이 표시 |
| **상태별 UI** | |
| | **미입력** — 좌측 border gray300 |
| | **일치** — border successGreen, 체크 아이콘, matchedHelper |
| | **차이** — border errorRed, 배지·오류 문구 errorRed |
| **주요 Props** | `productName`, `sku`, `systemQty`, `actualQtyController`, i18n 라벨 4종, `accentColor` |
| **내부** | `_ReadOnlyQtyBox`(시스템 수량) + `WmsQtyInputBox`(실물 수량) |

---

### 3.6 입력·기타

| 컴포넌트 | 파일 | 용도 |
|----------|------|------|
| `WmsTextInputField` | `wms_text_input_field.dart` | 로그인 ID/PW |
| `WmsCircleIconButton` | `wms_circle_icon_button.dart` | 홈 헤더 아이콘 버튼 |

---

## 4. 기능별 액센트 컬러 매핑

홈 `MenuGrid`와 각 `*Styles.accent` 기준:

| 기능 | routeKey | 액센트 | AppBar / Progress |
|------|----------|--------|-------------------|
| 상품정보조회 | `product_info` | `primaryGreen` | 녹색 |
| 상품입고검수 | `receiving_inspection` | `primaryGreen` / success | 녹색 |
| 입고적재 | `receiving_putaway` | `infoBlue` | 파랑 |
| 상품피킹 | `picking` | `warningOrange` (Amber) | 앰버 |
| 로케이션이동 | `location_move` | Purple (`HomeStyles.menuPurple`) | 보라 |
| **재고조사** | `stock_taking` | **`dangerRed`** | 빨강 |

> 재고조사만 메뉴·앱바·FAB·칩 강조색이 **빨강(error 계열)** 으로 통일되어 다른 작업 화면과 시각적으로 구분됩니다.

---

## 5. 기능 전용 위젯 (shared 아님) — 재고조사 예시

| 위젯 | 경로 | shared 의존 |
|------|------|----------------|
| `StockTakingInstructionCard` | `features/stock_taking/.../instruction_card.dart` | `WmsItemIconBox` |
| `StockTakingLocationScanSection` | `.../location_scan_section.dart` | `WmsScanSection`, `ScanRow` |
| `StockTakingLocationChips` | `.../location_chips.dart` | Material `ActionChip` (데모·빠른 전환) |

---

## 6. 신규 화면 추가 시 체크리스트

1. `AppTheme` / `CommonStyles` 토큰 우선 사용 (하드코딩 최소화)
2. 2회 이상 쓰일 UI는 `shared/widget`로 승격 검토
3. `PdaPageScaffold` + 기능 `*Styles` 생성
4. 바코드 UX는 `WmsScanSection` + `ScanRow` 조합
5. 수량 비교 UX는 `WmsStockCountLineCard` 재사용 검토
6. i18n 키 `assets/i18n/{ko,en,vi}.json` 3종 동시 추가
7. 퍼블리싱 단계: `data/mock_*_data.dart` + FAB **데모 시작**

---

## 7. 변경 이력

| 일자 | 내용 |
|------|------|
| 2026-05-20 | 재고조사 퍼블리싱 — `WmsProgressCard`, `WmsQtyInputBox`, `WmsStockCountLineCard` shared 추가 |
