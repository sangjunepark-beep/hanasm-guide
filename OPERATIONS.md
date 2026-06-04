# 운영 가이드

> 백업·복구·모니터링·사내망 이전 절차

---

## 1. 백업

### Firestore 데이터 자동 백업 (권장)

Firebase Spark 플랜에서는 자동 일일 백업이 없습니다. 다음 방법 중 택1.

#### 옵션 A. Firebase CLI 수동 export (간단)
```bash
# gcloud SDK 설치 후
gcloud auth login
gcloud config set project hanasm-chatbot-terms
gcloud firestore export gs://hanasm-chatbot-terms-backups/$(date +%Y%m%d) --collection-ids=erp_terminology,erp_term_logs,erp_term_order,erp_notes
```

#### 옵션 B. Cloud Scheduler + Cloud Function (자동)
GCP Cloud Scheduler 로 매일 02:00 KST 자동 export 등록. Blaze 플랜 필요.

#### 옵션 C. 페이지 내 수동 CSV 백업
현재 도구의 **📥 회의록 CSV** 버튼을 매주 1회 박상준 차장이 다운로드해서 사내 공유 폴더에 보관.

### 복구
```bash
gcloud firestore import gs://hanasm-chatbot-terms-backups/{YYYYMMDD}
```

---

## 2. 모니터링

### 정상 가동 지표
- Firestore 동시 접속자 수 (Console → Firestore → 사용량)
- 일일 read/write 횟수
- `erp_term_logs` 의 신규 추가 추세

### 비정상 패턴 감지 (운영 단계 진입 시 권장)

| 패턴 | 의심 |
|---|---|
| 5초 안에 같은 키 100회 이상 변경 | 자동화 봇 또는 무한 루프 |
| 비업무 시간(22:00~07:00) 대량 변경 | 외부 무단 접근 |
| `del-term` 액션이 1분 안에 10건 이상 | 데이터 파괴 시도 |
| UNDO 액션이 짧은 시간 30건 이상 | 작업자 실수 또는 충돌 |

### 알림 채널 (권장)
Firebase Cloud Function 트리거 → 사내 Slack/이메일 알림.

---

## 3. 회의 운영 절차

### 회의 시작 전
1. Firebase 보안규칙 임시 풀기 (또는 1단계 비밀번호 게이트 활성화)
2. 회의 진행자(박상준 차장)가 라이브 URL 공유
3. 5명 편집자 모두 본인 이름 선택

### 회의 진행 중
1. 행마다 1차 수정안 → 협의 → 최종 협의안 입력
2. 합의되면 **전체동의** 체크 (해당 행 잠금)
3. 사유 메모 (왜 이걸로 정했는지) 1줄 입력
4. 잘못된 결정이면 즉시 ↩ 되돌리기

### 회의 종료 후
1. **📥 회의록 CSV** 다운로드 → 사내 공유 폴더 보관
2. **📥 주입용 CSV** 다운로드 → 다운스트림 시스템 담당자에게 전달
3. Firebase 보안규칙 다시 잠금 (`allow read,write: if false;`)
4. `HANDOVER.md` 작성 (개발팀 인계 시)

---

## 4. 사내망 이전 옵션

### A. 정적 호스팅 (사내 Nginx) — 가장 단순
```nginx
server {
  listen 443 ssl;
  server_name guide.hanasm.internal;

  ssl_certificate /etc/ssl/hanasm.crt;
  ssl_certificate_key /etc/ssl/hanasm.key;

  # Basic Auth (1단계 임시)
  auth_basic "Internal Only";
  auth_basic_user_file /etc/nginx/.htpasswd;

  root /var/www/hanasm-guide;
  index erp-compare.html;
}
```

이전 시:
1. GitHub repo 클론 → `/var/www/hanasm-guide/` 에 배치
2. `.htpasswd` 로 5명 계정 생성
3. DNS `guide.hanasm.internal` 사내망에 매핑
4. GitHub Pages 페이지 비활성화 또는 410 Gone 처리

### B. 사내 SSO 프록시
Authelia · Caddy + OIDC · oauth2-proxy 등 사용. 기존 사내 인증 시스템과 통합.

### C. Firebase Hosting 유지 + Auth 강화
인프라 변경 최소. Firebase Auth + 도메인 한정 보안규칙으로 보안 확보.

---

## 5. Firestore 컬렉션 이전

GitHub repo 만 옮기면 Firebase 는 그대로 사용 가능. 그러나 운영 분리 원하면:

```bash
# 옛 project export
gcloud firestore export gs://hanasm-chatbot-terms-backups/migration --collection-ids=erp_terminology,erp_term_logs,erp_term_order,erp_notes

# 새 project (예: hanasm-internal-guide) 생성 후
gcloud firestore import gs://hanasm-chatbot-terms-backups/migration --project=hanasm-internal-guide
```

페이지의 `firebaseConfig` 만 새 project 정보로 교체.

---

## 6. 다운스트림 자동 연동 (Phase 2 시작 시)

### ERP 자동 반영
```javascript
// Cloud Function 예시
exports.syncToERP = functions.firestore
  .document('erp_terminology/{key}')
  .onUpdate(async (change, context) => {
    const after = change.after.data();
    const before = change.before.data();

    // confirmed 상태이고 final 이 변경된 경우만 ERP 푸시
    if(!after.confirmed?.by) return;
    if(after.final === before.final) return;

    // ERP API 호출 (개발팀 구현)
    await fetch('http://erp.internal/api/material-rename', {
      method: 'POST',
      headers: { 'Authorization': 'Bearer ' + ERP_TOKEN },
      body: JSON.stringify({
        erp_id: after.erp_id,        // ERP 코드 번호 (절대 변경 안 됨)
        display_name: after.final    // 화면에 보이는 이름만 변경
      })
    });

    // downstream.erp 자동 체크
    await change.after.ref.set({
      downstream: { erp: { by: 'auto-sync', at: new Date().toISOString() } }
    }, { merge: true });
  });
```

### 챗봇 동의어 사전 자동 갱신
```javascript
exports.syncToChatbot = functions.firestore
  .document('erp_terminology/{key}')
  .onWrite(async (change, context) => {
    const after = change.after.data();
    if(!after?.confirmed?.by) return;

    // synonyms 배열 생성: 옛이름·새이름·고객표현 모두
    const synonyms = [after.erp, after.mp, after.final];
    if(after.synonyms) synonyms.push(...after.synonyms);

    await fetch('http://chatbot.internal/api/synonyms-update', {
      method: 'POST',
      body: JSON.stringify({
        primary: after.final || after.erp,
        synonyms: synonyms.filter(Boolean)
      })
    });
  });
```

### 상품DB(Sheets) 자동 동기화
Google Apps Script 트리거. `OPERATIONS.md` 의 별도 섹션 참조 또는 별도 문서로 분리.

---

## 7. 문제 발생 시 대응

| 증상 | 가능 원인 | 대응 |
|---|---|---|
| 페이지에 "연결 중..." 만 표시되고 데이터 안 나옴 | 보안규칙으로 read 차단 | 콘솔에서 임시로 `allow read: if true` 로 변경 후 원인 분석 |
| `permission-denied` 토스트 | 보안규칙 미허용 컬렉션 | `SECURITY.md` 2단계 보안규칙 다시 확인 |
| 다른 클라이언트 변경이 화면에 안 나옴 | onSnapshot 구독 실패 (네트워크) | 페이지 새로고침 |
| 같은 행이 중복 표시 | race condition (드물게) | 페이지 새로고침. 그래도 보이면 한 쪽 행 삭제 |
| UNDO 가 안 됨 | 트랜잭션 claim 충돌 | 잠시 후 재시도. 그래도 안 되면 변경 이력 직접 확인 |

---

## 8. 정기 점검 (월 1회 권장)

- [ ] Firestore 사용량 확인 (Spark 플랜 한도 초과 여부)
- [ ] `erp_term_logs` 크기 — 100,000 행 넘으면 1년 이상 분량 별도 보관
- [ ] 백업 파일 무결성 확인 (실제 복구 시뮬레이션 분기 1회)
- [ ] 보안규칙 적용 상태 확인 (실수로 풀린 상태 방지)
- [ ] 적대적 검증 이슈 트래커(`ROADMAP.md`) 신규 등록 사항 검토
