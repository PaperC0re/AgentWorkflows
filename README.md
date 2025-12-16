# PaperC0re Agent Workflows
이 디렉토리는 PaperC0re 서비스의 핵심 자동화 로직을 담당하는 n8n 워크플로우 명세 및 구성 파일을 포함하고 있습니다. 시스템은 크게 논문을 수집하고 분석하는 **Collector(수집기)**와 사용자 맞춤형 뉴스레터를 발송하는 **Distributor(배포기)**로 구성되어 있습니다.

## 시스템 아키텍처 개요
PaperC0re의 에이전트 시스템은 n8n을 기반으로 구축되었으며, 다음과 같은 특징을 가집니다.

1. **비동기 처리:** 논문의 수집/분석과 메일 발송 로직을 분리하여 대량의 트래픽에도 전송 지연이 발생하지 않도록 설계되었습니다.
2. **이원화된 분석 파이프라인:**
* 일반 요약: 초록(Abstract) 기반의 경량화된 요약
* 심층 분석(Deep Dive): PDF 원문 기반의 상세 기술 분석 (Premium 전용)


3. **구독 모델 기반 분기 처리:** 사용자의 구독 등급(Basic/Premium)에 따라 차별화된 콘텐츠를 제공합니다.

---

<img width="2128" height="648" alt="스크린샷 2025-12-17 064144" src="https://github.com/user-attachments/assets/ccc0035a-42fd-4062-97de-4d3df81d3ae4" />

## 1. 논문 수집 및 분석 워크플로우 (Collector)
arXiv에서 최신 논문을 주기적으로 수집하고, 중복을 제거한 뒤 AI를 통해 분석하여 데이터베이스에 저장하는 워크플로우입니다.

### 주요 프로세스
1. **스케줄링 및 키워드 로드:** 지정된 주기(예: 3일)마다 실행되며, 시스템에 등록된 관심 키워드 목록을 불러옵니다.
2. **메타데이터 수집 (arXiv API):** 키워드별로 arXiv API를 호출하여 최신 논문의 메타데이터(제목, 저자, 초록 등)를 확보합니다.
3. **중복 검사:** 데이터베이스를 조회하여 이미 수집된 논문인지 확인하고 중복된 경우 스킵합니다.
4. **AI 기반 초록 요약:** 수집된 논문의 초록을 LLM(Ollama 또는 Gemini)에 전달하여 한국어 요약 및 핵심 기여점을 추출합니다.
5. **PDF 원문 다운로드 및 심층 분석 (Deep Dive):**
* 논문의 PDF 원문을 바이너리 형태로 다운로드합니다.
* Gemini 3 Pro 모델(Vision/Multimodal)을 활용하여 PDF 전체를 분석합니다.
* 연구 배경, 방법론(수식/아키텍처), 실험 결과, 실무 적용 포인트가 포함된 HTML 리포트를 생성합니다.


6. **데이터 저장:** 메타데이터, 초록 요약본, 심층 분석 리포트(HTML), PDF 링크를 통합하여 백엔드 DB에 저장합니다.

---

<img width="2087" height="629" alt="스크린샷 2025-12-17 064110" src="https://github.com/user-attachments/assets/56ce65f7-1712-4a1b-bc61-15b0018cd0f9" />

## 2. 뉴스레터 전송 워크플로우 (Distributor)
구독 중인 사용자 목록을 불러와 개인화된 논문을 추천하고, 구독 타입에 맞춰 이메일을 발송하는 워크플로우입니다.

### 주요 프로세스
1. **사용자 로드:** 활성화된 구독자 전체 목록을 백엔드에서 조회합니다.
2. **개인화 추천:** 각 사용자의 관심 키워드 및 히스토리를 기반으로 추천 논문 리스트를 백엔드에 요청합니다.
3. **예외 처리:** 추천된 논문이 없는 경우 해당 사용자를 건너뛰고 다음 사용자로 진행합니다(Bypass).
4. **구독 타입별 분기 (Switch Logic):**
* **Basic 플랜:**
* 추천 논문 5개를 선정합니다.
* 각 논문의 초록 요약본을 리스트 형태의 HTML로 변환합니다.
* 종합 요약 뉴스레터를 발송합니다.


* **Premium 플랜:**
* 가장 추천도가 높은 최상위 논문 1개를 선정합니다.
* 수집 단계에서 미리 생성된 **Deep Dive 분석 리포트(HTML)**를 로드합니다.
* PDF 상세 분석이 포함된 심층 리포트 메일을 발송합니다.




5. **로깅:** 발송 성공 여부와 발송된 논문 ID를 백엔드 로그 API로 전송하여 이력을 남깁니다.

---

## 기술 스택 및 환경
* **Workflow Engine:** n8n (Self-hosted via Docker)
* **LLM:**
* Abstract Summary: Ollama (e.g. GPT-OSS) 또는 Google Gemini 2.5 Flash Lite
* PDF Deep Analysis: Google Gemini 3 Pro (High Context Window)


* **Data Source:** arXiv API
* **Backend Integration:** Spring Boot REST API
* **Email Service:** Gmail SMTP (via n8n integration)

##설치 및 실행 방법
1. n8n 인스턴스를 실행합니다.
2. `.json` 워크플로우 파일을 Import 합니다.
3. `Credentials` 메뉴에서 다음 자격 증명을 설정합니다.
  * Backend API 연결 정보 (Header Auth 등)
  * Google Gemini API Key
  * Gmail OAuth2 또는 App Password


4. 환경 변수 또는 노드 내 하드코딩된 API 엔드포인트(`http://host.docker.internal:8080` 등)를 실제 배포 환경에 맞춰 수정합니다.
5. 워크플로우를 활성화(Active) 상태로 변경합니다.
---
