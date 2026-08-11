# 모듈 2. Furiosa AI RNGD 및 Software Stack 소개

> **파일 경로** : `01_교육내용_준비자료/02_Furiosa_RNGD_및_소프트웨어_스택.md`<br>
> **성격** : 이론 기반. RNGD 하드웨어의 세부 사양과 FuriosaAI 소프트웨어 스택의 역할을 이해하는 모듈.<br>
> **예상 시수** : 이론 2.5시간 + Q&A 0.5시간<br>

---

## 1. 학습 목표

1. FuriosaAI와 RNGD의 위치(세대, 용도, 차별점)를 설명할 수 있다.
2. RNGD의 주요 사양(아키텍처, 정밀도별 성능, 메모리, 인터페이스, 전력)을 정확히 답할 수 있다.
3. 소프트웨어 스택의 계층(드라이버/펌웨어 → 컴파일러 → 런타임 → LLM 서빙/도구)과 각 계층의 역할을 설명할 수 있다.
4. Furiosa Compiler가 만드는 ENF, furiosa-llm의 FXB 개념을 구분 설명할 수 있다.
5. furiosa-smi, furiosa-bench, furiosa-perf 등 제공 도구를 용도에 맞게 소개할 수 있다.

---

## 2. FuriosaAI 소개

- 2017년 설립된 국내 AI 반도체(Neural Processing Unit) 스타트업.
- **1세대 NPU : Warboy** — 데이터센터/에지 추론용. 첫 양산·실서비스 사례.
- **2세대 NPU : RNGD** ("레니게이드"로 발음) — LLM·멀티모달·비전 모델 등 고성능 추론 서빙용.
- RNGD의 강조점 : **고성능 + 저전력(TDP 150W 수준, 공랭 데이터센터 배포 가능)**, HBM3 대용량 메모리, TCP(텐서 축약 프로세서) 아키텍처.
- 차별점 : 동급 성능 대비 낮은 전력 소모, 추론 전용 최적화, 개방형 소프트웨어 스택(HF 허브 모델 배포, vLLM 호환 API).

---

## 3. RNGD 하드웨어 상세

### 3.1 기본 사양 요약 (암기 권장)

| 항목 | 사양 |
|------|------|
| 아키텍처 | Tensor Contraction Processor (TCP) |
| 공정 | TSMC 5nm |
| 클럭 | 1.0 GHz |
| 연산 성능 | BF16 256 TFLOPS / FP8 512 TFLOPS / INT8 512 TOPS / INT4 1,024 TOPS |
| 메모리 | HBM3 48GB (2개 스택), 대역폭 1.5 TB/s |
| 온칩 SRAM | 256MB (온칩 대역폭 약 384TB/s) |
| 호스트 인터페이스 | PCIe Gen5 x16 (단방향 최대 128GB/s 수준) |
| TDP | 150W (자료에 따라 180W 표기 — 배포 시 데이터시트 확인) |
| 쿨링 | 패시브(수동) 쿨링, 공랭 서버(최대 8개 카드/4U) 구성 가능 |
| 폼팩터 | PCIe 듀얼 슬롯 풀하이트 3/4 길이 |
| 멀티인스턴스 | 8개 (PE 단위) |
| 가상화 | SR-IOV 8 Virtual Functions 지원 |
| 신뢰성 | ECC 메모리, Secure Boot(Root of Trust) |

### 3.2 TCP 아키텍처 개념

- 기존 NPU가 "합성곱/행렬 곱 명령 단위"로 최적화된 것과 달리, TCP는 **텐서 축약(곱과 합) 자체를 하드웨어에서 직접 실행**하는 구조.
- 텐서 축약을 네이티브하게 실행 → 연산 유닛 활용률과 에너지 효율 최대화.
- 칩 내부는 **8개 PE(Processing Element)** 로 구성되며, 각 PE는 독립된 주소 공간과 자원을 가짐. (FP8 기준 PE당 약 64 TFLOPS)
- PE 내부·PE 간 통신은 메모리 라우터/NoC로 처리되며, HBM3 2개 스택으로 각 PE가 256GB/s 수준의 대역폭을 확보.
- 다중 칩 확장 : 칩 간 P2P 통신(PCIe 기반, 방향당 최대 64GB/s)을 통해 한 서버에서 여러 RNGD를 하나처럼 사용.

### 3.3 전력·신뢰성 설계

- DVFS(동적 전압·주파수 스케일링)로 전력 효율 최적화.
- 온다이/온인터포저 디커플링 커패시터로 급격한 전류 변화 대응.
- 타이밍 마진 모니터 등 데이터센터 안정성 설계 내장.

### 3.4 카드/서버 구성 예시

| 구성 | 용도 |
|------|------|
| RNGD 1카드 (1개 칩, 8 PE) | 8B급 LLM(예: Llama-3.1-8B, Qwen3-8B), BERT 등 |
| RNGD 2~4카드 | 32B~70B급 LLM(예: EXAONE-4.0-32B, Llama-2/3 70B) 텐서 병렬 |
| 8카드/4U 공랭 서버 | 다중 모델·다중 사용자 서빙, 데이터센터 표준 구성 |

---

## 4. 소프트웨어 스택 상세

```
┌──────────────────────────────────────────────────────────┐
│ 애플리케이션 (웹 서비스, 챗봇, RAG, 벤치마크)                │
├──────────────────────────────────────────────────────────┤
│ 서빙 계층   furiosa-llm (vLLM 호환 API, OpenAI 호환 서버)    │
│             furiosa-server / Triton · ONNX Runtime 백엔드    │
├──────────────────────────────────────────────────────────┤
│ 런타임 계층 Furiosa Runtime (실행·스케줄링·멀티NPU)           │
├──────────────────────────────────────────────────────────┤
│ 컴파일 계층 Furiosa Compiler (그래프 최적화 → ENF 생성)       │
├──────────────────────────────────────────────────────────┤
│ PE 런타임   PERT (PE 내부 스케줄링/실행)                     │
├──────────────────────────────────────────────────────────┤
│ 펌웨어/드라이버  커널 드라이버 · NPU 펌웨어                   │
├──────────────────────────────────────────────────────────┤
│ 하드웨어    RNGD NPU (TCP · PE ×8 · HBM3 · SRAM)           │
└──────────────────────────────────────────────────────────┘
```

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| OS/디바이스 | **커널 드라이버** | Linux가 NPU를 디바이스 파일로 인식하게 함. <br>인식 안 되면 재설치 |
| NPU 내부 | **펌웨어 + PERT** | PE 자원 관리·스케줄링, 호스트 런타임과 통신 |
| 컴파일 | **Furiosa Compiler** | 모델 그래프 최적화, 연산자 융합, 메모리 할당, <br>스케줄링, 데이터 이동 최적화 → **ENF** 실행 파일 생성 |
| 실행 | **Furiosa Runtime** | 컴파일된 프로그램 로드·실행, <br>NPU/호스트 메모리 할당, 멀티 NPU 통합 관리 |
| LLM 서빙 | **furiosa-llm** | LLM/멀티모달 고성능 추론 엔진. <br>vLLM 호환 API + OpenAI 호환 서버 |
| 모니터링/관리 | **furiosa-smi** | NPU 정보·상태·활용률·전력·온도 확인 (CLI) |
| 벤치마크 | **furiosa-bench / furiosa-perf / furiosa-mlperf** | 성능 측정, LLM 서빙 벤치마크, MLPerf 실행 |
| 메트릭 | **furiosa-npu-metrics-exporter** | OpenMetrics 형식으로 지표 노출<br> → Prometheus 수집 |
| 모델 배포 | **Hugging Face 허브 (furiosa-ai/*)** | 사전 컴파일된 모델(FXB 포함) 제공 |

### 4.1 Furiosa Compiler와 ENF

- 입력 : ONNX, TFLite(및 기타 IR). **양자화된 모델**이 NPU 가속에 필수.
- 주요 최적화 : 그래프 수준 최적화, 연산자 융합(fusion), 메모리 할당 최적화, 스케줄링, 계층 간 데이터 이동 최적화.
- 출력 : **ENF(Executable NPU Format)** — NPU 실행 가능 바이너리. 재사용 가능(캐시 `$HOME/.cache/furiosa/compiler`).
- 사용법 : `furiosa-compiler model.onnx -o model.enf` 또는 런타임/백엔드를 통해 자동 컴파일.

### 4.2 Furiosa Runtime

- ENF 로드 → NPU 실행. 세션 생성 후 동기/비동기 추론.
- 여러 NPU에 작업을 분산할 수 있는 통일된 진입점 제공.
- 사용 예(Python) :
  ```python
  from furiosa.runtime import sync
  with sync.create_runner("model.enf") as runner:
      outputs = runner.run(inputs)
  ```

### 4.3 furiosa-llm (LLM 서빙 엔진)

핵심 기능:
- **vLLM 호환 API** : `LLM`, `LLMEngine`, `AsyncLLMEngine` 클래스 → 기존 vLLM 워크플로를 그대로 이식 가능.
- **PagedAttention** : KV 캐시를 페이지 단위로 관리해 메모리 효율 극대화.
- **연속 배치(Continuous Batching)** : 요청이 도착하는 대로 동적 배치 구성 → 처리량 증대.
- **Hugging Face 허브 연동** : 사전 학습 모델을 그대로 로드.
- **OpenAI 호환 API 서버** : `furiosa-llm serve`로 기본 `http://localhost:8000` 서버 기동. 채팅 템플릿 지원.
- **스트리밍 추론** : `stream_generate()`로 토큰 생성 즉시 스트리밍 가능.

### 4.4 FXB (Furiosa Executable Bundle)

- furiosa-llm이 사용하는 **컴파일 산출물 포맷**. `.fxb` 파일 하나에 컴파일 바이너리+메타데이터를 묶음.
- GPU와 달리 RNGD는 **모델별로 커널이 컴파일**되므로, 사전 빌드된 FXB를 배포·재사용하는 워크플로가 중요.
- **아키텍처 지문(Fingerprint)** : hidden_size, attention 헤드 수, MoE 전문가 수, 양자화 포맷 등 커널 생성에 영향을 주는 config 필드로 결정. 지문이 같으면 다른 모델·파인튜닝 변형에도 FXB 재사용 가능.
- 명령어 : `fxb build`(빌드), `fxb add`(캐시 등록), `fxb check`(호환성 확인), `fxb inspect`.
- 텐서 병렬 크기(-tp)는 **빌드 시 고정**, 파이프라인 병렬(-pp)·데이터 병렬(-dp)은 **서빙 시 지정**.
- HF 허브의 `furiosa-ai/*` 모델(예: `furiosa-ai/Qwen3-8B-FP8`)은 FXB를 함께 배포해 그대로 서빙 가능.

### 4.5 제공 도구 요약

| 도구 | 용도 | 주요 명령/예시 |
|------|------|----------------|
| furiosa-smi | NPU 상태·활용률·전력·온도·토폴로지 확인 | `furiosa-smi info`, `furiosa-smi status`, `furiosa-smi topo`, `furiosa-smi ps` |
| furiosa-bench | 일반 모델(ONNX/TFLite) 벤치마크(지연·QPS) | `furiosa-bench model.onnx --workload L -n 1000` |
| furiosa-perf | **LLM 서빙** 벤치마크 자동화(서버 기동→부하→HW 모니터링→보고서) | `furiosa-perf run --backend furiosa-llm --hardware-type npu ...` |
| furiosa-mlperf | MLPerf Inference 공식 벤치마크 실행 | `furiosa-mlperf bert-offline ...`, `gpt-j-offline` |
| furiosa-npu-metrics-exporter | OpenMetrics 지표 노출(Prometheus 연동) | NPU 활용률·클럭 등 |

---

## 5. 실습 설계 (예시)

| 순서 | 실습 | 목적 |
|------|------|------|
| 1 | `furiosa-smi info` 로 RNGD 인식 확인, `topo` 로 토폴로지 확인 | 하드웨어 계층 이해 |
| 2 | `furiosa-compiler` 로 간단 ONNX 모델 → ENF 생성 후 런타임 실행 | 컴파일러/런타임 역할 체감 |
| 3 | HF에서 `furiosa-ai/Qwen3-8B-FP8` 로드해 furiosa-llm으로 1회 추론 | FXB 재사용·서빙 경험 |
| 4 | `furiosa-llm serve` 기동 후 curl로 OpenAI API 호출 | 서빙 계층 이해 |

---

## 6. 핵심 포인트 요약

- RNGD = 2세대 NPU. **TCP 아키텍처 + TSMC 5nm + HBM3 48GB/1.5TB/s + 256MB SRAM + PCIe Gen5**.
- 정밀도별 성능 : BF16 256TFLOPS / FP8 512TFLOPS / INT8 512TOPS / INT4 1,024TOPS. TDP 150W(자료에 따라 180W).
- 소프트웨어 스택 : 드라이버·펌웨어 → 컴파일러(ENF) → 런타임 → furiosa-llm(FXB) → 도구(furiosa-smi/perf 등).
- 컴파일러가 "모델별" 실행 파일(ENF/FXB)을 만들고, 이를 재사용하는 것이 RNGD 운영의 핵심.
- furiosa-llm은 vLLM 호환 API + OpenAI 호환 서버를 제공해 기존 생태계와 쉽게 통합된다.

---

## 7. 예상 Q&A

- **Q. RNGD와 GPU(NVIDIA)의 차이는 무엇인가요?** → A. 같은 AI 추론을 목표로 하나, RNGD는 추론 전용 설계로 단위 전력·단위 면적당 효율이 높습니다. NVIDIA는 학습·추론 겸용 생태계가 성숙합니다. RNGD는 150W 수준 TDP로 공랭 서버 배포가 가능한 점이 장점입니다.
- **Q. ENF와 FXB는 무엇이 다른가요?** → A. ENF는 Furiosa Runtime이 실행하는 단일 모델 컴파일 결과물이고, FXB는 furiosa-llm용으로 모델 아키텍처 지문과 함께 묶인 컴파일 산출물 포맷입니다. FXB는 지문이 같으면 여러 모델에 재사용할 수 있습니다.
- **Q. 모든 모델이 바로 실행되나요?** → A. 지원 아키텍처(현재 주요 오픈 LLM 계열)와 호환되는 모델은 그렇습니다. 그 외에는 컴파일/변환 과정이 필요할 수 있습니다. HF의 `furiosa-ai/*` 사전 컴파일 모델을 사용하면 빠르게 시작할 수 있습니다.
- **Q. 여러 RNGD 카드가 필요한 경우는?** → A. 모델 용량(가중치+KV 캐시)이 카드 메모리(48GB)를 초과하거나, 처리량 확장이 필요할 때입니다. 텐서 병렬로 분산하되, 텐서 병렬 크기는 FXB 빌드 시 결정합니다.
