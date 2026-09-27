# ViMax 전수조사 분석 정리 📽️✨

> 작성: 카리나 (Claude Code) · 정리일: 2026-09-27
> 이 문서는 ViMax 저장소 전수조사(187개 파일 / 코드 약 14,400줄) 결과와
> 설치·활용·수익화 논의를 한 곳에 정리한 기록입니다.

## 🔗 깃허브 주소

| 구분 | 주소 |
|---|---|
| **내 포크 (작업 저장소)** | https://github.com/bmshin94/vimax |
| **원본 (upstream)** | https://github.com/HKUDS/ViMax |
| 원본 릴리스 v1.2.0 | https://github.com/HKUDS/ViMax/releases/tag/v1.2.0 |
| Trendshift 등재 | https://trendshift.io/repositories/15299 |
| 기술 리포트 (arXiv) | https://arxiv.org/abs/2606.07649 |
| 데모 유튜브 | https://www.youtube.com/@AI-Creator-is-here |
| uv 설치 안내 | https://docs.astral.sh/uv/getting-started/installation/ |

---

## 1. ViMax는 무엇인가

**"아이디어 한 줄 → 완성된 영상"을 자율로 수행하는 에이전틱(agentic) 영상 제작 프레임워크.**
감독 / 시나리오작가 / 프로듀서 / 촬영감독 역할을 각각 LLM 에이전트로 만들어 한 팀처럼 오케스트레이션한다.

| 항목 | 내용 |
|---|---|
| 개발 | HKUDS (홍콩대 Data Intelligence Lab) — LightRAG, RAG-Anything, AutoAgent 제작팀 |
| 패키지명 | `autolongvideogeneration` v1.2.0 |
| 언어 | Python 3.12 (uv) + Node.js 18+ (TUI / Web UI) |
| 라이선스 | **MIT** (상업적 이용·수정·재판매 자유) |
| 커밋 수 | 88개 (주 저자 Xavier Huang 55, xvrrr_lxhuang 18) |
| 지원 OS | Linux, Windows (공식 표기 기준) |

### 해결하려는 문제 (README 첫 화면)
- ❌ 대부분의 AI 영상 도구는 몇 초짜리 클립만 생성
- ❌ 컷마다 캐릭터·배경이 딴 것으로 바뀜 (일관성 붕괴)
- ❌ 그림만 생성되고 각본·오디오·서사 구조가 없음

---

## 2. 폴더 구조 (전수조사 실측)

```
ViMax/
├── agents/          🎭 13개 LLM 에이전트 (핵심 두뇌)
├── pipelines/       🏭 3개 생산 라인 (idea2video / script2video / novel2movie)
├── agent_runtime/   🤖 자체 제작 에이전트 루프 (= 미니 Claude Code)
├── tools/           🔌 외부 AI API 어댑터 14개
├── interfaces/      📐 Pydantic 데이터 스키마 (Scene/Shot/Camera/Frame/Character…)
├── utils/           🧰 재시도·레이트리밋·robust JSON 파서·영상편집
├── configs/         ⚙️ YAML 설정 (API 키 입력 위치)
├── prompts/         📜 에이전트 행동 규범 (workflow.md, agent.md)
├── ui/              💻 터미널 UI (Ink = 터미널용 React)
├── web/             🌐 브라우저 워크스페이스 (React 18 + Vite + node server.mjs)
├── vimax_benchmark/ 📊 일관성 평가용 스토리 35편 (Type A/B/C)
└── tests/           ✅ pytest 25개 파일 (hang / crash / wrong-output 가드)
```

### 2-1. `agents/` — AI 스태프 13명

| 파일 | 직책 | 역할 |
|---|---|---|
| `screenwriter.py` | 시나리오 작가 | 아이디어 → 스토리 → 각본 |
| `script_planner.py` (434줄) | 총괄 기획 | 각본 → 장면 구조 설계 |
| `script_enhancer.py` | 각색 | 각본 디테일 보강 |
| `storyboard_artist.py` (275줄) | 스토리보드 | 각본 → 샷 리스트(콘티) |
| `character_extractor.py` | 캐스팅 | 등장인물 자동 추출(성별/나이/외모/의상) |
| `character_portraits_generator.py` | 프로필 촬영 | **정면/측면/후면 3면도** 생성 |
| `camera_image_generator.py` (273줄) | 촬영감독 | **카메라 트리** 구성 + 첫 프레임 생성 |
| `reference_image_selector.py` (237줄) | 자료 담당 | 컷마다 최적 레퍼런스 선택 |
| `best_image_selector.py` | 감수 | 후보 이미지 중 가장 일관된 것 채택 |
| `novel_compressor.py` | 편집 | 장편 소설 핵심 압축 |
| `event_extractor.py` | 플롯 분석 | 소설 → 사건 단위 분해 |
| `scene_extractor.py` | 각색 | 사건 → 장면 (FAISS RAG) |
| `global_information_planner.py` (368줄) | 연속성 관리 | 장면 간 동일 인물 병합 |

### 2-2. `pipelines/` — 3개 워크플로우

```
Idea2Video:
  아이디어 → 기획서 → 캐릭터 → 각본 → 스토리보드 → 샷분해
        → 카메라트리 → 프레임프롬프트 → 키프레임 → 영상클립 → 최종영상

Script2Video (822줄):
  각본 → 캐릭터 → 스토리보드 → 샷분해 → 카메라트리
      → 프레임 → 키프레임 → 클립 → 최종영상

Novel2Video (1,010줄, 최대):
  소설원문 → 압축 → 사건추출 → RAG청크검색
        → 장면화 → 전역캐릭터병합 → 장면별각본
```

### 2-3. `agent_runtime/` — 자체 에이전트 프레임워크 (가장 가치 높음)

| 파일 | 역할 |
|---|---|
| `loop.py` | 에이전트 메인 루프 (`MAX_TOOL_PASSES = 50`) |
| `tools.py` (407줄) | 툴 레지스트리 + JSON Schema 자동생성 + `permission_mode` + 병렬 실행 판정 |
| `tool_executor.py` | 툴 실행 / 취소(`cancel_event`) / 진행상황 스트리밍 |
| `context_compactor.py` (254줄) | 컨텍스트 자동 압축 (토큰 압박 감지 → 요약) |
| `session_index.py` (346줄) | 세션 저장/재개 (`.vimax/sessions.json`) |
| `vimax_adapters.py` (827줄) | 파이프라인을 에이전트 툴로 노출 |
| `llm.py` | OpenAI 호환 스트리밍 클라이언트 |
| `image_tools.py` | `view_image` — 에이전트가 생성 이미지를 직접 확인 |

**에이전트 툴 목록**
- 도메인 툴 3개: `vimax_narrative_planning`, `vimax_novel_planning`, `vimax_render_video`
- 기본 툴: `read_file`, `read_json`, `write_json`, `list_files`, `glob_files`, `search_text`,
  `memory_read/write`, `todo_read/write`, `sleep`, `run_shell`, `view_image`

**이벤트 스트림 타입**: `turn` / `token` / `tool_start` / `tool_progress` / `tool_result` /
`terminal` / `status` / `session` / `done` / `error`

### 2-4. `tools/` — 외부 AI 어댑터

| 종류 | 지원 |
|---|---|
| 이미지 | Nanobanana(Google 직결 / yunwu 경유), Doubao Seedream, OpenRouter GPT Image 2 |
| 영상 | Google **Veo 3.1**, Veo(yunwu), Doubao **Seedance**, Omni, OpenRouter |
| LLM | OpenAI 호환 전체 / OpenRouter / **MiniMax-M3** (프리셋 내장) |
| 리랭커 | BGE-reranker-v2-m3 (SiliconFlow) — 소설 RAG용 |

실측 엔드포인트: `yunwu.ai`(중국 API 중계), `openrouter.ai`, `ai.google.dev`

### 2-5. `vimax_benchmark/` — 자체 평가셋 35편

- **Type A**: 한 인물 × 환경 변화 (사계절 정원사, 극한기후 화가 등)
- **Type B**: 고정 공간 × 다중 요소 (보드게임카페, 빅토리안 저택 등)
- **Type C**: 2~3인 상호작용 (커플 요리, 밴드 합주, 형사 심문 등)

각 JSON에 샷별 `first_frame`(이미지 프롬프트) + `video_prompt`(영상 프롬프트) 수록.
"왼쪽 눈썹의 초승달 흉터"처럼 식별 특징을 매 샷에 반복 명시해 일관성을 강제하는 방식.

### 2-6. 프론트엔드 2종

- **TUI**: Ink(터미널 React) → `vimax tui` / `tui new` / `tui resume <id>`, 슬래시 명령 `/compact`
- **Web**: React 18 + Vite + `server.mjs`. REST API 13개
  (`/api/events` SSE, `/api/sessions`, `/api/history`, `/api/artifacts`, `/api/artifact`,
  `/api/uploads`, `/api/agent/start|stop`, `/api/messages`, `/api/config`, `/api/health`).
  기본 바인딩 `127.0.0.1:4173`, 다크모드 지원.

---

## 3. 일관성 확보 기술 (핵심 차별점)

AI 영상의 최대 난제인 "컷마다 인물이 바뀜"을 5중으로 방어한다.

1. **3면도 프로필 선생성** — 캐릭터 신분증 확보
2. **레퍼런스 자동 선택** — 컷마다 필요한 참조 이미지만 주입
3. **카메라 트리** — "부모(와이드샷)가 자식(클로즈업)을 포함" 관계를 LLM이 추론 → 공간 좌우 반전 방지
4. **N장 생성 → 최고작 선별** (`best_image_selector`: 인물 일관성 / 공간 일관성 / 설명 정확도 3축 채점)
5. **첫프레임 또는 첫+끝프레임 → 영상 변환**으로 클립 간 연결

---

## 4. 설치 및 사용법

### 준비물
- Python 3.12+ (uv), Node.js 18+, ffmpeg (moviepy / scenedetect 사용)

### STEP 1 — 클론 & 설치
```bash
git clone https://github.com/bmshin94/vimax.git
cd vimax
uv sync
```

### STEP 2 — 설정 (가장 중요)
```bash
cp configs/agent.example.yaml configs/agent.local.yaml
```
```yaml
llm:
  model_provider: openai      # OpenAI 호환은 모두 openai
  model: gpt-5.5
  base_url: https://openrouter.ai/api/v1
  api_key: <KEY>

image:
  model: gemini-3.1-flash-image-preview
  base_url: https://yunwu.ai
  api_key: <KEY>

video:
  model: veo3.1-fast
  base_url: https://openrouter.ai/api/v1
  api_key: <KEY>

# 소설(novel2video) 워크플로우에서만 필요
embedding: { model: text-embedding-3-small, ... }
reranker:  { model: BAAI/bge-reranker-v2-m3, ... }
```
- `configs/*.local.yaml`, `.env*` 는 `.gitignore` 등록됨 (커밋 안 됨)
- 환경변수가 YAML보다 우선: `VIMAX_LLM_API_KEY`, `VIMAX_IMAGE_API_KEY`, `VIMAX_VIDEO_API_KEY` 등

### STEP 3-A — 터미널 UI
```bash
cd ui && npm install && cd ..
vimax tui            # 현재 세션 재개
vimax tui new        # 새 세션
vimax tui resume <session_id>
```

### STEP 3-B — 웹 UI (권장)
```bash
cd web && npm install && npm run dev
# http://127.0.0.1:4173
ssh -N -L 4173:127.0.0.1:4173 user@server   # 원격 서버일 때 포트포워딩
VIMAX_WEB_PORT=4174 npm run dev             # 포트 변경
```

### STEP 3-C — 스크립트 직접 실행
`configs/idea2video.yaml`에 키 입력 후 `main_idea2video.py` 내부 변수 수정:
```python
idea = "고양이와 강아지가 새 고양이를 만나면?"
user_requirement = "아이들용, 3장면 이내"
style = "Cartoon"
```
```bash
uv run main_idea2video.py     # 결과: .working_dir/idea2video/
```

### 실사용 참고
- 모든 산출물은 `.working_dir/<session_id>/` 에 저장됨 (아티팩트의 authority)
- `.vimax/sessions.json` = 세션 인덱스, `.vimax/memory.md` = 사용자 선호만 저장
- 에이전트는 사용자가 워크플로우를 **명시적으로 선택**할 때까지 플래닝 툴을 호출하지 않음
- 기본 플랜은 의도적으로 작게: **1장면 3~5샷** (비용 방어)
- `vimax web start` = 빌드된 앱 서빙 (`npm run build` 선행)

---

## 5. 플러그인? 스킬? MCP? → 셋 다 아님

| 후보 | 판정 | 근거 |
|---|---|---|
| Claude Code 플러그인 | ❌ | `.claude-plugin/`, `plugin.json` 없음 |
| Skill | ❌ | `SKILL.md` 없음 (`CLAUDE.md`는 이 저장소에서 추가한 페르소나 파일) |
| MCP 서버 | ❌ | MCP SDK 의존성 없음, JSON-RPC/stdio 서버 구현 없음 |
| **독립 실행형 Python 앱** | ✅ | `pyproject.toml` + 자체 CLI/TUI/Web |
| **자체 에이전트 프레임워크** | ✅ | `agent_runtime/`에 루프·툴·세션 자체 구현 |

ViMax는 **에이전트 호스트 본인**이다. 남의 에이전트에 붙는 부품이 아니라, 스스로 LLM을 호출하고
툴을 실행하는 주체이므로 구조적으로 Claude Code와 같은 계층에 있다.

> 기회: `vimax_adapters.py`의 툴 3개를 MCP tool로 래핑하면 Claude Code / Cursor에서
> "영상 만들어줘" 한 마디로 동작하게 만들 수 있다. 예상 작업량 1~3일.

---

## 6. API 토큰 필요 여부 → **필수, 최소 3종**

| 용도 | 필수 | 후보 |
|---|---|---|
| LLM (각본/기획/판단) | ✅ | OpenAI 호환 전체, OpenRouter, MiniMax-M3 |
| 이미지 생성 | ✅ | Nanobanana, GPT Image 2, Seedream |
| 영상 생성 | ✅ | Veo 3.1, Seedance 2.0 Fast, Omni |
| Embedding | 🟡 소설모드 | text-embedding-3-small 등 |
| Reranker | 🟡 소설모드 | SiliconFlow BGE |

### 비용 구조
```
총비용 = LLM(소) + 이미지(중) + 영상(대) ← 영상이 원가의 80~95%
```
- Veo 3.1급은 초 단위 과금. 20샷 영상 1편이 수 달러~수십 달러 수준
- 재시도 / 다중 생성(best_image_selector)까지 포함하면 더 증가
- **정확한 단가는 시점마다 달라지므로 과금 전 각 provider 가격표 확인 필수**

### ViMax의 비용 방어 장치
```yaml
max_requests_per_minute: 2
max_requests_per_day: 10
```
`utils/rate_limiter.py`(레이트리밋), `utils/retry.py`+tenacity(재시도),
`tests/test_hang_guards.py`(무한 대기 방지), 기본 소규모 플랜, 아티팩트 캐시 재사용.

---

## 7. 왜 GitHub에서 유명한가 (7가지 이유)

1. **제작팀이 이미 스타 랩** — HKUDS (LightRAG, RAG-Anything, AutoAgent, MiniRAG)
2. **문제 제기가 정확** — AI 영상 써본 사람 전원이 겪는 고통을 README 3줄로 명시
3. **"Agentic"이 현 최대 유행 키워드** + 영상 생성 결합
4. **데모 영상 9개를 README에 직접 임베드** — 결과물 선공개가 최강 마케팅
5. **실제 완성도** — 파이프라인 3종 동작, 프론트 2종, 테스트 25파일, 벤치마크 35편, 기술 리포트
6. **MIT + uv + 영/중 이중 문서 + WeChat/Feishu 커뮤니티** — 기업 도입 장벽 낮음
7. **Trendshift 등재(#15299)** — 트렌딩 선순환 진입

---

## 8. 로컬 에이전트 구축에 도움이 되는가 → 매우 큼

`agent_runtime/`은 "에이전트 프레임워크 제작 교재"에 가깝다.

| 배울 것 | 위치 |
|---|---|
| 툴 레지스트리 + JSON Schema 자동생성 | `tools.py` |
| 권한 모드(`permission_mode`) | `tools.py` |
| 병렬 실행 판정(`concurrency_safe`, `partition_calls`) — 읽기 병렬 / 쓰기 순차 | `tools.py` |
| 컨텍스트 자동 압축 | `context_compactor.py` |
| 세션 저장/재개 | `session_index.py` |
| 이벤트 스트림 프로토콜 | `loop.py` |
| 취소 처리(`cancel_event`) | `tool_executor.py` |
| 프롬프트 조립 + `prompt_trace` 디버깅 | `prompts.py` |
| 환각 방지 규칙 | `prompts/agent.md` |
| 1 런타임 → 3 프론트엔드 | `main_agent.py` + `ui/` + `web/` |

### 특히 차용할 가치가 큰 3가지

**① 환각 차단 조항** (`prompts/agent.md`)
> "Do not claim that planning, rendering, or file edits happened unless a tool result or
> `.working_dir` state proves it."

**② 비용 작업 전 확인 게이트** (`prompts/workflow.md`)
> 사용자가 워크플로우를 명시적으로 선택하기 전에는 플래닝 툴 호출 금지.

**③ JSONL stdio 아키텍처**
```
main_agent.py --jsonl  ←stdio→  Ink TUI
                       ←stdio→  node server.mjs ──SSE──→ React Web
```

### 한계
- MCP 미지원 / 서브에이전트 개념 없음 / 툴 권한 모델 단순 / 영상 도메인에 결합됨

---

## 9. React / PHP로 만들 수 있는가

### React → 이미 React로 구현돼 있음
`web/` = React 18 + Vite + TypeScript
(`App.tsx` 893줄, `ArtifactViews.tsx` 479줄, `api.ts`, `events.ts`, `theme.ts`, vitest 테스트)
의존성: react 18.3, react-markdown, remark-gfm, lucide-react, yaml
→ **재작성 불필요. 브랜딩·결제 추가 커스터마이징만 하면 제품화 가능.**

### PHP → 전체 포팅은 비권장, 하이브리드 권장

전체 포팅이 어려운 이유: `moviepy`/`opencv`/ffmpeg 영상 합성, `asyncio` 대량 병렬,
`faiss-cpu` 벡터검색, `langchain`+Pydantic 구조화 출력, 장시간 작업(PHP-FPM 타임아웃 상성).

**권장 아키텍처**
```
React 프론트 (web/ 재활용)
        ↓ REST
PHP / Laravel  — 회원·인증·결제(토스/아임포트/Stripe)·크레딧 차감·관리자·작업 큐
        ↓ 잡 투입 (Redis / DB)
Python 워커 = ViMax 엔진 그대로 (main_agent.py --jsonl / 파이프라인 직접 호출)
        → 진행률 콜백 → PHP → SSE → 프론트
```
장점: 각 언어가 강점 영역만 담당 / 업스트림 `git pull` 동기화 유지 / 워커 분리 배치로 비용 최적화.
소요: 전체 포팅 수개월 + 업스트림 동기화 영구 포기 vs 하이브리드 MVP 2~3주.

---

## 10. 수익화 아이디어

### 전제 3가지
1. **원가 지배 요인은 영상 API** → 샷 수 통제 + 저렴한 영상 모델 + 재시도 최소화가 핵심.
   반드시 **선불 크레딧제**로 설계 (정액 무제한은 위험).
2. **법적 체크**: ViMax 코드 상업적 이용은 MIT로 자유(고지문 유지). 단
   **생성 영상의 상업적 사용 가능 여부는 각 provider 약관 확인 필수**.
   AI 생성물 표시 의무·워터마크(SynthID 등) 규제, 실존 인물/캐릭터 침해 금지,
   AutoCameo(얼굴 업로드)는 개인정보·딥페이크 규제 주의.
3. **"설치 대행"은 상품이 안 됨** (오픈소스라 누구나 무료). 결과물 또는 편의성을 팔아야 함.

### 아이디어 9개

| # | 아이디어 | 고객 | 가격대 | 난이도 | 마진 | 추천 |
|---|---|---|---|---|---|---|
| ① | **웹소설·웹툰 프로모 영상** | 작가·웹툰 스튜디오·출판사 | 편당 15~50만원 | ⭐⭐ | 🟢🟢 | ⭐⭐⭐⭐⭐ |
| ② | **커머스 상품 영상 SaaS** | 스마트스토어·쿠팡 셀러 | 크레딧제 / 월 3~10만원 | ⭐⭐⭐ | 🟢 | ⭐⭐⭐⭐⭐ |
| ③ | **AutoCameo 선물 영상** | 개인(생일·결혼·돌잔치) | 건당 2~5만원 | ⭐⭐ | 🟢 | ⭐⭐⭐⭐ |
| ④ | 숏폼 채널 운영 | (직접 콘텐츠 사업) | 애드센스·브랜드딜 | ⭐⭐ | 🟡 | ⭐⭐ |
| ⑤ | **B2B 사내 교육·안전 영상** | 제조·건설·병원·프랜차이즈 | 편당 100~500만원 | ⭐⭐⭐⭐ | 🟢🟢 | ⭐⭐⭐⭐ |
| ⑥ | 화이트라벨 / 구축형 SI | 기업 | 구축 2천만~1억 + 유지보수 | ⭐⭐⭐⭐ | 🟢🟢 | ⭐⭐⭐ |
| ⑦ | **ViMax MCP 서버** | 개발자 | 오픈소스 + 클라우드 유료 | ⭐⭐ | 🟡(간접 큼) | ⭐⭐⭐⭐ |
| ⑧ | 프리셋 / 템플릿 마켓 | 사용자 | 3~10만원 | ⭐ | 🟢🟢 | ⭐⭐⭐ |
| ⑨ | 강의·뉴스레터·컨설팅 | 개발자·기업 | 5~20만원 / 일 100~300만원 | ⭐⭐ | 🟢🟢 | ⭐⭐⭐⭐ |

**핵심 포인트**
- ① 은 ViMax의 Novel2Video가 정확히 이 용도로 설계돼 있고, 한국 웹소설/웹툰 시장 규모가 커서 적합도 최상.
- ② 는 셀러 영상 외주비(편당 20~50만원)와 원가(수천원) 차이가 커서 가격 경쟁력이 압도적.
- ③ 은 선물 시장이라 가격 저항이 낮고 결과물 자체가 SNS 바이럴이 됨.
- ⑤ 는 단가가 가장 크고 다국어(외국인 노동자 교육) 확장이 프롬프트 변경만으로 가능.
- ⑦ 은 아직 아무도 만들지 않아 선점 가치가 있고, 개인 브랜딩 효과가 직접 수익보다 큼.

### 실행 로드맵
```
1~2주   ViMax 실제 구동 + 원가 실측(샷당 단가 확인) + 데모 3편 포트폴리오
3~4주   ⑦ MCP 서버 오픈소스 공개(화제성) + ① 작가 5명 무료 제공(반응 측정)
2~3개월 ① 또는 ③ 유료화 (수동 운영으로 시작) + React 커스텀 + PHP 결제 연동
4~6개월 ② 커머스 SaaS 정식 런치 (하이브리드 아키텍처) + ⑨ 강의 제작
6개월+  ⑤ B2B 교육영상 / ⑥ 화이트라벨 영업
```

### 경고 3가지
1. 가격표를 만들기 전에 실제로 1편 생성해 **청구서로 원가를 실측**할 것
2. **정액 무제한 금지** — 선불 크레딧제 필수
3. 각 provider 약관에서 **생성물 상업적 이용 가능 여부·워터마크 정책** 확인

---

## 11. 한 줄 결론

> **ViMax는 "AI 영상 제작사"를 코드로 구현한 MIT 오픈소스이고,
> 영상 기능 못지않게 `agent_runtime/`(에이전트 런타임 설계)의 학습 가치가 크다.
> 수익화는 코드 판매가 아니라 "결과물 + 편의성" 판매로 접근하고,
> 성패는 영상 생성 API 원가 관리에 달려 있다.**

---

*생성 도구: [Claude Code](https://claude.com/claude-code)*
