# 📦 WMS PDA 시스템 기술 스택 정의서

---

## 1. 📌 프로젝트 개요

본 시스템은 WMS(창고관리시스템)와 연동되는 PDA 기반 업무 애플리케이션으로,
불안정한 네트워크 환경(베트남)을 고려한 **Offline-First 아키텍처**를 핵심으로 설계한다.

주요 기능:
- 상품 조회
- 입고 검수
- 입고 적재
- 상품 피킹
- 로케이션 이동
- 재고 조사

---

## 2. 🧱 전체 아키텍처 개요

```
[PDA App (Flutter)]
   ↓ (Sync / API)
[Spring Boot API Server]
   ↓
[PostgreSQL]
```

핵심 특징:
- 오프라인 우선 처리
- 로컬 저장 후 동기화
- Sync Queue 기반 재전송

---

## 3. 📱 PDA / 앱 기술 스택

### 3.1 기본 구성
- Flutter
- 상태관리: Riverpod
- 라우팅: go_router

### 3.2 데이터 처리
- 로컬 DB: SQLite (권장: Drift)
- 네트워크: Dio
- 직렬화: freezed + json_serializable

### 3.3 시스템 기능
- 백그라운드 작업: workmanager
- 연결 상태 감지: connectivity_plus
- UUID 생성: uuid

### 3.4 하드웨어 연동
- Platform Channel 기반 Scanner Adapter
- 벤더별 스캐너 분리 구조 (Zebra / Honeywell 대응)

---

## 4. 🖥 서버 기술 스택

- Java + Spring Boot
- Spring Security (인증/인가)
- MyBatis (SQL 중심 데이터 처리)
- PostgreSQL (메인 DB)
- Redis (선택: 캐시/락/세션)
- Swagger / OpenAPI (API 문서화)

---

## 5. 🔥 핵심 아키텍처 전략

### 5.1 Offline-First
- 모든 작업은 로컬 DB에 먼저 저장
- 서버 응답 없이도 업무 진행 가능

### 5.2 Outbox Pattern
- 서버 전송 전 데이터를 별도 큐에 저장
- 성공 시 제거, 실패 시 재시도

### 5.3 Sync 구조

#### 다운로드 Sync (Server → PDA)
- 피킹 리스트
- 상품 정보
- 로케이션 정보

#### 업로드 Sync (PDA → Server)
- 피킹 결과
- 검수 결과
- 이동 작업
- 재고 조사 결과

---

## 6. ⚠️ 필수 고려사항

### 6.1 네트워크 불안정 대응
- Offline 상태에서도 모든 작업 가능해야 함
- 네트워크 복구 시 자동 동기화
- 수동 Sync 버튼 제공

---

### 6.2 데이터 무결성
- 작업 단위별 고유 ID (UUID)
- 중복 처리 방지
- 서버 반영 시 version 체크

---

### 6.3 동기화 정책
- 앱 시작 시 Sync
- 작업 화면 진입 시 Sync
- 네트워크 복구 시 Sync
- 백그라운드 Sync (workmanager)

---

### 6.4 충돌 처리
- 서버 데이터와 PDA 데이터 불일치 시 처리 정책 필요

예:
- version mismatch
- 이미 처리된 작업

처리 방식:
- 자동 병합
- 사용자 재확인

---

### 6.5 스캐너 추상화
- ScannerService 인터페이스 정의
- 벤더별 구현 분리

```
ScannerService
  ├── ZebraScanner
  ├── HoneywellScanner
  └── CameraScanner (Fallback)
```

---

### 6.6 사용자 피드백
- 스캔 성공 시 즉시 반응
  - 진동
  - 사운드
  - UI 색상 변경

---

### 6.7 로그 및 모니터링
- 앱 로그 수집
- Sync 실패 로그
- 사용자 작업 로그

---

## 7. 🧩 데이터 구조 (예시)

### 로컬 DB 테이블
- picking_task
- picking_item
- scan_event
- sync_outbox
- sync_log

---

## 8. 🚀 향후 확장 고려

- Redis 도입 (락/캐시)
- WebSocket 기반 실시간 작업 할당
- 라벨 프린터 연동
- 이미지 첨부 기능
- 음성 안내 기능

---

## 9. 📌 최종 요약

본 시스템은 Flutter 기반 PDA 앱과 Spring Boot 서버를 중심으로 구성되며,
오프라인 환경에서도 안정적으로 동작하는 **Offline-First 구조**를 핵심으로 한다.

특히 Sync Queue, Scanner Adapter, 로컬 DB 기반 처리 구조가
시스템 안정성과 확장성을 결정짓는 핵심 요소이다.

---

✅ 핵심 키워드:
- Offline First
- Sync Queue
- Scanner Abstraction
- Local DB 중심 처리
- 안정적인 동기화 구조
