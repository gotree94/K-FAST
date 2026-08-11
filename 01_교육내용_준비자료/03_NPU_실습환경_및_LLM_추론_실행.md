# 모듈 3. NPU 실습환경 및 LLM 추론 실행

> **파일 경로** : `01_교육내용_준비자료/03_NPU_실습환경_및_LLM_추론_실행.md`
> **성격** : 실습 기반. RNGD 서버 환경을 확인하고 furiosa-llm으로 실제 LLM 추론을 실행·검증하는 모듈.
> **예상 시수** : 이론 1시간 + 실습 3시간 (교육에서 가장 실습 비중이 높은 모듈)

---

## 1. 학습 목표

1. 실습 서버의 GPU/NPU 환경을 확인하고 드라이버·패키지 설치 상태를 점검할 수 있다.
2. furiosa-llm 설치 및 모델 다운로드(HF 허브) 절차를 이해한다.
3. Python API(`LLM`, `SamplingParams`, `generate`, `stream_generate`, `chat`)로 오프라인 추론을 실행할 수 있다.
4. OpenAI 호환 API 서버(`furiosa-llm serve`)를 기동하고 curl/HTTP로 호출할 수 있다.
5. vLLM 호환 코드로 전환하는 방법을 이해한다.
6. 흔한 오류(드라이버 미인식, HF 토큰, 메모리 부족, 컴파일 시간)에 대응할 수 있다.

---

## 2. 실습 환경 구성

### 2.1 하드웨어/OS 요건

| 항목 | 요건 |
|------|------|
| OS | Linux (Ubuntu 계열 권장) |
| 가속기 | Furiosa RNGD NPU 1장 이상 |
| 드라이버 | Furiosa 커널 드라이버 설치 + 재부팅 후 디바이스 인식 확인 |
| 컨테이너 | Docker 사용 시 `--device /dev/rngd` 및 `--security-opt seccomp=unconfined` 필요 |
| 계정 | GPU/NPU 리소스가 개별 할당되는 경우 본인 장치 번호 사용 (예: npu0) |
| 인터넷 | HF 허브 모델 다운로드 필요. `HF_TOKEN` 환경변수(게이트 모델) 준비 |

### 2.2 환경 확인 명령

```bash
# NPU 인식/정보 확인
furiosa-smi info          # 각 NPU 전력·온도·PCI 정보
furiosa-smi status        # NPU 상태 + 코어(PE)별 활용률
furiosa-smi topo          # NPU 토폴로지 / NUMA
furiosa-smi ps            # NPU를 점유 중인 프로세스
```

- `furiosa-smi info`에 장치가 안 보이면 드라이버 설치/재부팅 점검.
- `status`에서 코어 활용률이 `-` 로 표시되면 현재 유휴 상태.

### 2.3 패키지 설치

```bash
# (예시) pip 기반 설치
pip install furiosa-smi furiosa-llm
# 또는 배포 채널(APT, Docker 이미지) 사용 — 공식 문서의 설치 안내 참고
```

- 설치 후 `furiosa-llm --version`, `fxb --help` 등으로 정상 설치 확인.

---

## 3. LLM 추론 실행 (실습 본론)

### 3.1 오프라인 배치 추론 (Python API)

```python
from furiosa_llm import LLM, SamplingParams

# 사전 컴파일된 FXB 포함 모델(HF 허브) 로드
llm = LLM(model="furiosa-ai/Qwen3-8B-FP8")

prompts = [
    "K-FAST 교육에서 NPU를 학습하는 이유를 한 문장으로 설명해 주세요.",
    "RNGD NPU의 주요 사양을 정리해 주세요.",
]

sampling_params = SamplingParams(
    temperature=0.7,
    max_tokens=256,
)

outputs = llm.generate(prompts, sampling_params)
for output in outputs:
    print("=" * 60)
    print("PROMPT :", output.prompt)
    print("ANSWER :", output.outputs[0].text)
```

**핵심 API** :
- `LLM(model=...)` : 모델 로드(컴파일 결과/FXB 자동 해석). `devices` 파라미터로 NPU 지정 가능.
- `SamplingParams(temperature, top_p, max_tokens, ...)` : 생성 옵션.
- `generate()` : 배치 추론.
- `stream_generate()` : 비동기 스트리밍(토큰 생성 즉시 반환).
- `chat()` : 채팅 템플릿을 내부 처리한 대화 추론.

### 3.2 스트리밍 추론

```python
import asyncio
from furiosa_llm import LLM, SamplingParams

async def main():
    llm = LLM(model="furiosa-ai/Qwen3-8B-FP8")
    params = SamplingParams(max_tokens=128)
    async for token in llm.stream_generate("점심 메뉴 추천해 주세요.", params):
        print(token.outputs[0].text, end="", flush=True)

asyncio.run(main())
```

- 스트리밍을 쓰면 첫 토큰(TTFT)부터 실시간 출력 → 사용자 체감 응답을 보여줄 때 유용.

### 3.3 채팅 방식

```python
from furiosa_llm import LLM, SamplingParams

llm = LLM(model="furiosa-ai/Qwen3-8B-FP8")
messages = [
    {"role": "system", "content": "당신은 AI 반도체 교육 조교입니다."},
    {"role": "user", "content": "NPU란 무엇인지 초보자에게 설명해 주세요."},
]
resp = llm.chat(messages, SamplingParams(max_tokens=200))
print(resp.outputs[0].text)
```

- 채팅 템플릿 적용을 엔진이 자동 처리.

### 3.4 OpenAI 호환 API 서버

```bash
# 서버 기동 (기본 포트 8000)
furiosa-llm serve furiosa-ai/Qwen3-8B-FP8 \
    --devices npu:0 \
    --host 0.0.0.0 --port 8000

# 다른 터미널에서 호출
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "furiosa-ai/Qwen3-8B-FP8",
    "messages": [{"role": "user", "content": "RNGD를 소개해 주세요."}],
    "max_tokens": 128
  }'
```

- 응답은 OpenAI 형식(`choices[].message.content`)으로 반환.
- `/v1/models`, `/v1/completions` 도 지원 → 기존 OpenAI SDK/애플리케이션과 호환.
- 참고 : 서버는 한 번에 하나의 모델만 호스팅(현재).

### 3.5 vLLM 호환 코드 전환

- 기존 vLLM 코드에서 import를 `vllm` → `furiosa_llm`으로 바꾸고, `model`을 `furiosa-ai/*` 형식으로 지정하는 정도로 이식 가능:
  ```python
  # vLLM: from vllm import LLM, SamplingParams
  from furiosa_llm import LLM, SamplingParams
  ```
- `LLM`, `LLMEngine`, `AsyncLLMEngine` API 호환.

---

## 4. 실행 결과 확인 방법

- **정상 동작 확인** :
  - 프롬프트별 출력 토큰이 생성됨.
  - `furiosa-smi status`에서 NPU 코어 활용률이 상승함(예: 90% 이상).
  - `furiosa-smi ps`에서 본인 프로세스가 NPU를 점유 중인지 확인.
- **성능 확인** : `--prompt-log` 옵션 또는 서버 로그에서 TTFT·tokens/s 확인 (모듈 4에서 상세).

---

## 5. 트러블슈팅 가이드 (교육 시 반드시 준비)

| 증상 | 원인/대응 |
|------|-----------|
| `furiosa-smi`에 장치 없음 | 드라이버 미설치/미로드 → 드라이버 재설치 후 재부팅 |
| `/dev/rngd` 접근 거부 | 컨테이너에 `--device /dev/rngd` 추가, 권한 확인 |
| 모델 다운로드 실패 / 401 | 게이트 모델은 `HF_TOKEN` 필요. 모델 라이선스 동의 확인 |
| "Device already in use" | 다른 사용자가 해당 NPU 점유 중 → `furiosa-smi ps`로 확인 후 빈 장치로 `--devices npu:N` 지정 |
| 첫 실행이 오래 걸림 | 컴파일/가중치 다운로드 시간(수 분~수십 분). 이후 FXB 캐시로 빠름 |
| CUDA 관련 오류 메시지 | GPU가 아니라 NPU용 코드인지 확인. `CUDA_VISIBLE_DEVICES` 불필요 |
| OOM / 메모리 부족 | 모델 크기·동시성·`max_model_len`(KV 캐시) 조정, 다른 장치로 분산 |
| 서버 ready가 안 됨 | 첫 실행 컴파일+다운로드로 수 분 소요. 로그 모니터링 후 대기 |
| 응답 품질 이상 | temperature/top_p 등 SamplingParams 조정, 프롬프트 재작성 |

---

## 6. 실습 설계 (교육 진행 시나리오)

> 실습 인원이 카드에 비해 많으면 **1명당 NPU 1장 또는 카드 공유(PE 분할)** 로 진행.
> 모델은 작은 것(Qwen3-0.5B/1.5B → 8B)부터 단계적으로.

| 단계 | 시간 | 활동 | 산출물 |
|------|------|------|--------|
| 1 | 20분 | 환경 점검: `furiosa-smi info/status/topo`, `furiosa-llm --version` | 점검 로그 |
| 2 | 30분 | 0.5B/1.5B 모델 오프라인 추론 (`generate`) | 출력 확인 |
| 3 | 30분 | 스트리밍(`stream_generate`) + 채팅(`chat`) | 토큰 스트리밍 확인 |
| 4 | 40분 | 8B 모델 로드, `furiosa-llm serve` 기동 | 서버 정상화 |
| 5 | 30분 | curl로 `/v1/chat/completions` 호출, OpenAI SDK 연동 | API 응답 |
| 6 | 30분 | 배치 크기·`max_tokens` 변경하며 `furiosa-smi status` 관찰 + 실습일지 작성 | 관찰 기록 |

**실습 평가 포인트** : 명령 실행 정확성, 오류 대응, NPU 활용률 확인, 결과 해석.

---

## 7. 핵심 포인트 요약

- 실습의 첫 단계는 **환경 확인**(furiosa-smi) → 장치가 안 보이면 드라이버부터.
- 모델은 HF 허브의 `furiosa-ai/*` 사전 컴파일 모델을 사용해 컴파일 없이 즉시 실행.
- 오프라인 → 스트리밍 → 채팅 → API 서버 순으로 확장하며 학습.
- vLLM 호환 API 덕분에 기존 코드 이식이 쉽다.
- 첫 실행의 "컴파일 시간"은 정상이며, 이후 FXB 캐시로 단축된다.

---

## 8. 예상 Q&A

- **Q. 반드시 사전 컴파일 모델만 써야 하나요?** → A. 아닙니다. 일반 HF 모델도 지원 아키텍처라면 furiosa-llm이 빌드/컴파일을 처리합니다. 다만 사전 컴파일 모델이 시간을 절약합니다.
- **Q. NPU를 안 쓰고 CPU로도 돌리면 되지 않나요?** → A. 가능은 하지만, 이 교육의 목적은 NPU를 활용한 고성능 추론입니다. NPU 미할당 시 CPU 실행은 교육 목표와 다릅니다.
- **Q. 여러 사람이 한 서버를 쓸 때 어떻게 나누나요?** → A. 카드 단위(`--devices npu:0,1,2,3` 등)로 분리하거나, PE 단위/멀티인스턴스로 분할합니다. `furiosa-smi status`로 점유 상태를 확인합니다.
- **Q. `furiosa-llm serve`와 `vLLM`은 어떤 관계인가요?** → A. furiosa-llm은 vLLM 호환 API를 제공하는 RNGD 전용 추론 엔진입니다. RNGD에서는 vLLM 대신 furiosa-llm을 사용합니다.
