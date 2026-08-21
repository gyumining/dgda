# PLAN.md — 파생 코드 실행 계획

데모 대시보드를 **실제 동작하는 진단 도구 세트**로 확장하는 계획.
어느 로컬에서든 이 문서를 보고 이어서 작업한다. 각 Phase는 독립적으로 완결되며,
**"PLAN.md Phase N 시작하자"** 한 문장으로 착수할 수 있게 작성했다.

> 진행하면서 각 Phase의 체크박스를 갱신하고, 설계가 바뀌면 이 문서와 CLAUDE.md를 함께 커밋할 것.

> **구현 세부 명세는 [docs/DESIGN.md](docs/DESIGN.md)에 있다** — 결과 JSON 스키마 전문,
> 판별식 규칙 파일 형식, 44개 항목별 엔드포인트·evaluator 표, CLI·환경변수 정의, 오류 처리 정책.
> 각 Phase 착수 전에 해당 섹션을 먼저 읽을 것.

## 목표

1. **`tools/collector`** — Okta·GitHub·Cloudflare·1Password API를 실제로 호출해, 팀이 설계한
   판별식대로 O/X/C/M을 판정하고 결과 JSON을 내놓는 Python CLI (Compliance-as-Code)
2. **`tools/report`** — 결과 JSON을 받아 Daily Security Posture Report(HTML → PDF)를
   자동 생성하는 코드

두 도구는 **동일한 결과 JSON 스키마**를 계약(contract)으로 공유한다. 스키마의 원형은
대시보드의 `MOCK_DATA` 항목 구조다 (CLAUDE.md의 "진단 데이터 모델" 참고).

## 최종 리포 구조 (목표)

```
tools/
  schema/
    result.schema.json      # 결과 JSON 스키마 (JSON Schema draft-07+)
  collector/
    cli.py                  # 진입점: python -m tools.collector --saas okta,github --out data/results/
    rules.py                # 판별식 로더 (rules/*.json을 읽어 판정 엔진에 공급)
    rules/                  # SaaS별 판별식 정의 — 코드가 아닌 데이터로 관리
      okta.json  github.json  cloudflare.json  onepassword.json
    saas/
      base.py               # 공통: 인증 헤더, 페이지네이션, 재시도, rate limit
      okta.py  github.py  cloudflare.py  onepassword.py
  report/
    generate.py             # 결과 JSON → HTML 보고서 (→ PDF)
    templates/report.html   # 보고서 템플릿 (대시보드 buildPrintReport 디자인 이식)
data/
  sample-results.json       # MOCK_DATA 44건을 스키마 형식으로 추출한 샘플 (테스트 픽스처 겸용)
  results/                  # 실제 수집 결과 (.gitignore — 커밋 금지)
```

## 공통 원칙 (모든 Phase 적용)

- **자격증명은 절대 커밋하지 않는다.** API 토큰은 `.env`(gitignore됨)로만 관리하고,
  read-only 최소 권한 스코프로 발급 (예: Okta Read-Only Admin, GitHub fine-grained read 권한)
- **판별식·임계치는 팀이 확정한 기준을 그대로 옮긴다.** 근거는 대시보드 `MOCK_DATA`의
  `criteria` 필드와 SSPM 통합 매핑표(Excel, 리포 외부). 임의 변경 금지
- 수집기는 **조회(GET)만 수행** — 대상 SaaS 설정을 절대 변경하지 않는다
- Python 3.11+, 의존성 최소화 (`httpx` 또는 `requests`, `python-dotenv` 정도)
- 문서·주석·커밋 메시지는 한국어

## Phase 0 — 스키마 확정 + 샘플 데이터 추출 *(기반 작업, 반나절)*

두 도구의 계약을 먼저 고정한다.

- [ ] `tools/schema/result.schema.json` 작성 — 항목 스키마는 `MOCK_DATA` 필드 그대로
      (`id, isms, control, area, target, item, measured, criteria, api, result, reason, rem`)
      + 실행 메타데이터를 감싸는 봉투(envelope) 추가:
      ```json
      {
        "run": { "timestamp": "ISO8601", "collector_version": "0.1.0",
                 "targets": ["okta"], "tenant": {"okta": "https://..."} },
        "items": [ { ...MOCK_DATA 항목 구조... } ]
      }
      ```
- [ ] 대시보드 HTML에서 `MOCK_DATA` 44건을 추출해 `data/sample-results.json` 생성
      (envelope 포함, 스키마 검증 통과 확인)
- [x] `.gitignore`에 `.env`, `data/results/` 추가 *(계획 수립 시 선반영 완료)*
- **완료 기준**: 샘플 JSON이 스키마 검증을 통과하고, 이후 Phase의 테스트 픽스처로 쓸 수 있다

## Phase 1 — 수집기 골격 + Okta 구현 *(1~2일)*

가장 판별식이 명확한 Okta(10개 항목)로 전체 파이프라인을 관통시킨다.

- [ ] `cli.py`: `--saas`, `--out`, `--dry-run`(수집 없이 rules 나열) 인자 처리
- [ ] `saas/base.py`: Bearer/토큰 헤더, 페이지네이션(Okta는 Link 헤더), 429 재시도
- [ ] `rules/okta.json`: OT-01~OT-10 판별식을 데이터로 정의 —
      항목당 `{ id, isms, control, area, item, api, criteria_desc, evaluator, threshold }`
- [ ] `saas/okta.py`: 엔드포인트 호출 → 실측값 산출 → 판별식 적용 → 스키마 형식 항목 생성
      - OT-04(MFA), OT-07(IP Zone), OT-08(Log Streaming), OT-10(Device Assurance)이 X 재현되는지 확인
      - C(수집완료) 항목(OT-01·06·09)은 실측값만 채우고 판정 보류로 처리
- [ ] 결과를 `data/results/{timestamp}.json`으로 저장
- **완료 기준**: 실제 Okta 트라이얼 테넌트에 돌려 10개 항목 결과 JSON이 나오고,
  기존 진단 보고서의 판정과 일치한다

## Phase 2 — GitHub · Cloudflare · 1Password 수집기 *(2~3일)*

Phase 1 패턴을 복제해 나머지 3개 SaaS를 채운다. SaaS별 주의점:

- [ ] **GitHub** (GH-01~12): REST API v3, `Accept: application/vnd.github+json`.
      2FA 미설정 멤버(`?filter=2fa_disabled`)는 Org owner 권한 필요.
      GH-06은 API가 마지막 로그인을 안 주므로 C 처리 (기존 판정과 동일)
- [ ] **Cloudflare** (CF-01~13): 계정 레벨(`/accounts/{id}/...`)과 존 레벨(`/zones/{id}/...`)
      토큰 권한이 다름 — `.env`에 account_id, zone_id 분리.
      Gateway/Access/Device Posture는 Zero Trust 활성화 필요 — 미활성 시 X가 아니라
      "기능 미사용" 판정을 어떻게 처리할지 팀 기준 확인 후 반영
- [ ] **1Password** (1P-01~09): Connect API(로컬 :8080) 사용.
      1P-07·08·09는 API 미지원으로 **M(수동확인) 고정** — 수집기는 M 항목에
      콘솔 확인 경로(`rem` 필드)를 안내로 출력
- [ ] SaaS 하나라도 자격증명이 없으면 해당 SaaS만 건너뛰고 나머지는 정상 수행
- **완료 기준**: 4개 SaaS 44개 항목 전체가 한 번의 실행으로 결과 JSON에 담긴다

## Phase 3 — 보고서 자동 생성기 *(1~2일)*

- [ ] `report/generate.py`: 결과 JSON(또는 `data/sample-results.json`) 입력 →
      `templates/report.html` 렌더링 → PDF 출력
      - 위험도 산정은 대시보드 `riskProfile()` 로직을 그대로 포팅
        (X + [인적 보안|인증 및 권한관리|암호화] → 상/P0, 그 외 X → 중/P1, O → 하, C/M → 중/P3)
      - 구성: 총평(보안 점수·상태별 건수) → SaaS별 현황 → 조치 우선순위(P0/P1/P3) →
        상세 항목 표 — 대시보드 `buildPrintReport()` 구성을 참고
- [ ] PDF 변환: 1순위 `무설치` 경로(HTML을 브라우저 인쇄로), 자동화 필요 시 `weasyprint` 검토
- [ ] 샘플 데이터로 생성한 보고서가 기존 Security Posture Report(PDF) 구성과 대응되는지 확인
- **완료 기준**: `python -m tools.report data/sample-results.json` 한 줄로 보고서 HTML/PDF가 나온다

## Phase 4 — 대시보드 연동 *(선택, 반나절)*

- [ ] 대시보드에 "결과 JSON 불러오기" 추가 (`<input type=file>` → 스키마 검증 → `liveData` 주입)
      — 정적 단일 파일 원칙 유지, fetch 서버 의존 금지
- [ ] 수집기 결과를 불러오면 `isDemoMode` 해제되고 실제 데이터 배지 표시
- **완료 기준**: 수집기 → JSON → 대시보드 → 보고서까지 파이프라인이 수동 개입 없이 이어진다

## 검증 원칙

- 각 Phase 완료 시 `data/sample-results.json` 기반 스키마 검증을 통과해야 한다
- 실제 테넌트 수집 결과의 판정(O/X)이 기존 진단 보고서·대시보드 데이터와 다르면,
  **코드를 의심하기 전에 팀 판별식 기준과 대조**하고 차이 사유를 기록한다
- 존재하지 않는 진단 결과를 지어내지 않는다 (CLAUDE.md 작업 규칙과 동일)

## 다른 로컬에서 시작하는 법

```
git pull
claude
> PLAN.md 읽고 Phase 0부터 시작하자
```

자격증명이 필요한 Phase 1부터는 `.env`에 토큰을 준비한다 (형식은 Phase 1에서
`.env.example`로 커밋할 것 — 값 없이 키 이름만).
