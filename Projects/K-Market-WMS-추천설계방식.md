# K-Market WMS 추천 설계 방식

#kmarket #wms #recommendation #vietnam

---

## 프로젝트 개요 요약

| 항목 | 내용 |
|------|------|
| **프로젝트명** | K-Market WMS (Warehouse Management System) |
| **고객사** | K-Market Vietnam |
| **비즈니스** | 베트남 최대 한국식품 유통기업 |
| **규모** | 100개 점포 (하노이 65 + 호치민 35) |
| **특수 요구사항** | 온도대 관리, FEFO, 다국어 지원, 수입 물류 |

---

# 추천 설계 방식 총괄

## 핵심 추천 사항

```
┌─────────────────────────────────────────────────────────────────────┐
│                   K-Market WMS 추천 설계 방향                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 아키텍처: 모놀리식 (Modular Monolith)                              │
│     └── 규모 대비 운영 효율성, 빠른 개발 속도                            │
│                                                                     │
│  2. 기술 스택: Spring Boot + React + Flutter                         │
│     └── 국내 인력 수급 용이, 안정성, 생태계                              │
│                                                                     │
│  3. 개발 방식: Agile (2주 Sprint)                                    │
│     └── 화면 기획 완료 상태 → 빠른 피드백 루프                           │
│                                                                     │
│  4. 우선순위: 마스터 → 입고 → 재고 → 출고 → PDA                         │
│     └── 데이터 의존성 기반 순차 개발                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 1단계: 시스템 분석과 설계

> **기간**: 4~6주 (화면 기획 완료 상태 기준)
> **목표**: 요구사항 확정, 아키텍처 설계, 개발 준비

---

## 1.1 현재 상태 분석

### 이미 완료된 사항 (AS-IS)

| 항목 | 상태 | 비고 |
|------|:----:|------|
| 화면 기획서 | ✅ 완료 | 39개 화면 |
| 기능 명세서 | ✅ 완료 | 4대 핵심 프로세스 정의 |
| DB 설계 (초안) | ✅ 완료 | 35개 테이블 |
| 디자인 가이드 | ✅ 완료 | UI/UX 원칙 정의 |
| 프로세스 플로우 | ✅ 완료 | 입고/출고/발주/점포발주 |

### 추가 필요 사항 (TO-DO)

| 항목 | 우선순위 | 담당 |
|------|:--------:|------|
| 물리 ERD 상세화 | 높음 | DBA |
| API 설계서 작성 | 높음 | PL |
| 코딩 표준 정의 | 중간 | PL |
| 인프라 설계 | 중간 | 인프라 |
| 테스트 계획서 | 중간 | QA |

---

## 1.2 추천 팀 구성

### 최소 팀 구성 (6명)

```
┌─────────────────────────────────────────────────────────────────┐
│                      K-Market WMS 개발팀                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐                                                │
│  │  PM/PL (1)  │ ← 프로젝트 관리 + 기술 리드 겸임                   │
│  └──────┬──────┘                                                │
│         │                                                       │
│  ┌──────┴──────────────────────────────────────────┐           │
│  │                                                  │           │
│  ▼                                                  ▼           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Backend (2) │  │Frontend (1) │  │  PDA (1)    │             │
│  │ - API 개발   │  │ - Web 화면  │  │ - Flutter   │             │
│  │ - 비즈니스   │  │ - 39개 화면 │  │ - 7개 화면  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐                                                │
│  │  DBA (0.5)  │ ← 파트타임 또는 Backend 겸임                     │
│  └─────────────┘                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 권장 팀 구성 (8~10명)

| 역할         |  인원   | 담당 업무                 |
| ---------- | :---: | --------------------- |
| PM         |   1   | 프로젝트 관리, 고객 커뮤니케이션    |
| PL/아키텍트    |   1   | 기술 리드, 아키텍처, 코드 리뷰    |
| Backend    |   3   | API 개발, 비즈니스 로직       |
| Frontend   |   2   | Web 화면 개발 (39개)       |
| Mobile/PDA |   1   | Flutter PDA 앱 (7개 화면) |
| DBA        |  0.5  | DB 설계, 쿼리 최적화         |
| QA         | 0.5~1 | 테스트 계획, 수행            |

---

## 1.3 추천 기술 스택

### Backend (Java + Spring Boot)

| 카테고리           | 기술                 | 버전       | 선정 이유            |
| -------------- | ------------------ | -------- | ---------------- |
| **Language**   | Java               | 17 (LTS) | 안정성, 인력 수급 용이    |
| **Framework**  | Spring Boot        | 3.2+     | 국내 표준, 풍부한 생태계   |
| **ORM**        | Spring Data JPA    | -        | 생산성, 유지보수성       |
| **Query**      | QueryDSL           | 5.0+     | 동적 쿼리, 타입 안정성    |
| **Security**   | Spring Security    | -        | 인증/인가 표준         |
| **API Docs**   | Springdoc OpenAPI  | 2.x      | Swagger UI 자동 생성 |
| **Mapper**     | MapStruct          | 1.5+     | Entity-DTO 변환    |
| **Validation** | Jakarta Validation | -        | 입력 검증            |
| **Batch**      | Spring Batch       | -        | 재고 마감, 정산 배치     |

**선정 이유:**
- 베트남 개발자 중 Java 경험자 다수
- Spring Boot 레퍼런스 풍부
- 엔터프라이즈 환경 검증 완료
- K-Market 본사(한국) 기술 지원 용이

### Frontend (React + TypeScript)

| 카테고리           | 기술               | 버전   | 선정 이유         |
| -------------- | ---------------- | ---- | ------------- |
| **Language**   | TypeScript       | 5.x  | 타입 안정성, 생산성   |
| **Framework**  | React            | 18.x | 컴포넌트 재사용, 생태계 |
| **State**      | Zustand          | 4.x  | 가볍고 직관적       |
| **Routing**    | React Router     | 6.x  | SPA 라우팅 표준    |
| **UI Library** | Ant Design       | 5.x  | 관리자 화면에 최적화   |
| **HTTP**       | TanStack Query   | 5.x  | 서버 상태 관리, 캐싱  |
| **Form**       | React Hook Form  | 7.x  | 폼 성능, 검증      |
| **Table**      | Ant Design Table | -    | 정렬, 필터, 페이징   |
| **Chart**      | ECharts          | 5.x  | 대시보드 차트       |
| **Excel**      | SheetJS          | -    | Excel 내보내기    |
| **Build**      | Vite             | 5.x  | 빠른 빌드, HMR    |

**선정 이유:**
- 화면 기획서에서 이미 기본 구조 정의
- Ant Design이 WMS 관리자 화면에 적합 (테이블 중심)
- React 인력 수급 용이

### Mobile/PDA (Flutter)

| 카테고리          | 기술             | 선정 이유 |              |
| ------------- | -------------- | ----- | ------------ |
| **Framework** | Flutter        | 3.x   | 크로스 플랫폼, 성능  |
| **Language**  | Dart           | -     | Flutter 네이티브 |
| **State**     | Riverpod       | 2.x   | 상태 관리        |
| **Local DB**  | Hive / SQLite  | -     | 오프라인 지원      |
| **Barcode**   | mobile_scanner | -     | 바코드/QR 스캔    |
| **HTTP**      | Dio            | -     | API 통신       |

**선정 이유:**
- Android PDA 기기 대응 필수
- 오프라인 피킹 지원 필요
- 단일 코드베이스로 유지보수 효율

### Database

| 카테고리      | 기술         | 버전  | 선정 이유              |
| --------- | ---------- | --- | ------------------ |
| **RDBMS** | PostgreSQL | 15+ | 화면 기획서 기준, JSON 지원 |
| **Cache** | Redis      | 7.x | 세션, 재고 캐싱          |
| **File**  | MinIO / S3 | -   | 송장, 문서 저장          |

### Infrastructure

| 카테고리              | 기술                   | 선정 이유          |
| ----------------- | -------------------- | -------------- |
| **Container**     | Docker               | 환경 일관성         |
| **Orchestration** | Docker Compose (초기)  | 소규모 운영         |
| **Web Server**    | Nginx                | 리버스 프록시, 정적 파일 |
| **CI/CD**         | GitHub Actions       | 무료, 간편         |
| **Monitoring**    | Grafana + Prometheus | 메트릭 시각화        |
| **Logging**       | Loki + Grafana       | 로그 수집/조회       |

---

## 1.4 추천 아키텍처

### 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Client Layer                                │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   WMS Admin     │  │   PDA App       │  │   Dashboard     │    │
│  │   (React)       │  │   (Flutter)     │  │   (React)       │    │
│  │   39 screens    │  │   7 screens     │  │   KPI/Charts    │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │ HTTPS (TLS 1.3)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Load Balancer (Nginx)                        │
│                  - SSL Termination                                  │
│                  - Rate Limiting                                    │
│                  - Static File Serving                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Application Layer (Spring Boot)                   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    WMS API Server                            │   │
│  │                                                              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │  Auth    │ │  Master  │ │ Inbound  │ │ Outbound │       │   │
│  │  │  Module  │ │  Module  │ │  Module  │ │  Module  │       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  │                                                              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │Inventory │ │  Order   │ │   PDA    │ │  Report  │       │   │
│  │  │  Module  │ │  Module  │ │  Module  │ │  Module  │       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  │                                                              │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │              Common Module                            │   │   │
│  │  │  - Security  - Exception  - Logging  - i18n          │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Batch Server                              │   │
│  │  - 재고 마감 (Daily)                                          │   │
│  │  - 유통기한 알림 (Daily)                                       │   │
│  │  - 리포트 생성 (Weekly/Monthly)                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Data Layer                                  │
│                                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │
│  │  PostgreSQL   │  │    Redis      │  │    MinIO      │          │
│  │  (Primary)    │  │   (Cache)     │  │   (Files)     │          │
│  │               │  │               │  │               │          │
│  │  - 35 Tables  │  │  - Session    │  │  - 송장 PDF   │          │
│  │  - Master     │  │  - Stock Qty  │  │  - 문서 파일  │          │
│  │  - Transaction│  │  - Code Cache │  │  - 이미지     │          │
│  └───────────────┘  └───────────────┘  └───────────────┘          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     External Integration                            │
│                                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │
│  │   K-Market    │  │   통관/선적   │  │     SMS      │          │
│  │     ERP       │  │   시스템 API  │  │   Gateway    │          │
│  └───────────────┘  └───────────────┘  └───────────────┘          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 모듈 구조 (Modular Monolith)

```
wms-backend/
├── src/main/java/com/kmarket/wms/
│   │
│   ├── KmarketWmsApplication.java
│   │
│   ├── common/                          # 공통 모듈
│   │   ├── config/
│   │   │   ├── SecurityConfig.java
│   │   │   ├── JpaConfig.java
│   │   │   ├── RedisConfig.java
│   │   │   └── MessageSourceConfig.java  # 다국어
│   │   ├── exception/
│   │   │   ├── GlobalExceptionHandler.java
│   │   │   ├── BusinessException.java
│   │   │   └── ErrorCode.java
│   │   ├── security/
│   │   │   ├── JwtTokenProvider.java
│   │   │   └── UserDetailsServiceImpl.java
│   │   ├── response/
│   │   │   ├── ApiResponse.java
│   │   │   └── PageResponse.java
│   │   └── util/
│   │       ├── DateUtils.java
│   │       ├── BarcodeUtils.java
│   │       └── ExcelUtils.java
│   │
│   ├── master/                          # 마스터 관리
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── entity/
│   │   │   ├── Item.java               # 품목
│   │   │   ├── Warehouse.java          # 창고
│   │   │   ├── Location.java           # 로케이션
│   │   │   ├── Customer.java           # 거래처
│   │   │   └── Category.java           # 카테고리
│   │   └── dto/
│   │
│   ├── inbound/                         # 입고 관리
│   │   ├── controller/
│   │   ├── service/
│   │   │   ├── InboundPlanService.java # 입고예정
│   │   │   ├── InboundService.java     # 입고확정
│   │   │   └── PutawayService.java     # 적치
│   │   ├── repository/
│   │   ├── entity/
│   │   │   ├── InboundPlan.java
│   │   │   ├── InboundPlanDetail.java
│   │   │   ├── Inbound.java
│   │   │   └── InboundDetail.java
│   │   └── dto/
│   │
│   ├── outbound/                        # 출고 관리
│   │   ├── controller/
│   │   ├── service/
│   │   │   ├── OutboundOrderService.java  # 출고지시
│   │   │   ├── AllocationService.java     # 재고할당 (FEFO)
│   │   │   ├── PickingService.java        # 피킹
│   │   │   └── ShippingService.java       # 출하
│   │   ├── repository/
│   │   ├── entity/
│   │   └── dto/
│   │
│   ├── inventory/                       # 재고 관리
│   │   ├── controller/
│   │   ├── service/
│   │   │   ├── StockService.java       # 재고 조회
│   │   │   ├── StockMoveService.java   # 재고 이동
│   │   │   ├── StockAdjustService.java # 재고 조정
│   │   │   └── InventoryService.java   # 재고 실사
│   │   ├── repository/
│   │   ├── entity/
│   │   │   ├── Stock.java              # 재고
│   │   │   ├── StockHistory.java       # 재고 이력
│   │   │   └── InventoryCount.java     # 실사
│   │   └── dto/
│   │
│   ├── order/                           # 발주 관리
│   │   ├── controller/
│   │   ├── service/
│   │   │   ├── StoreOrderService.java     # 점포 발주
│   │   │   ├── PurchaseOrderService.java  # 수입 발주
│   │   │   └── ShipmentService.java       # 선적 관리
│   │   ├── repository/
│   │   ├── entity/
│   │   └── dto/
│   │
│   ├── pda/                             # PDA API
│   │   ├── controller/
│   │   │   ├── PdaInboundController.java
│   │   │   ├── PdaPickingController.java
│   │   │   └── PdaInventoryController.java
│   │   ├── service/
│   │   └── dto/
│   │
│   ├── report/                          # 리포트
│   │   ├── controller/
│   │   ├── service/
│   │   └── dto/
│   │
│   ├── system/                          # 시스템 관리
│   │   ├── controller/
│   │   ├── service/
│   │   ├── entity/
│   │   │   ├── User.java
│   │   │   ├── Role.java
│   │   │   ├── Menu.java
│   │   │   └── CommonCode.java
│   │   └── dto/
│   │
│   └── batch/                           # 배치 작업
│       ├── job/
│       │   ├── DailyStockCloseJob.java    # 일마감
│       │   ├── ExpiryAlertJob.java        # 유통기한 알림
│       │   └── ReportGenerateJob.java     # 리포트 생성
│       └── config/
│           └── BatchConfig.java
```

---

## 1.5 DB 설계 보완 사항

### 핵심 테이블 추가 권장

기존 35개 테이블에 추가 권장:

| 테이블 | 용도 | 우선순위 |
|--------|------|:--------:|
| `STOCK_HISTORY` | 재고 이력 (입출고 추적) | 필수 |
| `AUDIT_LOG` | 감사 로그 | 필수 |
| `BATCH_JOB_LOG` | 배치 실행 이력 | 권장 |
| `NOTIFICATION` | 알림 (유통기한, 재고부족) | 권장 |
| `API_LOG` | API 호출 로그 | 선택 |

### FEFO 지원 재고 테이블

```sql
CREATE TABLE STOCK (
    STOCK_ID        BIGSERIAL PRIMARY KEY,
    ITEM_CD         VARCHAR(20) NOT NULL,          -- 품목코드
    LOC_CD          VARCHAR(20) NOT NULL,          -- 로케이션
    LOT_NO          VARCHAR(30),                   -- LOT번호
    WH_CD           VARCHAR(10) NOT NULL,          -- 창고코드
    TEMP_ZONE       VARCHAR(10) NOT NULL,          -- 온도대 (ROOM/COLD/FROZEN)

    -- 수량
    QTY             DECIMAL(15,3) DEFAULT 0,       -- 가용재고
    ALLOC_QTY       DECIMAL(15,3) DEFAULT 0,       -- 할당재고

    -- 유통기한 (FEFO 핵심)
    EXPIRE_DATE     DATE,                          -- 유통기한
    MFG_DATE        DATE,                          -- 제조일자
    INBOUND_DATE    DATE NOT NULL,                 -- 입고일자

    -- 상태
    STATUS          VARCHAR(10) DEFAULT 'NORMAL',  -- NORMAL/HOLD/DAMAGED
    USE_YN          CHAR(1) DEFAULT 'Y',

    -- 이력
    REG_USER        VARCHAR(50),
    REG_DATE        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UPD_USER        VARCHAR(50),
    UPD_DATE        TIMESTAMP,

    -- 인덱스
    CONSTRAINT UK_STOCK UNIQUE (ITEM_CD, LOC_CD, LOT_NO, EXPIRE_DATE)
);

-- FEFO 조회용 인덱스
CREATE INDEX IDX_STOCK_FEFO ON STOCK (ITEM_CD, EXPIRE_DATE, INBOUND_DATE)
    WHERE STATUS = 'NORMAL' AND QTY > 0;

-- 온도대별 조회 인덱스
CREATE INDEX IDX_STOCK_TEMP ON STOCK (WH_CD, TEMP_ZONE, ITEM_CD);
```

### 다국어 지원 설계

```sql
-- 방법 1: 컬럼 확장 (단순, 35개 테이블 기준 채택)
CREATE TABLE MST_ITEM (
    ITEM_CD         VARCHAR(20) PRIMARY KEY,
    ITEM_NM_KO      VARCHAR(200),                  -- 한국어
    ITEM_NM_EN      VARCHAR(200),                  -- English
    ITEM_NM_VI      VARCHAR(200),                  -- Tiếng Việt
    ...
);

-- 방법 2: 다국어 테이블 분리 (확장성, 향후 고려)
CREATE TABLE MST_ITEM_LANG (
    ITEM_CD         VARCHAR(20),
    LANG_CD         VARCHAR(5),                    -- KO, EN, VI
    ITEM_NM         VARCHAR(200),
    ITEM_DESC       TEXT,
    PRIMARY KEY (ITEM_CD, LANG_CD)
);
```

---

## 1.6 1단계 산출물 체크리스트

| 산출물 | 담당 | 상태 | 비고 |
|--------|------|:----:|------|
| 프로젝트 계획서 | PM | 📝 | WBS, 일정, 리소스 |
| 화면 설계서 | BA | ✅ | 39개 화면 완료 |
| 기능 명세서 | BA | ✅ | 프로세스 플로우 완료 |
| 물리 ERD | DBA | 📝 | 35개 → 40개 보완 |
| 아키텍처 설계서 | PL | 📝 | 본 문서 기반 |
| API 설계서 | PL | 📝 | OpenAPI 3.0 |
| 개발 표준 가이드 | PL | 📝 | 코딩 규칙, 명명 규칙 |
| 테스트 계획서 | QA | 📝 | |

---

# 2단계: 개발

> **기간**: 12~16주
> **목표**: 39개 화면 + 7개 PDA 화면 개발, 테스트 완료

---

## 2.1 개발 Sprint 계획

### Sprint 구성 (2주 단위)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    K-Market WMS 개발 로드맵                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Sprint 1-2 (4주): 환경 구축 + 공통 + 마스터                          │
│  ├── 개발 환경 셋업                                                  │
│  ├── 공통 모듈 (인증, 예외처리, 다국어)                                │
│  ├── 마스터 관리 (품목, 창고, 로케이션, 거래처)                         │
│  └── 시스템 관리 (사용자, 권한, 코드)                                  │
│                                                                     │
│  Sprint 3-4 (4주): 입고 관리                                        │
│  ├── 입고예정 등록/조회                                               │
│  ├── 입고확정 (검수)                                                 │
│  ├── 입고실적 조회                                                   │
│  └── PDA 입고검수, 적재                                              │
│                                                                     │
│  Sprint 5-6 (4주): 재고 관리                                        │
│  ├── 재고현황 조회                                                   │
│  ├── 재고이동, 재고조정                                               │
│  ├── 로케이션관리 (2D 맵)                                            │
│  └── 재고조사 (실사)                                                 │
│                                                                     │
│  Sprint 7-8 (4주): 출고 관리                                        │
│  ├── 출고지시 등록                                                   │
│  ├── 출고할당 (FEFO)                                                │
│  ├── 출고검수                                                       │
│  ├── 출고실적 조회                                                   │
│  └── PDA 피킹                                                       │
│                                                                     │
│  Sprint 9-10 (4주): 발주 관리 + 대시보드                              │
│  ├── 점포발주 등록/관리                                               │
│  ├── 수입발주 (구매요청, 선적정보)                                     │
│  ├── 대시보드                                                       │
│  └── 통합 테스트                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Sprint별 상세 일정

| Sprint | 기간     | Backend    | Frontend    | PDA  | QA      |
| :----: | ------ | ---------- | ----------- | ---- | ------- |
|   1    | 1-2주   | 환경구축, 공통모듈 | 환경구축, 레이아웃  | 환경구축 | 테스트 환경  |
|   2    | 3-4주   | 마스터 API    | 마스터 화면 (3개) | -    | 테스트 케이스 |
|   3    | 5-6주   | 입고 API (1) | 입고 화면 (2개)  | 입고검수 | 마스터 테스트 |
|   4    | 7-8주   | 입고 API (2) | 입고 화면 (2개)  | 입고적재 | 입고 테스트  |
|   5    | 9-10주  | 재고 API (1) | 재고 화면 (4개)  | 재고조사 | -       |
|   6    | 11-12주 | 재고 API (2) | 로케이션 화면     | 이동   | 재고 테스트  |
|   7    | 13-14주 | 출고 API (1) | 출고 화면 (2개)  | 피킹   | -       |
|   8    | 15-16주 | 출고 API (2) | 출고 화면 (2개)  | 피킹   | 출고 테스트  |
|   9    | 17-18주 | 발주 API     | 발주 화면 (6개)  | -    | -       |
|   10   | 19-20주 | 대시보드, ERP  | 대시보드 (2개)   | -    | 통합 테스트  |

---

## 2.2 모듈별 개발 상세

### 2.2.1 마스터 관리 (Sprint 1-2)

**API 목록**

| API | Method | Path | 설명 |
|-----|--------|------|------|
| 품목 목록 | GET | `/api/v1/items` | 페이징, 검색 |
| 품목 상세 | GET | `/api/v1/items/{itemCd}` | |
| 품목 등록 | POST | `/api/v1/items` | |
| 품목 수정 | PUT | `/api/v1/items/{itemCd}` | |
| 품목 삭제 | DELETE | `/api/v1/items/{itemCd}` | USE_YN='N' |
| 창고 목록 | GET | `/api/v1/warehouses` | |
| 로케이션 목록 | GET | `/api/v1/locations` | 창고별 필터 |
| 거래처 목록 | GET | `/api/v1/customers` | 유형별 필터 |
| 카테고리 트리 | GET | `/api/v1/categories/tree` | 3단계 계층 |

**화면 목록**

| 화면 | 기능 | 공수 |
|------|------|:----:|
| 품목마스터 | CRUD, 바코드, 온도대, 다국어 품명 | 0.2 M/M |
| 카테고리관리 | 트리뷰, 3단계 계층 | 0.1 M/M |
| 거래처관리 | 공급처/고객/점포 구분 | 0.15 M/M |

### 2.2.2 입고 관리 (Sprint 3-4)

**핵심 비즈니스 로직**

```java
@Service
@RequiredArgsConstructor
public class InboundService {

    private final StockService stockService;

    /**
     * 입고 확정 처리
     * - 입고예정 대비 실입고 수량 검증
     * - 온도대별 로케이션 자동 배정
     * - 재고 증가 처리
     */
    @Transactional
    public InboundResponse confirmInbound(InboundConfirmRequest request) {
        // 1. 입고예정 조회
        InboundPlan plan = inboundPlanRepository.findById(request.getPlanId())
            .orElseThrow(() -> new BusinessException(ErrorCode.PLAN_NOT_FOUND));

        // 2. 온도대 검증
        validateTempZone(request.getLocationCd(), plan.getItem().getTempZone());

        // 3. 재고 증가
        Stock stock = stockService.increaseStock(StockIncreaseRequest.builder()
            .itemCd(plan.getItemCd())
            .locationCd(request.getLocationCd())
            .qty(request.getConfirmQty())
            .expireDate(request.getExpireDate())
            .lotNo(generateLotNo())
            .build());

        // 4. 입고 이력 생성
        return createInboundHistory(plan, stock, request);
    }
}
```

**PDA 화면**

| 화면 | 기능 | 특이사항 |
|------|------|----------|
| 상품입고검수 | 입고예정서 스캔, 수량 검수 | 바코드 스캔 |
| 입고적재 | 품목/로케이션 스캔, 온도대 검증 | 온도대 불일치 알림 |

### 2.2.3 출고 관리 (Sprint 7-8)

**FEFO 재고 할당 로직**

```java
@Service
@RequiredArgsConstructor
public class AllocationService {

    /**
     * FEFO 기반 재고 할당
     * - 유통기한 임박 순 우선 할당
     * - 동일 유통기한인 경우 입고일 순
     */
    public List<AllocationResult> allocateByFEFO(AllocationRequest request) {
        List<AllocationResult> results = new ArrayList<>();
        BigDecimal remainQty = request.getOrderQty();

        // FEFO 순서로 재고 조회
        List<Stock> stocks = stockRepository.findAvailableStockByFEFO(
            request.getItemCd(),
            request.getWhCd()
        );

        for (Stock stock : stocks) {
            if (remainQty.compareTo(BigDecimal.ZERO) <= 0) break;

            // 할당 가능 수량 계산
            BigDecimal availableQty = stock.getQty().subtract(stock.getAllocQty());
            BigDecimal allocQty = remainQty.min(availableQty);

            // 재고 할당 처리
            stock.addAllocQty(allocQty);
            stockRepository.save(stock);

            results.add(AllocationResult.builder()
                .stockId(stock.getStockId())
                .locationCd(stock.getLocationCd())
                .allocQty(allocQty)
                .expireDate(stock.getExpireDate())
                .build());

            remainQty = remainQty.subtract(allocQty);
        }

        // 할당 부족 시 예외
        if (remainQty.compareTo(BigDecimal.ZERO) > 0) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK);
        }

        return results;
    }
}
```

**PDA 피킹 화면**

```
┌─────────────────────────────────────────┐
│  🔍 피킹 작업                           │
├─────────────────────────────────────────┤
│                                         │
│  출고번호: OUT-2026-0324-001            │
│  진행률: ████████░░ 80% (8/10)          │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 현재 피킹 품목                   │   │
│  │                                 │   │
│  │ 품목: 신라면 멀티팩              │   │
│  │ 바코드: 8801234567890           │   │
│  │ 수량: 10 EA                     │   │
│  │ 위치: A-01-02-03                │   │
│  │ 유통기한: 2026-06-30            │   │
│  └─────────────────────────────────┘   │
│                                         │
│  1단계: 로케이션 스캔                    │
│  ┌─────────────────────────────────┐   │
│  │ 📷 [스캔 대기중...]              │   │
│  └─────────────────────────────────┘   │
│                                         │
│  2단계: 상품 바코드 스캔                 │
│  ┌─────────────────────────────────┐   │
│  │ 📷 [1단계 완료 후 활성화]         │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [◀ 이전] [건너뛰기] [피킹완료 ▶]       │
│                                         │
└─────────────────────────────────────────┘
```

### 2.2.4 재고 관리 (Sprint 5-6)

**재고 이력 관리**

```java
@Service
@RequiredArgsConstructor
public class StockHistoryService {

    /**
     * 재고 변동 이력 기록
     * - 모든 재고 변동 시 자동 기록
     * - 추적성 확보
     */
    @Transactional
    public void recordHistory(StockHistoryRequest request) {
        StockHistory history = StockHistory.builder()
            .stockId(request.getStockId())
            .trxType(request.getTrxType())        // IN/OUT/MOVE/ADJUST
            .trxQty(request.getTrxQty())
            .beforeQty(request.getBeforeQty())
            .afterQty(request.getAfterQty())
            .refNo(request.getRefNo())            // 입고번호/출고번호/조정번호
            .refType(request.getRefType())        // INBOUND/OUTBOUND/ADJUST
            .remark(request.getRemark())
            .regUser(SecurityUtils.getCurrentUser())
            .build();

        stockHistoryRepository.save(history);
    }
}
```

---

## 2.3 공통 모듈 상세

### 2.3.1 다국어 지원 (i18n)

**Backend 설정**

```java
@Configuration
public class MessageSourceConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource source =
            new ReloadableResourceBundleMessageSource();
        source.setBasename("classpath:messages/messages");
        source.setDefaultEncoding("UTF-8");
        source.setCacheSeconds(60);
        return source;
    }
}

// messages/messages_ko.properties
error.stock.insufficient=재고가 부족합니다. (요청: {0}, 가용: {1})
error.temp_zone.mismatch=온도대가 일치하지 않습니다. (품목: {0}, 로케이션: {1})

// messages/messages_en.properties
error.stock.insufficient=Insufficient stock. (requested: {0}, available: {1})
error.temp_zone.mismatch=Temperature zone mismatch. (item: {0}, location: {1})

// messages/messages_vi.properties
error.stock.insufficient=Không đủ hàng tồn kho. (yêu cầu: {0}, có sẵn: {1})
error.temp_zone.mismatch=Vùng nhiệt độ không khớp. (mặt hàng: {0}, vị trí: {1})
```

**Frontend 설정**

```typescript
// i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

import ko from './locales/ko.json';
import en from './locales/en.json';
import vi from './locales/vi.json';

i18n.use(initReactI18next).init({
  resources: {
    ko: { translation: ko },
    en: { translation: en },
    vi: { translation: vi },
  },
  lng: localStorage.getItem('lang') || 'ko',
  fallbackLng: 'ko',
});
```

### 2.3.2 API 응답 표준

```java
@Getter
@Builder
public class ApiResponse<T> {
    private final boolean success;
    private final String code;
    private final String message;
    private final T data;
    private final LocalDateTime timestamp;

    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .code("SUCCESS")
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }

    public static <T> ApiResponse<T> error(ErrorCode errorCode) {
        return ApiResponse.<T>builder()
            .success(false)
            .code(errorCode.getCode())
            .message(errorCode.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
    }
}

// 응답 예시
{
    "success": true,
    "code": "SUCCESS",
    "message": null,
    "data": {
        "itemCd": "ITEM001",
        "itemNmKo": "신라면",
        "itemNmEn": "Shin Ramyun",
        "itemNmVi": "Mì Shin Ramyun"
    },
    "timestamp": "2026-03-24T10:30:00"
}
```

---

## 2.4 테스트 전략

### 테스트 커버리지 목표

| 테스트 유형 | 커버리지 | 담당 |
|------------|:--------:|------|
| 단위 테스트 | 70% | 개발자 |
| API 테스트 | 100% | 개발자 |
| 통합 테스트 | 주요 플로우 | QA |
| E2E 테스트 | 핵심 시나리오 | QA |

### 핵심 테스트 시나리오

| 시나리오 | 테스트 내용 |
|----------|------------|
| 입고 Full Cycle | 입고예정 → 입고확정 → 적치 → 재고 확인 |
| 출고 Full Cycle | 출고지시 → FEFO 할당 → 피킹 → 출하 |
| FEFO 검증 | 유통기한 순 할당 정확성 |
| 온도대 검증 | 품목-로케이션 온도대 일치 여부 |
| PDA 오프라인 | 네트워크 끊김 시 데이터 보존 |

---

## 2.5 2단계 산출물 체크리스트

| 산출물 | 담당 | 비고 |
|--------|------|------|
| 소스코드 (Git) | 개발팀 | Backend + Frontend + PDA |
| API 명세서 | Backend | Swagger UI |
| 단위 테스트 코드 | 개발팀 | JUnit, Jest |
| 테스트 결과서 | QA | 기능 테스트 |
| 결함 관리대장 | QA | Jira |

---

# 3단계: 시스템 구현

> **기간**: 4~6주
> **목표**: 운영 환경 구축, 데이터 이행, 사용자 교육, 시스템 오픈

---

## 3.1 운영 환경 구성

### 서버 구성 (권장)

**Option A: On-premise (자체 서버)**

| 서버 | 사양 | 수량 | 용도 |
|------|------|:----:|------|
| WAS | 4 Core, 16GB RAM, 200GB SSD | 2 | API 서버 (Active-Active) |
| DB | 8 Core, 32GB RAM, 500GB SSD | 2 | PostgreSQL (Primary-Standby) |
| Cache | 2 Core, 8GB RAM | 1 | Redis |
| File | 2 Core, 4GB RAM, 500GB HDD | 1 | MinIO |

**Option B: Cloud (AWS Vietnam)**

| 서비스 | 사양 | 용도 |
|--------|------|------|
| EC2 (t3.large) x 2 | 2 vCPU, 8GB | API 서버 |
| RDS PostgreSQL | db.t3.large | 메인 DB |
| ElastiCache | cache.t3.micro | Redis |
| S3 | - | 파일 저장 |
| ALB | - | 로드 밸런서 |

### Docker Compose 구성

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  wms-api:
    image: kmarket/wms-api:${VERSION}
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_HOST=wms-db
      - REDIS_HOST=wms-redis
    depends_on:
      - wms-db
      - wms-redis
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '2'
          memory: 4G

  wms-frontend:
    image: kmarket/wms-frontend:${VERSION}
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl

  wms-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=wms
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  wms-redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  wms-minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_PASSWORD}
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

---

## 3.2 데이터 이행

### 이행 대상

| 구분 | 데이터 | 건수 (예상) | 소스 | 방식 |
|------|--------|------------|------|------|
| 마스터 | 품목 | 5,000+ | ERP/Excel | ETL |
| 마스터 | 거래처 (점포) | 100 | Excel | 수동 |
| 마스터 | 창고/로케이션 | 500+ | 현장 조사 | 수동 |
| 마스터 | 카테고리 | 200+ | Excel | ETL |
| 트랜잭션 | 현재고 | 10,000+ | 실사 | 실사 등록 |
| 설정 | 공통코드 | 500+ | 설계서 | 수동 |
| 설정 | 사용자/권한 | 50+ | 명단 | 수동 |

### 이행 절차

```
┌─────────────────────────────────────────────────────────────────┐
│                      이행 절차 (2주)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Week 1: 이행 준비 + 1차 리허설                                   │
│  ├── D+1~2: 이행 데이터 준비 (Excel 정제)                         │
│  ├── D+3~4: 이행 프로그램 개발                                    │
│  └── D+5: 1차 리허설 (개발 환경)                                  │
│                                                                 │
│  Week 2: 2차 리허설 + 본 이행                                     │
│  ├── D+1~2: 1차 이슈 수정                                        │
│  ├── D+3: 2차 리허설 (스테이징 환경)                               │
│  ├── D+4: 본 이행 (운영 환경)                                     │
│  └── D+5: 데이터 검증                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 사용자 교육

### 교육 대상 및 일정

| 대상 | 인원 (예상) | 교육 내용 | 시간 | 방식 |
|------|:-----------:|----------|:----:|------|
| 시스템 관리자 | 2~3명 | 전체 기능, 설정, 권한 관리 | 8h | 집합 |
| 물류센터 사무직 | 5~10명 | 입고/출고/재고/발주 관리 | 4h | 집합 |
| 현장 작업자 (PDA) | 20~30명 | PDA 사용법, 피킹, 적치 | 2h | 현장 실습 |
| 점포 담당자 | 100명 | 점포 발주 등록 | 1h | 온라인/동영상 |

### 교육 자료

| 자료 | 대상 | 형식 | 언어 |
|------|------|------|------|
| 관리자 매뉴얼 | 시스템 관리자 | PDF | KO |
| 사용자 매뉴얼 (Web) | 사무직 | PDF | KO/VI |
| PDA 사용 가이드 | 현장 작업자 | PDF + 동영상 | KO/VI |
| Quick Card | 전체 | 인쇄물 | KO/VI |
| FAQ | 전체 | Online | KO/EN/VI |

---

## 3.4 오픈 체크리스트

```
┌─────────────────────────────────────────────────────────────────┐
│                   K-Market WMS 오픈 체크리스트                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  □ 인프라                                                       │
│    □ 서버 상태 확인 (CPU, Memory, Disk)                          │
│    □ 네트워크 연결 확인 (API ↔ DB, API ↔ Redis)                  │
│    □ SSL 인증서 확인 (만료일)                                     │
│    □ 백업 설정 확인 (DB, File)                                   │
│    □ 모니터링 알림 설정 확인                                      │
│                                                                 │
│  □ 애플리케이션                                                  │
│    □ 최신 버전 배포 확인                                          │
│    □ 환경 변수 확인 (SPRING_PROFILES_ACTIVE=prod)                │
│    □ 로그 레벨 확인 (INFO)                                       │
│    □ Health Check 확인 (/actuator/health)                       │
│                                                                 │
│  □ 데이터                                                       │
│    □ 품목 마스터 확인 (5,000+ 건)                                 │
│    □ 로케이션 마스터 확인                                         │
│    □ 현재고 데이터 확인 (실사 완료)                                │
│    □ 사용자/권한 확인                                            │
│    □ 공통코드 확인                                               │
│                                                                 │
│  □ 기능                                                         │
│    □ 로그인/로그아웃 확인                                         │
│    □ 입고 Full Cycle 테스트                                      │
│    □ 출고 Full Cycle 테스트 (FEFO 확인)                          │
│    □ PDA 바코드 스캔 확인                                        │
│    □ Excel 다운로드 확인                                         │
│    □ 다국어 전환 확인 (KO/EN/VI)                                 │
│                                                                 │
│  □ 보안                                                         │
│    □ 방화벽 규칙 확인                                            │
│    □ HTTPS 강제 리다이렉트 확인                                   │
│    □ 세션 타임아웃 확인 (30분)                                    │
│                                                                 │
│  □ 롤백 계획                                                     │
│    □ 이전 버전 Docker 이미지 보관                                 │
│    □ DB 백업 완료                                                │
│    □ 롤백 절차서 준비                                            │
│    □ 롤백 담당자 지정                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 안정화 운영

| 기간 | 활동 | 담당 |
|------|------|------|
| D-Day ~ D+3 | 현장 상주 지원 (09:00~21:00) | 개발팀 전원 |
| D+4 ~ D+14 | 주간 상주 + 야간 대기 | 개발팀 (순환) |
| D+15 ~ D+30 | 원격 지원 + 주 1회 방문 | PL + 담당자 |
| D+31 ~ | 유지보수 체제 전환 | 유지보수팀 |

---

## 3.6 3단계 산출물 체크리스트

| 산출물 | 담당 | 인수자 |
|--------|------|--------|
| 인프라 구성도 | 인프라 | 운영팀 |
| 배포 가이드 | DevOps | 운영팀 |
| 데이터 이행 결과서 | DBA | PM |
| 사용자 매뉴얼 | BA | 고객사 |
| 운영 매뉴얼 | PL | 운영팀 |
| 장애 대응 가이드 | PL | 운영팀 |
| 프로젝트 종료 보고서 | PM | 고객사 |

---

# 총 일정 요약

```
┌─────────────────────────────────────────────────────────────────────┐
│                   K-Market WMS 프로젝트 일정                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1단계: 시스템 분석과 설계 (4~6주)                                    │
│  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░              │
│                                                                     │
│  2단계: 개발 (12~16주)                                              │
│          ████████████████████████████████████████████░░░░░░        │
│                                                                     │
│  3단계: 시스템 구현 (4~6주)                                          │
│                                                  ████████████████   │
│                                                                     │
│  ─────────────────────────────────────────────────────────────     │
│  │  1   │  2   │  3   │  4   │  5   │  6   │  7   │  8   │ (월)   │
│                                                                     │
│  총 기간: 20~28주 (약 5~7개월)                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 단계 | 기간 | 주요 산출물 |
|------|------|------------|
| 1단계 | 4~6주 | 설계서, API 명세, ERD |
| 2단계 | 12~16주 | 소스코드, 테스트 결과 |
| 3단계 | 4~6주 | 운영 환경, 매뉴얼 |
| **총계** | **20~28주** | |

---

# 관련 문서

- [[K-Market-WMS-Overview]] - 프로젝트 개요
- [[일반적인-WMS-설계방식]] - 일반적인 WMS 설계 방식

---

**Last Updated**: 2026-03-24
**Document Version**: 1.0
