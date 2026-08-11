# 보충 6. RNGD 아키텍처 이해 Q&A 정리

> **파일 경로** : `01_교육내용_준비자료/06_RNGD_아키텍처_이해_QnA.md`
> **성격** : 온보딩 교육 준비 과정에서 다루었던 "레니게이드가 실제로 어떻게 동작하는가"에 대한
> 기초 개념 Q&A를 체계적으로 정리한 보충 자료.
> **대상** : 하드웨어·시스템 구조를 "개념→실행→비교" 관점에서 이해하고 싶은 학습자/교강사.

---

## 1. RNGD 기본 사항

### Q1. RNGD란 무엇인가?
- FuriosaAI의 **2세대 NPU**. "레니게이드(Renegade)"로 발음.
- LLM·멀티모달·비전 모델 등 **추론 서빙 전용** 칩.
- 특징: TCP 아키텍처, TSMC 5nm, HBM3 48GB(1.5TB/s), 온칩 SRAM 256MB, PCIe Gen5 x16, TDP 150W(자료에 따라 180W 표기).

### Q2. TDP는 무엇의 약자인가?
- **Thermal Design Power**. 칩이 최대 부하에서 소모하는 전력(발열) 기준값.
- 쿨링·전원 설계의 기준. RNGD는 150W 수준이라 공랭 데이터센터 배포 가능.

### Q3. KCCI-Furiosa AI Skills Training에서 KCCI는?
- **Korea Chamber of Commerce and Industry, 대한상공회의소**.
- 정부 지원 무료 반도체·AI 인력양성 교육을 운영하며, FuriosaAI와 협력해 NPU 인재를 양성하는 프로그램.

---

## 2. RNGD 내부 구조

### Q4. RNGD에는 CPU가 함께 들어있는가?
- **있다.** 다이 내부에 **온다이 CPU 코어(최대 2GHz, 0.75V)** 가 포함됨.
- 단, 용도는 **제어용(컨트롤 플레인)** : DVFS 전원 관리, 타이밍 모니터, 보안(RoT), 부팅 등.
- 사용자 연산은 하지 않음 → 실제 추론은 **8개 PE**가 담당.
- 애플리케이션·스케줄링은 **호스트 CPU(x86 서버)** 가 맡고 PCIe Gen5로 통신.

### Q5. PE란 무엇인가? (PCI Express? Performance Core?)
- 여기서 PE = **Processing Element (연산 유닛)**. PCI Express, Intel P-core가 아님.

| 약자 | 뜻 | RNGD에서 역할 |
|------|-----|---------------|
| PE | Processing Element | 칩 내부 **연산 유닛 8개** (MAC 배열 기반) |
| PCIe | PCI Express | 호스트 CPU와 칩을 연결하는 **외부 버스** (Gen5 x16) |
| P-core | Performance Core | (Intel CPU 한정) 고성능 코어 분류 |

### Q6. PE는 뉴런(노드·가중치) 구조를 모사한 것인가?
- **아니다.** 뉴런 모사는 뉴로모픽(예: IBM TrueNorth, Intel Loihi)의 개념.
- PE는 **텐서 축약 = 행렬 곱(MAC: 곱셈-누적)** 을 고속 수행하는 **MAC 배열 연산기**.
- 딥러닝 연산의 수학적 본질(행렬 곱·합)에 최적화한 전용 연산기.
- Furiosa의 독자성은 PE 존재 자체가 아니라, 이를 묶는 **TCP(Tensor Contraction Processor)** 설계 패러다임.

---

## 3. 실행 흐름 (메모리 번지 기반)

### Q7. CPU→레니게이드 실행 요청 흐름은?
```
1. 모델 적재  : 컴파일된 ENF/FXB + 가중치 → PCIe 통해 HBM(48GB) 메모리 번지에 DMA 전송
                (매 요청마다 재전송하지 않음, 상주)
2. 실행 요청  : 호스트 런타임이 펌웨어에 "실행 파일 + 입력 데이터 번지"를 담은
                커맨드 디스크립터 제출
3. PE 연산    : PE들이 SRAM/HBM에서 데이터를 읽어 추론 수행 → 결과를 디바이스
                메모리 지정 번지에 기록
4. 결과 반환  : 완료 신호(인터럽트/도어벨) 후 결과를 DMA로 호스트 메모리에 복사
                → CPU가 값을 받아 최종 응답 생성
```
- **핵심** : CPU가 데이터를 일일이 쓰는(PIO) 방식이 아니라, 번지·명령만 지정하고
  **대량 데이터 이동은 DMA 엔진**이 처리. 개념적으로 "번지에 올리고 요청 → PE 실행 → 결과 반환"이 맞음.

### Q8. 텐서플로우를 GPU에서 쓰듯, 텐서플로우에 레니게이드 API를 추가하는 방식인가?
- **아니다.** GPU(CUDA 커널을 프레임워크가 즉석 호출)와 다른 **오프라인 컴파일 방식**:

| GPU 방식 | RNGD 방식 |
|----------|-----------|
| 프레임워크(텐서플로우/파이토치)가 CUDA 커널 호출 | 모델을 ONNX/TFLite로 **export** |
| 즉석 실행 | **Furiosa Compiler로 미리 컴파일** → ENF/FXB |
| - | **Furiosa Runtime**이 실행 (Python/C SDK) |
| - | LLM은 **furiosa-llm**(vLLM 호환 API)으로 서빙 |
- 텐서플로우 직접 지원보다 **ONNX/TFLite 경유가 주 경로**. PyTorch는 `torch.compile()`의 **FuriosaBackend** 제공.

### Q9. 전체 워크플로 (PC/서버 ↔ RNGD)
```
파이토치/텐서플로우로 모델 생성
        │  ONNX/TFLite export
        ▼
Furiosa Compiler → ENF / FXB 생성 (호스트에서 수행)
        │  PCIe로 적재
        ▼
RNGD HBM에 모델 상주 (가중치 + KV 캐시)
        │
호스트 애플리케이션 + Furiosa Runtime(Python/C SDK) → 추론 요청
        │  LLM의 경우
        ▼
furiosa-llm(vLLM 호환 API + OpenAI 호환 서버) → 최종 사용자와 HTTP 인터페이스
```
- 주의: RNGD와 호스트(Runtime이 도는 서버)는 **한 서버 안에 PCIe로 연결**된 관계.
- 컴파일도 보통 RNGD가 장착된 동일 서버 호스트에서 수행.

### Q10. furiosa-llm은 PC 메모리에서 도는가, 레니게이드 안에서 도는가?
- **호스트 CPU(서버 메인 CPU 메모리)에서 실행**되는 Python 소프트웨어.

| 요소 | 실행 위치 |
|------|-----------|
| furiosa-llm 코드(스케줄링·배치·토크나이저·최종 디코딩) | **호스트 CPU 메모리** |
| 모델 가중치 + KV 캐시 | **RNGD HBM(48GB)** |
| 신경망 연산(Attention·FFN 계층) | **RNGD PE** |
| 사용자와의 API 통신 | 호스트의 HTTP 서버 |
- furiosa-llm = "지휘자"가 호스트에서 돌고, 무거운 행렬 연산과 모델 저장만 RNGD에 위임.
- GPU 사용과 같은 개념 (CUDA 프로그램도 CPU에서 시작, 커널만 GPU로 전송).

---

## 4. 생태계 비유

### Q11. STM32 생태계와 유사한가?
- **유사점** : 둘 다 "벤더 하드웨어 + 벤더 툴체인 + 개방 생태계" 구조.

| | STM32 생태계 | Furiosa RNGD 생태계 |
|---|---|---|
| 하드웨어 | STM32 MCU | RNGD NPU 카드 |
| 벤더 툴 | CubeIDE/CubeMX, HAL 라이브러리 | Furiosa Compiler, Runtime, furiosa-smi |
| 개방 생태계 | ARM/CMSIS, GCC, FreeRTOS | Linux, ONNX, PyTorch, vLLM |
| 개발 대상 | MCU에서 돌릴 펌웨어를 직접 작성 | 모델을 컴파일해 NPU에 실행 요청 |

- **핵심 차이** :
  - STM32: 사용자가 칩 위에서 도는 **펌웨어(코드)를 직접 작성·제어** (코딩형)
  - RNGD: 사용자가 NPU 코드를 작성하지 않고 **모델을 컴파일 산출물(ENF/FXB)로 바꿔 런타임에 위임** (모델 컴파일형)
  - RNGD 내부 CPU 코어는 사용자 프로그래밍 대상이 아닌 고정 제어용.

---

## 5. 핵심 포인트 요약

- RNGD = 호스트 서버에 PCIe로 장착된 **추론 전용 가속 카드** (호스트와 한 서버에 공존).
- 칩 내부: **제어용 온다이 CPU + 8개 PE(연산 유닛) + SRAM(256MB) + HBM3(48GB/1.5TB/s)**.
- 실행 방식: **오프라인 컴파일(ENF/FXB) → DMA 적재 → 커맨드 제출 → PE 연산 → 결과 DMA 반환**.
- 프레임워크 플러그인이 아니라 **ONNX/TFLite 경유 + Furiosa Runtime + furiosa-llm** 구조.
- furiosa-llm은 호스트에서 실행되며, RNGD는 무거운 연산만 위임받는 "지휘자-일꾼" 관계.
- 개발 패러다임은 STM32 같은 펌웨어 코딩이 아니라 **모델 컴파일·위임** 방식.
