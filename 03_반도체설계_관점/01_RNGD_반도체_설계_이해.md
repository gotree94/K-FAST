# 반도체 설계 관점에서 바라본 RNGD

> **파일 경로** : `03_반도체설계_관점/01_RNGD_반도체_설계_이해.md`
> **성격** : 기존 자료(01~07)가 "사용/운영" 관점이라면, 이 섹터는 **칩 설계자·하드웨어 엔지니어의
> 관점**에서 RNGD가 왜 이런 구조인지, 내부가 어떻게 설계됐는지를 이해하는 보충 자료.
> **대상** : 반도체 설계(디지털 로직·아키텍처·전력·패키징) 배경지식을 쌓고 싶은 교강사/학습자.
> **주의** : 문서 중 "추정/계산 예시"로 표시된 수치는 공식 사양이 아닌 유도값입니다.

---

## 1. 문서 구성

| 장 | 내용 |
|----|------|
| 1 | 반도체 설계 흐름과 RNGD의 위치 |
| 2 | 디지털 회로 기초 — MAC·PE가 왜 이 모양인가 |
| 3 | TCP 마이크로아키텍처 — 8개 PE를 어떻게 묶었나 |
| 4 | 메모리 계층 설계 — SRAM/HBM3/대역폭 |
| 5 | 전력 설계 — TDP, DVFS, 디커플링 커패시터 |
| 6 | 공정·패키징 — TSMC 5nm, CoWoS-S, HBM 적층 |
| 7 | 검증·신뢰성·가상화 — ECC, RoT, SR-IOV |
| 8 | 컴파일러-하드웨어 공동 설계 — TCP의 진짜 의미 |
| 9 | 설계 관점의 성능 산술 — TOPS·대역폭·연산 밀도 |
| 10 | 핵심 요약 및 설계 관점 체크리스트 |

---

## 2. 반도체 설계 흐름과 RNGD의 위치

### 2.1 전형적인 반도체 설계 흐름

```
사양/아키텍처(스펙)
   → RTL 설계 (SystemVerilog/HLS로 PE·SRAM·NoC 로직 기술)
   → 검증 (시뮬레이션, UVM, 전력 검증)
   → 합성(Synthesis) → 물리 설계(P&R) → DRC/LVS
   → GDS 테이프아웃 → 파운드리(TSMC 5nm) 양산
   → 패키징(CoWoS-S + HBM3) → 테스트 → 카드/서버 구성
```

- RNGD는 이 흐름의 **디지털 SoC(가속기)** 편에 해당.
- 동시에 진행되는 **하드웨어/소프트웨어 공동 설계(HW-SW codesign)**가 핵심 차별점
  → 8장에서 상세.

### 2.2 설계 관점에서 RNGD가 목표로 한 것

1. **추론 전용** : 학습(역전파) 불필요 → 연산 유닛을 순전파(텐서 축약)에만 최적화.
2. **전력 효율** : 150W급 TDP로 공랭 데이터센터 배포 가능(수랭 불필요).
3. **메모리 병목 해소** : HBM3 48GB + 온칩 256MB SRAM으로 대용량 LLM도 수용.
4. **유연한 확장** : PCIe Gen5 P2P로 카드 여러 장을 하나처럼 사용.

---

## 3. 디지털 회로 기초 — MAC·PE가 왜 이 모양인가

### 3.1 MAC(Multiply-Accumulate) 연산

- 딥러닝의 기본 연산은 `y = Σ wᵢ·xᵢ` — 즉 **곱셈 한 번 + 덧셈 누적 한 번(MAC)**.
- 하드웨어에서 MAC은 **곱셈기(Multiplier) + 덧셈기(Adder) + 누산기(Accumulator)** 회로로 구현.
- MAC 1회 = 연산 2회(곱+덧셈) → TOPS 단위와 MAC 수의 관계는 9장에서 계산.

### 3.2 MAC 배열(Array)과 PE

- MAC 하나는 느리므로 **수천~수만 개의 MAC을 배열(행렬)로 배치** → 한 클럭에 행렬 곱을 병렬 처리.
- 이 MAC 배열을 묶은 것이 **PE(Processing Element)**.
- RNGD는 **PE 8개**, PE마다 8PE 전체로는 512TOPS(INT8) 수준(9장 산술 참고).
- 데이터 흐름 방식(가중치 고정/입력 고정 등)에 따라 서스톨릭 어레이(Systolic Array) 등
  다양한 배열 설계가 존재 — TCP는 "텐서 축약을 네이티브 실행"하도록 이 배열을 설계.

### 3.3 왜 정밀도가 낮으면 TOPS가 커지는가

- INT8는 INT32 대비 **비트 폭이 작아 같은 면적/클럭에 더 많은 MAC을 배치** 가능.
- 결과적으로 정밀도 낮출수록 TOPS 상승(BF16 256TFLOPS → INT8 512TOPS → INT4 1,024TOPS).
- **면적·전력·정확도의 트레이드오프**가 설계의 핵심 결정.

---

## 4. TCP 마이크로아키텍처 — 8개 PE를 어떻게 묶었나

### 4.1 칩 전체 구조 (논문 기준 요약)

| 구성 요소 | 설계 내용 |
|-----------|-----------|
| PE | 8개, 각각 독립 주소 공간 + 연산/자원. FP8 기준 PE당 약 64TFLOPS |
| 온칩 SRAM | 256MB, 온칭 대역폭 약 384TB/s — 가중치 재사용의 핵심 |
| 메모리 라우터/NoC | PE↔SRAM↔HBM 데이터 이동 담당, HBM 풀 인터리빙으로 활용 극대화 |
| HBM3 | 칩당 2스택, 48GB, 1.5TB/s — PE당 약 256GB/s 확보 |
| ATU(주소 변환 유닛) | PE 주소 공간을 추상화 → PE 간 동적 할당·통신 지원 |
| 호스트 연결 | PCIe Gen5 x16, 칩 간 P2P(방향당 최대 64GB/s) |
| 온다이 CPU 코어 | 제어용(전원·클럭·보안·부팅), 최대 2GHz, 0.75V |

### 4.2 설계 관점의 핵심 아이디어

1. **텐서 축약을 명령 단위로 하지 않음** : 기존 NPU는 합성곱/행렬 곱을 "명령 시퀀스"로
   처리하지만, TCP는 축약 연산을 하드웨어에서 직접 실행 → 활용률·에너지 효율↑.
2. **SRAM 중심 데이터 재사용** : 가중치를 SRAM에 올려 두고 재사용해 HBM 접근을 최소화
   → "메모리 병목"을 설계 단계에서 줄임.
3. **PE 독립 + 라우터** : PE가 독립 실행되므로 워크로드에 따라 동적으로 할당 가능
   (멀티인스턴스 8개 = 8개 PE를 시간/공간 분할).
4. **주소 추상화(ATU)** : PE 자원을 하나의 주소 공간처럼 노출 → 소프트웨어(런타임)가
   쉽게 멀티PE·멀티칩을 관리.

---

## 5. 메모리 계층 설계 — SRAM/HBM3/대역폭

### 5.1 메모리 계층 비교

| 구분 | 온칩 SRAM | HBM3 (외장) |
|------|-----------|-------------|
| 용량 | 256MB | 48GB |
| 대역폭 | 약 384TB/s(온칩) | 1.5TB/s |
| 지연 | 매우 낮음 | 상대적으로 높음 |
| 역할 | 가중치·중간 결과 재사용 버퍼 | 모델 전체(가중치+KV 캐시) 저장 |
| 설계 트레이드오프 | 면적·전력 큼 → 용량 제한 | 용량 큼 → 접근 비용 큼 |

### 5.2 왜 SRAM 256MB가 큰 의미인가

- LLM 가중치 중 자주 재사용되는 부분(지속 레이어)을 SRAM에 상주시키면 HBM 접근 급감.
- 대역폭 차이(384TB/s vs 1.5TB/s)를 보면, 재사용률을 조금만 높여도 병목이 줄어든다.
- **"연산은 싸고, 데이터 이동은 비싸다"** — 이 원칙이 NPU 설계의 모토.

### 5.3 KV 캐시와 메모리

- 디코드 단계는 KV 캐시(시퀀스 길이에 비례)를 매 토큰마다 읽음 → 메모리 대역폭 소모 큼.
- 48GB가 큰 이유 : 8B~70B급 모델 가중치 + 긴 시퀀스 KV 캐시를 한 카드/소수 카드에 수용.
- 설계 관점에서 "용량(48GB) + 대역폭(1.5TB/s)" 두 축을 모두 확보한 것.

---

## 6. 전력 설계 — TDP, DVFS, 디커플링 커패시터

### 6.1 TDP(Thermal Design Power) 의미

- 칩이 최대 부하에서 소모하는 전력 = 곧 **발열량**. 쿨링·전원 공급 장치 설계의 기준.
- RNGD는 150W(일부 자료 180W 표기)로 설계 → 패시브 쿨링 + 공랭 서버 배포 가능.
- 설계 관점 : TDP를 낮추려면 전압·클럭·연산 구조·공정을 함께 최적화해야 함.

### 6.2 전력 최적화 설계 기법 (RNGD 채택)

| 기법 | 내용 | 효과 |
|------|------|------|
| DVFS | 동적 전압·주파수 스케일링 | 워크로드에 따라 전력-성능 균형 |
| 저전압 설계 | 노미널 0.75V 수준 동작 | 정적·동적 전력 감소 |
| 디커플링 커패시터(온다이/온인터포저) | 순간 전류 급변(최대 1kA 수준)을 흡수 | 전압 강하(IR drop) 방지·안정성 |
| 타이밍 마진 모니터 | 공정/온도/전압 변동 감시 | 신뢰성 확보 |

### 6.3 설계 관점의 관찰 포인트

- **최대 1kA급 전류 변동**을 감당하려면 패키지·인터포저·PCB 전원 계통까지 고려 필요.
- 절전(아이들) 상태 관리·전력 측정(furiosa-smi)도 전력 아키텍처의 일부.

---

## 7. 공정·패키징 — TSMC 5nm, CoWoS-S, HBM 적층

### 7.1 공정

| 항목 | 값 | 의미 |
|------|-----|------|
| 공정 노드 | TSMC 5nm | 고집적·저전력에 유리한 최신급 공정 |
| 다이 크기 | 약 24.77×26.38mm (653.4mm²) | 대형 다이 → 많은 PE/SRAM 수용 |
| 클럭 | 1.0GHz | 전력 효율 우선(고클럭 지향이 아님) |
| 코어 전압 | 0.75V 수준 | 저전압 동작 |

- 공정·클럭 선택이 "고클럭 + 고전력"이 아니라 "낮은 클럭 + 넓은 병렬 배열"로 갔다는 것이
  **가속기 설계의 일반 원칙**(GPU/NPU 공통)을 보여줌.

### 7.2 패키징 — CoWoS-S + HBM

- **CoWoS-S(Chip on Wafer on Substrate)** : 로직 다이와 HBM 스택을 하나의 실리콘
  인터포저 위에 올려 연결하는 2.5D 패키징.
- RNGD는 HBM3 2스택(48GB)을 CoWoS-S로 칩과 결합 → 대역폭 1.5TB/s 확보.
- 설계 관점 : HBM은 로직과 별도 파운드리/공정에서 만들어지므로 **패키징이 곧 시스템 성능**을
  좌우하는 요소. 인터포저 배선·신호 무결성·열 관리가 설계 과제.

---

## 8. 검증·신뢰성·가상화 — ECC, RoT, SR-IOV

| 항목 | 설계 목적 |
|------|-----------|
| ECC 메모리 | HBM 데이터 오류 검출·정정 → 데이터센터 신뢰성 |
| Secure Boot(Root of Trust) | 부팅 단계부터 펌웨어/실행 무결성 보장 |
| SR-IOV(8 VF) | 하나의 물리 장치를 여러 가상 함수로 분리 → 가상화 환경 지원 |
| 멀티인스턴스 8 | 8개 PE를 사용자/워크로드별로 분할 배정 |
| ATU | PE 주소 공간 분리 → 테넌트 간 격리·동적 자원 할당 |

- 설계 관점 : "단순 가속"을 넘어 **서버급(ECC·보안·가상화·다중 테넌트)** 요구를 반영한 설계.
- 이 때문에 소프트웨어(런타임)가 PE를 안전하게 나눠 쓸 수 있음.

---

## 9. 컴파일러-하드웨어 공동 설계 — TCP의 진짜 의미

### 9.1 하드웨어만으로는 안 되는 이유

- GPU는 커널을 프레임워크가 소프트웨어로 컴파일해 실행.
- RNGD는 **모델별로 커널이 다르게 생성**됨(ENF/FXB) → 컴파일러가 곧 성능의 절반.

### 9.2 공동 설계 요소

| 하드웨어 | 컴파일러/소프트웨어 | 공동 목표 |
|----------|---------------------|-----------|
| TCP(축약 네이티브 실행) | 그래프 최적화·연산자 융합 | 연산 유닛 활용률↑ |
| SRAM 256MB | 메모리 할당·스케줄링 | 재사용 극대화(대역폭 절감) |
| PE 라우터/ATU | 멀티PE·멀티칩 실행 계획 | 확장성 |
| FXB/캐시 | 사전 컴파일·재사용(아키텍처 지문) | 배포 효율 |
| PERT(PE 런타임) | 호스트 런타임과 협력 | 실행 스케줄링 |

- **교수 포인트** : "칩만 봐서는 RNGD를 이해할 수 없다. 컴파일러 설계가 곧 아키텍처의 일부"라는
  것이 이 섹터의 결론.

---

## 10. 설계 관점의 성능 산술 — TOPS·대역폭·연산 밀도

> 아래 수치는 **공식 사양 아님**. 공개 사양(TOPS·클럭·PE 수·대역폭)으로부터 유도한
> "설계 이해용 추정값"입니다. 교육 시 반드시 "추정"임을 명시하세요.

### 10.1 TOPS → 클럭당 MAC 수 유도 (추정)

- INT8 512 TOPS = 512×10¹² ops/s
- 클럭 1GHz = 10⁹ cycle/s
- → **클럭당 512,000 ops/cycle** (= 512×10¹² ÷ 10⁹)
- MAC 1회 = 2 ops(곱+덧셈) → **약 256,000 MAC/cycle**
- PE 8개 → **PE당 약 32,000 MAC/cycle** (추정)

### 10.2 연산 밀도(Compute Intensity) (추정)

- INT8: 512 TOPS ÷ 1.5TB/s ≈ **약 341 ops/byte**
- BF16: 256 TFLOPS ÷ 1.5TB/s ≈ **약 170 ops/byte**
- 연산 밀도가 높을수록 "데이터 이동보다 연산이 많다"는 뜻 → 메모리 병목에 유리.
- 즉 RNGD는 컴퓨트-인텐시브(연산 집약) 워크로드에 최적화된 설계.

### 10.3 대역폭 관점의 이해

- PE당 평균 HBM 대역폭: 1.5TB/s ÷ 8 ≈ 187.5GB/s
- 온칩 SRAM 대역폭(384TB/s)과 비교하면 "재사용할수록 100배 이상 빠르다"는 것을 숫자로 확인.

---

## 11. 핵심 요약 및 설계 관점 체크리스트

### 핵심 요약
- RNGD는 "저클럭·광병렬(8PE)·저전압·대용량 메모리"로 설계한 **추론 전용 가속기**.
- 성능의 비밀은 단일 연산 유닛이 아니라 **MAC 배열(PE) + SRAM 재사용 + NoC + HBM 대역폭**의 조합.
- 전력(150W)을 낮추기 위해 공정·전압·클럭·패키징까지 함께 설계됨.
- 소프트웨어(컴파일러·런타임)가 아키텍처의 절반 → HW-SW 공동 설계 관점 필수.

### 설계 관점 체크리스트 (교강사 자기 점검)
- [ ] "MAC = 곱셈+누적" 회로와 TOPS 관계를 계산으로 설명할 수 있는가
- [ ] PE/SRAM/NoC/HBM3/ATU 각각의 설계 목적을 말할 수 있는가
- [ ] SRAM 재사용이 왜 대역폭 병목을 줄이는지 숫자로 설명할 수 있는가
- [ ] TDP·DVFS·디커플링 커패시터가 왜 필요한지 설명할 수 있는가
- [ ] TSMC 5nm·CoWoS-S·HBM 적층이 성능에 주는 영향(대역폭·전력)을 말할 수 있는가
- [ ] "추정" 수치(§10)는 추정임을 명시하고 교육하는가
- [ ] 컴파일러가 하드웨어 설계의 일부라는 점(HW-SW 공동 설계)을 강조하는가

---

## 12. 참고 자료 (문서·논문·영상)

> 링크는 작성 시점 기준입니다. 최신 정보는 FuriosaAI 공식 사이트/공식 문서에서 재확인하세요.
> 논문·슬라이드 등 일부 PDF는 최신 버전과 수치가 다를 수 있습니다.

### 12.1 논문·학술 발표 (아키텍처 근거)

| 자료 | 내용 | 링크 |
|------|------|------|
| TCP: A Tensor Contraction Processor for AI Workloads (ISCA 2024) | TCP 아키텍처 제안 논문 (FuriosaAI CTO 김한준 발표) | [ISCA 2024 발표 영상](https://www.youtube.com/watch?v=XKTWKCh9XvU) |
| FuriosaAI RNGD: A Tensor Contraction Processor for Sustainable AI Computing (MICRO 2025) | RNGD 마이크로아키텍처 상세(PE/TU/슬라이스/회로 스위치 페치 네트워크 등) | [PDF](https://web.ist.utl.pt/nuno.lopes/pubs/tcp-micro25.pdf) |
| RNGD – Tensor Contraction Processor for Sustainable AI Computing (Hot Chips 2024) | CEO 백준호 발표 슬라이드(HW 아키텍처·칩 설계·SW 풀스택) | [PDF](https://hc2024.hotchips.org/assets/program/conference/day1/83_HC2024.Furiosa.JunePaik.Final.pdf) |

### 12.2 공식 데이터시트·제품 문서

| 자료 | 내용 | 링크 |
|------|------|------|
| RNGD Datasheet (2025.3.0) | 정식 사양표(성능·메모리·전력·폼팩터) | [PDF](https://static.furiosa.info/Media%20Kit%20and%20Resources/RNGD-Datasheet-2025.3.0.pdf) |
| RNGD 제품 페이지 | 개요·특징·발음 안내 | [furiosa.ai/rngd](https://furiosa.ai/rngd) |
| RNGD 사양 페이지 | 사양 요약 | [furiosa.ai/renegade-spec](https://furiosa.ai/renegade-spec) |

### 12.3 공식 개발 문서 (소프트웨어·시스템)

| 자료 | 내용 | 링크 |
|------|------|------|
| RNGD 개요 (Developer Center) | 아키텍처·사양·관련 논문 안내 | [developer.furiosa.ai](https://developer.furiosa.ai/latest/en/overview/rngd.html) |
| Software Stack 개요 | 드라이버→컴파일러→런타임→furiosa-llm 계층 | [developer.furiosa.ai](https://developer.furiosa.ai/latest/en/overview/software_stack.html) |
| furiosa-llm 소개 | LLM 서빙 엔진 기능(PagedAttention·연속 배치·OpenAI 호환) | [developer.furiosa.ai](https://developer.furiosa.ai/latest/en/furiosa_llm/intro.html) |
| furiosa-smi | 시스템 관리 인터페이스(상태·활용률·전력·온도) | [developer.furiosa.ai](https://developer.furiosa.ai/latest/en/device_management/system_management_interface.html) |

### 12.4 공식 블로그·기술 설명

| 자료 | 내용 | 링크 |
|------|------|------|
| Tensor Contraction Processor: AI Chip Architecture | TCP 아키텍처 개념 설명 (블로그) | [furiosa.ai/blog](https://furiosa.ai/blog/tensor-contraction-processor-ai-chip-architecture) |
| RNGD Preview | RNGD 소개·GPU와의 차이 | [furiosa.ai/blog](https://furiosa.ai/blog/rngd-preview-furiosa-ai) |
| Architecture TCP | TCP 설계 철학(텐서 지오메트리·컴파일러 공동 설계) | [furiosa.ai/architecture](https://furiosa.ai/architecture) |

### 12.5 동영상 자료

| 자료 | 내용 | 링크 |
|------|------|------|
| ISCA 2024: TCP 발표 | CTO가 설명하는 TCP 아키텍처(PE·슬라이스·컴파일러) | [YouTube](https://www.youtube.com/watch?v=XKTWKCh9XvU) |
| High-Speed Inference Throughput (Furiosa SDK) | 1카드(RNGD 180W)로 동시 LLM 요청 처리 시연 | [YouTube](https://www.youtube.com/watch?v=t21GKPzM6eg) |

### 12.6 참고 시 유의사항

- **TDP 표기 차이** : 공식 개발 문서는 150W, 일부 데이터시트/영상은 180W로 표기 — 배포 대상 카드 기준으로 확인.
- **성능 비교 수치** : 논문(ISCA/MICRO/Hot Chips)의 GPU 대비 성능/와트 수치는 시점·워크로드 의존 — 교육 시 "당시 발표 기준"임을 명시.
- **링크 변동** : 문서 버전(2025.x/2026.x)이 바뀌면 경로가 변경될 수 있음 — 접속 불가 시 `developer.furiosa.ai`에서 재검색.

