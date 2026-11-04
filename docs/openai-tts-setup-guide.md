# OpenAI TTS (Text-to-Speech) 연동 가이드

> Daily Study 프로젝트 — 실생활 영어 발음 음원 생성용 OpenAI TTS API 연동 절차  
> 작성일: 2026-11-04  
> 버전: v1.0

---

## 목차

1. [OpenAI TTS 개요](#1-openai-tts-개요)
2. [OpenAI 계정 가입](#2-openai-계정-가입)
3. [결제 수단 등록](#3-결제-수단-등록)
4. [API 키 발급](#4-api-키-발급)
5. [TTS API 호출 방법](#5-tts-api-호출-방법)
6. [사용 가능한 모델](#6-사용-가능한-모델)
7. [음성(Voice) 옵션](#7-음성voice-옵션)
8. [출력 포맷](#8-출력-포맷)
9. [비용 구조](#9-비용-구조)
10. [Daily Study 적용 설계](#10-daily-study-적용-설계)
11. [Spring Boot 연동 코드 예시](#11-spring-boot-연동-코드-예시)
12. [Vue 프론트엔드 연동 예시](#12-vue-프론트엔드-연동-예시)
13. [캐싱 전략](#13-캐싱-전략)
14. [Rate Limit 관리](#14-rate-limit-관리)
15. [무료 대안: Edge TTS](#15-무료-대안-edge-tts)
16. [트러블슈팅](#16-트러블슈팅)

---

## 1. OpenAI TTS 개요

OpenAI TTS(Text-to-Speech) API는 텍스트를 자연스러운 음성 오디오로 변환하는 서비스입니다.

**Daily Study 적용 분야:**
- 실생활 영어 문장의 원어민 발음 음원 생성
- 학생이 영어 문장 옆 `🔊 재생` 버튼 클릭 → MP3 오디오 재생
- 크롤링 파이프라인에서 문장 정제 시 미리 TTS 음원 생성 (배치 처리)

**핵심 특징:**
- 13개 내장 음성(Voice)
- 50+ 언어 지원 (한국어 포함, 영어에 최적화)
- 6가지 출력 포맷 (MP3, Opus, AAC, FLAC, WAV, PCM)
- 실시간 스트리밍 지원
- 음성 톤/감정/속도 제어 (gpt-4o-mini-tts 모델)

---

## 2. OpenAI 계정 가입

### 2.1 회원가입 절차

1. **OpenAI Platform 접속**
   - URL: https://platform.openai.com/
   - "Sign up" 버튼 클릭

2. **가입 방법 선택**
   - 이메일 가입 또는
   - Google 계정 연동 또는
   - Microsoft 계정 연동 또는
   - Apple 계정 연동

3. **이메일 가입 시**
   - 이메일 주소 입력 → "Continue"
   - 비밀번호 설정 (8자 이상, 영문+숫자+특수문자 조합 권장)
   - 이메일 인증 메일 수신 → "Verify email address" 클릭

4. **본인 확인**
   - 이름 입력 (First Name / Last Name)
   - 휴대전화 번호 입력 (SMS 인증)
   - 인증 코드 6자리 입력

5. **가입 완료**
   - OpenAI Platform 대시보드로 이동
   - URL: https://platform.openai.com/usage

> **주의**: 한국 휴대전화 번호로 SMS 인증 가능 (국가 코드 +82 선택)

---

## 3. 결제 수단 등록

TTS API는 유료 서비스이므로 결제 수단 등록이 필요합니다.

### 3.1 결제 수단 등록 절차

1. **Settings → Billing 메뉴 이동**
   - URL: https://platform.openai.com/account/billing
   - 좌측 사이드바 "Settings" → "Billing"

2. **Add payment method 클릭**
   - 신용카드/직불카드 등록
   - 지원 카드: Visa, Mastercard, Amex
   - 한국 카드 사용 가능

3. **충전 금액 설정 (Prepaid)**
   - "Add to credit balance"로 선충전 가능
   - 최소 $5부터 충전 가능
   - 사용한 만큼 차감 (Postpaid가 아닌 Prepaid 방식)

4. **사용 한도 설정 (선택)**
   - "Usage limits"에서 월별 지출 한도 설정 가능
   - Hard limit (강제 중단) / Soft limit (이메일 알림)

> **권장**: 초기에는 $10~$20 충전 후 사용량 모니터링  
> Daily Study TTS 사용량 예상: 영어 문장 1,000개 × 평균 50자 = 50,000자/월 ≈ $0.75/월

---

## 4. API 키 발급

### 4.1 API 키 생성 절차

1. **API Keys 메뉴 이동**
   - URL: https://platform.openai.com/api-keys
   - "Create new secret key" 버튼 클릭

2. **키 이름 설정**
   - Name: `daily-study-tts` (식별 가능한 이름)
   - 권한: "All" 또는 "Restricted" → Audio API만 필요하면 "Restricted" 권장

3. **키 생성**
   - "Create secret key" 클릭
   - **`sk-proj-xxxxxxxxxxxxxxxxxxxx`** 형태의 키 발급
   - ⚠️ **키는 생성 시 한 번만 표시됨 — 즉시 안전한 곳에 복사 저장**

4. **키 보안 관리**
   - 절대 소스코드에 하드코딩하지 않음
   - 환경변수(`OPENAI_API_KEY`)로 관리
   - `.env` 파일은 `.gitignore`에 등록
   - 키 유출 시 즉시 "Revoke" 후 재발급

### 4.2 환경변수 설정

```bash
# /opt/dailystudy/config/application-prod.yml
openai:
  api:
    key: ${OPENAI_API_KEY}
    tts:
      model: gpt-4o-mini-tts
      voice: coral
      response-format: mp3
```

```bash
# systemd 서비스 파일
[Service]
Environment=OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxx
```

---

## 5. TTS API 호출 방법

### 5.1 cURL 예시

```bash
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini-tts",
    "input": "Could you tell me how to get to the station?",
    "voice": "coral",
    "response_format": "mp3"
  }' \
  --output speech.mp3
```

### 5.2 Python 예시

```python
from pathlib import Path
from openai import OpenAI

client = OpenAI()  # OPENAI_API_KEY 환경변수 자동 참조

speech_file_path = Path(__file__).parent / "speech.mp3"

with client.audio.speech.with_streaming_response.create(
    model="gpt-4o-mini-tts",
    voice="coral",
    input="Could you tell me how to get to the station?",
    response_format="mp3",
) as response:
    response.stream_to_file(speech_file_path)

print(f"음원 저장 완료: {speech_file_path}")
```

### 5.3 JavaScript (Node.js) 예시

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI(); // OPENAI_API_KEY 환경변수 자동 참조

const mp3 = await openai.audio.speech.create({
  model: "gpt-4o-mini-tts",
  voice: "coral",
  input: "Could you tell me how to get to the station?",
  response_format: "mp3",
});

const buffer = Buffer.from(await mp3.arrayBuffer());
fs.writeFileSync("speech.mp3", buffer);
console.log("음원 저장 완료: speech.mp3");
```

---

## 6. 사용 가능한 모델

| 모델 | 특징 | 비용 (1,000자당) | 비고 |
|------|------|-----------------|------|
| **gpt-4o-mini-tts** | 최신 모델, 감정/톤/속도 제어 가능, 13개 음성 | $0.015 | **권장** |
| **tts-1** | 표준 모델, 낮은 지연 시간 | $0.015 | 2027.01 폐지 예정 |
| **tts-1-hd** | 고음질 모델, 약간 높은 지연 | $0.030 | 2027.01 폐지 예정 |

> **참고**: `tts-1`과 `tts-1-hd`는 2027년 1월 6일에 폐지 예정. 새 프로젝트는 `gpt-4o-mini-tts` 사용 권장.

### gpt-4o-mini-tts 추가 기능

`instructions` 파라미터로 음성 스타일 제어:

```json
{
  "model": "gpt-4o-mini-tts",
  "voice": "coral",
  "input": "Welcome to Daily Study!",
  "instructions": "Speak in a cheerful and positive tone."
}
```

제어 가능 항목:
- Accent (억양)
- Emotional range (감정 범위)
- Intonation (억양)
- Impressions (모방)
- Speed of speech (속도)
- Tone (톤)
- Whispering (속삭임)

---

## 7. 음성(Voice) 옵션

### 13개 내장 음성

| 음성 | 특징 | 권장 용도 |
|------|------|----------|
| `alloy` | 중성적, 균형잡힌 | 일반적인 안내 |
| `ash` | 차분한 남성 | 뉴스, 설명 |
| `ballad` | 서정적 | 스토리텔링 |
| `coral` | 밝고 친근한 여성 | **학습용 권장** |
| `echo` | 단정한 남성 | 공지사항 |
| `fable` | 서사적 | 이야기 |
| `nova` | 따뜻한 여성 | 친근한 안내 |
| `onyx` | 깊은 남성 | 공식 안내 |
| `sage` | 지적인 여성 | 교육용 |
| `shimmer` | 부드러운 여성 | 감성적 |
| `verse` | 시적인 | 운율 있는 텍스트 |
| `marin` | 고품질 여성 | **최고 음질 권장** |
| `cedar` | 고품질 남성 | **최고 음질 권장** |

> **Daily Study 권장**: `coral` (밝고 친근한 여성 — 학생 친화적)  
> **최고 음질 필요 시**: `marin` 또는 `cedar`

### 음성 미리 듣기

OpenAI.fm (https://openai.fm/) 에서 13개 음성을 직접 들어볼 수 있습니다.

---

## 8. 출력 포맷

| 포맷 | 확장자 | 특징 | 권장 용도 |
|------|--------|------|----------|
| **MP3** | `.mp3` | 기본값, 범용 | **Daily Study 기본** |
| Opus | `.opus` | 저지연 스트리밍 | 실시간 통신 |
| AAC | `.aac` | 압축률 높음 | 모바일 |
| FLAC | `.flac` | 무손실 압축 | 아카이브 |
| WAV | `.wav` | 비압축, 저지연 | 즉시 재생 |
| PCM | `.pcm` | raw 24kHz 16-bit | 커스텀 처리 |

> **Daily Study**: MP3 사용 (브라우저 호환성 최고, 파일 크기 적당)

---

## 9. 비용 구조

### 9.1 가격표

| 모델 | 1,000자당 비용 | 1분 음성 (약 150자) | 1시간 음성 (약 9,000자) |
|------|---------------|---------------------|----------------------|
| gpt-4o-mini-tts | $0.015 | $0.00225 | $0.135 |
| tts-1 | $0.015 | $0.00225 | $0.135 |
| tts-1-hd | $0.030 | $0.00450 | $0.270 |

### 9.2 Daily Study 월 예상 비용

| 항목 | 계산 | 월 비용 |
|------|------|---------|
| 영어 문장 1,000개 (신규) | 1,000 × 50자 × $0.015/1K | $0.75 |
| 매일 10문장 추가 (월 300개) | 300 × 50자 × $0.015/1K | $0.23 |
| **월 예상 총비용** | | **약 $1~2** |

> 캐싱 적용 시 동일 문장 재생성 비용 발생 안 함 → 실제 비용은 더 낮음

---

## 10. Daily Study 적용 설계

### 10.1 아키텍처

```
[학생]                    [백엔드]                      [OpenAI API]
  │                        │                              │
  │ POST /api/english/tts   │                              │
  │ {sentenceId: 42}       │                              │
  │ ────────────────────→   │                              │
  │                        │ 1. Redis 캐시 확인            │
  │                        │    key: tts:{sentence_id}     │
  │                        │    hit → audio_url 즉시 반환   │
  │                        │                              │
  │                        │ 2. 캐시 미스 → DB 확인        │
  │                        │    ENGLISH_SENTENCE.tts_url   │
  │                        │    hit → Redis 캐싱 후 반환    │
  │                        │                              │
  │                        │ 3. DB 미스 → OpenAI 호출      │
  │                        │    POST /v1/audio/speech      │
  │                        │ ─────────────────────────→    │
  │                        │                              │
  │                        │    ← MP3 바이너리 응답        │
  │                        │    ← ──────────────────────   │
  │                        │                              │
  │                        │ 4. 파일 저장                  │
  │                        │    /uploads/tts/{hash}.mp3    │
  │                        │                              │
  │                        │ 5. DB + Redis 캐싱            │
  │                        │    tts_url 저장                │
  │                        │    Redis TTL 24h              │
  │                        │                              │
  │  ← audioUrl 응답       │                              │
  │  {"audioUrl":           │                              │
  │   "/uploads/tts/x.mp3"}│                              │
  │                        │                              │
  │ <audio src="url" />    │                              │
  │ ▶ 재생                 │                              │
```

### 10.2 API 엔드포인트

```
POST /api/english/tts
Authorization: Bearer <JWT>
Content-Type: application/json

요청:
{
  "sentenceId": 42,
  "text": "Could you tell me how to get to the station?"
}

응답 (200):
{
  "audioUrl": "/uploads/tts/a3f2b1c8.mp3"
}

에러 (429):
{
  "error": "Too Many Requests",
  "message": "TTS 요청 한도를 초과했습니다. (시간당 30회)"
}

에러 (503):
{
  "error": "Service Unavailable",
  "message": "TTS 서비스 일시 오류. 잠시 후 다시 시도해주세요."
}
```

---

## 11. Spring Boot 연동 코드 예시

### 11.1 TtsService.java

```java
package org.shaloman.pj.dailystudy.english.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.security.MessageDigest;
import java.time.Duration;
import java.util.Map;

@Service
public class TtsService {

    @Value("${openai.api.key}")
    private String apiKey;

    @Value("${openai.api.tts.model:gpt-4o-mini-tts}")
    private String ttsModel;

    @Value("${openai.api.tts.voice:coral}")
    private String defaultVoice;

    @Value("${holylife.upload.path:/opt/dailystudy/uploads}")
    private String uploadPath;

    private final RestTemplate restTemplate;
    private final RedisTemplate<String, String> redisTemplate;

    public TtsService(RestTemplate restTemplate, RedisTemplate<String, String> redisTemplate) {
        this.restTemplate = restTemplate;
        this.redisTemplate = redisTemplate;
    }

    /**
     * TTS 음성 합성
     * 1. Redis 캐시 확인 → 2. DB 확인 → 3. OpenAI API 호출
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
        String filePath = uploadPath + "/tts/" + fileName;
        String audioUrl = "/uploads/tts/" + fileName;

        // 3. 이미 파일이 존재하면 반환
        if (new File(filePath).exists()) {
            redisTemplate.opsForValue().set(cacheKey, audioUrl, Duration.ofHours(24));
            return audioUrl;
        }

        // 4. OpenAI TTS API 호출
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(apiKey);

        Map<String, Object> requestBody = Map.of(
            "model", ttsModel,
            "voice", defaultVoice,
            "input", text,
            "response_format", "mp3"
        );

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);

        ResponseEntity<byte[]> response = restTemplate.exchange(
            "https://api.openai.com/v1/audio/speech",
            HttpMethod.POST,
            entity,
            byte[].class
        );

        // 5. 음원 파일 저장
        Path dir = Paths.get(uploadPath, "tts");
        Files.createDirectories(dir);
        Files.write(Paths.get(filePath), response.getBody());

        // 6. Redis 캐싱 (24시간)
        redisTemplate.opsForValue().set(cacheKey, audioUrl, Duration.ofHours(24));

        return audioUrl;
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

### 11.2 EnglishController.java (TTS 엔드포인트)

```java
@RestController
@RequestMapping("/api/english")
public class EnglishController {

    private final TtsService ttsService;
    private final EnglishSentenceService sentenceService;

    @PostMapping("/tts")
    public ResponseEntity<Map<String, String>> generateTts(@RequestBody TtsRequestDto request) {
        try {
            String audioUrl = ttsService.generateTts(request.getSentenceId(), request.getText());
            return ResponseEntity.ok(Map.of("audioUrl", audioUrl));
        } catch (Exception e) {
            return ResponseEntity.status(503)
                .body(Map.of("error", "Service Unavailable",
                             "message", "TTS 서비스 오류. 잠시 후 다시 시도해주세요."));
        }
    }
}
```

### 11.3 TtsRequestDto.java

```java
package org.shaloman.pj.dailystudy.english.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class TtsRequestDto {
    @NotNull
    private Long sentenceId;

    @NotBlank
    private String text;
}
```

### 11.4 application.yml 설정

```yaml
openai:
  api:
    key: ${OPENAI_API_KEY}
    tts:
      model: gpt-4o-mini-tts
      voice: coral
      response-format: mp3
      timeout-ms: 30000
```

---

## 12. Vue 프론트엔드 연동 예시

### 12.1 TtsPlayer.vue

```vue
<template>
  <div class="tts-player">
    <button
      @click="playTts"
      :disabled="loading"
      class="btn btn-primary btn-sm"
    >
      <span v-if="loading" class="loading loading-spinner loading-sm"></span>
      <span v-else>🔊 재생</span>
    </button>
    <audio ref="audioPlayer" @ended="onEnded" />
  </div>
</template>

<script setup>
import { ref } from 'vue';
import axios from 'axios';

const props = defineProps({
  sentenceId: { type: Number, required: true },
  text: { type: String, required: true }
});

const loading = ref(false);
const audioPlayer = ref(null);
const audioUrl = ref(null);

const playTts = async () => {
  if (audioUrl.value) {
    // 캐시된 URL이 있으면 즉시 재생
    audioPlayer.value.src = audioUrl.value;
    audioPlayer.value.play();
    return;
  }

  loading.value = true;
  try {
    const res = await axios.post('/api/english/tts', {
      sentenceId: props.sentenceId,
      text: props.text
    });
    audioUrl.value = res.data.audioUrl;
    audioPlayer.value.src = audioUrl.value;
    audioPlayer.value.play();
  } catch (err) {
    if (err.response?.status === 429) {
      alert('TTS 요청 한도를 초과했습니다. 잠시 후 다시 시도해주세요.');
    } else {
      alert('음성 재생 중 오류가 발생했습니다.');
    }
  } finally {
    loading.value = false;
  }
};

const onEnded = () => {
  // 재생 완료
};

defineExpose({ playTts });
</script>
```

### 12.2 EnglishDetailView.vue 사용 예시

```vue
<template>
  <div class="english-detail">
    <div class="sentence-card">
      <p class="english-text">{{ sentence.english }}</p>
      <p class="korean-text">{{ sentence.korean }}</p>
      <TtsPlayer :sentenceId="sentence.id" :text="sentence.english" />
    </div>
  </div>
</template>

<script setup>
import TtsPlayer from '@/components/TtsPlayer.vue';
</script>
```

---

## 13. 캐싱 전략

### 3계층 캐싱 구조

| 계층 | 위치 | TTL | 목적 |
|------|------|-----|------|
| **L1: Redis** | `tts:{sentence_id}` | 24시간 | 빠른 응답 (1ms) |
| **L2: DB** | `ENGLISH_SENTENCE.tts_url` | 영구 | 영속 저장 |
| **L3: 파일 시스템** | `/uploads/tts/{hash}.mp3` | 영구 | 실제 음원 파일 |

### 캐싱 플로우

1. **Redis Hit** → audio_url 즉시 반환 (API 호출 없음)
2. **Redis Miss → DB Hit** → Redis 캐싱 후 반환 (API 호출 없음)
3. **DB Miss → 파일 존재** → Redis 캐싱 후 반환 (API 호출 없음)
4. **전부 Miss** → OpenAI API 호출 → 파일 저장 → DB + Redis 캐싱

### 크롤링 배치 연동

크롤링 파이프라인에서 문장 수집 → AI 정제 → **TTS 음원 미리 생성**:

```
Spring Batch (매일 새벽 4시)
  → Jsoup 크롤링 (영어 문장 수집)
  → GPT-4o-mini 정제 (문법 검증, 번역)
  → OpenAI TTS 음원 생성 (gpt-4o-mini-tts)
  → 파일 저장 + DB tts_url 등록
  → 관리자 검수 큐 전달
```

> 사용자가 재생 버튼을 누를 때는 이미 음원이 준비되어 있음 → 실시간 API 호출 불필요

---

## 14. Rate Limit 관리

### 백엔드 Rate Limiting

| 엔드포인트 | 사용자당 한도 | 비고 |
|-----------|--------------|------|
| `POST /api/english/tts` | 30회/시간 | Redis 기반 Rate Limiter |

### OpenAI API Rate Limit

| 등급 | TTS RPM (분당 요청) |
|------|-------------------|
| Free | 50 RPM |
| Tier 1 ($5+ 사용) | 100 RPM |
| Tier 2 ($50+ 사용) | 500 RPM |
| Tier 3 ($100+ 사용) | 1,000 RPM |

> Daily Study 초기에는 Free/Tier 1로 충분 (월 $1~2 사용량)

---

## 15. 무료 대안: Edge TTS

비용을 완전히 zero로 만드는 대안:

### OpenAI Edge TTS (오픈소스)

- GitHub: https://github.com/travisvn/openai-edge-tts
- Microsoft Edge TTS 엔진을 OpenAI 호환 API로 래핑
- **완전 무료** — API 키 불필요
- 로컬 Docker 컨테이너로 실행
- OpenAI TTS와 동일한 요청/응답 구조 → 코드 변경 최소화

### Docker 실행

```bash
docker run -d -p 5050:5050 \
  --name edge-tts \
  ghcr.io/travisvn/openai-edge-tts:latest
```

### 호출 예시

```bash
# OpenAI TTS와 동일한 구조 — URL만 변경
curl http://localhost:5050/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tts-1",
    "input": "Could you tell me how to get to the station?",
    "voice": "en-US-JennyNeural"
  }' \
  --output speech.mp3
```

### OpenAI vs Edge TTS 비교

| 항목 | OpenAI TTS | Edge TTS |
|------|-----------|----------|
| 비용 | $0.015/1K자 | 무료 |
| 음질 | 우수 (자연스러움) | 양호 (약간 기계음) |
| 음성 수 | 13개 | 100+개 (다국어) |
| 셋업 | API 키만 있으면 즉시 | Docker 컨테이너 실행 |
| 안정성 | 99.9% SLA | 자체 관리 필요 |
| 권장 | 프로덕션 | 개발/테스트, 비용 절감 |

> **Daily Study 권장**: 개발 단계는 Edge TTS, 프로덕션은 OpenAI TTS  
> 설정 파일에서 쉽게 전환 가능하도록 설계

---

## 16. 트러블슈팅

### 16.1 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| 401 Unauthorized | API 키 오류 | 키 재확인, 환경변수 확인 |
| 429 Too Many Requests | Rate Limit 초과 | 캐싱 강화, Edge TTS 전환 |
| 음원 파일 재생 안됨 | 포맷 미지원 | MP3 포맷 사용 |
| 지연 시간 김 | 네트워크, 모델 | tts-1 사용 (gpt-4o-mini-tts보다 빠름) |
| 한국어 발음 이상 | 영어 최적화 모델 | 한국어는 Edge TTS (ko-KR-voice) 고려 |

### 16.2 음원 파일 관리

```bash
# TTS 음원 디렉토리
/opt/dailystudy/uploads/tts/

# 파일 정리 스크립트 (월 1회 — 30일 이상 미사용 파일 삭제)
find /opt/dailystudy/uploads/tts/ -name "*.mp3" -mtime +30 -delete
```

---

## 참고 자료

- [OpenAI TTS 공식 문서](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI API 키 발급](https://platform.openai.com/api-keys)
- [OpenAI 가격 페이지](https://openai.com/pricing)
- [음성 미리 듣기 (OpenAI.fm)](https://openai.fm/)
- [OpenAI Edge TTS (무료 대안)](https://github.com/travisvn/openai-edge-tts)

---

> **Daily Study 프로젝트** — OpenAI TTS 연동 가이드 v1.0  
> 기술 스택: Spring Boot 4.0 + Vue 3 + Redis + OpenAI gpt-4o-mini-tts