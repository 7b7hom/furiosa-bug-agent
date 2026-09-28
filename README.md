# 🐛 furiosa-bug-agent

**Bug Memory** — FuriosaAI RNGD 기반 **사내 버그 지식 공유 Agent**

![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-1C3C3C?logo=langgraph&logoColor=white)
![Streamlit](https://img.shields.io/badge/Streamlit-FF4B4B?logo=streamlit&logoColor=white)
![LangSmith](https://img.shields.io/badge/LangSmith-1C3C3C?logo=langchain&logoColor=white)
![FuriosaAI RNGD](https://img.shields.io/badge/FuriosaAI-RNGD-615CED)

에러 스크린샷(또는 텍스트)을 넣으면, 팀이 쌓아온 과거 버그 기록에서 비슷한 사례를 찾아 원인과 해결법을 정리해주는 에이전트입니다.

Claude나 ChatGPT에 그때그때 물어보면 대화가 끝나는 순간 답도 사라집니다. 이 에이전트는 **사람이 승인한 분석 결과를 팀이 검색할 수 있는 지식으로 계속 쌓아서**, 같은 에러가 다시 발생하면 즉시 재발로 감지합니다.

> 4인 팀 미니 프로젝트 · 본인 담당: **Streamlit UI · HITL 승인 · LangSmith 연동** (`ui.py`)

## 실행 화면

에러 스크린샷 입력부터 재발 감지, 원인 분석, 과거 사례 참조, 사람 승인까지 이어지는 실제 실행 화면입니다.

<p align="center">
  <img src="assets/demo-showcase.png" alt="Bug Memory 실행 화면 — ① 에러 스크린샷 입력 ② 재발 감지 · 원인 분석 ③ 과거 사례 참조 · 사람 승인" width="100%">
</p>

## 주요 기능

| 기능 | 설명 |
|---|---|
| **스크린샷 입력** | 터미널 캡처를 올리면 비전 모델(Qwen3-VL)이 에러 타입·메시지·문제 코드를 추출합니다. 텍스트를 붙여넣어도 됩니다. |
| **과거 사례 검색** | 팀 버그 코퍼스를 임베딩으로 검색한 뒤 리랭커로 재정렬해, 가장 비슷한 과거 사례를 찾습니다. |
| **멀티 에이전트 분석** | LangGraph로 `원인 분석`과 `유사 사례 판단`을 병렬로 실행한 뒤, 결과를 합쳐 재발 판정과 예방 조치 생성으로 넘깁니다. |
| **재발 감지** | 리랭커 점수가 기준(0.5) 이상인 과거 사례가 있으면 `재발`로 판정하고, 해당 사례의 발생 횟수를 올립니다. |
| **웹 검색 보완** | 기준을 넘는 내부 사례가 없으면 에이전트가 DuckDuckGo로 공식 문서·알려진 이슈를 찾아봅니다. |
| **사람 승인 (HITL)** | 분석 결과는 사람이 **[승인하고 저장]**을 눌러야만 코퍼스에 반영됩니다. 잘못된 분석이 팀 지식으로 쌓이지 않도록 막습니다. |
| **실행 추적** | LangSmith로 각 LLM 호출과 노드별 소요 시간을 추적합니다. |

## 아키텍처

<p align="center">
  <img src="docs/architecture.svg" alt="furiosa-bug-agent 아키텍처" width="100%">
</p>

| 모듈 | 파일 | 역할 |
|---|---|---|
| RAG | `app/rag.py` | 코퍼스(`app/bugs.json`) 검색·저장, 임베딩 + 리랭커 |
| Vision | `app/vision.py` | 스크린샷 OCR, 텍스트·이미지 입력 통합 |
| Agent | `app/agent.py` | LangGraph 기반 멀티 에이전트 분석 (원인 분석 · 재발 판정 · 예방 조치) |
| **UI (본인 담당)** | `app/ui.py` | Streamlit 화면, HITL 승인, LangSmith 연동 |

### 사용 모델

모든 모델은 FuriosaAI RNGD 추론 엔드포인트에서 서빙됩니다.

| 모델 | 용도 |
|---|---|
| Qwen3-VL-32B-Instruct | 에러 스크린샷 OCR |
| GPT-OSS-120B | 원인 분석 · 유사 사례 판단 · 예방 조치 생성 |
| Qwen3-Embedding-8B | 코퍼스 벡터 검색 |
| Qwen3-Reranker-8B | 검색 결과 재정렬 |

## 개발하며 해결한 문제

- **도구 호출을 강제하면 응답이 비어버리는 문제**: 이 서빙 환경에서 `tool_choice`를 `required`나 특정 함수로 강제하면 응답이 통째로 빈 채 돌아왔습니다. `tool_choice="auto"`만 쓰고, "리랭커 점수가 0.5 미만이면 반드시 웹 검색을 호출하라"는 규칙을 프롬프트에 명시하는 방식으로 해결했습니다.
- **추론 모델의 답변 잘림**: GPT-OSS-120B는 답하기 전에 추론 토큰을 먼저 소모해서, `max_tokens`를 짧게 잡으면 JSON이 중간에 끊겼습니다. 호출 목적에 맞게 토큰 한도를 넉넉하게 조정했습니다.
- **재발 판정의 불안정성**: 처음에는 LLM이 `재발 / 재발 가능성 / 신규` 세 단계로 판정했지만, 같은 입력에도 결과가 매번 흔들렸습니다. 데모 신뢰성을 위해 리랭커 점수를 기준으로 재발 여부를 판정하도록 단순화했습니다.
- **프롬프트 품질 편차**: 모든 프롬프트를 역할·목표·맥락·출력형식(Role/Task/Context/Format) 구조와 few-shot 예시로 통일했습니다.

## 프로젝트 구조

```
furiosa-bug-agent/
├── app/            실행 코드 — agent.py / rag.py / ui.py / vision.py / bugs.json
│                   stubs.py (UI 개발용 가짜 구현), test_rag.py (RAG 유닛 테스트)
├── docs/           모듈별 설계 메모 (AGENT.md), 아키텍처 다이어그램
├── assets/         README용 실행 화면
├── requirements.txt
├── README.md
└── CLAUDE.md       AI 코딩 도구 작업 규칙 (루트 고정 — 도구가 자동으로 읽음)
```

## 설치 및 실행

> **참고**: 이 프로젝트는 FuriosaAI RNGD 강좌 기간에 한시적으로 제공된 API 엔드포인트와 키를 사용했습니다. 강좌가 끝나 현재는 해당 API에 접근할 수 없으므로, 직접 실행하려면 FuriosaAI RNGD API 키(또는 호환 엔드포인트)를 새로 발급받아야 합니다. 위 실행 화면은 강좌 기간 중 실제로 실행한 결과입니다.

### 1. 설치

```bash
git clone https://github.com/sojjeoi/furiosa-bug-agent.git
cd furiosa-bug-agent
python -m pip install -r requirements.txt
```

> **Windows에서 `pip install`이나 `streamlit` 명령이 "command not found"라고 나오면**, PATH에 잡혀 있지 않은 것입니다. `pip install ...` 대신 `python -m pip install ...`처럼 `python -m`을 붙여서 실행하세요. 아래 실행 명령도 마찬가지입니다.

### 2. API 키 설정

프로젝트 루트에 `.env` 파일을 만들고 아래 내용을 채우세요. 모델별 키는 강좌에서 배포한 "FuriosaAI RNGD 실습 가이드" 문서에서 확인할 수 있습니다.

```
FURIOSA_VL_API_KEY=...          # Qwen3-VL-32B-Instruct (화면 OCR)
FURIOSA_LLM_API_KEY=...         # GPT-OSS-120B (분석)
FURIOSA_EMBEDDING_API_KEY=...   # Qwen3-Embedding-8B (검색)
FURIOSA_RERANKER_API_KEY=...    # Qwen3-Reranker-8B (검색)
```

**(선택) LangSmith 추적**: 실행 과정을 [smith.langchain.com](https://smith.langchain.com)에서 시각적으로 보고 싶다면 아래 줄을 추가하세요. 없어도 앱은 정상 작동합니다.

```
LANGSMITH_API_KEY=...
```

`.env`는 `.gitignore`에 포함되어 있어 커밋되지 않습니다. **API 키를 코드나 GitHub에 직접 올리지 마세요.**

### 3. 실행

```bash
python -m streamlit run app/ui.py
```

브라우저가 자동으로 열립니다. 에러 메시지를 텍스트로 붙여넣거나 스크린샷을 업로드한 뒤 **[분석 시작]**을 누르면 됩니다. 분석 결과를 확인하고 **[승인하고 저장]**을 눌러야만 팀 코퍼스(`app/bugs.json`)에 반영됩니다.

> 첫 분석은 여러 LLM 호출이 순차적으로 이어져서 시간이 다소 걸립니다. 정상 동작이며, LangSmith 트레이스에서 단계별 소요 시간을 확인할 수 있습니다.

### 4. 데모 시나리오

1. **신규 사례**: 코퍼스에 없는 새 에러 입력 → `신규` 판정 확인
2. **재발 감지**: 방금 승인한 것과 동일한 에러를 다시 입력 → `기존 사례와 동일한 원인으로 재발` 경고와 함께 발생 횟수가 올라가는지 확인

## Non-goals (이번 범위에서 하지 않는 것)

- 실제 Slack/Notion 채널 연동, 사용자 인증, 여러 프로젝트·조직 동시 지원
- 예방 조치의 자동 적용·자동 검증(테스트 실행 등) — 제안까지만 하고 실제 적용은 사람이 직접 함
- 자동 민감정보 마스킹 — 업로드 전 육안 확인 원칙으로 대체
