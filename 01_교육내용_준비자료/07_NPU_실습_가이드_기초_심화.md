# NPU 실습 가이드 — 기초부터 심화까지 (Step-by-Step)

> **파일 경로** : `01_교육내용_준비자료/07_NPU_실습_가이드_기초_심화.md`
> **성격** : 모듈 3(실습환경·LLM 추론 실행)을 **기초→중급→중상급→심화→프로젝트** 5단계로
> 나눈 실습 중심 가이드. 각 실습은 "목표 → 진행 → 예상 결과 → 성공 기준 → 이해 포인트 → 자주 나는 오류" 순.
> **전제** : 모든 실습은 **노트북(SSH)에서 RNGD 서버에 접속**해 진행. 1인당 NPU 1장 또는 카드 공유.

---

## 실습 로드맵 요약

| 스테이지 | 수준 | 내용 | 예상 시간 |
|----------|------|------|-----------|
| Stage 0 | 준비 | SSH 접속·환경 점검·Python 환경 | 20분 |
| Stage 1 | 기초 | 첫 LLM 추론 (generate, SamplingParams) | 40분 |
| Stage 2 | 중급 | 스트리밍·채팅·배치·NPU 활용률 관찰 | 40분 |
| Stage 3 | 중상급 | OpenAI 호환 서버 배포·API 연동 | 50분 |
| Stage 4 | 심화 | 성능 측정·모니터링 (furiosa-perf 등) | 60분 |
| Stage 5 | 응용 | FXB 빌드·ONNX 전환·vLLM 이식·조별 프로젝트 | 90분+ |

> 한 세션에 다 못 하면 Stage 1~2(기초) → Stage 3~4(실무)로 나누어 진행해도 좋다.

---

## Stage 0. 시작하기 전에 (준비)

### 0-1. 목표
- RNGD 서버에 SSH 접속하고, NPU·SDK가 정상인지 확인한다.
- Python 가상환경을 만들 수 있다.

### 0-2. 진행
```bash
# 1) 서버 접속 (IP/계정은 운영진 안내)
ssh <user>@<server_ip>

# 2) NPU 인식 확인
furiosa-smi info      # 전력·온도·PCI 정보 출력
furiosa-smi status    # NPU 상태 + 코어(PE)별 활용률
furiosa-smi topo      # 토폴로지 / NUMA
furiosa-smi ps        # NPU를 점유 중인 프로세스 (처음엔 없음)

# 3) SDK 버전 확인
furiosa-llm --version
fxb --help | head -20

# 4) 실습용 가상환경 생성 (서버에 설치 권한이 제한된 경우)
python3 -m venv ~/llm_lab
source ~/llm_lab/bin/activate
pip install furiosa-llm   # 필요 시 (서버에 이미 설치돼 있으면 생략)
```

### 0-3. 예상 결과
- `furiosa-smi info`에 `rngd` 아키텍처의 `npu0`(및 추가 장치)가 보인다.
- `status`의 코어 활용률이 `-` (유휴)로 표시된다.
- `furiosa-llm --version`이 정상 버전을 출력한다.

### 0-4. 성공 기준
- [ ] `ssh` 접속 후 터미널 프롬프트에 서버 이름이 보인다
- [ ] `furiosa-smi info`에서 장치가 1개 이상 인식된다
- [ ] 가상환경 활성화 후 `which python`이 `~/llm_lab`을 가리킨다

### 0-5. 자주 나는 오류
| 증상 | 대응 |
|------|------|
| SSH 연결 안 됨 | IP·계정·VPN·방화벽 확인, `ssh -v`로 원인 출력 |
| furiosa-smi: command not found | 드라이버/SDK 미설치 → 운영진 문의 |
| 장치 없음 | 드라이버 미로드 → 서버 재부팅 필요할 수 있음 |

---

## Stage 1. 기초 — 첫 LLM 추론

### 1-1. 목표
- furiosa-llm의 `LLM`, `SamplingParams`, `generate` 흐름을 이해한다.
- 작은 모델(0.5B/1.5B)로 첫 추론을 실행하고 결과를 해석한다.

### 1-2. 진행
```bash
# 가상환경 안에서 실행 (Stage 0의 터미널 그대로)
python
```
```python
from furiosa_llm import LLM, SamplingParams

# 0.5B급 소형 모델 로드 (처음이면 다운로드로 수 분 소요)
llm = LLM(model="furiosa-ai/Qwen3-0.5B")

params = SamplingParams(temperature=0.7, max_tokens=64)
output = llm.generate("NPU란 무엇인가요?", params)[0]
print(output.outputs[0].text)
```

### 1-3. 예상 결과
- 로드 시 HF 허브 다운로드/컴파일 로그가 보인 뒤,
- "NPU는 신경망 연산을 가속하는 반도체..." 류의 문장이 출력된다.
- 이후 다시 로드하면 FXB 캐시로 훨씬 빨라진다.

### 1-4. 성공 기준
- [ ] 출력 토큰(문장)이 생성된다
- [ ] `furiosa-smi status`를 다른 터미널에서 확인하면 코어 활용률이 상승 구간을 보인다

### 1-5. 파라미터 이해하기 (핵심 실습)
```python
# 같은 프롬프트로 max_tokens/temperature 변화를 비교
for mt in (16, 64, 256):
    out = llm.generate("한국의 수도는?", SamplingParams(max_tokens=mt))[0]
    print(mt, "->", out.outputs[0].text)

# temperature: 0.0(결정적) vs 1.5(다양성↑)
for t in (0.0, 0.8, 1.5):
    out = llm.generate("숫자 1부터 3까지 나열하세요.", SamplingParams(temperature=t, max_tokens=32))[0]
    print("temp", t, "->", out.outputs[0].text)
```

### 1-6. 이해 포인트
- `LLM(model=...)` : 모델 로드. `devices="npu:0"`처럼 장치 지정 가능.
- `SamplingParams` : 생성 옵션(온도·최대 토큰·top_p).
- `generate()` : 프롬프트 리스트를 받아 배치 처리 → 출력 리스트 반환.

### 1-7. 자주 나는 오류
| 증상 | 대응 |
|------|------|
| 다운로드 401/403 | `HF_TOKEN` 환경변수 설정, 라이선스 동의 |
| OOM | 더 작은 모델(0.5B) 사용, `max_model_len` 낮추기 |
| 첫 실행이 오래 걸림 | 컴파일+다운로드 정상 → FXB 캐시로 단축 |

---

## Stage 2. 중급 — 다양한 추론 방식

### 2-1. 목표
- 스트리밍·채팅·배치 추론을 구분해 사용할 수 있다.
- 추론 전후 NPU 활용률 변화를 관찰·해석한다.

### 2-2. 진행 A — 스트리밍 추론
```python
import asyncio
from furiosa_llm import LLM, SamplingParams

async def main():
    llm = LLM(model="furiosa-ai/Qwen3-1.5B")
    params = SamplingParams(max_tokens=128)
    # 토큰이 생성되는 즉시 출력 → 첫 토큰(TTFT) 체감
    async for token in llm.stream_generate("이야기 초반부를 시작해 주세요.", params):
        print(token.outputs[0].text, end="", flush=True)

asyncio.run(main())
```

### 2-3. 진행 B — 채팅 방식
```python
from furiosa_llm import LLM, SamplingParams

llm = LLM(model="furiosa-ai/Qwen3-1.5B")
messages = [
    {"role": "system", "content": "당신은 AI 반도체 교육 조교입니다."},
    {"role": "user", "content": "RNGD를 초보자에게 설명해 주세요."},
]
resp = llm.chat(messages, SamplingParams(max_tokens=200))
print(resp.outputs[0].text)
```

### 2-4. 진행 C — 배치 추론
```python
from furiosa_llm import LLM, SamplingParams

llm = LLM(model="furiosa-ai/Qwen3-1.5B")
prompts = [
    "첫 번째 질문: 1+1은?",
    "두 번째 질문: 대한민국의 수도는?",
    "세 번째 질문: 10보다 큰 소수 하나는?",
]
outputs = llm.generate(prompts, SamplingParams(max_tokens=32))
for o in outputs:
    print("PROMPT:", o.prompt)
    print("ANSWER:", o.outputs[0].text)
    print("-" * 40)
```

### 2-5. 진행 D — NPU 활용률 관찰 (핵심)
```bash
# 터미널 2개를 띄운다
# 터미널 1 : 실습 코드 실행
# 터미널 2 : 실행 직전/직후 상태 관찰
furiosa-smi status --loop 1      # 1초 간격 갱신
furiosa-smi ps
```
- 실행 중에는 코어(PE) 8개가 `occupied` + 활용률 90% 이상 구간이 보인다.
- 실행 종료 후 다시 유휴(`-`)로 돌아온다.

### 2-6. 성공 기준
- [ ] 스트리밍: 문자가 점진적으로 출력된다 (한꺼번에 안 나옴)
- [ ] 채팅: 시스템 프롬프트 반영된 답변이 온다
- [ ] 배치: 3개 프롬프트 모두 답이 온다
- [ ] `furiosa-smi status`에서 활용률 상승 구간을 관찰했다

### 2-7. 이해 포인트
- `generate`(한 번에 결과) vs `stream_generate`(토큰 즉시) vs `chat`(템플릿 자동 적용).
- NPU 활용률 상승 = 실제로 NPU가 연산을 수행 중이라는 증거.

### 2-8. 자주 나는 오류
| 증상 | 대응 |
|------|------|
| 스트리밍 코드 실행 오류 | `asyncio.run(main())` 구조 유지, 함수 정의 확인 |
| 활용률이 안 보임 | 관찰 터미널의 `--loop` 옵션 확인, 실행 중일 때만 보임 |

---

## Stage 3. 중상급 — OpenAI 호환 서버 배포

### 3-1. 목표
- `furiosa-llm serve`로 API 서버를 띄우고, 노트북에서 원격으로 호출한다.
- curl과 OpenAI Python SDK로 채팅/완성/스트리밍을 호출한다.

### 3-2. 진행 A — 서버 기동
```bash
# 서버 터미널에서 (기본 포트 8000, 8B 모델)
furiosa-llm serve furiosa-ai/Qwen3-8B-FP8 \
    --devices npu:0 \
    --host 0.0.0.0 --port 8000
```
- 로그에 모델 로드/컴파일 진행 후 `Uvicorn running on http://0.0.0.0:8000` 확인.
- 첫 기동은 수 분 소요 → **잠시 대기**(FXB 캐시 후 재시작은 빠름).

### 3-3. 진행 B — 노트북에서 포트포워딩
```bash
# 노트북의 새 터미널에서
ssh -L 8000:localhost:8000 <user>@<server_ip>
# 접속이 유지되는 동안 노트북 localhost:8000 == 서버의 8000
```

### 3-4. 진행 C — curl 호출 (노트북 터미널)
```bash
curl http://localhost:8000/v1/models
# -> {"data":[{"id":"furiosa-ai/Qwen3-8B-FP8", ...}]}

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "furiosa-ai/Qwen3-8B-FP8",
    "messages": [{"role": "user", "content": "RNGD를 소개해 주세요."}],
    "max_tokens": 128
  }'
# -> OpenAI 형식 JSON (choices[].message.content)
```

### 3-5. 진행 D — OpenAI Python SDK 연동
```bash
pip install openai
```
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

# 채팅
resp = client.chat.completions.create(
    model="furiosa-ai/Qwen3-8B-FP8",
    messages=[{"role": "user", "content": "Furiosa RNGD의 메모리는?"}],
    max_tokens=128,
)
print(resp.choices[0].message.content)

# 스트리밍
stream = client.chat.completions.create(
    model="furiosa-ai/Qwen3-8B-FP8",
    messages=[{"role": "user", "content": "1부터 5까지 세어 주세요."}],
    max_tokens=64,
    stream=True,
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### 3-6. 성공 기준
- [ ] `/v1/models`가 200 응답을 반환한다
- [ ] `/v1/chat/completions`가 OpenAI 형식 JSON을 반환한다
- [ ] OpenAI SDK(stream 포함)로 정상 응답을 받는다
- [ ] `furiosa-smi ps`에 서버 프로세스가 보인다

### 3-7. 이해 포인트
- furiosa-llm 서버는 OpenAI 호환 인터페이스 → 기존 앱/SDK를 바꾸지 않고 연동 가능.
- 서버는 현재 **한 번에 한 모델만** 호스팅.
- 포트포워딩 덕분에 노트북 로컬에서 실제 배포처럼 테스트할 수 있다.

### 3-8. 자주 나는 오류
| 증상 | 대응 |
|------|------|
| connection refused | 서버 기동 완료 확인(로그), 포트포워딩 유지 확인 |
| Device in use | 다른 사용자 점유 → `--devices npu:다른번호` |
| 서버 기동이 오래 걸림 | 첫 컴파일 정상 → 대기, 이후 캐시로 빠름 |

---

## Stage 4. 심화 — 성능 측정·모니터링

### 4-1. 목표
- furiosa-perf로 LLM 서빙 벤치마크를 돌리고, TTFT·TPOT·tokens/s·활용률을 해석한다.
- furiosa-bench로 일반 모델(ONNX) 성능을 측정한다.

### 4-2. 진행 A — furiosa-perf 벤치마크
```bash
# 시나리오 YAML(입력/출력 길이, 동시성) 준비 후
furiosa-perf run \
  --backend furiosa-llm \
  --hardware-type npu \
  --server-config ./server_config.yaml \
  --benchmark-config ./bench_scenario.yaml \
  --model-id furiosa-ai/Qwen3-8B-FP8
```
- 서버 자동 기동 → 벤치마크 → 전력·온도·활용률·호스트 CPU/메모리 로그 수집 → 요약 생성.

### 4-3. 진행 B — furiosa-bench (일반 모델)
```bash
# (예시) ONNX 모델 대상 지연 중심 벤치마크
furiosa-bench ./model.onnx --workload L -n 1000 -w 8 -t 2
# -> QPS, min/max/mean, p50/p95/p99 지연 출력
```

### 4-4. 진행 C — 실시간 모니터링
```bash
furiosa-smi status --loop 1          # 1초 갱신 (벤치마크 중 관찰)
furiosa-smi ps                       # 점유 프로세스
furiosa-smi info --full              # 전력·온도 상세
```

### 4-5. 결과 해석 연습 (핵심)
| 지표 | 의미 | 관찰 팁 |
|------|------|---------|
| TTFT | 첫 토큰까지 시간 | 동시성↑ → TTFT 상승 경향 |
| TPOT | 출력 토큰 1개당 시간 | 동시성↑ → TPOT 상승 |
| tokens/s | 초당 생성 토큰 | 동시성↑ → 처리량 증가 (지연과 상충) |
| 코어 활용률 | NPU 사용률 | 벤치마크 중 90% 이상이면 최적화 양호 |
| 전력/온도 | 운영 한계 | TDP(150W 수준) 근접 확인 |

### 4-6. 성공 기준
- [ ] furiosa-perf 요약(CSV/MD)이 생성된다
- [ ] 동시성 1 vs 8 vs 32 결과에서 "처리량↑ · 지연↑" 트레이드오프를 확인했다
- [ ] 모니터링 로그에서 전력·활용률 시계열을 본다

### 4-7. 자주 나는 오류
| 증상 | 대응 |
|------|------|
| furiosa-perf: NPU 모니터링 없음 | `furiosa_smi_py` 설치 여부 확인 |
| 벤치마크 결과 없음 | 시나리오 YAML 경로·포맷 확인 |
| 서버 ready 타임아웃 | 첫 컴파일이 오래 걸림 → 대기 시간 늘리기 |

---

## Stage 5. 응용·프로젝트

### 5-1. FXB 빌드 (자체 모델 컴파일)
```bash
# HF 모델을 FXB로 빌드
fxb build <hf-model-id> ./output
# 텐서 병렬 크기는 빌드 시 결정
fxb build <hf-model-id> ./output -tp 2
# 호환성 확인
fxb check ./output
```
- 성공 시 `./output.fxb` 생성 → `furiosa-llm serve`에서 재사용 가능.

### 5-2. ONNX 모델 → 컴파일 → 실행
```bash
# ONNX 모델을 ENF로 컴파일
furiosa-compiler ./model.onnx -o model.enf
```
```python
from furiosa.runtime import sync
with sync.create_runner("model.enf") as runner:
    outputs = runner.run(inputs)
```
- 일반(비LLM) 모델의 RNGD 실행 경로 체험.

### 5-3. vLLM 코드 이식
```python
# vLLM: from vllm import LLM, SamplingParams
from furiosa_llm import LLM, SamplingParams
llm = LLM(model="furiosa-ai/Qwen3-8B-FP8", devices="npu:0")
```
- import만 바꾸면 되는 수준 → 기존 코드 이식 실습.

### 5-4. 조별 프로젝트 아이디어
1. **RAG 챗봇** : 문서 검색 + furiosa-llm 서버 연동 챗봇 구축·발표
2. **벤치마크 리포트** : 모델/동시성/정밀도별 TTFT·TPOT·tokens/s·전력 비교 보고서
3. **서빙 SLO 설계** : "p99 TTFT < 2s" 등 목표 설정 → 실측 → 최적화 제안
4. **양자화 비교** : FP8 vs INT8 모델 품질·속도 비교

### 5-5. 성공 기준
- [ ] FXB 빌드→서빙까지 한 번 성공
- [ ] ONNX→ENF→Runtime 실행 성공
- [ ] 조별 프로젝트 발표물(코드+결과+해석) 완성

---

## 부록. 공통 오류 대응표 (전체 실습용)

| 증상 | 원인/대응 |
|------|-----------|
| SSH 접속 불가 | IP·계정·VPN·방화벽 / `ssh -v` |
| 장치 없음 | 드라이버 미로드 → 재부팅 필요 |
| `/dev/rngd` 거부 | 컨테이너 `--device /dev/rngd` 추가 |
| HF 다운로드 401 | `HF_TOKEN`·라이선스 동의 |
| Device in use | `furiosa-smi ps` → 빈 장치로 `--devices npu:N` |
| 첫 실행 지연 | 컴파일+다운로드 정상 → FXB 캐시로 단축 |
| CUDA 메시지 | NPU 코드인지 확인, `CUDA_VISIBLE_DEVICES` 불필요 |
| OOM | 모델 축소·`max_model_len`·동시성 조정 |
| connection refused | 서버 기동/포트포워딩 확인 |
| furiosa-perf 모니터링 없음 | `furiosa_smi_py` 설치 확인 |

---

## 부록. 실습 완료 체크리스트

- [ ] Stage 0: SSH 접속·NPU 인식·가상환경
- [ ] Stage 1: 0.5B 모델 첫 추론 + 파라미터 실험
- [ ] Stage 2: 스트리밍·채팅·배치 + 활용률 관찰
- [ ] Stage 3: 서버 기동·포트포워딩·curl·OpenAI SDK
- [ ] Stage 4: furiosa-perf 벤치마크 + 결과 해석
- [ ] Stage 5: FXB 빌드 또는 ONNX 실행 + 조별 프로젝트

> **교육 운영 팁** : 각 Stage 끝마다 "성공 기준"을 확인받고 넘어가면 낙오자를 줄일 수 있다.
> 시간이 부족하면 Stage 1~3(기초~서빙)을 필수로, Stage 4~5를 심화로 운영하세요.

