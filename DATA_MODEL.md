# 데이터 모델 정의

> Firestore project: `hanasm-chatbot-terms`

본 문서는 회의 도구가 사용하는 Firestore 컬렉션·필드 구조를 정의합니다. 개발팀이 인계받아 ERP·챗봇·상품DB와 자동 연동 시 본 구조를 참조하면 됩니다.

---

## 컬렉션 일람

| 컬렉션 | 역할 | doc ID 형식 |
|---|---|---|
| `erp_terminology` | 용어 데이터 본체 (1행 = 1 doc) | `{group}::{erp}` (예: `material::포맥스3T`) |
| `erp_term_logs` | 모든 변경 이력 (UNDO 근거 + 감사 로그) | auto ID |
| `erp_term_order` | 행 순서 · 소구분 순서 (드래그앤드롭 결과) | `{group}::{subgroup}` 또는 `__sub__::{group}` |
| `erp_notes/{section}/entries` | 섹션별 협의 메모 + 댓글 | auto ID, parentId 로 트리 |

> `erp_subgroup_order` 라는 별도 컬렉션은 보안규칙 제약으로 인해 `erp_term_order` 안에 `__sub__::` prefix doc id 로 통합 저장됨.

---

## 1. `erp_terminology` — 용어 본체

### 필드 정의

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `group` | string | ✓ | `machine` / `stage` / `material` 중 하나 |
| `subgroup` | string |  | 소구분 이름 (예: `공정 1 · UV인쇄 관련 장비`). 정규화 — 끝의 `(N)` 카운트 제거됨 |
| `erp` | string | ✓ | ERP 원본 명칭 (예: `포맥스3T`) |
| `mp` | string |  | 가이드 표준명 (예: `포맥스`) |
| `custom` | boolean |  | 신규 추가된 행이면 `true`. 정적 HTML 행은 미설정 |
| `draft` | string |  | 1차 수정안 |
| `draft_by` | string |  | 1차 입력자 이름 |
| `draft_at` | timestamp |  | 1차 입력 시각 |
| `final` | string |  | 최종 협의안 |
| `final_by` | string |  | 최종 입력자 이름 |
| `final_at` | timestamp |  | 최종 입력 시각 |
| `confirmed` | object \| null |  | 전체동의 정보. `{ by: string, at: ISO 문자열 }` |
| `rationale` | string |  | 결정 사유 메모 |
| `rationale_by` | string |  | 사유 작성자 |
| `rationale_at` | timestamp |  | 사유 작성 시각 |
| `downstream` | object |  | 다운스트림 반영 상태. 키 = `erp`/`productdb`/`chatbot`, 값 = `{ by, at }` |
| `copies` | array\<string\> |  | 공정(stage) 그룹 — 이 행이 복사된 소구분 이름들 |
| `updatedAt` | timestamp |  | 마지막 갱신 |
| `updatedBy` | string |  | 마지막 갱신자 |

### 예시

```json
{
  "group": "material",
  "subgroup": "포맥스(폼보드) 계열",
  "erp": "포맥스3T",
  "mp": "포맥스",
  "draft": "포맥스 3T",
  "draft_by": "송기용 팀장",
  "draft_at": "2026-06-02T13:28:00Z",
  "final": "포맥스 3T",
  "final_by": "박상준 차장",
  "final_at": "2026-06-04T11:35:00Z",
  "confirmed": {
    "by": "박상준 차장",
    "at": "2026-06-04T11:48:00.000Z"
  },
  "rationale": "두께별 분리 유지 — 생산팀 자재 관리 정확도 우선",
  "downstream": {
    "erp": { "by": "박상준 차장", "at": "..." },
    "productdb": { "by": "홍재이 주임", "at": "..." }
  },
  "updatedAt": "...",
  "updatedBy": "박상준 차장"
}
```

### 다운스트림 반영 대상

다운스트림 처리(ERP·상품DB·챗봇 자동 반영)는 **`confirmed.by` 가 존재하는 doc 만** 대상으로 합니다.

---

## 2. `erp_term_logs` — 변경 이력

모든 변경 작업이 자동으로 기록됩니다. UNDO 기능의 근거이자 감사 로그.

### 필드 정의

| 필드 | 타입 | 설명 |
|---|---|---|
| `key` | string | 대상 doc id (예: `material::포맥스3T`) |
| `group` | string | machine/stage/material |
| `erp` | string | 대상 ERP 명칭 |
| `subgroup` | string | 대상 소구분 |
| `field` | string | 변경된 필드 라벨 (예: `draft`, `(전체동의)`, `(복사)`) |
| `before` | string | 변경 전 값 |
| `after` | string | 변경 후 값 |
| `beforeData` | object | `del-term` 시 doc 전체 보존 (UNDO 복원용) |
| `beforeOrder` / `afterOrder` | array | 순서 변경 시 |
| `action` | string | 액션 종류 (아래 표) |
| `editor` | string | 작업자 이름 |
| `ts` | timestamp | 작업 시각 |
| `undoOf` | string | UNDO 액션이면 원본 로그 id |
| `undoTargetAction` | string | UNDO 대상 액션 종류 |

### action 종류

| action | 의미 |
|---|---|
| `create` / `update` / `delete` | 1차·최종 협의안 입력·수정·삭제 |
| `add-term` | 신규 항목 추가 |
| `del-term` | 신규 항목 삭제 |
| `edit-term` | 신규 항목 수정 |
| `confirm` / `unconfirm` | 전체동의 토글 |
| `deploy` / `undeploy` | 다운스트림 반영 토글 |
| `rationale-create` / `rationale-update` / `rationale-clear` | 사유 변경 |
| `reorder` | 행 순서 변경 |
| `reorder-subgroup` | 소구분 순서 변경 |
| `copy-term` / `uncopy-term` | 복사본 추가 · 제거 |
| `undo-claim` / `undo` | UNDO 진행 · 완료 |

---

## 3. `erp_term_order` — 행 순서 · 소구분 순서

### 필드 정의

| 필드 | 타입 | 설명 |
|---|---|---|
| `group` | string | machine/stage/material |
| `subgroup` | string | 소구분 (행 순서 doc), `__sub__` (소구분 순서 doc) |
| `order` | array\<string\> | ERP 명칭 배열 (행 순서) 또는 소구분 이름 배열 (소구분 순서) |
| `isSubgroupOrder` | boolean | 소구분 순서 doc 표시 |
| `updatedAt` / `updatedBy` | | |

### doc id 패턴

- 행 순서: `{group}::{subgroup}` 예: `material::포맥스(폼보드) 계열`
- 소구분 순서: `__sub__::{group}` 예: `__sub__::stage`

---

## 4. `erp_notes/{section}/entries` — 섹션별 협의 메모

### 섹션 (`section` 키)

| 섹션 | 의미 |
|---|---|
| `machine` | 1. 사용장비 |
| `stage` | 2. 공정 |
| `material` | 3. 재질 (4. 정합성 검토 포함) |
| `conflict` | 5. 주요 불일치 |

### 필드 정의

| 필드 | 타입 | 설명 |
|---|---|---|
| `editor` | string | 작성자 |
| `text` | string | 본문 |
| `ts` | timestamp | 작성 시각 |
| `parentId` | string \| null | 댓글이면 부모 메모 id. 최상위 메모면 미설정 |

---

## 권장 인덱스

```javascript
// erp_term_logs 에 대해
collection: erp_term_logs, field: ts (Descending), Query scope: Collection
collection: erp_term_logs, field: undoOf (Ascending) + ts (Ascending), Query scope: Collection

// erp_notes/{section}/entries 에 대해
collection: erp_notes/{section}/entries, field: ts (Ascending), Query scope: Collection
```

---

## 권장 보안규칙

`SECURITY.md` 참조. 현재는 임시 전체 허용 상태이며 운영 단계 진입 시 즉시 강화 필요.

---

## 데이터 무결성 보장

다음 race condition·데이터 손실 시나리오에 대한 방어 코드가 적용되어 있습니다 (적대적 검증 결과 13건 수정 완료):

| 시나리오 | 방어 코드 |
|---|---|
| `editTerm` rename 시 newKey 데이터 덮어쓰기 | `getDoc(newKey)` + nested merge |
| `copies` 배열 동시 변경 | `arrayUnion` / `arrayRemove` |
| `toggleConfirm` · `toggleDownstream` · `saveRationale` race | `runTransaction` |
| 다른 클라이언트의 doc 삭제 후 좀비 행 | `_lastSeenKeys` diff 정리 |
| UNDO 동시 클릭 | claim 메커니즘 (트랜잭션) |
| `del-term` 복원 정보 부족 | `beforeData` 에 doc 전체 보존 |
| 부정 입력 (제어문자 · 길이) | `validateTermName` 가드 |
| 드래그 중 onSnapshot 재배치 | `_dragSrc` 확인 후 일시중지 |

자세한 내역은 `ROADMAP.md` 의 "적대적 검증 이슈 트래커" 참조.
