# data-analysis-project

[![lang: English](https://img.shields.io/badge/lang-English-lightgrey.svg)](README.md)
[![lang: 한국어](https://img.shields.io/badge/lang-한국어-2b6cb0.svg)](README.ko.md)
&nbsp;
[![methodology: CRISP-DM](https://img.shields.io/badge/methodology-CRISP--DM-2f855a.svg)](#crisp-dm)
[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](#라이선스)

> 실제 문제를 정의하는 일부터 공유 가능한 HTML 리포트를 내놓기까지, 각 단계를 게이트로 검증하는 엔드투엔드 데이터 분석 워크플로우. **[CRISP-DM](#crisp-dm)을 구현한** Claude Code **플러그인**(단일 스킬 + 서브에이전트 3 + 스크립트 3).

🇬🇧 **English README → [README.md](README.md)**

### 무엇인가
`data-analysis-project` 는 **하나의 스킬을 패키징한 Claude Code 플러그인**입니다. "제품"은 코드가 아니라 **프롬프트 내용** — 미래의 Claude 가 데이터 분석 프로젝트를 수행할 때 읽는 지침입니다.

핵심 주장: **약한 분석은 모델링이 아니라 문제 정의·목표 설정에서 실패한다.** 그래서 각 스테이지 끝에 **게이트**를 두고, 실제로 통과해야만 다음 단계로 넘어갑니다. "재프레임(문제가 생각과 달랐다)"은 실패가 아니라 강한 결과로 취급합니다.

### 9단계 게이트 워크플로우
1. **문제 제기** — 3 테스트(구체성·실재성·실행가능성) + 사용자 확인 + 경량 상황 점검(데이터·법/라이선스·비용·리스크)
2. **분석 목표** — 한 문장 목표 + 의사결정 + **비즈니스 합격선** + **모델 무관 정량 목표**(baseline-상대 기본)
3. **데이터 선정** — 실제 샘플 확보, 스키마·키·접근·라이선스 확인
4. **데이터 이해** — `data_quality.py` 프로파일(null률·이상치·중복·분포), 규모·grain·계절성·불균형 측정
5. **분석 방법** — 하위질문→방법→검증 계획, 명명 지표 + 임계 확정(예측: baseline-first / 서술형: method-appropriate validity)
6. **데이터 준비** — 결측·이상치 처리(판단은 `data-quality` 에이전트), 게이트 = `data_quality.py --strict`, join 검증·leakage 차단·변환 로그
7. **코드 작성** — 노트북(한 셀=한 동작, 셀 사이 마크다운 해석); 예측 모델 시 fit·튜닝·후보 랭크·승자 선택
8. **결론** — **MET/PARTIAL/NOT-MET 판정**(달성 vs 임계) + 거짓 성공 금지 + 미달 시 사용자 go/no-go
9. **리포트** — 자체 완결형 **bilingual(ko/en) HTML 데이터 에세이**, 스크린샷 시각 검토 게이트 통과

진행도는 **스테이지별 Claude TODO**로 추적합니다(진입 시 in-progress, 게이트 통과 시 completed).

### <a id="crisp-dm"></a>CRISP-DM
이 플러그인은 **CRISP-DM**(Cross-Industry Standard Process for Data Mining)을 **의견을 담아 구현**한 것입니다. 9개 게이트 스테이지는 CRISP-DM 6단계에 다음과 같이 매핑됩니다:

| CRISP-DM 단계 | 이 플러그인의 스테이지 |
|---|---|
| 1. 비즈니스 이해 | **1** 문제 제기 · **2** 분석 목표 |
| 2. 데이터 이해 | **3** 데이터 선정 · **4** 데이터 이해(프로파일링) |
| 3. 데이터 준비 | **6** 데이터 준비 |
| 4. 모델링 | **5** 분석 방법 · **7** 코드 작성 |
| 5. 평가 | **8** 결론·판정 (MET/NOT-MET + go/no-go) |
| 6. 배포 | **9** 리포트 작성 |

CRISP-DM 의 **정신에 충실**합니다: 비즈니스 이해와 평가가 핵심이고, 평가 기준은 모델 지표가 아니라 **비즈니스 목표**이며, 미달 시 거짓 성공을 내보내지 않고 **되돌아갑니다**(재프레임·보완) — CRISP-DM 의 비선형 핵심 동작. 배포는 의도적으로 **최종 리포트 산출**(분석/리서치 산출물)까지로 한정하며, 모델 서빙/모니터링은 범위 밖입니다.

### 구성 요소
- **스킬** `skills/data-analysis-project/` — `SKILL.md`(항상 로드되는 진입점) + `references/`(스테이지별 상세) + `assets/report-template.html` + `scripts/`
- **서브에이전트** `agents/`
  - `data-source-scout` — Stage 3: 공개 데이터 탐색·접근 probe → 정제된 후보 shortlist 반환(메인 컨텍스트 보호)
  - `data-quality` — Stage 6: `data_quality.py` 리포트 해석 → 컬럼별 처리(impute/drop/cap/keep) 판단, leakage 플래그
  - `analysis-critic` — Stage 5·8: 방법 계획과 결론을 적대적으로 검토(거짓 성공·과대주장 탐지)
- **스크립트** `skills/.../scripts/`
  - `data_quality.py` — 엄격 프로파일러/게이트(null률·IQR+robust-z 이상치·중복·상수열)
  - `execute_notebook.py` — 노트북 end-to-end 실행·출력 임베드(Stage 7 게이트)
  - `screenshot_report.py` — 리포트 8 PNG 렌더(데스크톱/모바일 × 라이트/다크 × ko/en, Stage 9 게이트)

### 설치
```bash
# 마켓플레이스 등록 후 설치
/plugin marketplace add sciencemj/data-analysis-project
/plugin install data-analysis-project
```

### 요구 사항 (스크립트 실행 시)
```bash
uv add pandas numpy                                            # Stage 4·6  data_quality.py
uv add --dev nbclient nbformat ipykernel                       # Stage 7    execute_notebook.py
uv add --dev playwright && uv run playwright install chromium  # Stage 9    screenshot_report.py
```

### 저장소 구조
```
.claude-plugin/{plugin,marketplace}.json   # 매니페스트 + 마켓플레이스 엔트리
agents/                                     # 서브에이전트 3
skills/data-analysis-project/              # 스킬 본체 (SKILL.md · references · scripts · assets)
evals/                                      # 개발용 행동 테스트(미배포)
CLAUDE.md                                   # 저장소 개발 가이드(미배포)
```

### 개발 · 평가
`evals/evals.json` 은 외부 eval 하네스(`skill-creator`)가 소비하는 **행동 테스트 케이스**입니다(이 저장소엔 test runner 없음; assertion 은 LLM 판정). 스테이지 행동을 바꾸면 대응 assertion 도 갱신하세요.

### 라이선스
MIT
