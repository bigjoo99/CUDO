# VLM Scene Analysis API 서버 설명서

이 문서는 `vlm_api_server_ver4.py`를 개발팀에 전달하기 위한 설명서입니다.  
본 코드는 AI 연구팀에서 작성한 Qwen3-VL 기반 CCTV/감시 영상 분석용 FastAPI 서버 예시이며, 실제 운영 API 연동, 인증, 배포 구조, callback 수신부 구현 등은 개발팀 환경에 맞게 조정할 수 있습니다.

---

## 1. 전체 개요

이 서버는 CUVIA 시스템에서 전달한 비디오 폴더를 입력으로 받아, 폴더 하위의 모든 비디오 파일을 Qwen3-VL 모델로 분석합니다.

분석 방식은 크게 두 단계입니다.

1. **Single video analysis**
   - `video_dir` 하위 비디오를 하나씩 분석합니다.
   - 각 비디오 분석이 끝날 때마다 CUVIA callback URL로 결과를 전송합니다.
   - 결과는 `results` 배열의 단일 item 형태로 전달됩니다.

2. **Multi-video summary**
   - 모든 single video 분석이 끝난 뒤, 전체 비디오를 한 번에 모델에 입력합니다.
   - 대상 인물이 여러 비디오에서 어떻게 나타나는지 순서대로 종합 요약합니다.
   - 최종 결과는 `summary.summary_message`에 포함됩니다.

---

## 2. 지원하는 query_type

### 2.1 query_type = 0: Text Query

사용자가 텍스트로 분석 대상을 전달하는 방식입니다.

예:

```json
{
  "video_dir": "/data/share/sample/videos",
  "response_url": "http://callback-server/api/v2/scene/analyses/JOB_001/result",
  "query_type": 0,
  "message": "회색 후드티를 입은 남자"
}
```

처리 흐름:

```text
message
→ normalize_query_text()
→ job.query_text
→ single video prompt에 "분석 대상"으로 주입
→ multi-video summary prompt에도 "분석 대상"으로 주입
```

---

### 2.2 query_type = 1: Image Query

사용자가 crop image 경로를 전달하는 방식입니다.  
이 경우 모델이 먼저 crop image를 분석하여 자연어 text query를 추출합니다.

예:

```json
{
  "video_dir": "/data/share/sample/videos",
  "response_url": "http://callback-server/api/v2/scene/analyses/JOB_001/result",
  "query_type": 1,
  "image_path": "/data/share/sample/query/person_crop.png"
}
```

처리 흐름:

```text
image_path
→ resolve_media_file()
→ infer_image_caption_sync()
→ IMAGE_SYSTEM_PROMPT + IMAGE_QUERY_TO_TEXT_PROMPT
→ "검은색 상의와 검은색 하의를 입고 헬멧을 착용한 사람" 형태의 text query 생성
→ job.query_text
→ single video prompt에 "분석 대상"으로 주입
→ multi-video summary prompt에도 "분석 대상"으로 주입
```

---

## 3. 지원 API

### 3.1 VLM 헬스 체크

```http
GET /api/v2/metis/analysis/health
```

서버의 기본 상태를 확인합니다.

응답 예:

```json
{
  "error": 0,
  "message": "success",
  "data": {
    "vlm": "ok"
  }
}
```

`vlm` 값은 아래 기준으로 결정됩니다.

- `ok`: 모델 이름이 설정되어 있고 CUDA 사용 가능
- `error`: 모델 이름이 비어 있거나 CUDA 사용 불가

---

### 3.2 영상 분석 작업 요청

```http
POST /api/v2/scene/analysis
```

영상 분석 작업을 요청합니다.  
이 API는 즉시 `job_id`를 반환하고, 실제 분석은 background task에서 수행됩니다.

#### Request Body

| 필드 | 타입 | 필수 여부 | 설명 |
|---|---|---:|---|
| `video_dir` | string | 필수 | 분석할 비디오 폴더 경로 |
| `response_url` | string | 필수 | 분석 진행 상태와 결과를 받을 callback URL |
| `query_type` | int | 필수 | `0`: text query, `1`: image query |
| `message` | string | query_type=0일 때 필수 | 분석 대상 text query |
| `image_path` | string | query_type=1일 때 필수 | 분석 대상 crop image 경로 |

#### Response

```json
{
  "error": 0,
  "message": "success",
  "data": {
    "job_id": "JOB_..."
  }
}
```

---

### 3.3 영상 분석 작업 취소

```http
DELETE /api/v2/scene/analyses/{job_id}
```

실행 중이거나 대기 중인 분석 작업을 취소합니다.

- 실행 중인 작업: `cancel_event`를 set하여 generate 중단을 유도합니다.
- 대기 중인 작업: queue에서 제거하고 상태를 `cancel`로 기록합니다.

응답 예:

```json
{
  "error": 0,
  "message": "success"
}
```

---

### 3.4 영상 분석 작업 조회

```http
GET /api/v2/scene/analyses/{job_id}
```

특정 작업의 현재 상태와 결과를 조회합니다.

응답 예:

```json
{
  "error": 0,
  "message": "success",
  "data": {
    "job_id": "JOB_...",
    "status": "processing",
    "progress_rate": "75",
    "sequence": "2/5",
    "current_file": "/workspace/data/sample/video_002.mp4",
    "job_message": "VLM generate 진행중",
    "results": [
      {
        "video_file": {
          "path": "/workspace/data/sample/video_001.mp4",
          "name": "video_001.mp4"
        },
        "message": "..."
      }
    ]
  }
}
```

---

### 3.5 영상 분석 작업 목록 조회

```http
GET /api/v2/scene/analyses
```

현재 실행 중인 작업과 대기 중인 작업 목록을 반환합니다.

---

## 4. Callback 구조

분석 진행 중 서버는 `response_url`로 callback을 보냅니다.

### 4.1 Progress Callback 예시

```json
{
  "job_id": "JOB_001",
  "status": "processing",
  "progress_rate": "50",
  "sequence": "0/3",
  "current_file": "/workspace/data/sample/video_001.mp4",
  "job_message": "VLM 추론 시작"
}
```

### 4.2 Single Video Result Callback 예시

```json
{
  "job_id": "JOB_001",
  "status": "processing",
  "progress_rate": "100",
  "sequence": "1/3",
  "current_file": "/workspace/data/sample/video_001.mp4",
  "job_message": "비디오 분석 완료 (1/3)",
  "results": [
    {
      "video_file": {
        "path": "/workspace/data/sample/video_001.mp4",
        "name": "video_001.mp4"
      },
      "message": "회색 후드티를 입은 사람이 화면 왼쪽에서 오른쪽으로 이동했다."
    }
  ]
}
```

### 4.3 Final Summary Callback 예시

```json
{
  "job_id": "JOB_001",
  "status": "completed",
  "progress_rate": "100",
  "sequence": "3/3",
  "current_file": "MULTI_VIDEO_SUMMARY",
  "job_message": "전체 영상 분석 및 종합 요약 완료",
  "summary": {
    "total_cnt": 4,
    "summary_message": "대상 인물은 회색 후드티를 입은 사람으로 보인다. 1번째 비디오에서는 ..."
  },
  "results": [
    {
      "video_file": {
        "path": "/workspace/data/sample/video_001.mp4",
        "name": "video_001.mp4"
      },
      "message": "..."
    }
  ]
}
```

> 현재 코드에서는 `summary.total_cnt = 비디오 개수 + 1`로 설정되어 있습니다.  
> 즉, 비디오 3개를 분석하면 `summary.total_cnt`는 4가 됩니다. 이는 기존 연동 규격을 따른 것으로 보입니다.

---

## 5. AI 모델 부분 설명

### 5.1 사용 모델

현재 코드는 Hugging Face Transformers 기반으로 Qwen3-VL 모델을 사용합니다.

```python
AutoModelForImageTextToText.from_pretrained(...)
AutoProcessor.from_pretrained(...)
```

기본 모델 이름:

```text
Qwen/Qwen3-VL-8B-Instruct
```

환경 변수 `QWEN_MODEL_NAME`을 통해 모델 경로 또는 모델 이름을 바꿀 수 있습니다.

예:

```bash
export QWEN_MODEL_NAME="/models/Qwen3-VL-32B-Instruct"
```

현재 코드는 `local_files_only=True`로 모델을 로딩합니다.  
따라서 폐쇄망/오프라인 환경에서는 모델 파일이 사전에 로컬 경로 또는 Hugging Face cache에 존재해야 합니다.

---

### 5.2 Video Analysis 입력/출력

비디오 분석 함수:

```python
infer_sync(...)
```

#### 입력

- `video_paths`: 비디오 파일 경로 리스트
  - single video 분석: 길이 1
  - multi-video summary: 전체 비디오 경로 리스트
- `user_prompt`
  - single video prompt 또는 multi-video summary prompt
- `VIDEO_SYSTEM_PROMPT`
  - 영상 분석용 system prompt

Qwen3-VL에 들어가는 user content 구조:

```python
[
  {
    "type": "video",
    "video": "file:///workspace/data/sample/video_001.mp4",
    "min_pixels": ...,
    "max_pixels": ...,
    "total_pixels": ...,
    "fps": ...
  },
  {
    "type": "text",
    "text": "분석 prompt"
  }
]
```

#### 출력

모델 출력은 별도 parsing 없이 문자열 그대로 사용합니다.

예:

```text
회색 후드티를 입은 사람이 화면 왼쪽에서 오른쪽으로 이동한 뒤 잠시 멈춰 주변을 바라봤다.
```

이 값은 callback의 `results[].message` 또는 `summary.summary_message`에 들어갑니다.

---

### 5.3 Query Image Caption 입력/출력

query image 분석 함수:

```python
infer_image_caption_sync(...)
```

#### 입력

- `image_path`: crop image 파일 경로
- `IMAGE_SYSTEM_PROMPT`
  - 이미지 분석용 system prompt
- `IMAGE_QUERY_TO_TEXT_PROMPT`
  - crop image를 text query로 변환하기 위한 user prompt

Qwen3-VL에 들어가는 user content 구조:

```python
[
  {
    "type": "image",
    "image": "file:///workspace/data/sample/query/person_crop.png"
  },
  {
    "type": "text",
    "text": "이미지에서 사람의 외형과 복장 특징을 설명하라는 prompt"
  }
]
```

#### 출력

query image caption 출력은 이후 video 분석의 text query로 사용됩니다.

예:

```text
검은색 상의와 검은색 하의를 입고 헬멧을 착용한 사람
```

처리 흐름:

```text
query image
→ Qwen3-VL image caption
→ job.query_text
→ single video prompt
→ multi-video summary prompt
```

---

## 6. 주요 환경 변수

| 환경 변수 | 기본값 | 설명 |
|---|---|---|
| `CUDA_VISIBLE_DEVICES` | 코드 상단에서 `"1"`로 설정 | 사용할 GPU 번호 |
| `VLM_INBOUND_TOKEN` | `metis-demo-token` | inbound API token |
| `VLM_OUTBOUND_TOKEN` | `metis-demo-token` | callback 전송 시 사용할 token |
| `VLM_REQUIRE_ACCESS_TOKEN` | `0` | `1`이면 x-access-token 검증 활성화 |
| `HOST_VIDEO_PREFIX` | `/data/share` | 외부/호스트 기준 미디어 루트 |
| `CONTAINER_VIDEO_PREFIX` | `/workspace/data` | 컨테이너 내부 미디어 루트 |
| `HF_HOME` | `/root/.cache/huggingface` | Hugging Face cache 경로 |
| `QWEN_MODEL_NAME` | `Qwen/Qwen3-VL-8B-Instruct` | 사용할 Qwen3-VL 모델 이름 또는 로컬 경로 |
| `QWEN_DEFAULT_FPS` | `1.0` | 비디오 프레임 샘플링 FPS |
| `QWEN_MIN_PIXELS` | `256 * 28 * 28` | 모델 입력 최소 픽셀 설정 |
| `QWEN_MAX_PIXELS` | `960 * 540` | 모델 입력 최대 픽셀 설정 |
| `QWEN_TOTAL_PIXELS` | `1000000000` | 전체 비디오 입력 픽셀 budget |
| `QWEN_MAX_NEW_TOKENS` | `512` | 비디오 분석 최대 생성 토큰 수 |
| `QWEN_IMAGE_CAPTION_MAX_NEW_TOKENS` | `64` | query image caption 최대 생성 토큰 수 |
| `QWEN_FLASH_ATTN2` | `1` | flash attention 2 사용 여부 |
| `HTTP_TIMEOUT` | `10.0` | callback HTTP timeout |
| `VLM_MAX_QUEUE` | `0` | 대기열 최대 길이. 0이면 제한 없음 |
| `GENERATE_PROGRESS_INTERVAL_SEC` | `3.0` | generate 진행중 callback 주기 |
| `VIDEO_EXTS` | `.mp4,.ts,.avi,...` | 분석 대상 비디오 확장자 목록 |

---

## 7. 작업 상태값

현재 코드에서 사용되는 주요 status는 다음과 같습니다.

| status | 의미 |
|---|---|
| `pending` | 대기 중 |
| `processing` | 처리 중 |
| `completed` | 전체 분석 완료 |
| `failed` | 분석 실패 |
| `cancel` | 취소됨 |

---

## 8. 개발팀 연동 시 주의사항

### 8.1 모델 파일 위치

현재 모델 로딩은 다음 옵션을 사용합니다.

```python
local_files_only=True
```

따라서 운영 서버에서는 다음 중 하나가 필요합니다.

1. `QWEN_MODEL_NAME`을 로컬 모델 폴더로 설정
2. Hugging Face cache에 모델을 미리 다운로드
3. 네트워크가 가능한 환경에서만 `local_files_only=False`로 변경

권장 방식:

```bash
export QWEN_MODEL_NAME="/models/Qwen3-VL-8B-Instruct"
```

---

### 8.2 파일 경로 매핑

CUVIA에서 전달하는 경로가 호스트 경로이고, VLM 서버가 컨테이너에서 실행된다면 경로 매핑이 필요합니다.

현재 기본 매핑:

```text
/data/share      → /workspace/data
```

관련 환경 변수:

```bash
export HOST_VIDEO_PREFIX="/data/share"
export CONTAINER_VIDEO_PREFIX="/workspace/data"
```

---

### 8.3 callback URL 처리

`response_url`이 이미 `/api/v2/scene/analyses/{job_id}/result`까지 포함한 전체 URL이면 그대로 사용합니다.  
그렇지 않고 base URL만 들어오면 서버가 아래 path를 붙입니다.

```text
/api/v2/scene/analyses/{job_id}/result
```

---

### 8.4 출력 parsing 없음

현재 모델 출력은 JSON parsing 없이 string 그대로 전달됩니다.  
개발팀에서 downstream 처리가 필요하면 `message`와 `summary_message` 문자열을 그대로 저장/표시하는 형태가 가장 안전합니다.

---

### 8.5 단일 작업 처리와 queue

현재 전역 변수 기반으로 하나의 current job과 pending queue를 관리합니다.

- 현재 작업이 없으면 즉시 실행
- 현재 작업이 있으면 pending queue에 추가
- 작업 종료 후 다음 작업 자동 시작
- 대기열 제한은 `VLM_MAX_QUEUE`로 설정 가능

운영 규모가 커지면 Redis, DB, Celery/RQ 같은 외부 queue 시스템으로 교체하는 것이 좋습니다.

---

### 8.6 프로세스 메모리 기반 상태 관리

현재 `job_records_by_id`, `active_jobs_by_id`, `pending_jobs`는 모두 프로세스 메모리에 저장됩니다.

따라서 서버 프로세스가 재시작되면 기존 job 상태가 사라집니다.  
운영 환경에서는 DB 저장 또는 외부 job store 연동을 고려해야 합니다.

---

## 9. 실행 예시

```bash
uvicorn vlm_api_server_ver4:app --host 0.0.0.0 --port 8000
```

GPU/모델 설정 예:

```bash
export CUDA_VISIBLE_DEVICES=1
export QWEN_MODEL_NAME="/models/Qwen3-VL-8B-Instruct"
export HOST_VIDEO_PREFIX="/data/share"
export CONTAINER_VIDEO_PREFIX="/workspace/data"

uvicorn vlm_api_server_ver4:app --host 0.0.0.0 --port 8000
```

---

## 10. 개발팀에서 확인하면 좋은 부분

- 실제 CUVIA callback URL 형식
- `summary.total_cnt = video_count + 1` 규칙 유지 여부
- 작업 상태 저장을 메모리로 유지할지 DB로 분리할지
- 파일 경로 매핑 규칙
- 인증 토큰 적용 여부
- 모델 서버를 API 서버와 같은 프로세스로 둘지 별도 inference server로 분리할지
- callback 실패 시 재시도 정책
- 대용량 비디오/다수 작업 요청 시 queue 및 timeout 정책
