# CLAUDE.md — 프로젝트 컨텍스트

이 파일은 어느 로컬에서든 Claude Code가 이 프로젝트의 맥락을 갖고 시작할 수 있도록 유지하는 문서다.
**중요한 결정이나 구조 변경이 생기면 이 파일도 함께 갱신하고 커밋할 것.**

## 프로젝트 개요

**SSPM Console** — 기업용 SaaS 보안 컴플라이언스 진단 체계의 웹 데모 콘솔.
글로벌 SSPM이 다루지 않는 국내 컴플라이언스(ISMS-P) 기준으로, 4개 SaaS(Okta·GitHub·Cloudflare·1Password)의
보안 설정을 API로 수집해 적합/부적합을 자동 판정하는 진단 체계를 시연한다.

- **팀**: Team Dgda (디그다) — 2026 시큐리티아카데미 7기 컨설팅 트랙 (이서윤 PM, 오은희, 남규민, 멘토 박창현)
- **배포**: GitHub Pages → https://gyumining.github.io/dgda/
- **배경·설계·성과의 상세 설명은 [README.md](README.md) 참고** (판별식 예시, 5단계 프로세스, 멀티 에이전트 협업 방식 포함)

## 파일 구조

| 파일 | 역할 |
|---|---|
| `웹 진단 대시보드.html` | **핵심 산출물.** 빌드 없는 단일 파일 웹 데모 콘솔 (HTML+CSS+JS 전부 인라인, ~240KB) |
| `index.html` | GitHub Pages 진입점. meta refresh로 대시보드로 리다이렉트 + 카카오톡/슬랙 공유용 OG 메타 보유 (크롤러는 refresh를 안 따라가므로 OG는 여기 있어야 함) |
| `README.md` | 프로젝트 소개 문서 (GitHub 랜딩 페이지 겸용) |
| `assets/` | README용 도식·스크린샷 PNG 8장 |
| `og-image.png` | 링크 공유 미리보기 이미지 (1200×630) |
| `.claude/launch.json` | 로컬 프리뷰: `python -m http.server 8123` (`static-preview`) |

## 대시보드 내부 구조 (`웹 진단 대시보드.html`)

단일 파일이지만 내부는 명확히 구획되어 있다. 주요 지점:

### 진단 데이터 모델 — `MOCK_DATA` (약 2385행)

**다른 코드를 만들 때 재사용할 핵심 데이터셋.** 총 44개 진단 항목
(Okta 10 · GitHub 12 · Cloudflare 13 · 1Password 9). 항목 스키마:

```js
{
  id: 'OT-04',                    // SaaS 접두어 + 순번 (OT/GH/CF/1P)
  isms: '2.5.3',                  // ISMS-P 보호대책 요구사항 항목 번호
  control: '사용자 인증',          // ISMS-P 통제항목명
  area: '인증 및 권한관리',        // 분류 (인적 보안/인증 및 권한관리/접근 통제/암호화/로그 관리/기타 보안/사고 예방/시스템 개발 보안)
  target: 'Okta',                 // 대상 SaaS
  item: 'MFA Authenticator 활성화', // 점검 항목명
  measured: 'INACTIVE',           // 실측값
  criteria: 'MFA status==ACTIVE → O', // O/X 판별식 (팀이 설계한 임계치)
  api: '/api/v1/authenticators',  // 판정에 사용한 API 엔드포인트
  result: 'X',                    // O(적합)/X(부적합)/C(수집완료)/M(수동확인) — RESULT_META 참고
  reason: '...',                  // 판정 근거
  rem: '...'                      // 조치 권고 (SaaS 콘솔 경로 포함)
}
```

- 이 데이터는 실제 가상 기업 환경의 진단 결과를 각색한 것 (`SSPM_Report_최종본.xlsx`의 Detail 시트 기준). 실 자격증명 없음.
- 결과 코드: `RESULT_META` (약 3154행) — O=적합, X=부적합, C=수집완료, M=수동확인
- 위험도 산정: `riskProfile()` (약 3279행) — X이면서 area가 [인적 보안, 인증 및 권한관리, 암호화]면 '상'(8.5점, P0), 그 외 X는 '중'(6.5, P1), O는 '하'(0.3), C/M은 '중'(2.4, P3)

### 주요 함수 지도

| 영역 | 함수 (대략 행) |
|---|---|
| 진단 실행 | `runAssessment` (2628), `renderAssessmentResult` (2711) |
| 요약/차트 | `updateSummaryView` (2788), `renderCharts` (2998, Chart.js), `renderMasterPlan` (2907) |
| 테이블 | `renderActionItemTable` (3169), `renderDetailedTable` (3189) |
| PDF 보고서 | `buildPrintReport` (3292), `exportPdf` (3412) |
| 이력 | `recordAssessmentRun` (3431), `assessmentHistory` — **존재하지 않는 과거 진단을 만들어 넣지 않는다** (보고서와 교차 확인되는 공개 채널이므로) |
| UI 셸 | 리본/사이드바/탭 (`toggleSidebar` 2569, `switchTab` 2502), 다크모드 `applyDarkMode` (3791), 가이드 투어 `TOURS` (3979) |

## 작업 규칙 · 결정사항

- **빌드 도구 도입 금지** — 대시보드는 정적 단일 파일 유지 (GitHub Pages에서 그대로 서빙)
- UI·문서 언어는 한국어
- 데모 데이터를 수정할 때 **실제 진단 보고서와 어긋나는 값을 지어내지 말 것** (수치는 SSPM_Report_최종본.xlsx와 교차 확인됨)
- `index.html`의 OG 메타는 대시보드 파일과 별개로 유지 (링크 크롤러 대응)
- 로컬 미리보기: `.claude/launch.json`의 `static-preview` (포트 8123) 또는 파일 직접 열기
- 커밋 메시지는 한국어 요약체 (기존 이력 참고)

## 진행 이력 요약

1. 웹 진단 대시보드 구현 (진단 수행 → 결과 요약 → 조치 우선순위, 가이드 투어·다크모드 포함)
2. 가이드 UX 개선, 사이드바 토글, 레이아웃 버그 수정
3. GitHub Pages 배포 + `index.html` OG 메타 정비
4. README 작성·보강 (배경/진단 체계/성과/팀 + 도식 8장, 멀티 에이전트 협업 방식 섹션)

## 다음 작업 방향

대시보드의 진단 데이터(`MOCK_DATA`)와 기존 산출물(매핑표, 판별식, 보고서 체계)을 기반으로 **파생 코드 제작 예정**.
새 코드를 만들 때는:

- 진단 항목 스키마(위 데이터 모델)를 공통 포맷으로 삼을 것
- 판별식·임계치는 팀이 설계한 기준을 따르고 임의로 바꾸지 말 것
- 관련 원본 산출물: SSPM 통합 매핑표(Excel), SaaS 보안 상태 진단 보고서, Security Posture Report (리포 외부 — 필요 시 사용자에게 요청)
