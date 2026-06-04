# 인수인계서

> 박상준 차장(웹팀) → 하나사인몰 개발팀
> 작성: ________________ (회의 완료 시 박상준 차장이 채워 넣음)

---

## 1. 인수인계 범위

| 항목 | 인계 |
|---|---|
| 소스코드 | GitHub repo `sangjunepark-beep/hanasm-guide` (Private 전환 후) |
| Firestore project | `hanasm-chatbot-terms` (Firebase Console 멤버 권한) |
| 데이터 | 회의 완료 시점의 Firestore export (별도 zip 또는 GCS) |
| 문서 | README · DATA_MODEL · SECURITY · OPERATIONS · ROADMAP · CHANGELOG · 본 문서 |

---

## 2. 회의 운영 완료 상태 (회의 종료 시 박상준 차장 작성)

### 통일 작업 진행률

- [ ] 1. 사용장비 (Machine) — 12 항목 중 ____ 합의 완료
- [ ] 2. 공정 (Stage) — 15+ 항목 중 ____ 합의 완료
- [ ] 3. 재질 (Material) — 63 항목 중 ____ 합의 완료
- [ ] 5. 주요 불일치 — 9 항목 중 ____ 합의 완료

### 진행 중인 작업

(회의 진행 중 미합의 항목, 추후 결정 필요 사항 등)

| 항목 | 상태 | 비고 |
|---|---|---|
|  |  |  |

### 보류 사항

(생산팀 확인 필요, 외주사 협의 필요 등)

| 항목 | 보류 사유 | 결정 책임자 |
|---|---|---|
|  |  |  |

---

## 3. 개발팀이 시작할 작업

### 즉시 작업 (보안)
- [ ] GitHub repo Private 전환 + 개발팀 멤버 권한 부여
- [ ] Firebase Console 개발팀 멤버 초대
- [ ] **Firestore 보안규칙 임시 잠금**: `allow read, write: if false;` (회의 외 시간 무방비 상태)
- [ ] `SECURITY.md` 2단계 — Firebase Auth 5명 계정 생성

### 1주 안에
- [ ] Firestore 자동 백업 설정 (`OPERATIONS.md` 1번 참조)
- [ ] 모니터링 알림 채널 구성 (Slack/이메일)

### Phase 2 시작 전
- [ ] `SECURITY.md` 3단계 — 회사 도메인 한정
- [ ] ERP 내부 API endpoint 작성: `POST /api/internal/material-rename`
- [ ] 챗봇 동의어 사전 갱신 API: `POST /api/internal/synonyms-update`
- [ ] Cloud Function 작성: Firestore confirmed 변경 → ERP·챗봇 푸시

### 운영 진입 시
- [ ] `SECURITY.md` 4단계 — 사내망 이전 또는 인증 프록시
- [ ] 정기 점검 일정 (`OPERATIONS.md` 8번)

---

## 4. 연락처 (회의 종료 시 채움)

| 역할 | 이름 | 연락 |
|---|---|---|
| 운영 책임 (인수인계 전) | 박상준 차장 (웹팀) | jimrn22@gmail.com |
| 인수인계 대상 | 개발팀 (담당자: __________ ) |  |
| 회의 참가자 5명 | 대표 · 박상준 차장 · 송기용 팀장 · 천현정 차장 · 강두원 팀장 |  |
| 외부 연관 시스템 담당 | ERP 운영 (__________) · 챗봇 외주사 (__________) · 상품DB (홍재이 주임) |  |

---

## 5. 데이터 export 절차

회의 완료 시점 Firestore 데이터를 개발팀에 인계하려면:

### 옵션 A. Firebase Console (가장 간단)
1. Firebase Console → Firestore → `가져오기/내보내기` 탭
2. `내보내기` 클릭
3. Cloud Storage 버킷 선택 (없으면 생성)
4. 컬렉션 선택: `erp_terminology`, `erp_term_logs`, `erp_term_order`, `erp_notes`
5. 내보내기 완료 후 zip 다운로드 → 개발팀 전달

### 옵션 B. gcloud CLI (개발팀이 직접)
```bash
gcloud config set project hanasm-chatbot-terms
gcloud firestore export gs://hanasm-chatbot-terms-handover/$(date +%Y%m%d)
```

---

## 6. 인수인계 완료 체크

- [ ] 본 문서의 1·2·3·4 모두 작성 완료
- [ ] 데이터 export 완료 + 개발팀 전달
- [ ] GitHub repo Private + 권한 이관
- [ ] Firebase Console 권한 이관
- [ ] 회의 종료 후 임시 보안 잠금 적용
- [ ] 개발팀과 첫 협의 미팅 일정 잡음 (Phase 2 킥오프)

---

## 7. 박상준 차장 사후 역할

인수인계 후 박상준 차장은 다음 역할로 전환:
- 회의 도구 사용자 (Phase 1~4 진행 중에도 미팅 진행)
- 개발팀-운영 간 협업 인터페이스 (의사결정·요구사항 전달)
- 회의 결과 검수 및 추가 통일 작업 진행

기술적 운영 책임(인증·인프라·DB·모니터링)은 개발팀으로 완전 이관.

---

서명:

박상준 차장 (인계자) ________________ 일자: __________
개발팀 담당 (인수자) ________________ 일자: __________
