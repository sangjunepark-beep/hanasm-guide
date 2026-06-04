# CHANGELOG

> 인수인계 시점까지의 주요 변경 이력. 자세한 commit 은 `git log` 참조.

---

## 2026-06-04 — 데이터 무결성 강화 + 기능 안정화

### 적대적 검증 패치 (대규모)
- **#1 editTerm rename 시 newKey 데이터 머지** — 덮어쓰기 방지
- **#2 copies 배열 `arrayUnion`/`arrayRemove`** — race 제거
- **#3 toggleConfirm·toggleDownstream·saveRationale `runTransaction`** — 동시 토글 race 차단
- **#4 다른 클라이언트 doc 삭제 시 좀비 행 자동 정리** — `_lastSeenKeys` diff
- **#5 UNDO 동시 클릭 claim 메커니즘** — 트랜잭션
- **#10 del-term beforeData 에 doc 전체 보존** — confirmed·downstream·rationale·copies·approvals 까지 복원 가능
- **#7 신규 추가 동명 ERP 자동 연동 경고** confirm
- **#8 빈 final + 전체동의 차단** alert
- **#9 UNDO 본인 액션 아닐 때 강조 경고**
- **#11 `validateTermName`** — 제어문자·길이·예약어 가드 (charCodeAt 비교로 NULL byte 사고 회피)
- **#12 드래그 중 onSnapshot DOM 재배치 일시중지**
- **#16 사유 textarea max-height 200px**
- **#19 beforeunload 미저장 입력 확인**

### 기능 추가·UI 개선
- **공정(stage) 그룹 — 행 복사 기능** — 다른 공정 흐름으로 복사, 데이터 자동 연동 (`copies` 배열)
- **5번 주요 불일치 표 제안 문구** — 비전공자용 평어체로 풀어쓰기
- **승인 5칸 → 전체동의 1버튼** 통합 (UI 단순화)
- **반영 4칸 → 3칸** (ERP·상품DB·챗봇, CS답변 제거)
- **row-actions 별도 행(actions-row, colspan=4) 분리** — 좁은 모니터에서 셀 침범 버그 근본 해결
- **컬럼 비율 조정** — 본문·메모 영역 폭 조정
- **table-layout:fixed** 적용 범위 좁힘 — data 표에만
- 복사 버튼 시각 축소 (호버 시에만 드러남)
- 복사본 행 첫 셀 이탤릭 제거
- 본문·메모 영역 확장 (wrap 1320→1686px)

### 버그 수정
- **validateTermName regex NULL byte** 박힌 사고 수정 — 모듈 전체 실행 실패 원인 제거
- `tr.confirmed::before` pseudo-element 제거 — 첫 셀 폭 늘리던 버그
- `setDoc + merge` 에서 dot notation 미지원 → nested 객체로 변경
- 셀렉터 escape 의존 제거 → `dataset` 직접 비교 (`forEachByData`)
- `cssEscape` 순서 버그 수정
- 신규 항목 수정/삭제 race condition 차단 (DOM-first + `_pendingDeletes` blacklist)
- 보안규칙 미허용 컬렉션(`erp_subgroup_order`) 제거 → `erp_term_order` 안 `__sub__::` prefix 통합

---

## 2026-06-04 — P1 기능 도입

- **승인·사유·다운스트림 반영** 칸 도입 (이후 단순화)
- **CSV 2종** — 회의록용 + 주입용 분리
- **승인 권한 본인 한정** 시도 후 회의 운영 편의로 풀기
- **반영 4칸 잠금 제거** — 5명 승인 가드 빼고 자유롭게

---

## 2026-06-03 ~ 06-04 — 드래그·UNDO·복사 시스템

- **드래그앤드롭 행 순서 변경** — `erp_term_order` 컬렉션
- **소구분 헤더 드래그** — 소구분 자체 순서 변경
- **UNDO 시스템** — 최대 20회 연속, 모든 변경 역추적 가능
- **신규 항목 수정 기능** + 삭제 버튼 시인성 강화
- 신규 항목 추가 — 소구분별 인라인 버튼, 빨간색 신규 행, CSV 별도추가건 표시
- 중복 이름 차단 제거 (`/` 만 차단)

---

## 2026-06-03 — UI v2

- **단락별 우측 sticky 메모** 분리
- **분류별 [항목 추가]** 버튼 (Firestore 공유)
- ERP 검증 반영: 배면UV인쇄 공정 편입, 포마트10T 정정, 불일치 항목 추가
- 전역 `clearAll` 함수 제거 (보안 위험)

---

## 2026-06-02 — 분류 구조 정리

- **3 그룹 6 공정 흐름** 구조로 정리 (UV / 솔벤 / 라텍스 / 수성 / 레이저 / 슈마 / DTF)
- **재질 63종** 카테고리 재구성
- 공통 장비 그룹 분리
- 비고/메모에 댓글(parentId 모델) 추가

---

## 2026-06-01 — 초기 구축

- Firebase 연동 (별도 project `hanasm-chatbot-terms`)
- 실시간 동기화 + 편집자 드롭다운
- 1차 수정안 / 최종 협의안 분리
- 섹션별 비고/협의 기록
- DB 변경 로그 (`erp_term_logs` 컬렉션)
- 행마다 마지막 편집자/시각 표시
- 모바일 대응
- CSV 내보내기
- 초기 ERP↔가이드 v2 용어 비교 작성

---

## 향후 변경 기록 (개발팀 인계 후)

이 섹션은 개발팀이 인계받은 후의 변경을 기록하기 위한 자리입니다.

```
## YYYY-MM-DD — 변경 요약
- 변경 1
- 변경 2
```
