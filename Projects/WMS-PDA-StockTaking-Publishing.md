# WMS PDA — 재고조사 화면 퍼블리싱 정리

#project #kmarket #wms #pda #stock-taking #publishing

> **상태**: UI 퍼블리싱 완료 (API·Provider 미연동)  
> **코드 경로**: `wms_pda_app/lib/features/stock_taking/`  
> **관련 문서**: [WMS-PDA-UI-Component-Spec.md](WMS-PDA-UI.md), [WMS-PDA-UX-Design-Spec.md](WMS-PDA-UX.md)

---

## 1. 작업 범위 요약

| 구분             | 완료      | 미완료        |
| -------------- | ------- | ---------- |
| 지시 목록 UI       | ✅       | API 연동     |
| 로케이션 스캔 UI     | ✅       | 실제 스캐너 SDK |
| 예상 재고·수량 입력 UI | ✅       | 저장 API     |
| 차이(오류 수량) 표시   | ✅       | —          |
| 저장·완료 Dialog   | ✅ (UI만) | 서버 반영      |
| 다국어 (ko/en/vi) | ✅       | —          |
| 더미 데이터 화면 이동   | ✅       | —          |
| 홈 메뉴·라우터 연결    | ✅       | —          |

---

## 2. 디렉터리 구조

```
lib/features/stock_taking/
├── data/
│   └── mock_stock_taking_data.dart      # 더미 지시·헬퍼·데모 수량
├── domain/model/
│   ├── stock_taking_instruction.dart    # 조사 지시
│   ├── stock_taking_location.dart       # 로케이션 + 라인 목록
│   └── stock_taking_line.dart           # 상품 라인 (systemQty)
└── presentation/
    ├── style/
    │   └── stock_taking_styles.dart     # accent=dangerRed, CommonStyles 매핑
    ├── screen/
    │   ├── stock_taking_list_screen.dart
    │   └── stock_taking_detail_screen.dart
    └── widget/
        ├── stock_taking_instruction_card.dart
        ├── stock_taking_location_scan_section.dart
        └── stock_taking_location_chips.dart   # 데모·로케이션 빠른 전환
```

### 2.1 Shared로 승격한 컴포넌트 (재고조사 작업 중 추가)

| 파일 | 재고조사에서의 역할 |
|------|---------------------|
| `shared/widget/wms_progress_card.dart` | 목록 상단 조사 진행률 |
| `shared/widget/wms_qty_input_box.dart` | 실물 수량 입력 |
| `shared/widget/wms_stock_count_line_card.dart` | 품목별 시스템/실물/차이 카드 |

기존 shared 재사용: `PdaPageScaffold`, `WmsListHeader`, `WmsEmptyState`, `WmsScanSection`, `ScanRow`, `WmsItemIconBox`, `AppFooter`

---

## 3. 화면별 구현 요약

### 3.1 지시 목록 (`StockTakingListScreen`)

- **경로**: `/stock_taking` (`app_router.dart`)
- **구성**: ProgressCard → ListHeader(미완료 N) → InstructionCard × 3 (mock)
- **인터랙션**: 카드 탭 → 상세 / FAB **데모 시작** → 첫 지시+로케이션 프리로드 상세

### 3.2 지시 상세 (`StockTakingDetailScreen`)

- **구성**: LocationScan → (Chips) → 품목 카드 or Empty
- **하단 바**: 저장 | 로케이션 완료 (수량 전부 입력 시 활성)
- **FAB**: 로케이션 미선택 시 데모 로케이션 로드

---

## 4. 더미 데이터·데모 가이드

### 4.1 상수

```dart
stockTakingDemoInstructionId = 'ST-2024-0515-001'
stockTakingDemoLocationCode    = 'A-A01-01-01'
```

### 4.2 mock 지시 (3건)

| ID | 구역 | 로케이션 수 |
|----|------|-------------|
| ST-2024-0515-001 | A존 냉장 | 2 (`A-A01-01-01`, `A-A01-02-01`) |
| ST-2024-0515-002 | B존 상온 | 1 (`B-B02-01-01`) |
| ST-2024-0514-003 | C존 냉동 | 2 |

### 4.3 데모 수량 (의도적 차이)

| 로케이션 | SKU | 시스템 | 데모 실물 | 비고 |
|----------|-----|--------|-----------|------|
| A-A01-01-01 | SKU-BN-001 | 120 | 120 | 일치 |
| A-A01-01-01 | SKU-MG-014 | 85 | 80 | 오류 -5 |
| A-A01-02-01 | SKU-DR-210 | 45 | 50 | 오류 +5 |
| 기타 | — | N | N | 일치 |

### 4.4 QA 시나리오

1. 홈 → 재고조사 → **데모 시작**
2. 망고 젤리 오류 수량·경고 문구 확인
3. **로케이션 완료** → 목록 진행률 1/5 증가 확인
4. 칩 `A-A01-02-01` 선택 → 3품목 카드 확인
5. 스캔 필드에 `B-B02-01-01` 입력 후 Enter

---

## 5. i18n 키 목록 (`stock_taking.*`)

| 키 | 용도 |
|----|------|
| `stock_taking.title` | 앱바 |
| `stock_taking.progress.title` | 진행 카드 |
| `stock_taking.list.header` / `list.pending` | 목록 헤더 |
| `stock_taking.instruction.locations` / `counted` / `products` | 카드 배지 |
| `stock_taking.scan.location.*` | 스캔 섹션 |
| `stock_taking.items.header` | 품목 목록 헤더 |
| `stock_taking.line.*` | 수량 라벨·차이·일치 |
| `stock_taking.action.*` | 저장·완료·데모 |
| `stock_taking.dialog.*` | 확인 Dialog |
| `stock_taking.empty.*` | EmptyState |
| `stock_taking.error.location_not_found` | 잘못된 바코드 |
| `stock_taking.info.demo_loaded` | 데모 SnackBar |
| `stock_taking.message.variance_exists` | 하단 경고 |

---

## 6. 다음 개발 단계 (기능 연동 시)

1. `stock_taking_provider` + Repository (지시 목록 API)
2. 로케이션 스캔 → API 검증 / 로컬 캐시
3. 저장·완료 → Outbox + Sync
4. `InspectionItemCard`와 `WmsQtyInputBox` 통합 리팩터 (선택)
5. 서버 i18n에 `stock_taking.*` 한국어 등록 (앱 번들과 동기)

---

## 7. 변경 이력

| 일자 | 작성 |
|------|------|
| 2026-05-20 | 재고조사 퍼블리싱·shared 추출·데모 UX·문서화 |
