# 보안 현황 및 강화 가이드

> **⚠️ 본 문서는 인수인계 시점 기준으로 도구의 현재 보안 상태를 정직하게 기록합니다. 운영 단계 진입 전 반드시 강화 작업 수행 필요.**

---

## 현재 상태 (회의 운영 단계)

### 임시 무방비 상태
도구가 회의용 협업 도구로 빠르게 작동하도록 **보안을 의도적으로 풀어둔 상태**입니다.

| 항목 | 현재 | 위험 |
|---|---|---|
| 페이지 URL | GitHub Pages 공개 (`sangjunepark-beep.github.io/hanasm-guide/erp-compare.html`) | URL만 알면 누구든 접속 |
| Firestore 보안규칙 | `allow read, write: if true;` | 인증 없이 모든 컬렉션 읽기/쓰기/삭제 가능 |
| 페이지 내 인증 | 없음 — 이름 드롭다운만 | "박상준 차장" 선택만 하면 본인 행세 가능 |
| Firebase API 키 | 페이지 소스에 노출 | 콘솔에서 직접 Firestore 조작 가능 |
| 세션 관리 | `localStorage.hanasm_erp_editor` 만 | 추적 불가 |
| 감사 로그 | `editor` 필드 (사용자 자가 신고) | 위조 가능 |

### 노출 시 위험 자산

- **자재·공정·장비 전체 목록** — 경쟁사가 우리 생산 구조 파악
- **외주처 코드** (시나젯·아론·해광·신애드 등)
- **ERP 내부 코드 번호** (예: 104·105·107)
- **회의 사유·결정 과정** — 영업비밀
- **Phase 2 이후** ERP·챗봇 자동 연동 시 → **외부에서 ERP·챗봇에 영향 줄 수 있는 통로**

---

## 단계별 강화 방안

### 1단계 — 임시 비밀번호 게이트 (회의 종료 직후 즉시)
**소요**: 30분
**방법**: 페이지 진입 시 비밀번호 입력 (회의 끝나면 변경)

```javascript
// erp-compare.html 상단에 추가
(function(){
  const PW = 'hanasm-2026-internal';  // 정기 변경
  if(sessionStorage.getItem('pw_ok')!=='1'){
    const v = prompt('내부 통일 도구 — 비밀번호 입력:');
    if(v !== PW){ document.body.innerHTML = '접근 거부'; throw new Error('blocked'); }
    sessionStorage.setItem('pw_ok','1');
  }
})();
```

**한계**: 비밀번호 노출 위험 항상 존재. Firestore 보안규칙 자체는 그대로 열려 있어서 Firebase 콘솔로 직접 접근하면 우회됨.

### 2단계 — Firebase Auth + 보안규칙 잠금 (Phase 2 시작 전 필수)
**소요**: 4~6시간
**방법**:
1. Firebase Console → Authentication → Sign-in method → Email/Password 활성화
2. 5명 계정 생성:
   - 대표님
   - 박상준 차장 (jimrn22@gmail.com)
   - 송기용 팀장
   - 천현정 차장
   - 강두원 팀장
3. 페이지에 Firebase Auth UI 추가 (이름 드롭다운 → 로그인 화면 대체)
4. 보안규칙 강화:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // 회사 5명만
    function isAllowed(){
      return request.auth != null && request.auth.token.email in [
        'representative@hanasignmall.com',
        'jimrn22@gmail.com',
        'song@hanasignmall.com',
        'cheon@hanasignmall.com',
        'kang@hanasignmall.com'
      ];
    }
    match /erp_terminology/{doc=**} { allow read, write: if isAllowed(); }
    match /erp_term_order/{doc=**} { allow read, write: if isAllowed(); }
    match /erp_term_logs/{doc=**} { allow read, write: if isAllowed(); }
    match /erp_notes/{section}/entries/{doc=**} { allow read, write: if isAllowed(); }
    // 외부 모든 접근 차단
    match /{document=**} { allow read, write: if false; }
  }
}
```

5. 도구의 `editor` 필드는 `request.auth.token.name` 또는 email 기반 자동 입력 (자가 신고 → 인증 기반 변경)

### 3단계 — 회사 도메인 한정 (Google Workspace SSO)
**소요**: 1일
**방법**: Firebase Auth Identity Provider 로 Google Workspace 연동. `@hanasm.kr` 또는 `@hanasignmall.com` 도메인 사용자만 허용.

```javascript
function isAllowed(){
  return request.auth != null
    && request.auth.token.email_verified
    && request.auth.token.email.matches('.*@hanasm\\.kr');
}
```

### 4단계 — 사내망 호스팅 또는 인증 프록시 (운영 진입 시)
**소요**: 1~2주
**방법** (택1):
- **A. 정적 호스팅 (사내 Nginx)** + Nginx Basic Auth 또는 IP 화이트리스트
- **B. 사내 SSO 프록시** (예: Authelia · Caddy + OIDC) 뒤로 이전
- **C. Firebase Hosting + Domain Restriction** (`hanasm.kr` 만 허용)

GitHub Pages 에서 사내 환경으로 이전 시 코드 변경 거의 없음 — `index.html` 만 그대로 옮기면 됨. Firebase 연결은 환경 변수/설정으로 분리하면 더 깔끔.

---

## 가장 위험한 시점

**Phase 2 (ERP·챗봇 자동 반영) 시작 직전.**

이때 보안 강화 안 하면:
1. 외부에서 회의 도구에 가짜 항목 박음
2. 5명 동의 위조 (가짜 confirmed 필드 설정)
3. 자동으로 ERP·챗봇에 가짜 데이터 박힘 → **실제 상품·고객 응대에 영향**

데이터 손실이 아니라 **서비스 자체에 침투할 수 있는 통로**가 됩니다.

### 권장 일정

| 시점 | 작업 | 우선순위 |
|---|---|---|
| 회의 직후 | 1단계 비밀번호 게이트 + Firebase 규칙 임시 잠금 (`allow if false`) | 즉시 |
| 인수인계 후 1주 | 2단계 Firebase Auth | Phase 1 시작 전 |
| Phase 2 시작 전 | 3단계 도메인 한정 + 인증 기반 editor 필드 | 필수 |
| Phase 2 가동 후 | 4단계 사내망 이전 또는 인증 프록시 | 운영 정착 |

---

## 인수인계 시점 운영 규칙 (응급 조치)

개발팀이 보안 강화 작업을 시작하기 전까지의 임시 운영 규칙:

1. **회의 도구 URL을 외부에 절대 공유 안 함** — DM·메일·메신저·문서 X
2. **회의 끝나면 즉시 Firebase 규칙 임시 잠금**: 콘솔에서 `allow read,write: if false;` 게시 (1분, 누구도 못 들어옴)
3. **다음 회의 직전 다시 풀고 진행 → 끝나면 다시 잠금** (회의 외 시간은 항상 잠금)
4. **회의 화면 캡쳐를 외부 노출 안 되게** (소셜미디어 X, 외부 메일 첨부 X)
5. **Firebase API 키 노출** 은 어쩔 수 없으나, 보안규칙으로 인증 게이트만 잘 박으면 무의미해짐

---

## 추가 권장 사항

### `.gitignore` 추가
Firebase API 키가 페이지 소스에 직접 박혀 있습니다. 운영 단계로 가면 별도 config 파일로 분리하고 `.gitignore` 처리 권장.

### Audit Log 모니터링
`erp_term_logs` 컬렉션의 변경 패턴을 주기적으로 검토. 비정상 패턴(짧은 시간 대량 변경·비업무 시간대 변경 등) 감지 시 알림.

### 백업 정책
`OPERATIONS.md` 참조.
