# 무료 TTS 솔루션 가이드

> Daily Study 프로젝트 — 비용 없이 Text-to-Speech를 구현하는 방법  
> 작성일: 2026-11-04  
> 버전: v1.0

---

## 목차

1. [무료 TTS 솔루션 개요](#1-무료-tts-솔루션-개요)
2. [Edge TTS (가장 추천)](#2-edge-tts-가장-추천-)
3. [Kokoro / SpeakCrate (완전 오프라인)](#3-kokoro--speakcrate-완전-오프라인)
4. [XTTS-v2 (음성 복제 지원)](#4-xtts-v2-음성-복제-지원)
5. [Google Cloud TTS (무료 한도)](#5-google-cloud-tts-무료-한도)
6. [비교표](#6-비교표)
7. [Daily Study 추천 전략](#7-daily-study-추천-전략)
8. [멀티 프로바이더 전환 설계](#8-멀티-프로바이더-전환-설계)
9. [Spring Boot 멀티 TtsService 구현](#9-spring-boot-멀티-ttsservice-구현)
10. [Docker Compose 통합](#10-docker-compose-통합)
11. [참고 자료](#11-참고-자료)

---

## 1. 무료 TTS 솔루션 개요

OpenAI TTS는 월 $1~2의 저비용이지만, 개발 단계나 초기 운영에서는 **완전 무료**로 TTS를 사용하는 것이 유리합니다. 다음 4가지 무료 솔루션을 검토했습니다.

| 솔루션 | 비용 | 음질 | 오프라인 | 한국어 | 셋업 난이도 |
|--------|------|------|---------|--------|------------|
| Edge TTS | 무료 | 양호 | 불가 (인터넷 필요) | 지원 | 매우 쉬움 |
| Kokoro | 무료 | 우수 | 가능 | 미지원 | 보통 |
| XTTS-v2 | 무료 | 우수 | 가능 | 지원 | 어려움 |
| Google Cloud TTS | 400만자/월 무료 | 매우 우수 | 불가 | 지원 | 보통 |

---

## 2. Edge TTS (가장 추천) ⭐

Microsoft Edge 브라우저의 TTS 엔진을 OpenAI 호환 API로 래핑한 오픈소스입니다.

### 2.1 특징

| 항목 | 내용 |
|------|------|
| 비용 | 완전 무료 |
| 음질 | 양호 (약간 기계음이 있지만 실용적) |
| 음성 | 100+개 (다국어, 한국어 `ko-KR-` 음성 지원) |
| 셋업 | Docker 1줄로 실행 |
| API 호환 | OpenAI TTS와 동일한 구조 (`/v1/audio/speech`) |
| GitHub | github.com/travisvn/openai-edge-tts (2,100+ ⭐) |
| 라이선스 | GPL v3.0 |

### 2.2 설치 및 실행

**Docker (권장):**

```bash
docker run -d -p 5050:5050 \
  --name edge-tts \
  --restart unless-stopped \
  travisvn/openai-edge-tts:latest
```

**Python 직접 실행:**

```bash
git clone https://github.com/travisvn/openai-edge-tts.git
cd openai-edge-tts
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python app/server.py
# 서버: http://localhost:5050
```

### 2.3 API 호출 예시

```bash
# OpenAI TTS와 동일한 구조 — URL만 다름
curl -X POST http://localhost:5050/v1/audio/speech \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer any-key-works" \
  -d '{
    "model": "tts-1",
    "input": "Could you tell me how to get to the station?",
    "voice": "en-US-JennyNeural",
    "response_format": "mp3"
  }' \
  --output speech.mp3
```

### 2.4 사용 가능한 음성

**OpenAI 호환 음성 (자동 매핑):**

| OpenAI 음성 | Edge TTS 매핑 | 특징 |
|------------|-------------|------|
| `alloy` | en-US-AvaNeural | 중성적, 균형 |
| `echo` | en-US-AndrewNeural | 차분한 남성 |
| `fable` | en-US-EmmaNeural | 서사적 여성 |
| `onyx` | en-US-EricNeural | 깊은 남성 |
| `nova` | en-US-JennyNeural | 밝은 여성 |
| `shimmer` | en-US-MichelleNeural | 부드러운 여성 |

**한국어 음성 (직접 지정):**

| 음성 ID | 특징 |
|---------|------|
| `ko-KR-SunHiNeural` | 여성, 밝고 친근 |
| `ko-KR-InJoonNeural` | 남성, 차분 |
| `ko-KR-HyunsuNeural` | 남성, 자연스러움 |
| `ko-KR-JiMinNeural` | 여성, 부드러움 |

> **Daily Study 권장 음성**: `en-US-JennyNeural` (영어 학습용 — 밝고 친근한 여성)

### 2.5 장단점

**장점:**
- API 키 불필요 — 즉시 사용 가능
- OpenAI TTS 코드에서 URL만 변경하면 전환 가능
- 한국어 음성 지원
- 100+개 다국어 음성
- Docker 1줄 실행

**단점:**
- Microsoft 온라인 서비스 의존 (인터넷 연결 필수)
- 상용 SLA 없음 (서비스 중단 가능성)
- OpenAI TTS보다 음질 약간 낮음

---

## 3. Kokoro / SpeakCrate (완전 오프라인)

로컬 서버에서 완전히 돌아가는 오픈소스 TTS 엔진입니다.

### 3.1 특징

| 항목 | 내용 |
|------|------|
| 비용 | 완전 무료 |
| 음질 | 우수 (Kokoro-82M 모델, 자연스러움) |
| 음성 | 54개 (9개 언어 — 영어, 일본어, 중국어, 스페인어, 프랑스어, 이탈리아어 등) |
| 오프라인 | 인터넷 없이 동작 가능 (모델 사전 캐싱) |
| GPU | CPU만으로 동작, NVIDIA GPU 가속 옵션 (`:cuda` 이미지) |
| API 호환 | OpenAI 호환 (`/v1/audio/speech`) |
| GitHub | github.com/hwdsl2/speakcrate |
| 라이선스 | Apache 2.0 |

### 3.2 설치 및 실행

**CPU 버전:**

```bash
docker run -d -p 8880:8880 \
  --name kokoro-tts \
  --restart unless-stopped \
  hwdsl2/kokoro-server:latest
```

**GPU 버전 (NVIDIA 필요):**

```bash
docker run -d -p 8880:8880 \
  --name kokoro-tts \
  --gpus all \
  --restart unless-stopped \
  hwdsl2/kokoro-server:cuda
```

### 3.3 API 호출 예시

```bash
# API 키 획득 (최초 1회)
speak_api_key=$(docker exec kokoro-tts kokoro_manage --getkey)

# 음성 생성
curl http://localhost:8880/v1/audio/speech \
  -H "Authorization: Bearer $speak_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tts-1",
    "input": "Could you tell me how to get to the station?",
    "voice": "af_heart"
  }' \
  --output speech.mp3
```

### 3.4 사용 가능한 음성 (일부)

| 음성 ID | 특징 |
|---------|------|
| `af_heart` | 여성, 따뜻한 |
| `af_bella` | 여성, 부드러운 |
| `bm_george` | 남성, 깊은 |
| `bm_lewis` | 남성, 차분한 |

> OpenAI 음성 이름(`alloy`, `nova` 등)도 자동 매핑됨

### 3.5 장단점

**장점:**
- 완전 오프라인 동작 (데이터 외부 유출 없음)
- 고품질 자연스러운 음성
- CPU만으로 동작 (GPU 불필요)
- OpenAI API 호환

**단점:**
- 한국어 미지원 (영어, 일본어, 중국어 등 9개 언어)
- 최초 모델 다운로드 필요 (약 200MB)
- CPU만 사용 시 긴 텍스트 처리 약간 느림

---

## 4. XTTS-v2 (음성 복제 지원)

Coqui XTTS-v2 기반, 음성 복제(voice cloning)가 가능한 무료 TTS입니다.

### 4.1 특징

| 항목 | 내용 |
|------|------|
| 비용 | 무료 (HuggingFace Spaces에 배포 시) |
| 음질 | 우수 (자연스러운 음성) |
| 음성 복제 | 6~30초 샘플 오디오로 새 음성 생성 |
| 언어 | 17개 언어 (한국어 포함) |
| 배포 | HuggingFace Spaces (무료) 또는 자체 서버 |
| GitHub | github.com/anubhav-n-mishra/xtts-api |

### 4.2 HuggingFace 무료 배포

1. https://huggingface.co/spaces 접속
2. "Create new Space" 클릭
3. Space name: `dailystudy-tts`, SDK: Docker, Visibility: Public
4. 파일 업로드 후 자동 배포

### 4.3 API 호출 예시

```bash
curl -X POST "https://your-space.hf.space/tts" \
  -H "key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Could you tell me how to get to the station?",
    "voice": "default",
    "language": "en",
    "format": "mp3",
    "speed": 1.0
  }' \
  --output speech.mp3
```

### 4.4 장단점

**장점:**
- 음성 복제 가능 (특정 목소리로 학습)
- 17개 언어 (한국어 포함)
- HuggingFace 무료 배포

**단점:**
- 설정 복잡 (GPU 권장, CPU는 느림)
- 음성 복제 시 윤리적 고려 필요
- HuggingFace 무료 tier에 리소스 제한

---

## 5. Google Cloud TTS (무료 한도)

Google Cloud의 TTS 서비스로 매월 무료 한도를 제공합니다.

### 5.1 특징

| 항목 | 내용 |
|------|------|
| 비용 | 매월 400만 자 무료 (약 1시간 분량) |
| 초과 시 | $4/100만 자 |
| 음질 | 매우 우수 (WaveNet, Neural2, Journey) |
| 음성 | 220+개 (40+ 언어, 한국어 포함) |
| API | Google Cloud REST API / SDK |
| URL | cloud.google.com/text-to-speech |

### 5.2 설정 절차

1. **Google Cloud 프로젝트 생성**
   - https://console.cloud.google.com/ 접속
   - "프로젝트 만들기" → 이름: `daily-study`

2. **Text-to-Speech API 활성화**
   - "API 및 서비스" → "API 사용 설정"
   - "Cloud Text-to-Speech API" 검색 → 활성화

3. **서비스 계정 키 생성**
   - "IAM 및 관리자" → "서비스 계정"
   - 계정 생성 → "키" 탭 → "키 추가" → JSON 형식
   - 다운로드한 JSON 파일을 서버에 저장

4. **환경변수 설정**
   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS=/opt/dailystudy/config/google-tts-key.json
   ```

### 5.3 API 호출 예시

```bash
# 액세스 토큰 획득
TOKEN=$(gcloud auth application-default print-access-token)

# 음성合成
curl -X POST \
  "https://texttospeech.googleapis.com/v1/text:synthesize" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "text": "Could you tell me how to get to the station?"
    },
    "voice": {
      "languageCode": "en-US",
      "name": "en-US-Neural2-F"
    },
    "audioConfig": {
      "audioEncoding": "MP3",
      "speakingRate": 1.0,
      "pitch": 0.0
    }
  }'
# 응답: base64 인코딩된 audioContent
```

### 5.4 한국어 음성

| 음성 | 특징 |
|------|------|
| `ko-KR-Neural2-A` | 여성, 자연스러움 |
| `ko-KR-Neural2-B` | 남성, 자연스러움 |
| `ko-KR-Neural2-C` | 여성, 밝음 |
| `ko-KR-Wavenet-A` | 여성, WaveNet (최고 음질) |

### 5.5 장단점

**장점:**
- 가장 음질이 우수 (WaveNet/Neural2)
- 한국어 최고 음질
- 매월 400만 자 무료 (Daily Study 사용량 충분)
- Google Cloud 안정성

**단점:**
- 가입/인증 절차 복잡 (서비스 계정 JSON)
- 응답이 base64 인코딩 (디코딩 필요)
- 400만 자 초과 시 유료

---

## 6. 비교표

| 항목 | Edge TTS | Kokoro | XTTS-v2 | Google Cloud TTS | OpenAI TTS |
|------|----------|--------|---------|-----------------|-----------|
| **비용** | 무료 | 무료 | 무료 | 400만자/월 무료 | $0.015/1K자 |
| **음질** | 양호 | 우수 | 우수 | 매우 우수 | 우수 |
| **한국어** | 지원 | 미지원 | 지원 | 지원 | 지원 |
| **오프라인** | 불가 | 가능 | 가능 | 불가 | 불가 |
| **음성 수** | 100+ | 54 | 17개 언어 | 220+ | 13 |
| **음성 복제** | 불가 | 불가 | 가능 | 불가 | 가능 (Custom) |
| **API 호환** | OpenAI 호환 | OpenAI 호환 | 자체 | Google REST | 자체 |
| **셋업** | 매우 쉬움 | 보통 | 어려움 | 보통 | 매우 쉬움 |
| **안정성** | 낮음 (MS 의존) | 중간 (자체 관리) | 낮음 | 높음 (Google SLA) | 높음 (OpenAI SLA) |
| **스트리밍** | 지원 (SSE) | 지원 | 미지원 | 미지원 | 지원 |
| **출력 포맷** | mp3/opus/aac/flac/wav/pcm | mp3/wav/flac/opus/aac/pcm | mp3/wav | mp3/wav/ogg | mp3/opus/aac/flac/wav/pcm |

---

## 7. Daily Study 추천 전략

| 환경 | 추천 솔루션 | 이유 |
|------|------------|------|
| **개발/테스트** | Edge TTS (Docker) | 즉시 실행, 비용 0원, API 키 불필요 |
| **프로덕션 (초기)** | Google Cloud TTS | 무료 한도 400만 자/월 충분, 음질 우수, 한국어 지원 |
| **프로덕션 (고품질)** | OpenAI TTS | $1~2/월, 가장 자연스러운 영어 발음 |
| **오프라인/보안** | Kokoro (SpeakCrate) | 완전 로컬, 데이터 유출 없음 |

### 단계별 마이그레이션

```
Phase 1 (MVP 개발)
  └── Edge TTS — 비용 0원, Docker 1줄, 즉시 사용

Phase 2 (프로덕션 오픈)
  └── Google Cloud TTS — 무료 한도 내 사용, 안정적

Phase 3 (사용자 증가)
  └── OpenAI TTS — 고품질 영어 발음, 월 $1~2

Phase 4 (상용화)
  └── 캐싱 + 혼용 — 자주 재생되는 문장은 Edge TTS, 신규/고품질은 OpenAI
```

---

## 8. 멀티 프로바이더 전환 설계

설정 파일 한 줄로 TTS 프로바이더를 전환할 수 있는 설계입니다.

### 8.1 application.yml

```yaml
tts:
  provider: ${TTS_PROVIDER:edge}  # edge | google | openai | kokoro
  
  edge:
    url: http://localhost:5050/v1/audio/speech
    voice: en-US-JennyNeural
    api-key: any-key-works  # 더미 키
  
  google:
    credentials-path: ${GOOGLE_APPLICATION_CREDENTIALS}
    language-code: en-US
    voice: en-US-Neural2-F
  
  openai:
    key: ${OPENAI_API_KEY}
    model: gpt-4o-mini-tts
    voice: coral
  
  kokoro:
    url: http://localhost:8880/v1/audio/speech
    voice: af_heart
    api-key: ${KOKORO_API_KEY}
  
  # 공통 설정
  response-format: mp3
  cache-ttl-hours: 24
  upload-dir: /opt/dailystudy/uploads/tts
```

### 8.2 환경변수

```bash
# systemd 서비스 파일
[Service]
Environment=TTS_PROVIDER=edge
Environment=OPENAI_API_KEY=sk-proj-xxxx
Environment=GOOGLE_APPLICATION_CREDENTIALS=/opt/dailystudy/config/google-tts-key.json
Environment=KOKORO_API_KEY=your-kokoro-key
```

---

## 9. Spring Boot 멀티 TtsService 구현

### 9.1 TtsProvider 인터페이스

```java
package org.shaloman.pj.dailystudy.english.tts;

public interface TtsProvider {
    
    /**
     * 텍스트를 음성으로 변환하여 MP3 바이트 배열 반환
     */
    byte[] synthesize(String text, String voice) throws Exception;
    
    /**
     * 프로바이더 이름
     */
    String getName();
}
```

### 9.2 EdgeTtsProvider

```java
package org.shaloman.pj.dailystudy.english.tts;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

@Component
public class EdgeTtsProvider implements TtsProvider {

    @Value("${tts.edge.url:http://localhost:5050/v1/audio/speech}")
    private String apiUrl;

    @Value("${tts.edge.voice:en-US-JennyNeural}")
    private String defaultVoice;

    @Value("${tts.edge.api-key:any-key-works}")
    private String apiKey;

    private final RestTemplate restTemplate = new RestTemplate();

    @Override
    public byte[] synthesize(String text, String voice) throws Exception {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(apiKey);

        Map<String, Object> body = Map.of(
            "model", "tts-1",
            "input", text,
            "voice", voice != null ? voice : defaultVoice,
            "response_format", "mp3"
        );

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(body, headers);
        ResponseEntity<byte[]> response = restTemplate.exchange(
            apiUrl, HttpMethod.POST, entity, byte[].class
        );
        return response.getBody();
    }

    @Override
    public String getName() { return "edge"; }
}
```

### 9.3 OpenAiTtsProvider

```java
package org.shaloman.pj.dailystudy.english.tts;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

@Component
public class OpenAiTtsProvider implements TtsProvider {

    @Value("${tts.openai.key:${OPENAI_API_KEY:}}")
    private String apiKey;

    @Value("${tts.openai.model:gpt-4o-mini-tts}")
    private String model;

    @Value("${tts.openai.voice:coral}")
    private String defaultVoice;

    private final RestTemplate restTemplate = new RestTemplate();

    @Override
    public byte[] synthesize(String text, String voice) throws Exception {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(apiKey);

        Map<String, Object> body = Map.of(
            "model", model,
            "input", text,
            "voice", voice != null ? voice : defaultVoice,
            "response_format", "mp3"
        );

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(body, headers);
        ResponseEntity<byte[]> response = restTemplate.exchange(
            "https://api.openai.com/v1/audio/speech",
            HttpMethod.POST, entity, byte[].class
        );
        return response.getBody();
    }

    @Override
    public String getName() { return "openai"; }
}
```

### 9.4 GoogleTtsProvider

```java
package org.shaloman.pj.dailystudy.english.tts;

import com.google.auth.oauth2.GoogleCredentials;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.io.FileInputStream;
import java.util.Base64;
import java.util.Map;

@Component
public class GoogleTtsProvider implements TtsProvider {

    @Value("${tts.google.credentials-path:${GOOGLE_APPLICATION_CREDENTIALS:}}")
    private String credentialsPath;

    @Value("${tts.google.voice:en-US-Neural2-F}")
    private String defaultVoice;

    @Value("${tts.google.language-code:en-US}")
    private String languageCode;

    private final RestTemplate restTemplate = new RestTemplate();

    @Override
    public byte[] synthesize(String text, String voice) throws Exception {
        // Google Cloud 액세스 토큰 획득
        GoogleCredentials credentials = GoogleCredentials
            .fromStream(new FileInputStream(credentialsPath))
            .createScoped("https://www.googleapis.com/auth/cloud-platform");
        credentials.refreshIfExpired();
        String accessToken = credentials.getAccessToken().getTokenValue();

        // API 호출
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(accessToken);

        String v = voice != null ? voice : defaultVoice;
        Map<String, Object> body = Map.of(
            "input", Map.of("text", text),
            "voice", Map.of("languageCode", languageCode, "name", v),
            "audioConfig", Map.of("audioEncoding", "MP3")
        );

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(body, headers);
        ResponseEntity<Map> response = restTemplate.exchange(
            "https://texttospeech.googleapis.com/v1/text:synthesize",
            HttpMethod.POST, entity, Map.class
        );

        // base64 디코딩
        String audioContent = (String) response.getBody().get("audioContent");
        return Base64.getDecoder().decode(audioContent);
    }

    @Override
    public String getName() { return "google"; }
}
```

### 9.5 TtsService (프로바이더 라우팅)

```java
package org.shaloman.pj.dailystudy.english.service;

import org.shaloman.pj.dailystudy.english.tts.TtsProvider;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.security.MessageDigest;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class TtsService {

    @Value("${tts.provider:edge}")
    private String providerName;

    @Value("${tts.response-format:mp3}")
    private String responseFormat;

    @Value("${tts.cache-ttl-hours:24}")
    private int cacheTtlHours;

    @Value("${tts.upload-dir:/opt/dailystudy/uploads/tts}")
    private String uploadDir;

    private final RedisTemplate<String, String> redisTemplate;
    private final List<TtsProvider> providers;
    private final Map<String, TtsProvider> providerMap = new ConcurrentHashMap<>();

    public TtsService(RedisTemplate<String, String> redisTemplate,
                       List<TtsProvider> providers) {
        this.redisTemplate = redisTemplate;
        this.providers = providers;
        for (TtsProvider p : providers) {
            providerMap.put(p.getName(), p);
        }
    }

    /**
     * TTS 음성 합성 — 캐싱 + 프로바이더 라우팅
     */
    public String generateTts(Long sentenceId, String text) throws Exception {
        String cacheKey = "tts:" + sentenceId;

        // 1. Redis 캐시 확인
        String cachedUrl = redisTemplate.opsForValue().get(cacheKey);
        if (cachedUrl != null) {
            return cachedUrl;
        }

        // 2. 파일명 생성 (text 해시)
        String hash = sha256(text).substring(0, 16);
        String fileName = hash + ".mp3";
        String filePath = uploadDir + "/" + fileName;
        String audioUrl = "/uploads/tts/" + fileName;

        // 3. 파일이 이미 존재하면 반환
        if (new File(filePath).exists()) {
            redisTemplate.opsForValue().set(cacheKey, audioUrl, Duration.ofHours(cacheTtlHours));
            return audioUrl;
        }

        // 4. 프로바이더 선택
        TtsProvider provider = providerMap.get(providerName);
        if (provider == null) {
            throw new IllegalStateException("Unknown TTS provider: " + providerName);
        }

        // 5. 음성 합성
        byte[] audioData = provider.synthesize(text, null);

        // 6. 파일 저장
        Path dir = Paths.get(uploadDir);
        Files.createDirectories(dir);
        Files.write(Paths.get(filePath), audioData);

        // 7. Redis 캐싱
        redisTemplate.opsForValue().set(cacheKey, audioUrl, Duration.ofHours(cacheTtlHours));

        return audioUrl;
    }

    /**
     * 프로바이더 전환 (런타임)
     */
    public void switchProvider(String name) {
        if (providerMap.containsKey(name)) {
            // 설정값 변경은 application.yml 또는 환경변수로
            // 런타임 전환은 관리자 API에서 처리
        }
    }

    private String sha256(String input) throws Exception {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(input.getBytes());
        StringBuilder hex = new StringBuilder();
        for (byte b : hash) {
            hex.append(String.format("%02x", b));
        }
        return hex.toString();
    }
}
```

---

## 10. Docker Compose 통합

Daily Study의 `docker-compose.yml`에 TTS 서비스를 통합합니다.

### 10.1 Edge TTS 통합

```yaml
version: '3.8'

services:
  # 백엔드 (Spring Boot)
  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - TTS_PROVIDER=edge
      - SPRING_PROFILES_ACTIVE=prod
    depends_on:
      - mariadb
      - redis
      - edge-tts

  # 프론트엔드 (Vue 3 — Nginx)
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend

  # MariaDB
  mariadb:
    image: mariadb:11
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: dailystudy
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mariadb-data:/var/lib/mysql

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Edge TTS (무료)
  edge-tts:
    image: travisvn/openai-edge-tts:latest
    ports:
      - "5050:5050"
    restart: unless-stopped

volumes:
  mariadb-data:
```

### 10.2 프로바이더별 전환

```yaml
# Edge TTS (무료, 개발)
environment:
  - TTS_PROVIDER=edge

# Google Cloud TTS (무료 한도)
environment:
  - TTS_PROVIDER=google
  - GOOGLE_APPLICATION_CREDENTIALS=/opt/dailystudy/config/google-tts-key.json
volumes:
  - ./config/google-tts-key.json:/opt/dailystudy/config/google-tts-key.json:ro

# OpenAI TTS (유료, 고품질)
environment:
  - TTS_PROVIDER=openai
  - OPENAI_API_KEY=sk-proj-xxxx

# Kokoro (무료, 오프라인)
environment:
  - TTS_PROVIDER=kokoro
  - KOKORO_API_KEY=your-key
# + kokoro 서비스 추가:
  kokoro:
    image: hwdsl2/kokoro-server:latest
    ports:
      - "8880:8880"
    restart: unless-stopped
```

---

## 11. 참고 자료

### 오픈소스 프로젝트

| 프로젝트 | GitHub | 라이선스 |
|----------|--------|---------|
| OpenAI Edge TTS | github.com/travisvn/openai-edge-tts | GPL v3.0 |
| SpeakCrate (Kokoro) | github.com/hwdsl2/speakcrate | Apache 2.0 |
| XTTS-v2 API | github.com/anubhav-n-mishra/xtts-api | MPL 2.0 |
| EdgeVoice | github.com/OOMrConrado/EdgeVoice | MIT |

### 클라우드 TTS

| 서비스 | URL | 무료 한도 |
|--------|-----|----------|
| Google Cloud TTS | cloud.google.com/text-to-speech | 400만 자/월 |
| OpenAI TTS | platform.openai.com/docs/guides/text-to-speech | 없음 (유료) |
| Azure TTS | azure.microsoft.com/products/ai-services/text-to-speech | 50만 자/월 |
| AWS Polly | aws.amazon.com/polly | 500만 자/월 (12개월) |

### 음성 미리 듣기

| 서비스 | URL |
|--------|-----|
| OpenAI.fm | https://openai.fm/ |
| Edge TTS 음성 목록 | https://www.tts.best/ |
| Google Cloud 음성 데모 | https://cloud.google.com/text-to-speech#section-3 |

---

> **Daily Study 프로젝트** — 무료 TTS 솔루션 가이드 v1.0  
> 개발: Edge TTS (무료) → 프로덕션: Google Cloud TTS (무료 한도) → 고품질: OpenAI TTS ($1~2/월)