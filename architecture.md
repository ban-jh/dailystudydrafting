# Daily Study — 상세 아키텍처 설계서

> **중·고등학생 학습 보조 플랫폼**  
> Spring Boot 4.0 + MyBatis + MariaDB 11 + Redis + Vue 3 + Vite + Pinia + TailwindCSS  
> 작성일: 2026-11-04 / v1.0

---

## 목차

1. [전체 시스템 아키텍처](#1-전체-시스템-아키텍처)
2. [백엔드 패키지 구조](#2-백엔드-패키지-구조)
3. [API 엔드포인트 설계](#3-api-엔드포인트-설계)
4. [데이터베이스 ERD](#4-데이터베이스-erd)
5. [AI 콘텐츠 생성 파이프라인](#5-ai-콘텐츠-생성-파이프라인)
6. [크롤링 파이프라인 설계](#6-크롤링-파이프라인-설계)
7. [보안 설계](#7-보안-설계)
8. [Redis 활용 전략](#8-redis-활용-전략)
9. [프론트엔드 아키텍처](#9-프론트엔드-아키텍처)
10. [배포 아키텍처](#10-배포-아키텍처)

---

## 1. 전체 시스템 아키텍처

### 1.1 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              클라이언트 (Browser)                         │
│                    Vue 3 + Vite + Pinia + TailwindCSS                    │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ HTTPS
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Nginx (Reverse Proxy)                          │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────────────┐ │
│  │ 정적 파일    │  │ /api/* →     │  │ /swagger-ui/* →                 │ │
│  │ 서빙 (Vue)  │  │ Backend:8080 │  │ Backend:8080                    │ │
│  └─────────────┘  └──────────────┘  └─────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Spring Boot 4.0 Backend (:8080)                      │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Controller Layer (REST API)                                      │  │
│  │  Auth / Subject / Summary / Quiz / English / Dashboard / Admin    │  │
│  ├───────────────────────────────────────────────────────────────────┤  │
│  │  Service Layer (Business Logic)                                   │  │
│  │  UserService / SummaryService / QuizService / AiContentService    │  │
│  │  EnglishService / StudyLogService / BadgeService / CrawlService   │  │
│  ├───────────────────────────────────────────────────────────────────┤  │
│  │  AI Content Pipeline                                              │  │
│  │  ┌──────────────┐  ┌──────────────────┐  ┌─────────────────────┐ │  │
│  │  │ GLM-5.2      │  │ GPT-4o-mini      │  │ PromptTemplateMgr   │ │  │
│  │  │ (한국어 콘텐츠) │  │ (영어/보조)       │  │ (템플릿 관리)         │ │  │
│  │  └──────────────┘  └──────────────────┘  └─────────────────────┘ │  │
│  ├───────────────────────────────────────────────────────────────────┤  │
│  │  Data Access Layer (MyBatis XML Mapper)                           │  │
│  ├───────────────────────────────────────────────────────────────────┤  │
│  │  Infrastructure                                                   │  │
│  │  Spring Security + JWT / OAuth2 / Spring Batch / Scheduler        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────┬──────────────┬──────────────┬──────────────┬─────────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────┐
│ MariaDB  │   │  Redis   │   │ TTS API  │   │ External Sources │
│  11      │   │ (Cache + │   │ (음성     │   │ (교과서, 뉴스,    │
│ (RDBMS)  │   │ Session) │   │ 합성)     │   │ 영어 문장)        │
└──────────┘   └──────────┘   └──────────┘   └──────────────────┘
```

### 1.2 핵심 설계 원칙

| 원칙 | 설명 |
|------|------|
| **Domain-Grouped Packaging** | holylife 프로젝트와 동일 — 각 도메인이 `controller/service/domain/dto/mapper` 하위 패키지를 가짐 |
| **MyBatis XML Mapper 방식** | JPA 미사용 — 복잡한 JOIN/동적 SQL을 XML에서 관리 |
| **Stateless JWT 인증** | 서버 세션 미사용 — Redis로 토큰 블랙리스트/Rate Limiting 관리 |
| **AI 콘텐츠 캐싱** | 동일 요약/문제 재생성 방지 — Redis + DB 이중 캐싱 |
| **Phase별 점진적 개발** | Phase 1 MVP → Phase 2 → Phase 3 → Phase 4 순차 구현 |

### 1.3 기술 스택 상세

| 레이어 | 기술 | 버전 | 비고 |
|--------|------|------|------|
| Backend Framework | Spring Boot | 4.0.0 | webmvc starter |
| ORM | MyBatis Spring Boot Starter | 4.0.1 | **3.0.x 사용 불가** (Spring Boot 4.0 호환성) |
| Database | MariaDB | 11.x | utf8mb4_unicode_ci |
| Cache/Session | Redis | 7.x | Docker 컨테이너 |
| API Docs | springdoc-openapi | 2.6.0 | Swagger UI |
| Security | Spring Security + JWT (jjwt 0.12.6) | - | STATELESS |
| Batch | Spring Batch | 5.x | 크롤링/통계 배치 |
| HTML Parsing | JSoup | 1.18.1 | 크롤링용 |
| Frontend Framework | Vue 3 | 3.5.x | Composition API |
| Build Tool | Vite | 6.x | - |
| State Management | Pinia | 2.x | - |
| CSS Framework | TailwindCSS | 3.4.x | - |
| Router | Vue Router | 4.x | History mode |
| HTTP Client | Axios | 1.7.x | - |
| AI - 한국어 | GLM-5.2 cloud | - | 요약, 문제 출제, 설명 |
| AI - 영어/보조 | OpenAI GPT-4o-mini | - | 영어 문장, 보조 생성 |
| TTS | OpenAI TTS / Edge TTS | - | 실생활 영어 발음 |
| Auth | OAuth2 (Google, Kakao) | - | Phase 2 추가 |
| Server OS | Ubuntu 24.04 LTS | - | - |
| Web Server | Nginx | 1.25+ | Reverse proxy + 정적 서빙 |
| Container | Docker + Docker Compose | - | - |
| JDK | OpenJDK | 17 | holylife와 동일 |

---

## 2. 백엔드 패키지 구조

### 2.1 Organizational 패키지 방식 (holylife 패턴 계승)

```
org.shaloman.pj.dailystudy/
│
├── DailyStudyApplication.java              # Spring Boot Main 클래스
│
├── config/                                 # 인프라 설정
│   ├── SecurityConfig.java                 # Spring Security + JWT 필터 체인
│   ├── JwtUtil.java                        # JWT 토큰 생성/검증
│   ├── JwtAuthenticationFilter.java        # JWT 인증 필터
│   ├── RoleAuthorizationFilter.java        # URL 패턴 기반 권한 필터
│   ├── OAuth2Config.java                   # OAuth2 클라이언트 설정 (Phase 2)
│   ├── OAuth2SuccessHandler.java           # OAuth2 로그인 성공 핸들러
│   ├── MyBatisConfig.java                  # @MapperScan("org.shaloman.pj.dailystudy")
│   ├── TransactionConfig.java             # @EnableTransactionManagement
│   ├── RedisConfig.java                    # Redis 접속 + RedisTemplate 설정
│   ├── SwaggerConfig.java                  # OpenAPI 메타데이터
│   ├── WebMvcConfig.java                   # CORS, 인터셉터, 정적 리소스
│   ├── PasswordConfig.java                 # BCryptPasswordEncoder
│   ├── FileUploadConfig.java              # 파일 업로드 경로 설정
│   ├── BatchConfig.java                    # Spring Batch 설정
│   ├── SchedulerConfig.java               # @EnableScheduling
│   ├── GlobalExceptionHandler.java        # @ControllerAdvice 예외 처리
│   └── RateLimitConfig.java               # API Rate Limiting 설정
│
├── common/                                 # 공통 모듈
│   ├── response/
│   │   ├── ApiResponse.java               # 통일 응답 래퍼 {success, data, message, error}
│   │   └── PageResponse.java              # 페이징 응답 {items, totalCount, page, pageSize}
│   ├── exception/
│   │   ├── BusinessException.java         # 비즈니스 예외
│   │   ├── AuthException.java             # 인증/인가 예외
│   │   ├── AiGenerationException.java     # AI 생성 실패 예외
│   │   └── ErrorCode.java                 # 에러 코드 Enum
│   ├── fileupload/
│   │   ├── controller/FileUploadController.java
│   │   └── service/FileUploadService.java
│   ├── util/
│   │   ├── DateUtils.java
│   │   └── TextUtils.java
│   └── interceptor/
│       └── RateLimitInterceptor.java       # Redis 기반 Rate Limiting
│
├── auth/                                   # 인증 도메인
│   ├── controller/
│   │   └── AuthController.java            # POST /api/auth/login, /api/auth/me, /api/auth/refresh
│   ├── service/
│   │   ├── AuthService.java               # 로그인/회원가입/토큰 갱신
│   │   └── OAuth2Service.java             # OAuth2 사용자 연동 (Phase 2)
│   ├── domain/
│   │   └── LoginRequest.java, LoginResponse.java, SignupRequest.java
│   └── dto/
│       └── TokenDto.java
│
├── user/                                  # 사용자 도메인
│   ├── controller/
│   │   └── UserController.java            # /api/users CRUD, /api/users/me
│   ├── service/
│   │   ├── UserService.java
│   │   └── UserSettingsService.java       # 학습 설정 (선호 과목, 알림, 난이도)
│   ├── domain/
│   │   ├── User.java
│   │   ├── UserSettings.java
│   │   └── UserStats.java                 # 학습 통계 집계용
│   ├── dto/
│   │   ├── UserResponseDto.java
│   │   ├── UserUpdateDto.java
│   │   └── UserSettingsDto.java
│   └── mapper/
│       ├── UserMapper.java
│       └── UserSettingsMapper.java
│
├── subject/                               # 과목/단원 도메인
│   ├── controller/
│   │   └── SubjectController.java         # /api/subjects, /api/subjects/{id}/units
│   ├── service/
│   │   ├── SubjectService.java
│   │   └── UnitService.java
│   ├── domain/
│   │   ├── Subject.java                   # 과목 (수학, 영어, 과학, ...)
│   │   └── Unit.java                      # 단원 (1단원, 2단원, ...)
│   ├── dto/
│   │   ├── SubjectResponseDto.java
│   │   └── UnitResponseDto.java
│   └── mapper/
│       ├── SubjectMapper.java
│       └── UnitMapper.java
│
├── summary/                               # 단원 요약 도메인
│   ├── controller/
│   │   └── SummaryController.java         # /api/summaries, /api/summaries/{id}
│   ├── service/
│   │   ├── SummaryService.java            # 요약 CRUD
│   │   └── SummaryCacheService.java       # Redis 캐싱
│   ├── domain/
│   │   └── Summary.java
│   ├── dto/
│   │   ├── SummaryResponseDto.java
│   │   └── SummaryRequestDto.java
│   └── mapper/
│       └── SummaryMapper.java
│
├── quiz/                                  # 문제/풀이 도메인 (관리자 생성→승인→학생 제공)
│   ├── controller/
│   │   ├── QuizController.java            # /api/quizzes (학생용 — 승인된 문제만)
│   │   ├── AdminQuizController.java       # /api/admin/quizzes/* (관리자용 — 생성/승인/반려)
│   │   └── QuizAttemptController.java     # /api/quiz-attempts, /api/quiz-attempts/wrong
│   ├── service/
│   │   ├── QuizPoolService.java           # 승인된 문제 풀 조회 (학생용, Redis 캐싱)
│   │   ├── AiQuizGenerator.java           # AI 문제 일괄 생성 (GLM-5.2) → PENDING
│   │   ├── CrawlingQuizCollector.java     # 크롤링 수집 + AI 정제 → PENDING
│   │   ├── ManualQuizService.java         # 관리자 직접 입력/Excel 업로드 → PENDING
│   │   ├── QuizApprovalService.java       # 승인/반려 워크플로우 (PENDING→PUBLISHED/REJECTED)
│   │   ├── QuizAttemptService.java        # 풀이 기록, 채점
│   │   ├── WrongNoteService.java          # 오답노트
│   │   └── GradingService.java            # 객관식/주관식/서술형 채점
│   ├── domain/
│   │   ├── Quiz.java                      # 문제 (source_type, status, reviewed_by, reviewed_at)
│   │   ├── QuizOption.java               # 보기
│   │   └── QuizAttempt.java               # 풀이 기록
│   ├── dto/
│   │   ├── QuizResponseDto.java
│   │   ├── QuizSubmitDto.java
│   │   ├── QuizAttemptResultDto.java
│   │   ├── WrongNoteDto.java
│   │   ├── AiGenerateDto.java             # AI 생성 요청 (subject_id, unit_id, difficulty, type, count)
│   │   ├── CrawlDto.java                  # 크롤링 요청 (source_url, subject_id, unit_id)
│   │   ├── ManualQuizDto.java             # 직접 입력 요청
│   │   └── ApproveDto.java                # 승인/반려 요청 (reason 옵션)
│   └── mapper/
│       ├── QuizMapper.java
│       └── QuizAttemptMapper.java
│
├── english/                               # 실생활 영어 도메인
│   ├── controller/
│   │   └── EnglishController.java         # /api/english/sentences, /api/english/tts
│   ├── service/
│   │   ├── EnglishSentenceService.java
│   │   └── TtsService.java                # TTS 음성 합성
│   ├── domain/
│   │   └── EnglishSentence.java
│   ├── dto/
│   │   ├── EnglishSentenceDto.java
│   │   └── TtsRequestDto.java
│   └── mapper/
│       └── EnglishSentenceMapper.java
│
├── ai/                                    # AI 콘텐츠 생성 파이프라인
│   ├── controller/
│   │   └── AiController.java              # /api/ai/summary, /api/ai/quiz, /api/ai/english
│   ├── service/
│   │   ├── AiContentService.java          # AI 생성 orchestration (공통)
│   │   ├── SummaryGenerationService.java  # 요약 생성
│   │   ├── QuizGenerationService.java     # 문제 출제 (객관식/주관식)
│   │   ├── EnglishGenerationService.java  # 영어 문장 생성
│   │   └── PromptTemplateManager.java     # 프롬프트 템플릿 관리
│   ├── client/
│   │   ├── GlmClient.java                 # GLM-5.2 cloud API 클라이언트
│   │   ├── OpenAiClient.java              # GPT-4o-mini API 클라이언트
│   │   └── AiClient.java                  # 공통 인터페이스
│   ├── domain/
│   │   ├── AiRequest.java                 # AI 생성 요청
│   │   ├── AiResponse.java                # AI 생성 응답
│   │   ├── SummaryPrompt.java             # 요약 프롬프트 파라미터
│   │   ├── QuizPrompt.java                # 문제 프롬프트 파라미터
│   │   └── EnglishPrompt.java             # 영어 프롬프트 파라미터
│   ├── dto/
│   │   ├── SummaryGenerateDto.java
│   │   ├── QuizGenerateDto.java
│   │   └── EnglishGenerateDto.java
│   └── config/
│       └── AiApiConfig.java               # API 키, 타임아웃, 모델 설정
│
├── study/                                 # 학습 기록/통계 도메인
│   ├── controller/
│   │   ├── DashboardController.java       # /api/dashboard
│   │   └── StudyLogController.java        # /api/study-logs
│   ├── service/
│   │   ├── DashboardService.java          # 대시보드 데이터 집계
│   │   ├── StudyLogService.java           # 학습 기록
│   │   └── StatsService.java              # 학습 통계 분석
│   ├── domain/
│   │   └── StudyLog.java
│   ├── dto/
│   │   ├── DashboardDto.java
│   │   ├── StudyLogDto.java
│   │   └── StatsDto.java
│   └── mapper/
│       └── StudyLogMapper.java
│
├── gamification/                          # 게이미피케이션 도메인 (Phase 3)
│   ├── controller/
│   │   └── BadgeController.java           # /api/badges, /api/users/me/badges
│   ├── service/
│   │   ├── BadgeService.java              # 배지 획득 로직
│   │   ├── PointService.java             # 포인트/레벨
│   │   └── StreakService.java            # 연속 학습일
│   ├── domain/
│   │   ├── Badge.java
│   │   └── UserBadge.java
│   ├── dto/
│   │   └── BadgeResponseDto.java
│   └── mapper/
│       ├── BadgeMapper.java
│       └── UserBadgeMapper.java
│
├── crawl/                                 # 크롤링 파이프라인
│   ├── batch/
│   │   ├── EnglishSentenceBatchJob.java   # 영어 문장 크롤링 Batch Job
│   │   ├── EnglishSentenceReader.java     # Batch ItemReader
│   │   ├── EnglishSentenceProcessor.java  # Batch ItemProcessor
│   │   └── EnglishSentenceWriter.java     # Batch ItemWriter
│   ├── scheduler/
│   │   └── CrawlScheduler.java            # @Scheduled 크롤링 스케줄
│   ├── service/
│   │   └── CrawlService.java              # 크롤링 공통 서비스
│   └── parser/
│       ├── EnglishSentenceParser.java     # 영어 문장 파서
│       └── NewsParser.java                # 뉴스 기사 파서
│
└── admin/                                 # 관리자 도메인 (Phase 3)
    ├── controller/
    │   └── AdminController.java           # /api/admin/dashboard, /api/admin/users
    ├── service/
    │   ├── AdminUserService.java
    │   ├── AdminContentService.java       # 요약/문제 승인
    │   └── AdminStatsService.java         # 플랫폼 전체 통계
    ├── dto/
    │   ├── AdminDashboardDto.java
    │   └── AdminUserDto.java
    └── mapper/
        └── AdminMapper.java
```

### 2.2 MyBatis XML Mapper 위치

```
src/main/resources/
├── mapper/
│   ├── auth/AuthMapper.xml
│   ├── user/UserMapper.xml
│   ├── user/UserSettingsMapper.xml
│   ├── subject/SubjectMapper.xml
│   ├── subject/UnitMapper.xml
│   ├── summary/SummaryMapper.xml
│   ├── quiz/QuizMapper.xml
│   ├── quiz/QuizAttemptMapper.xml
│   ├── english/EnglishSentenceMapper.xml
│   ├── study/StudyLogMapper.xml
│   ├── gamification/BadgeMapper.xml
│   ├── gamification/UserBadgeMapper.xml
│   └── admin/AdminMapper.xml
├── application.yml
├── application-prod.yml
└── application-dev.yml
```

> `application.yml` 설정 (holylife 패턴 계승):
> ```yaml
> mybatis:
>   mapper-locations: classpath:mapper/**/*.xml
>   type-aliases-package: org.shaloman.pj.dailystudy
>   configuration:
>     map-underscore-to-camel-case: true
>     cache-enabled: true
>     default-fetch-size: 100
>     default-statement-timeout: 30
> ```

---

## 3. API 엔드포인트 설계

### 3.1 API 설계 원칙

- RESTful 리소스 기반 URL 설계
- `/api` 접두어 + 복수형 명사
- HTTP 메서드语义: GET(조회), POST(생성), PUT(수정), DELETE(삭제)
- 응답 래퍼: `{ "success": boolean, "data": T, "message": string, "error": string }`
- 페이지네이션: `?page=1&size=20` → `PageResponse<T>`

### 3.2 전체 API 목록

#### 3.2.1 인증 (Auth)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| POST | `/api/auth/signup` | 이메일 회원가입 | Public | 1 |
| POST | `/api/auth/login` | 이메일 로그인 (JWT 발급) | Public | 1 |
| GET | `/api/auth/me` | 현재 사용자 정보 | Authenticated | 1 |
| POST | `/api/auth/refresh` | 토큰 갱신 | Authenticated | 1 |
| POST | `/api/auth/logout` | 로그아웃 (Redis 토큰 블랙리스트) | Authenticated | 1 |
| GET | `/api/auth/oauth2/google` | Google OAuth2 로그인 시작 | Public | 2 |
| GET | `/api/auth/oauth2/kakao` | Kakao OAuth2 로그인 시작 | Public | 2 |
| GET | `/api/auth/oauth2/callback` | OAuth2 콜백 처리 | Public | 2 |

#### 3.2.2 사용자 (User)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/users/me` | 내 프로필 조회 | Authenticated | 1 |
| PUT | `/api/users/me` | 내 프로필 수정 | Authenticated | 1 |
| PUT | `/api/users/me/password` | 비밀번호 변경 | Authenticated | 1 |
| GET | `/api/users/me/settings` | 학습 설정 조회 | Authenticated | 1 |
| PUT | `/api/users/me/settings` | 학습 설정 수정 | Authenticated | 1 |
| GET | `/api/users/me/stats` | 내 학습 통계 | Authenticated | 1 |
| DELETE | `/api/users/me` | 회원 탈퇴 | Authenticated | 2 |

#### 3.2.3 과목/단원 (Subject/Unit)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/subjects` | 과목 목록 | Authenticated | 1 |
| GET | `/api/subjects/{subjectId}` | 과목 상세 | Authenticated | 1 |
| GET | `/api/subjects/{subjectId}/units` | 단원 목록 | Authenticated | 1 |
| GET | `/api/subjects/{subjectId}/units/{unitId}` | 단원 상세 | Authenticated | 1 |
| POST | `/api/subjects` | 과목 생성 | Admin | 3 |
| POST | `/api/subjects/{subjectId}/units` | 단원 생성 | Admin | 3 |
| PUT | `/api/subjects/{subjectId}` | 과목 수정 | Admin | 3 |
| PUT | `/api/subjects/{subjectId}/units/{unitId}` | 단원 수정 | Admin | 3 |

#### 3.2.4 단원 요약 (Summary)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/summaries` | 요약 목록 (필터: subjectId, unitId) | Authenticated | 1 |
| GET | `/api/summaries/{id}` | 요약 상세 | Authenticated | 1 |
| POST | `/api/summaries/generate` | AI 요약 생성 요청 | Authenticated | 1 |
| GET | `/api/summaries/by-unit/{unitId}` | 단원별 요약 | Authenticated | 1 |
| PUT | `/api/summaries/{id}` | 요약 수정 (관리자) | Admin | 3 |
| DELETE | `/api/summaries/{id}` | 요약 삭제 (관리자) | Admin | 3 |

#### 3.2.5 문제/풀이 (Quiz)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/quizzes` | 문제 목록 — **승인된 문제만** (status=PUBLISHED, 필터: subjectId, unitId, type, difficulty) | Authenticated | 1 |
| GET | `/api/quizzes/{id}` | 문제 상세 (승인된 문제만) | Authenticated | 1 |
| GET | `/api/quizzes/by-unit/{unitId}` | 단원별 승인된 문제 목록 | Authenticated | 1 |
| GET | `/api/quizzes/random` | 랜덤 문제 (승인된 풀에서, 필터: subjectId, count) | Authenticated | 1 |
| POST | `/api/admin/quizzes/generate-ai` | **관리자 AI 문제 일괄 생성** (subject_id, unit_id, difficulty, type, count) | Admin | 1 |
| POST | `/api/admin/quizzes/crawl` | **관리자 크롤링 수집** (source_url, subject_id, unit_id) | Admin | 1 |
| POST | `/api/admin/quizzes/manual` | **관리자 직접 입력** (type, difficulty, question, options, answer, explanation) | Admin | 1 |
| POST | `/api/admin/quizzes/batch` | **관리자 Excel/CSV 일괄 업로드** | Admin | 1 |
| GET | `/api/admin/quizzes` | 관리자 문제 목록 (전체 상태, 필터: status, source_type) | Admin | 1 |
| GET | `/api/admin/quizzes/pending` | 검수 대기 문제 목록 (status=PENDING) | Admin | 1 |
| POST | `/api/admin/quizzes/{id}/approve` | **문제 승인** (PENDING → PUBLISHED) | Admin | 1 |
| POST | `/api/admin/quizzes/{id}/reject` | **문제 반려** (PENDING → REJECTED, reason 옵션) | Admin | 1 |
| POST | `/api/admin/quizzes/batch-approve` | 일괄 승인 (quizIds 배열) | Admin | 1 |
| PUT | `/api/admin/quizzes/{id}` | 문제 수정 (관리자) | Admin | 1 |
| DELETE | `/api/admin/quizzes/{id}` | 문제 삭제 (관리자) | Admin | 1 |
| POST | `/api/quiz-attempts` | 문제 풀이 제출 (채점) | Authenticated | 1 |
| GET | `/api/quiz-attempts` | 내 풀이 기록 (필터: subjectId, dateRange) | Authenticated | 1 |
| GET | `/api/quiz-attempts/wrong` | 오답노트 목록 | Authenticated | 2 |
| GET | `/api/quiz-attempts/wrong/by-unit/{unitId}` | 단원별 오답노트 | Authenticated | 2 |
| DELETE | `/api/quiz-attempts/wrong/{attemptId}` | 오답노트에서 제거 | Authenticated | 2 |
| POST | `/api/quiz-attempts/wrong/retry` | 오답 재풀이 | Authenticated | 2 |

#### 3.2.6 실생활 영어 (English)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/english/sentences` | 영어 문장 목록 (필터: category, difficulty) | Authenticated | 2 |
| GET | `/api/english/sentences/{id}` | 영어 문장 상세 | Authenticated | 2 |
| GET | `/api/english/sentences/today` | 오늘의 영어 문장 | Authenticated | 2 |
| GET | `/api/english/sentences/random` | 랜덤 영어 문장 | Authenticated | 2 |
| POST | `/api/english/tts` | TTS 음성 합성 요청 | Authenticated | 2 |
| GET | `/api/english/categories` | 영어 문장 카테고리 목록 | Authenticated | 2 |

#### 3.2.7 학습 기록/대시보드 (Study/Dashboard)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/dashboard` | 대시보드 종합 데이터 | Authenticated | 1 |
| GET | `/api/dashboard/weekly` | 주간 학습 통계 | Authenticated | 1 |
| GET | `/api/dashboard/monthly` | 월간 학습 통계 | Authenticated | 1 |
| GET | `/api/study-logs` | 학습 기록 목록 (필터: dateRange, type) | Authenticated | 1 |
| POST | `/api/study-logs` | 학습 기록 수동 추가 | Authenticated | 1 |
| GET | `/api/study-logs/calendar` | 캘린더용 일일 학습 여부 | Authenticated | 2 |

#### 3.2.8 게이미피케이션 (Gamification)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/badges` | 배지 목록 | Authenticated | 3 |
| GET | `/api/users/me/badges` | 내 획득 배지 | Authenticated | 3 |
| GET | `/api/users/me/points` | 내 포인트/레벨 | Authenticated | 3 |
| GET | `/api/users/me/streak` | 연속 학습일 | Authenticated | 3 |
| GET | `/api/leaderboard` | 랭킹 (주간/월간) | Authenticated | 3 |

#### 3.2.9 관리자 (Admin)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| GET | `/api/admin/dashboard` | 관리자 대시보드 (전체 통계) | Admin | 3 |
| GET | `/api/admin/users` | 전체 사용자 목록 | Admin | 3 |
| GET | `/api/admin/users/{id}` | 사용자 상세 | Admin | 3 |
| PUT | `/api/admin/users/{id}/status` | 사용자 상태 변경 | Admin | 3 |
| GET | `/api/admin/contents` | 전체 콘텐츠 목록 | Admin | 3 |
| PUT | `/api/admin/contents/{type}/{id}/approve` | 콘텐츠 승인 | Admin | 3 |
| GET | `/api/admin/stats/content` | 콘텐츠 생성 통계 | Admin | 3 |
| GET | `/api/admin/stats/ai` | AI API 사용량 통계 | Admin | 3 |

#### 3.2.10 파일 업로드 (File)

| Method | Endpoint | 설명 | 권한 | Phase |
|--------|----------|------|------|-------|
| POST | `/api/files` | 파일 업로드 (단일/다수) | Authenticated | 1 |
| DELETE | `/api/files` | 파일 삭제 | Authenticated | 1 |

### 3.3 API 응답 형식

```json
// 성공
{
  "success": true,
  "data": { ... },
  "message": "조회 성공"
}

// 실패
{
  "success": false,
  "data": null,
  "error": "INVALID_CREDENTIALS",
  "message": "이메일 또는 비밀번호가 올바르지 않습니다."
}

// 페이지네이션
{
  "success": true,
  "data": {
    "items": [ ... ],
    "totalCount": 150,
    "page": 1,
    "pageSize": 20,
    "totalPages": 8
  }
}
```

---

## 4. 데이터베이스 ERD

### 4.1 ERD 다이어그램 (텍스트)

```
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│      USER        │       │     SUBJECT      │       │      UNIT        │
├──────────────────┤       ├──────────────────┤       ├──────────────────┤
│ PK user_id       │       │ PK subject_id    │       │ PK unit_id       │
│    email         │       │    subject_name  │       │ FK subject_id ───┼─→ SUBJECT
│    password      │       │    grade_level   │       │    unit_number   │
│    name          │       │    icon          │       │    unit_title    │
│    role          │       │    color         │       │    description   │
│    grade         │       │    sort_order    │       │    sort_order    │
│    school        │       │    created_at    │       │    created_at    │
│    status        │       │    updated_at    │       │    updated_at    │
│    provider      │       └──────────────────┘       └──────────────────┘
│    provider_id   │            │                          │
│    created_at    │            │ 1                        │ 1
│    updated_at    │            │                          │
│    last_login_at │            ▼                          ▼
└──────────────────┘     ┌──────────────────┐       ┌──────────────────┐
         │ 1             │  USER_SETTINGS   │       │    SUMMARY       │
         │               ├──────────────────┤       ├──────────────────┤
         ├──────────────→│ PK setting_id    │       │ PK summary_id    │
         │               │ FK user_id ──────┼─→ USER│ FK unit_id ──────┼─→ UNIT
         │               │    preferred_     │       │    title         │
         │               │      subjects    │       │    content       │
         │               │    difficulty    │       │    ai_model      │
         │               │    notification  │       │    ai_generated  │
         │               │    daily_goal    │       │    status        │
         │               │    created_at    │       │    created_at    │
         │               └──────────────────┘       │    updated_at    │
         │                                          └──────────────────┘
         │ 1                                               │ 1
         │                                                 │
         ▼                                                 ▼
┌──────────────────┐     ┌──────────────────┐       ┌──────────────────┐
│   STUDY_LOG      │     │  QUIZ_ATTEMPT    │       │     QUIZ         │
├──────────────────┤     ├──────────────────┤       ├──────────────────┤
│ PK log_id        │     │ PK attempt_id    │       │ PK quiz_id       │
│ FK user_id ──────┼─→ USER│ FK user_id ─────┼─→ USER│ FK unit_id ──────┼─→ UNIT
│    log_type      │     │ FK quiz_id ──────┼─→ QUIZ│    quiz_type     │
│    subject_id    │     │    user_answer   │       │    question      │
│    ref_id        │     │    is_correct    │       │    difficulty    │
│    study_date    │     │    time_spent    │       │    answer        │
│    duration      │     │    attempted_at  │       │    explanation   │
│    created_at    │     │    is_in_wrong_  │       │    ai_model      │
└──────────────────┘     │      note        │       │    ai_generated  │
                         │    created_at    │       │    status        │
                         └──────────────────┘       │    created_at    │
                                                    └──────────────────┘
                                                          │ 1
                                                          │
                                                          ▼
┌──────────────────┐     ┌──────────────────┐       ┌──────────────────┐
│  ENGLISH_        │     │     BADGE        │       │  QUIZ_OPTION     │
│  SENTENCE        │     ├──────────────────┤       ├──────────────────┤
├──────────────────┤     │ PK badge_id      │       │ PK option_id     │
│ PK sentence_id   │     │    badge_name    │       │ FK quiz_id ──────┼─→ QUIZ
│    category      │     │    description   │       │    option_number │
│    english_      │     │    icon          │       │    option_text   │
│      text        │     │    condition_    │       │    is_answer     │
│    korean_text   │     │      type        │       │    sort_order    │
│    difficulty    │     │    condition_    │       └──────────────────┘
│    situation     │     │      value       │
│    source_url    │     │    created_at    │       ┌──────────────────┐
│    created_at    │     └──────────────────┘       │  USER_BADGE      │
└──────────────────┘          │ 1                    ├──────────────────┤
                              │                      │ PK user_badge_id │
                              ▼                      │ FK user_id ──────┼─→ USER
                    ┌──────────────────┐              │ FK badge_id ─────┼─→ BADGE
                    │   USER_BADGE     │              │    earned_at     │
                    ├──────────────────┤              └──────────────────┘
                    │ PK user_badge_id │
                    │ FK user_id ──────┼─→ USER
                    │ FK badge_id ─────┼─→ BADGE
                    │    earned_at     │
                    └──────────────────┘
```

### 4.2 테이블 정의서

#### 4.2.1 USER (사용자)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| user_id | BIGINT | PK, AUTO_INCREMENT | 사용자 ID |
| email | VARCHAR(255) | UNIQUE, NOT NULL | 이메일 (로그인 ID) |
| password | VARCHAR(255) | NULL | BCrypt 해시 (OAuth2 사용자는 NULL) |
| name | VARCHAR(100) | NOT NULL | 사용자명 |
| role | VARCHAR(20) | NOT NULL, DEFAULT 'STUDENT' | STUDENT / ADMIN |
| grade | INT | NULL | 학년 (1~6: 중1~고3) |
| school | VARCHAR(100) | NULL | 학교명 |
| status | VARCHAR(10) | NOT NULL, DEFAULT 'ACTIVE' | ACTIVE / INACTIVE / WITHDRAWN |
| provider | VARCHAR(20) | NULL | OAuth2 제공자 (GOOGLE / KAKAO / NULL) |
| provider_id | VARCHAR(255) | NULL | OAuth2 제공자 사용자 ID |
| total_points | INT | NOT NULL, DEFAULT 0 | 누적 포인트 |
| current_level | INT | NOT NULL, DEFAULT 1 | 현재 레벨 |
| study_streak | INT | NOT NULL, DEFAULT 0 | 연속 학습일 |
| last_study_date | DATE | NULL | 마지막 학습일 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |
| last_login_at | DATETIME | NULL | 최종 로그인 일시 |

#### 4.2.2 USER_SETTINGS (사용자 학습 설정)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| setting_id | BIGINT | PK, AUTO_INCREMENT | 설정 ID |
| user_id | BIGINT | FK → USER, UNIQUE | 사용자 ID (1:1) |
| preferred_subjects | VARCHAR(255) | NULL | 선호 과목 (CSV: "1,2,3") |
| difficulty | VARCHAR(10) | DEFAULT 'MEDIUM' | EASY / MEDIUM / HARD |
| notification_enabled | BOOLEAN | DEFAULT TRUE | 알림 활성화 |
| daily_goal_minutes | INT | DEFAULT 30 | 일일 학습 목표 (분) |
| tts_enabled | BOOLEAN | DEFAULT TRUE | TTS 자동 재생 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

#### 4.2.3 SUBJECT (과목)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| subject_id | BIGINT | PK, AUTO_INCREMENT | 과목 ID |
| subject_name | VARCHAR(50) | NOT NULL | 과목명 (수학, 영어, 과학, ...) |
| grade_level | VARCHAR(10) | NOT NULL | MIDDLE / HIGH / ALL |
| icon | VARCHAR(50) | NULL | 아이콘명 (Tailwind/SVG) |
| color | VARCHAR(7) | NULL | 테마 색상 (#hex) |
| sort_order | INT | DEFAULT 0 | 정렬 순서 |
| status | VARCHAR(10) | DEFAULT 'ACTIVE' | ACTIVE / INACTIVE |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

#### 4.2.4 UNIT (단원)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| unit_id | BIGINT | PK, AUTO_INCREMENT | 단원 ID |
| subject_id | BIGINT | FK → SUBJECT, NOT NULL | 과목 ID |
| unit_number | INT | NOT NULL | 단원 번호 |
| unit_title | VARCHAR(200) | NOT NULL | 단원 제목 |
| description | TEXT | NULL | 단원 설명 |
| sort_order | INT | DEFAULT 0 | 정렬 순서 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

#### 4.2.5 SUMMARY (단원 요약)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| summary_id | BIGINT | PK, AUTO_INCREMENT | 요약 ID |
| unit_id | BIGINT | FK → UNIT, NOT NULL | 단원 ID |
| title | VARCHAR(200) | NOT NULL | 요약 제목 |
| content | LONGTEXT | NOT NULL | 요약 내용 (Markdown/HTML) |
| ai_model | VARCHAR(50) | NULL | AI 모델명 (GLM-5.2 / GPT-4o-mini) |
| ai_generated | BOOLEAN | DEFAULT FALSE | AI 생성 여부 |
| status | VARCHAR(10) | DEFAULT 'PUBLISHED' | DRAFT / PUBLISHED / ARCHIVED |
| view_count | INT | DEFAULT 0 | 조회수 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

#### 4.2.6 QUIZ (문제)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| quiz_id | BIGINT | PK, AUTO_INCREMENT | 문제 ID |
| unit_id | BIGINT | FK → UNIT, NOT NULL | 단원 ID |
| quiz_type | VARCHAR(20) | NOT NULL | MULTIPLE_CHOICE / SHORT_ANSWER / ESSAY |
| question | LONGTEXT | NOT NULL | 문제 내용 |
| difficulty | VARCHAR(10) | DEFAULT 'MEDIUM' | EASY / MEDIUM / HARD |
| answer | VARCHAR(500) | NOT NULL | 정답 |
| explanation | TEXT | NULL | 해설 |
| source_type | VARCHAR(10) | NOT NULL, DEFAULT 'AI' | AI / CRAWL / MANUAL — 문제 생성 방식 |
| ai_model | VARCHAR(50) | NULL | AI 모델명 (AI/CRAWL 정제 시) |
| ai_generated | BOOLEAN | DEFAULT FALSE | AI 생성 여부 (AI/CRAWL true, MANUAL false) |
| status | VARCHAR(10) | NOT NULL, DEFAULT 'PENDING' | PENDING / PUBLISHED / REJECTED — 승인 워크플로우 |
| created_by | BIGINT | FK → USER, NULL | 생성자 ID (관리자) |
| reviewed_by | BIGINT | FK → USER, NULL | 검수자 ID (관리자) |
| reviewed_at | DATETIME | NULL | 검수 완료 일시 |
| reject_reason | TEXT | NULL | 반려 사유 (REJECTED 시) |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

#### 4.2.7 QUIZ_OPTION (문제 보기)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| option_id | BIGINT | PK, AUTO_INCREMENT | 보기 ID |
| quiz_id | BIGINT | FK → QUIZ, NOT NULL | 문제 ID |
| option_number | INT | NOT NULL | 보기 번호 (1~5) |
| option_text | VARCHAR(500) | NOT NULL | 보기 내용 |
| is_answer | BOOLEAN | NOT NULL, DEFAULT FALSE | 정답 여부 |
| sort_order | INT | DEFAULT 0 | 정렬 순서 |

#### 4.2.8 QUIZ_ATTEMPT (문제 풀이 기록)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| attempt_id | BIGINT | PK, AUTO_INCREMENT | 풀이 ID |
| user_id | BIGINT | FK → USER, NOT NULL | 사용자 ID |
| quiz_id | BIGINT | FK → QUIZ, NOT NULL | 문제 ID |
| user_answer | VARCHAR(500) | NOT NULL | 사용자 답안 |
| is_correct | BOOLEAN | NOT NULL | 정답 여부 |
| time_spent | INT | NULL | 풀이 시간 (초) |
| is_in_wrong_note | BOOLEAN | DEFAULT FALSE | 오답노트 포함 여부 |
| attempted_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 풀이 일시 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |

#### 4.2.9 ENGLISH_SENTENCE (실생활 영어 문장)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| sentence_id | BIGINT | PK, AUTO_INCREMENT | 문장 ID |
| category | VARCHAR(50) | NOT NULL | 카테고리 (RESTAURANT, SHOPPING, TRAVEL, SCHOOL, ...) |
| english_text | TEXT | NOT NULL | 영어 문장 |
| korean_text | TEXT | NOT NULL | 한국어 번역 |
| difficulty | VARCHAR(10) | DEFAULT 'MEDIUM' | EASY / MEDIUM / HARD |
| situation | VARCHAR(200) | NULL | 상황 설명 |
| source_url | VARCHAR(500) | NULL | 크롤링 소스 URL |
| source_type | VARCHAR(20) | NULL | CRAWL / AI / MANUAL |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |

#### 4.2.10 STUDY_LOG (학습 기록)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| log_id | BIGINT | PK, AUTO_INCREMENT | 로그 ID |
| user_id | BIGINT | FK → USER, NOT NULL | 사용자 ID |
| log_type | VARCHAR(20) | NOT NULL | SUMMARY_READ / QUIZ_ATTEMPT / ENGLISH_STUDY |
| subject_id | BIGINT | NULL | 과목 ID (영어 학습 시 NULL 가능) |
| ref_id | BIGINT | NULL | 참조 ID (summary_id / quiz_id / sentence_id) |
| study_date | DATE | NOT NULL | 학습일 |
| duration | INT | NULL | 학습 시간 (초) |
| points_earned | INT | DEFAULT 0 | 획득 포인트 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |

#### 4.2.11 BADGE (배지)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| badge_id | BIGINT | PK, AUTO_INCREMENT | 배지 ID |
| badge_name | VARCHAR(100) | NOT NULL | 배지명 |
| description | VARCHAR(500) | NULL | 배지 설명 |
| icon | VARCHAR(50) | NOT NULL | 아이콘명 |
| color | VARCHAR(7) | NULL | 색상 |
| condition_type | VARCHAR(50) | NOT NULL | 획득 조건 타입 (QUIZ_COUNT / STREAK_DAYS / SUBJECT_COMPLETE / POINTS) |
| condition_value | INT | NOT NULL | 획득 조건 값 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일시 |

#### 4.2.12 USER_BADGE (사용자 배지)

| 컬럼명 | 타입 | 제약 | 설명 |
|--------|------|------|------|
| user_badge_id | BIGINT | PK, AUTO_INCREMENT | ID |
| user_id | BIGINT | FK → USER, NOT NULL | 사용자 ID |
| badge_id | BIGINT | FK → BADGE, NOT NULL | 배지 ID |
| earned_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 획득 일시 |

### 4.3 인덱스 설계

```sql
-- USER
CREATE UNIQUE INDEX idx_user_email ON USER(email);
CREATE INDEX idx_user_status ON USER(status);
CREATE INDEX idx_user_provider ON USER(provider, provider_id);

-- UNIT
CREATE INDEX idx_unit_subject ON UNIT(subject_id, sort_order);

-- SUMMARY
CREATE INDEX idx_summary_unit ON SUMMARY(unit_id);
CREATE INDEX idx_summary_status ON SUMMARY(status);

-- QUIZ
CREATE INDEX idx_quiz_unit ON QUIZ(unit_id);
CREATE INDEX idx_quiz_difficulty ON QUIZ(difficulty);
CREATE INDEX idx_quiz_status ON QUIZ(status);

-- QUIZ_ATTEMPT
CREATE INDEX idx_attempt_user ON QUIZ_ATTEMPT(user_id, attempted_at);
CREATE INDEX idx_attempt_quiz ON QUIZ_ATTEMPT(quiz_id);
CREATE INDEX idx_attempt_wrong ON QUIZ_ATTEMPT(user_id, is_in_wrong_note);

-- ENGLISH_SENTENCE
CREATE INDEX idx_english_category ON ENGLISH_SENTENCE(category, difficulty);

-- STUDY_LOG
CREATE INDEX idx_studylog_user_date ON STUDY_LOG(user_id, study_date);
CREATE INDEX idx_studylog_type ON STUDY_LOG(log_type);
```

### 4.4 관계 요약

| 관계 | 타입 | 설명 |
|------|------|------|
| USER → USER_SETTINGS | 1:1 | 사용자별 학습 설정 |
| SUBJECT → UNIT | 1:N | 과목 내 단원 |
| UNIT → SUMMARY | 1:N | 단원별 요약 |
| UNIT → QUIZ | 1:N | 단원별 문제 |
| QUIZ → QUIZ_OPTION | 1:N | 문제별 보기 (객관식) |
| USER → QUIZ_ATTEMPT ← QUIZ | M:N | 사용자의 문제 풀이 기록 |
| USER → STUDY_LOG | 1:N | 사용자의 학습 기록 |
| USER → USER_BADGE ← BADGE | M:N | 사용자의 배지 획득 |

---

## 5. AI 콘텐츠 생성 파이프라인

> **문제 출제 정책 변경:** 문제 출제는 관리자 주도로 운영됩니다. 학생은 실시간 AI 생성을 호출하지 않으며, 관리자가 AI 생성/크롤링/직접 입력으로 미리 생성하고 승인한 문제 풀에서 제공받습니다. 아래 파이프라인은 요약 생성(학생 직접 호출 가능)과 관리자용 문제 일괄 생성에 공통으로 적용됩니다.

### 5.1 파이프라인 아키텍처

```
┌──────────────────────────────────────────────────────────────────┐
│                        클라이언트 요청                              │
│   POST /api/summaries/generate  { unitId, difficulty }           │
│   POST /api/admin/quizzes/generate-ai { unitId, difficulty, ... }│  ← 관리자용
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Controller → AiController                                       │
│  • 요청 파라미터 검증 (unitId 존재 여부, difficulty 범위)           │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Service → AiContentService                                      │
│  • 캐시 키 생성: "ai:summary:{unitId}:{difficulty}"              │
│  • Redis 캐시 조회 → HIT 시 즉시 반환                              │
│  • MISS 시 AI 생성 프로세스 진행                                   │
└────────────┬─────────────────────────────┬──────────────────────┘
             │ Cache HIT                   │ Cache MISS
             ▼                             ▼
┌──────────────────────────┐  ┌──────────────────────────────────────┐
│  Redis에서 캐시된         │  │  PromptTemplateManager               │
│  응답 반환                │  │  • 단원 정보 조회 (UNIT + SUBJECT)   │
│                          │  │  • 프롬프트 템플릿 선택               │
└──────────────────────────┘  │  • 변수 주입 (단원명, 과목, 난이도)    │
                              └──────────────────┬───────────────────┘
                                                 ▼
                              ┌──────────────────────────────────────┐
                              │  AI Client 라우팅                     │
                              │  • 한국어 콘텐츠 → GLM-5.2 cloud     │
                              │  • 영어 콘텐츠   → GPT-4o-mini        │
                              └──────────────────┬───────────────────┘
                                                 ▼
                              ┌──────────────────────────────────────┐
                              │  AI API 호출                          │
                              │  • HTTP POST (timeout: 30s)          │
                              │  • Retry: 3회 (exponential backoff)  │
                              │  • 응답 파싱 (JSON)                   │
                              └──────────────────┬───────────────────┘
                                                 ▼
                              ┌──────────────────────────────────────┐
                              │  응답 후처리                          │
                              │  • JSON 스키마 검증                   │
                              │  • HTML sanitize (XSS 방지)          │
                              │  • DB 저장 (SUMMARY / QUIZ)          │
                              │  • Redis 캐시 저장 (TTL: 24h)        │
                              └──────────────────┬───────────────────┘
                                                 ▼
                              ┌──────────────────────────────────────┐
                              │  클라이언트 응답                      │
                              │  • ApiResponse 래핑                  │
                              │  • STUDY_LOG 기록                    │
                              └──────────────────────────────────────┘
```

### 5.2 AI 클라이언트 라우팅 전략

```java
public interface AiClient {
    String generate(String prompt, AiModel model);
    AiResponse generateStructured(String prompt, Class<?> responseType);
}

@Component
public class GlmClient implements AiClient {
    // GLM-5.2 cloud — 한국어 요약, 한국어 문제 출제, 한국어 해설
    // Endpoint: https://api.zhipu.ai/api/paas/v4/chat/completions
    // Model: glm-5.2
}

@Component
public class OpenAiClient implements AiClient {
    // GPT-4o-mini — 영어 문장 생성, 보조 생성, 품질 검증
    // Endpoint: https://api.openai.com/v1/chat/completions
    // Model: gpt-4o-mini
}
```

| 콘텐츠 타입 | 1순위 (Primary) | 2순위 (Fallback) | 비고 |
|-------------|-----------------|-------------------|------|
| 단원 요약 (한국어) | GLM-5.2 | GPT-4o-mini | 한국어 품질 우수 |
| 객관식 문제 (한국어) | GLM-5.2 | GPT-4o-mini | 관리자 일괄 생성 — 학생 실시간 호출 아님 |
| 주관식 문제 (한국어) | GLM-5.2 | GPT-4o-mini | 관리자 일괄 생성 |
| 실생활 영어 문장 | GPT-4o-mini | GLM-5.2 | 영어 자연스러움 |
| 영어 해설 | GPT-4o-mini | GLM-5.2 | - |
| 품질 검증 (교차 검증) | 상대 모델 | - | 1순위 모델 응답을 2순위 모델로 검증 |

### 5.3 프롬프트 템플릿

#### 5.3.1 단원 요약 프롬프트

```text
[시스템 프롬프트]
당신은 중·고등학생을 위한 학습 요약 전문가입니다.
학생이 이해하기 쉽고, 핵심 개념을 놓치지 않는 요약을 작성합니다.

[사용자 프롬프트]
다음 단원에 대한 학습 요약을 작성해주세요.

- 과목: {subjectName}
- 단원: {unitNumber}단원. {unitTitle}
- 단원 설명: {unitDescription}
- 대상 학생: {gradeLevel} (중학생/고등학생)
- 요약 난이도: {difficulty} (EASY: 핵심만 / MEDIUM: 균형 / HARD: 심화 포함)

[요구사항]
1. Markdown 형식으로 작성할 것
2. 핵심 개념을 3~5개로 구분하여 작성할 것
3. 각 개념마다 정의, 예시, 주의점을 포함할 것
4. 학생이 3분 이내에 읽을 수 있는 분량으로 작성할 것
5. 실생활 예시를 최소 1개 포함할 것
6. 마지막에 "핵심 요약" 섹션으로 3줄 요약을 추가할 것

[출력 형식]
JSON: { "title": "요약 제목", "content": "Markdown 내용" }
```

#### 5.3.2 객관식 문제 출제 프롬프트

```text
[시스템 프롬프트]
당신은 중·고등학생을 위한 문제 출제 전문가입니다.
교과서 수준에 맞는 정확하고 공정한 문제를 출제합니다.

[사용자 프롬프트]
다음 단원에 대한 객관식 문제를 {count}개 출제해주세요.

- 과목: {subjectName}
- 단원: {unitNumber}단원. {unitTitle}
- 단원 설명: {unitDescription}
- 대상 학생: {gradeLevel}
- 문제 난이도: {difficulty}
  (EASY: 개념 확인 / MEDIUM: 적용 / HARD: 심화 응용)

[요구사항]
1. 4지선다 객관식 문제로 작성할 것
2. 각 문제마다 정답이 정확히 1개만 있을 것
3. 오답은 그럴듯하지만 명확히 틀린 보기로 작성할 것
4. 각 문제마다 해설을 포함할 것 (왜 정답이고 왜 오답인지)
5. 추측으로 풀 수 없도록 개념 이해를 요구하는 문제로 작성할 것
6. 중복되지 않는 다양한 개념을 다룰 것

[출력 형식]
JSON 배열:
[
  {
    "question": "문제 내용",
    "options": ["보기1", "보기2", "보기3", "보기4"],
    "answerIndex": 0,
    "explanation": "해설",
    "difficulty": "MEDIUM"
  }
]
```

#### 5.3.3 실생활 영어 문장 프롬프트

```text
[시스템 프롬프트]
You are an expert in creating practical English learning sentences
for Korean middle and high school students.
Sentences should be natural, commonly used in real-life situations,
and appropriate for the students' level.

[사용자 프롬프트]
Generate {count} practical English sentences for the given category.

- Category: {category} (RESTAURANT / SHOPPING / TRAVEL / SCHOOL / DAILY_LIFE)
- Difficulty: {difficulty}
  (EASY: basic expressions / MEDIUM: intermediate / HARD: advanced)
- Target: Korean {gradeLevel} students

[Requirements]
1. Each sentence should be natural American English
2. Include Korean translation
3. Describe the situation where the sentence is used
4. Vary sentence patterns — not all the same structure
5. Include at least one question, one statement, and one request per batch

[Output Format]
JSON array:
[
  {
    "englishText": "English sentence",
    "koreanText": "Korean translation",
    "situation": "Situation description",
    "difficulty": "MEDIUM"
  }
]
```

### 5.4 캐싱 전략

| 캐시 레이어 | 키 패턴 | TTL | 목적 |
|-------------|---------|-----|------|
| Redis L1 | `ai:summary:{unitId}:{difficulty}` | 24시간 | 동일 요약 재생성 방지 |
| Redis L1 | `ai:quiz:{unitId}:{difficulty}:{quizType}:{hash}` | 12시간 | 동일 문제 재생성 방지 |
| Redis L1 | `ai:english:{category}:{difficulty}:{date}` | 7일 | 일일 영어 문장 캐싱 |
| Redis L1 | `ai:response:{requestHash}` | 6시간 | 동일 요청 중복 호출 방지 |
| DB L2 | SUMMARY, QUIZ 테이블 | 영구 | AI 생성 콘텐츠 영속화 |

```
캐싱 흐름:
1. 요청 수신 → 요청 파라미터로 캐시 키 생성 (SHA-256 hash)
2. Redis 조회 → HIT: 즉시 반환 (평균 < 10ms)
3. MISS → DB 조회 (이미 생성된 콘텐츠인지) → HIT: 반환 + Redis 갱신
4. MISS → AI API 호출 → 응답 DB 저장 → Redis 캐싱 → 반환
```

### 5.5 에러 처리 및 Fallback

```java
@Service
public class AiContentService {

    public Summary generateSummary(SummaryGenerateDto dto) {
        try {
            // 1. GLM-5.2 시도
            return generateWithClient(glmClient, dto);
        } catch (AiGenerationException e) {
            log.warn("GLM-5.2 실패, GPT-4o-mini로 fallback: {}", e.getMessage());
            try {
                // 2. GPT-4o-mini Fallback
                return generateWithClient(openAiClient, dto);
            } catch (AiGenerationException e2) {
                // 3. 최종 실패 — 사용자에게 에러 반환
                throw new BusinessException(ErrorCode.AI_GENERATION_FAILED,
                    "AI 콘텐츠 생성에 실패했습니다. 잠시 후 다시 시도해주세요.");
            }
        }
    }

    // Retry: 3회, exponential backoff (1s → 2s → 4s)
    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000, multiplier = 2))
    private Summary generateWithClient(AiClient client, SummaryGenerateDto dto) {
        // ...
    }
}
```

### 5.6 Rate Limiting (AI API)

| 제한 항목 | 값 | 비고 |
|----------|-----|------|
| 사용자당 AI 생성 요청 | 10회/시간 | Redis 카운터 |
| 사용자당 일일 AI 생성 한도 | 50회/일 | 일일 리셋 |
| 전체 AI API 호출 | 100회/분 | 글로벌 제한 |
| 단일 요청 타임아웃 | 30초 | HTTP timeout |
| 재시도 횟수 | 3회 | Exponential backoff |

---

## 6. 크롤링 파이프라인 설계

### 6.1 아키텍처

```
┌──────────────────────────────────────────────────────────────────┐
│                    Spring Batch 크롤링 파이프라인                  │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐           │
│  │ Scheduler │───→│  Batch Job   │───→│  DB 저장      │           │
│  │ (매일 6AM)│    │              │    │              │           │
│  └──────────┘    └──────┬───────┘    └──────────────┘           │
│                         │                                        │
│          ┌──────────────┼──────────────┐                        │
│          ▼              ▼              ▼                        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│  │  ItemReader  │ │ ItemProcessor│ │  ItemWriter   │             │
│  │  (웹페이지   │ │ (파싱 + 중복 │ │ (DB 저장 +    │             │
│  │   크롤링)    │ │  제거 + AI   │ │  Redis 갱신)  │             │
│  │              │ │  보정)       │ │              │             │
│  └──────────────┘ └──────────────┘ └──────────────┘             │
└──────────────────────────────────────────────────────────────────┘
```

### 6.2 크롤링 대상

| 대상 | URL/소스 | 주기 | 파싱 내용 | Phase |
|------|----------|------|-----------|-------|
| 실생활 영어 문장 | 뉴스 사이트, 영어 학습 사이트 | 매일 06:00 | 영어 문장 + 한국어 번역 | 2 |
| 오늘의 영단어 | dictionary API / 웹사이트 | 매일 06:00 | 단어 + 뜻 + 예문 | 2 |
| 과목 뉴스/시사 | 교육 뉴스 사이트 | 주간 | 시사 자료 (심화 학습용) | 4 |

### 6.3 Spring Batch Job 구성

```java
@Configuration
public class EnglishSentenceBatchConfig {

    // Job: englishSentenceCrawlJob
    // Step: crawl → process → write

    @Bean
    public Job englishSentenceCrawlJob(JobRepository jobRepository,
                                        Step crawlStep) {
        return new JobBuilder("englishSentenceCrawlJob", jobRepository)
            .start(crawlStep)
            .build();
    }

    @Bean
    public Step crawlStep(JobRepository jobRepository,
                          PlatformTransactionManager transactionManager,
                          ItemReader<RawEnglishSentence> reader,
                          ItemProcessor<RawEnglishSentence, EnglishSentence> processor,
                          ItemWriter<EnglishSentence> writer) {
        return new StepBuilder("crawlStep", jobRepository)
            .<RawEnglishSentence, EnglishSentence>chunk(10, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .retryLimit(3)
            .retry(IOException.class)
            .build();
    }
}
```

### 6.4 ItemReader — 크롤링

```java
public class EnglishSentenceReader implements ItemReader<RawEnglishSentence> {

    @Override
    public RawEnglishSentence read() throws Exception {
        // 1. JSoup로 웹페이지 크롤링
        Document doc = Jsoup.connect(targetUrl)
            .userAgent("Mozilla/5.0 (DailyStudy Bot)")
            .timeout(10000)
            .get();

        // 2. CSS Selector로 문장 추출
        Elements sentences = doc.select(".sentence-class");

        // 3. RawEnglishSentence 객체로 변환
        // 4. 다음 아이템 반환 (null이면 Job 종료)
    }
}
```

### 6.5 ItemProcessor — 파싱 + 중복 제거 + AI 보정

```java
public class EnglishSentenceProcessor
    implements ItemProcessor<RawEnglishSentence, EnglishSentence> {

    @Override
    public EnglishSentence process(RawEnglishSentence raw) {
        // 1. 중복 제거 — DB + Redis에서 기존 문장 확인
        if (isDuplicate(raw.getEnglishText())) {
            return null; // skip
        }

        // 2. 품질 검증 — 너무 짧거나 의미 없는 문장 필터링
        if (!isValidSentence(raw.getEnglishText())) {
            return null;
        }

        // 3. AI 보정 — GPT-4o-mini로 한국어 번역 품질 개선
        if (raw.getKoreanText() == null || raw.getKoreanText().isEmpty()) {
            String translation = openAiClient.translate(raw.getEnglishText());
            raw.setKoreanText(translation);
        }

        // 4. 난이도 자동 분류
        String difficulty = classifyDifficulty(raw.getEnglishText());

        // 5. EnglishSentence 도메인 객체 반환
        return EnglishSentence.builder()
            .category(raw.getCategory())
            .englishText(raw.getEnglishText())
            .koreanText(raw.getKoreanText())
            .difficulty(difficulty)
            .sourceUrl(raw.getSourceUrl())
            .sourceType("CRAWL")
            .build();
    }
}
```

### 6.6 스케줄러

```java
@Component
public class CrawlScheduler {

    private final JobLauncher jobLauncher;
    private final Job englishSentenceCrawlJob;

    // 매일 새벽 6시 — 실생활 영어 문장 크롤링
    @Scheduled(cron = "0 0 6 * * ?")
    public void runEnglishSentenceCrawl() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addDate("runDate", new Date())
            .toJobParameters();
        jobLauncher.run(englishSentenceCrawlJob, params);
    }

    // 매주 일요일 새벽 5시 — 주간 시사 자료 크롤링 (Phase 4)
    @Scheduled(cron = "0 0 5 * * SUN")
    public void runWeeklyNewsCrawl() throws Exception {
        // ...
    }
}
```

### 6.7 크롤링 주의사항

| 항목 | 조치 |
|------|------|
| robots.txt 준수 | 크롤링 전 robots.txt 확인 |
| User-Agent 명시 | "DailyStudy-Bot/1.0" 사용 |
| 요청 간격 | 3~5초 간격 (DoS 방지) |
| 중복 콘텐츠 | 영어 문장 해시(SHA-256)로 중복 판별 |
| 에러 처리 | Retry 3회, Skip Limit 10 |
| Spring Batch 메타테이블 | BATCH_JOB_INSTANCE, BATCH_JOB_EXECUTION 등 8개 테이블 사용 |

---

## 7. 보안 설계

### 7.1 인증 아키텍처

```
┌──────────────────────────────────────────────────────────────────┐
│                    인증 플로우                                    │
│                                                                  │
│  [이메일 로그인]                                                  │
│  Client → POST /api/auth/login {email, password}                 │
│       → BCrypt 검증 → JWT 발급 (Access + Refresh)                │
│       → 응답: { token, refreshToken, user }                      │
│                                                                  │
│  [OAuth2 로그인 — Phase 2]                                       │
│  Client → GET /api/auth/oauth2/google                            │
│       → Google 로그인 페이지 리다이렉트                            │
│       → 콜백 → 사용자 정보 조회 → DB 연동 → JWT 발급              │
│                                                                  │
│  [모든 API 요청]                                                  │
│  Client → Authorization: Bearer {token}                          │
│       → JwtAuthenticationFilter (토큰 검증)                      │
│       → RoleAuthorizationFilter (URL 패턴 권한 체크)             │
│       → Controller                                               │
└──────────────────────────────────────────────────────────────────┘
```

### 7.2 Spring Security 설정

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Value("${spring.profiles.active:default}")
    private String activeProfile;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> {
                // 공개 엔드포인트
                auth.requestMatchers("/api/auth/login").permitAll();
                auth.requestMatchers("/api/auth/signup").permitAll();
                auth.requestMatchers("/api/auth/oauth2/**").permitAll();
                // Swagger (dev만)
                if (!"prod".equals(activeProfile)) {
                    auth.requestMatchers("/swagger-ui/**").permitAll();
                    auth.requestMatchers("/v3/api-docs/**").permitAll();
                }
                // 정적 리소스
                auth.requestMatchers("/uploads/**").permitAll();
                // 관리자 전용
                auth.requestMatchers("/api/admin/**").hasRole("ADMIN");
                // 나머지 API — 인증 필요
                auth.requestMatchers("/api/**").authenticated();
                auth.anyRequest().permitAll();
            })
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterAfter(new RoleAuthorizationFilter(), JwtAuthenticationFilter.class)
            // OAuth2 (Phase 2)
            .oauth2Login(oauth -> oauth
                .successHandler(oAuth2SuccessHandler));

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of(
            "http://localhost",
            "http://localhost:3000",
            "http://localhost:80",
            "https://dailystudy.example.com"  // 프로덕션 도메인
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

### 7.3 JWT 토큰 설계

```java
public class JwtUtil {
    // Access Token: 24시간 (86400000ms)
    // Refresh Token: 7일 (604800000ms)
    // 알고리즘: HS256
    // 클레임: userId, email, role, iat, exp

    public String generateAccessToken(User user) {
        return Jwts.builder()
            .subject(String.valueOf(user.getUserId()))
            .claim("email", user.getEmail())
            .claim("role", user.getRole())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 86400000))
            .signWith(getSigningKey())
            .compact();
    }

    public String generateRefreshToken(User user) {
        return Jwts.builder()
            .subject(String.valueOf(user.getUserId()))
            .claim("type", "refresh")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 604800000))
            .signWith(getSigningKey())
            .compact();
    }
}
```

### 7.4 RoleAuthorizationFilter — URL 패턴 권한

```java
public class RoleAuthorizationFilter extends OncePerRequestFilter {

    // ADMIN 전용 URL 패턴
    private static final Map<String, String> ADMIN_ONLY_PATTERNS = Map.of(
        "/api/admin/",           "ADMIN",
        "/api/subjects (POST)",  "ADMIN",
        "/api/subjects (PUT)",   "ADMIN",
        "/api/summaries (PUT)",  "ADMIN",
        "/api/summaries (DELETE)","ADMIN",
        "/api/quizzes (PUT)",    "ADMIN",
        "/api/quizzes (DELETE)", "ADMIN"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) {
        String path = request.getRequestURI();
        String method = request.getMethod();

        // ADMIN 전용 엔드포인트 권한 체크
        if (isAdminOnlyPath(path, method)) {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null || !auth.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
                response.setStatus(403);
                response.getWriter().write("{\"success\":false,\"error\":\"FORBIDDEN\"}");
                return;
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

### 7.5 OAuth2 설정 (Phase 2)

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: profile, email
            redirect-uri: "{baseUrl}/api/auth/oauth2/callback/google"
          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            client-secret: ${KAKAO_CLIENT_SECRET}
            scope: profile_nickname, account_email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/api/auth/oauth2/callback/kakao"
            client-authentication-method: client_secret_post
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

### 7.6 보안 체크리스트

| # | 항목 | 조치 | Phase |
|---|------|------|-------|
| 1 | 비밀번호 암호화 | BCrypt (strength=10) | 1 |
| 2 | JWT 시크릿 키 | 환경변수 주입 (비하드코딩) | 1 |
| 3 | CORS 제한 | 허용 origin 화이트리스트 | 1 |
| 4 | SQL Injection | MyBatis #{} 사용 (=${} 금지) | 1 |
| 5 | XSS | HTML sanitize (AI 생성 콘텐츠) | 1 |
| 6 | Swagger 차단 | 프로덕션 환경 비활성화 | 1 |
| 7 | Rate Limiting | Redis 기반 API 호출 제한 | 1 |
| 8 | 비밀번호 복잡성 | 최소 8자, 영문+숫자+특수문자 | 1 |
| 9 | 계정 잠금 | 5회 실패 시 30분 잠금 (Redis) | 1 |
| 10 | 사용자 열거 방지 | 로그인 실패 메시지 통일 | 1 |
| 11 | 토큰 블랙리스트 | 로그아웃 시 Redis에 토큰 등록 | 1 |
| 12 | HTTPS 강제 | Nginx에서 HTTP→HTTPS 리다이렉트 | 1 |
| 13 | 파일 업로드 검증 | MIME 타입 + 확장자 + 크기 제한 | 1 |
| 14 | OAuth2 | Google, Kakao (PKCE) | 2 |
| 15 | Nginx 보안 헤더 | X-Frame-Options, X-Content-Type-Options 등 | 1 |

### 7.7 Nginx 보안 헤더

```nginx
server {
    # 보안 헤더
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # HTTPS 강제
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    }
}
```

---

## 8. Redis 활용 전략

### 8.1 Redis 아키텍처 개요

```
┌──────────────────────────────────────────────────────────────────┐
│                         Redis 7.x                                │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Cache       │  │ Session     │  │ Rate Limit  │             │
│  │ (AI 응답,   │  │ (JWT 블랙   │  │ (API 호출   │             │
│  │  인기 요약) │  │  리스트,    │  │  제한,      │             │
│  │             │  │  계정 잠금) │  │  AI 생성)   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐                              │
│  │ Real-time   │  │ Ranking     │                              │
│  │ Stats       │  │ (리더보드)  │                              │
│  └─────────────┘  └─────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 Redis 키 네이밍 규칙

| 용도 | 키 패턴 | 타입 | TTL | 설명 |
|------|---------|------|-----|------|
| **AI 캐시 - 요약** | `ai:summary:{unitId}:{difficulty}` | String (JSON) | 24h | AI 생성 요약 캐싱 |
| **AI 캐시 - 문제** | `ai:quiz:{unitId}:{difficulty}:{type}:{hash}` | String (JSON) | 12h | AI 생성 문제 캐싱 |
| **AI 캐시 - 영어** | `ai:english:{category}:{difficulty}:{date}` | String (JSON) | 7d | 일일 영어 문장 캐싱 |
| **AI 캐시 - 응답** | `ai:resp:{requestHash}` | String (JSON) | 6h | 동일 요청 중복 방지 |
| **JWT 블랙리스트** | `jwt:blacklist:{token}` | String | 24h | 로그아웃 토큰 무효화 |
| **Refresh 토큰** | `jwt:refresh:{userId}` | String | 7d | 리프레시 토큰 저장 |
| **계정 잠금** | `auth:lock:{email}` | String | 30m | 로그인 실패 5회 시 잠금 |
| **로그인 실패 카운트** | `auth:fail:{email}` | Integer | 30m | 로그인 실패 횟수 |
| **Rate Limit - 일반** | `rate:{userId}:{endpoint}` | Integer | 1h | API 호출 카운트 |
| **Rate Limit - AI** | `rate:ai:{userId}:hour` | Integer | 1h | AI 생성 시간당 제한 |
| **Rate Limit - AI 일일** | `rate:ai:{userId}:day:{date}` | Integer | 24h | AI 생성 일일 제한 |
| **인기 요약** | `popular:summaries` | ZSET | 1h | 조회수 기반 TOP 20 |
| **오늘의 영어** | `english:today:{date}` | String (JSON) | 24h | 오늘의 영어 문장 |
| **대시보드 캐시** | `dashboard:{userId}` | String (JSON) | 5m | 대시보드 데이터 캐싱 |
| **연속 학습일** | `streak:{userId}` | Integer | 영구 | 연속 학습 일수 |
| **리더보드 - 주간** | `leaderboard:weekly` | ZSET | 7d | 주간 포인트 랭킹 |
| **리더보드 - 월간** | `leaderboard:monthly` | ZSET | 30d | 월간 포인트 랭킹 |
| **온라인 사용자** | `online:users` | SET | - | 현재 온라인 사용자 (Phase 3) |

### 8.3 RedisConfig 설정

```java
@Configuration
public class RedisConfig {

    @Value("${spring.data.redis.host}")
    private String redisHost;

    @Value("${spring.data.redis.port}")
    private int redisPort;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(redisHost);
        config.setPort(redisPort);
        config.setDatabase(0);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public StringRedisTemplate stringRedisTemplate() {
        return new StringRedisTemplate(redisConnectionFactory());
    }
}
```

### 8.4 application.yml Redis 설정

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
          max-wait: 3000ms
```

### 8.5 캐싱 패턴 구현

```java
@Service
public class SummaryCacheService {

    private static final String KEY_PREFIX = "ai:summary:";
    private static final long TTL_HOURS = 24;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // 캐시 조회
    public Optional<Summary> get(Long unitId, String difficulty) {
        String key = KEY_PREFIX + unitId + ":" + difficulty;
        Object cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return Optional.of((Summary) cached);
        }
        return Optional.empty();
    }

    // 캐시 저장
    public void put(Long unitId, String difficulty, Summary summary) {
        String key = KEY_PREFIX + unitId + ":" + difficulty;
        redisTemplate.opsForValue().set(key, summary, TTL_HOURS, TimeUnit.HOURS);
    }

    // 캐시 무효화 (관리자 수정 시)
    public void evict(Long unitId, String difficulty) {
        String key = KEY_PREFIX + unitId + ":" + difficulty;
        redisTemplate.delete(key);
    }
}
```

### 8.6 Rate Limiting 구현

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    @Autowired
    private StringRedisTemplate redisTemplate;

    // 엔드포인트별 제한 설정
    private static final Map<String, RateLimit> LIMITS = Map.of(
        "/api/ai/",      new RateLimit(10, 3600),   // AI: 10회/시간
        "/api/summaries/generate", new RateLimit(10, 3600),
        "/api/quizzes/generate",    new RateLimit(20, 3600),
        "/api/english/tts",         new RateLimit(30, 3600)
    );

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws Exception {
        String path = request.getRequestURI();
        String userId = getCurrentUserId(request);

        RateLimit limit = findMatchingLimit(path);
        if (limit == null) return true;

        String key = "rate:" + userId + ":" + path;
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, limit.getWindowSeconds(), TimeUnit.SECONDS);
        }

        if (count > limit.getMaxRequests()) {
            response.setStatus(429);
            response.getWriter().write(
                "{\"success\":false,\"error\":\"RATE_LIMIT_EXCEEDED\","
                + "\"message\":\"요청 한도를 초과했습니다. 잠시 후 다시 시도해주세요.\"}"
            );
            return false;
        }
        return true;
    }

    private record RateLimit(int maxRequests, int windowSeconds) {}
}
```

---

## 9. 프론트엔드 아키텍처

### 9.1 디렉토리 구조

```
dailyStudyView/
├── public/
│   ├── favicon.ico
│   └── index.html
├── src/
│   ├── api/                               # API 클라이언트
│   │   ├── index.ts                       # Axios 인스턴스 + 인터셉터
│   │   ├── authApi.ts                     # 인증 API
│   │   ├── userApi.ts                     # 사용자 API
│   │   ├── subjectApi.ts                  # 과목/단원 API
│   │   ├── summaryApi.ts                  # 요약 API
│   │   ├── quizApi.ts                     # 문제/풀이 API
│   │   ├── englishApi.ts                  # 실생활 영어 API
│   │   ├── dashboardApi.ts               # 대시보드 API
│   │   ├── badgeApi.ts                    # 배지 API
│   │   ├── adminApi.ts                    # 관리자 API
│   │   └── fileApi.ts                     # 파일 업로드 API
│   │
│   ├── assets/                            # 정적 자산
│   │   ├── images/
│   │   └── styles/
│   │       └── tailwind.css               # TailwindCSS entry
│   │
│   ├── components/                        # 재사용 컴포넌트
│   │   ├── common/
│   │   │   ├── BaseButton.vue             # 버튼 (variant: primary, secondary, danger)
│   │   │   ├── BaseCard.vue               # 카드
│   │   │   ├── BaseModal.vue              # 모달
│   │   │   ├── BaseInput.vue              # 입력 필드
│   │   │   ├── BaseSelect.vue             # 셀렉트
│   │   │   ├── BaseBadge.vue              # 배지/태그
│   │   │   ├── BaseProgress.vue           # 진행률 바
│   │   │   ├── BaseSpinner.vue            # 로딩 스피너
│   │   │   ├── BaseEmpty.vue              # 데이터 없음
│   │   │   ├── BaseToast.vue              # 토스트 알림
│   │   │   └── Pagination.vue             # 페이지네이션
│   │   ├── layout/
│   │   │   ├── AppSidebar.vue             # 사이드바 네비게이션
│   │   │   ├── AppHeader.vue              # 헤더 (햄버거, 프로필)
│   │   │   └── AppFooter.vue              # 푸터
│   │   ├── study/
│   │   │   ├── SummaryCard.vue            # 요약 카드
│   │   │   ├── SummaryContent.vue         # 요약 내용 렌더러 (Markdown)
│   │   │   ├── QuizCard.vue               # 문제 카드
│   │   │   ├── QuizOption.vue             # 문제 보기
│   │   │   ├── QuizResult.vue             # 풀이 결과
│   │   │   ├── WrongNoteItem.vue          # 오답노트 아이템
│   │   │   └── DifficultyBadge.vue        # 난이도 배지
│   │   ├── english/
│   │   │   ├── EnglishSentenceCard.vue    # 영어 문장 카드
│   │   │   └── TtsPlayer.vue              # TTS 음성 재생기
│   │   ├── dashboard/
│   │   │   ├── StatCard.vue               # 통계 카드
│   │   │   ├── WeeklyChart.vue            # 주간 학습 차트
│   │   │   ├── StudyCalendar.vue          # 학습 캘린더
│   │   │   └── RecentActivity.vue         # 최근 활동
│   │   └── gamification/
│   │       ├── BadgeDisplay.vue           # 배지 표시
│   │       ├── LevelProgress.vue          # 레벨 진행률
│   │       └── StreakCounter.vue          # 연속 학습일 카운터
│   │
│   ├── composables/                       # Vue Composables
│   │   ├── useAuth.ts                     # 인증 상태 관리
│   │   ├── useApi.ts                      # API 호출 공통 로직
│   │   ├── useToast.ts                    # 토스트 알림
│   │   ├── usePagination.ts              # 페이지네이션
│   │   ├── useTimer.ts                    # 타이머 (문제 풀이 시간)
│   │   └── useTheme.ts                    # 테마 (다크/라이트)
│   │
│   ├── layouts/                           # 레이아웃 컴포넌트
│   │   ├── DefaultLayout.vue              # 사이드바 + 헤더 + RouterView
│   │   ├── AuthLayout.vue                 # 로그인/회원가입용 (센터 정렬)
│   │   └── AdminLayout.vue                # 관리자 전용 레이아웃
│   │
│   ├── router/                            # Vue Router
│   │   └── index.ts                       # 라우트 정의 + 가드
│   │
│   ├── stores/                            # Pinia Stores
│   │   ├── auth.ts                        # 인증 상태 (token, user, role)
│   │   ├── user.ts                        # 사용자 정보 + 설정
│   │   ├── subject.ts                     # 과목/단원 상태
│   │   ├── quiz.ts                        # 문제 풀이 상태
│   │   ├── dashboard.ts                   # 대시보드 데이터
│   │   └── notification.ts               # 알림 상태
│   │
│   ├── views/                             # 페이지 뷰
│   │   ├── auth/
│   │   │   ├── LoginView.vue
│   │   │   └── SignupView.vue
│   │   ├── DashboardView.vue              # 홈/대시보드
│   │   ├── subject/
│   │   │   ├── SubjectListView.vue        # 과목 목록
│   │   │   └── UnitListView.vue           # 단원 목록
│   │   ├── summary/
│   │   │   ├── SummaryListView.vue        # 요약 목록
│   │   │   └── SummaryDetailView.vue      # 요약 상세
│   │   ├── quiz/
│   │   │   ├── QuizListView.vue           # 문제 목록
│   │   │   ├── QuizSolveView.vue          # 문제 풀이
│   │   │   └── WrongNoteView.vue          # 오답노트
│   │   ├── english/
│   │   │   ├── EnglishListView.vue        # 실생활 영어 목록
│   │   │   └── EnglishDetailView.vue      # 영어 문장 상세 + TTS
│   │   ├── stats/
│   │   │   └── StatsView.vue              # 학습 통계
│   │   ├── MyPageView.vue                 # 마이페이지
│   │   └── admin/
│   │       ├── AdminDashboardView.vue     # 관리자 대시보드
│   │       ├── AdminUserListView.vue      # 사용자 관리
│   │       ├── AdminContentView.vue       # 콘텐츠 관리
│   │       └── AdminStatsView.vue         # 플랫폼 통계
│   │
│   ├── App.vue                            # 루트 컴포넌트
│   ├── main.ts                            # 앱 진입점 (Pinia + Router + Tailwind)
│   └── env.d.ts                           # 타입 정의
│
├── docker/
│   └── nginx.conf                         # Nginx 설정 (Docker용)
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
└── Dockerfile
```

### 9.2 라우터 설계

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import DefaultLayout from '@/layouts/DefaultLayout.vue'
import AuthLayout from '@/layouts/AuthLayout.vue'
import AdminLayout from '@/layouts/AdminLayout.vue'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    // 인증 라우트 (사이드바 없음)
    {
      path: '/login',
      component: AuthLayout,
      children: [
        { path: '', name: 'login', component: () => import('@/views/auth/LoginView.vue') },
      ],
    },
    {
      path: '/signup',
      component: AuthLayout,
      children: [
        { path: '', name: 'signup', component: () => import('@/views/auth/SignupView.vue') },
      ],
    },

    // 메인 라우트 (사이드바 + 헤더)
    {
      path: '/',
      component: DefaultLayout,
      children: [
        { path: '', redirect: '/dashboard' },

        // 대시보드
        { path: 'dashboard', name: 'dashboard',
          component: () => import('@/views/DashboardView.vue') },

        // 과목/단원
        { path: 'subjects', name: 'subjectList',
          component: () => import('@/views/subject/SubjectListView.vue') },
        { path: 'subjects/:subjectId/units', name: 'unitList',
          component: () => import('@/views/subject/UnitListView.vue') },

        // 요약
        { path: 'summaries', name: 'summaryList',
          component: () => import('@/views/summary/SummaryListView.vue') },
        { path: 'summaries/:id', name: 'summaryDetail',
          component: () => import('@/views/summary/SummaryDetailView.vue') },

        // 문제
        { path: 'quizzes', name: 'quizList',
          component: () => import('@/views/quiz/QuizListView.vue') },
        { path: 'quizzes/solve/:id', name: 'quizSolve',
          component: () => import('@/views/quiz/QuizSolveView.vue') },
        { path: 'quizzes/wrong-note', name: 'wrongNote',
          component: () => import('@/views/quiz/WrongNoteView.vue') },

        // 실생활 영어
        { path: 'english', name: 'englishList',
          component: () => import('@/views/english/EnglishListView.vue') },
        { path: 'english/:id', name: 'englishDetail',
          component: () => import('@/views/english/EnglishDetailView.vue') },

        // 통계
        { path: 'stats', name: 'stats',
          component: () => import('@/views/stats/StatsView.vue') },

        // 마이페이지
        { path: 'mypage', name: 'mypage',
          component: () => import('@/views/MyPageView.vue') },
      ],
    },

    // 관리자 라우트
    {
      path: '/admin',
      component: AdminLayout,
      meta: { requiresAdmin: true },
      children: [
        { path: '', redirect: '/admin/dashboard' },
        { path: 'dashboard', name: 'adminDashboard',
          component: () => import('@/views/admin/AdminDashboardView.vue') },
        { path: 'users', name: 'adminUsers',
          component: () => import('@/views/admin/AdminUserListView.vue') },
        { path: 'contents', name: 'adminContents',
          component: () => import('@/views/admin/AdminContentView.vue') },
        { path: 'stats', name: 'adminStats',
          component: () => import('@/views/admin/AdminStatsView.vue') },
      ],
    },
  ],
})

// 라우터 가드
router.beforeEach((to, from, next) => {
  const auth = useAuthStore()

  // 공개 라우트
  if (to.path === '/login' || to.path === '/signup') {
    next()
    return
  }

  // 인증 확인
  if (!auth.isSessionValid()) {
    next('/login')
    return
  }

  // 관리자 권한 확인
  if (to.meta.requiresAdmin && !auth.isAdmin) {
    next('/dashboard')
    return
  }

  next()
})

export default router
```

### 9.3 Pinia Store 설계

```typescript
// src/stores/auth.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { authApi } from '@/api/authApi'

export const useAuthStore = defineStore('auth', () => {
  // State
  const token = ref<string | null>(localStorage.getItem('ds_token'))
  const refreshToken = ref<string | null>(localStorage.getItem('ds_refresh_token'))
  const user = ref<any>(JSON.parse(localStorage.getItem('ds_user') || 'null'))

  // Getters
  const isAuthenticated = computed(() => !!token.value)
  const isAdmin = computed(() => user.value?.role === 'ADMIN')
  const userName = computed(() => user.value?.name || '')
  const userGrade = computed(() => user.value?.grade || null)

  // Actions
  async function login(email: string, password: string) {
    const res = await authApi.login({ email, password })
    token.value = res.data.token
    refreshToken.value = res.data.refreshToken
    user.value = res.data.user
    localStorage.setItem('ds_token', res.data.token)
    localStorage.setItem('ds_refresh_token', res.data.refreshToken)
    localStorage.setItem('ds_user', JSON.stringify(res.data.user))
    localStorage.setItem('ds_last_activity', String(Date.now()))
  }

  function logout() {
    authApi.logout()  // 서버에 블랙리스트 등록
    token.value = null
    refreshToken.value = null
    user.value = null
    localStorage.removeItem('ds_token')
    localStorage.removeItem('ds_refresh_token')
    localStorage.removeItem('ds_user')
    localStorage.removeItem('ds_last_activity')
  }

  function isSessionValid(): boolean {
    if (!token.value) return false
    const lastActivity = localStorage.getItem('ds_last_activity')
    if (!lastActivity) return false
    // 1시간 세션 타임아웃
    return Date.now() - parseInt(lastActivity) < 60 * 60 * 1000
  }

  return {
    token, refreshToken, user,
    isAuthenticated, isAdmin, userName, userGrade,
    login, logout, isSessionValid,
  }
})
```

```typescript
// src/stores/quiz.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { quizApi } from '@/api/quizApi'

export const useQuizStore = defineStore('quiz', () => {
  const quizzes = ref<any[]>([])
  const currentQuiz = ref<any>(null)
  const attempts = ref<any[]>([])
  const wrongNotes = ref<any[]>([])
  const loading = ref(false)

  const correctCount = computed(() =>
    attempts.value.filter(a => a.isCorrect).length
  )
  const accuracy = computed(() =>
    attempts.value.length > 0
      ? Math.round((correctCount.value / attempts.value.length) * 100)
      : 0
  )

  async function fetchQuizzes(params: { unitId?: number; difficulty?: string; type?: string }) {
    loading.value = true
    // 학생용 — 승인된 문제만 조회 (status=PUBLISHED)
    const res = await quizApi.getAll(params)
    quizzes.value = res.data.items
    loading.value = false
  }

  // 학생은 실시간 AI 생성을 호출하지 않습니다 — 승인된 문제 풀에서만 제공
  // 관리자 생성/승인은 별도 adminApi를 통해 처리됩니다

  async function submitAnswer(quizId: number, answer: string, timeSpent: number) {
    const res = await quizApi.submit({
      quizId, userAnswer: answer, timeSpent
    })
    attempts.value.push(res.data)
    if (!res.data.isCorrect) {
      wrongNotes.value.push(res.data)
    }
    return res.data
  }

  async function fetchWrongNotes() {
    const res = await quizApi.getWrongNotes()
    wrongNotes.value = res.data.items
  }

  return {
    quizzes, currentQuiz, attempts, wrongNotes, loading,
    correctCount, accuracy,
    fetchQuizzes, generateQuiz, submitAnswer, fetchWrongNotes,
  }
})
```

### 9.4 Axios 인터셉터 (JWT + 세션 연장)

```typescript
// src/api/index.ts
import axios from 'axios'
import { useAuthStore } from '@/stores/auth'

const api = axios.create({
  baseURL: '/api',
  timeout: 15000,
  headers: { 'Content-Type': 'application/json' },
})

// 요청 인터셉터 — JWT 헤더 추가 + 활동 시간 업데이트
api.interceptors.request.use((config) => {
  const auth = useAuthStore()
  if (auth.token) {
    config.headers.Authorization = `Bearer ${auth.token}`
    localStorage.setItem('ds_last_activity', String(Date.now()))
  }
  return config
})

// 응답 인터셉터 — 401/403 처리 + 토큰 갱신
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const auth = useAuthStore()
    const originalRequest = error.config

    // 401 — 토큰 만료 시 리프레시 시도
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true
      try {
        const res = await axios.post('/api/auth/refresh', {
          refreshToken: auth.refreshToken,
        })
        auth.token = res.data.token
        originalRequest.headers.Authorization = `Bearer ${res.data.token}`
        return api(originalRequest)
      } catch {
        auth.logout()
        window.location.href = '/login'
      }
    }

    // 403 — 권한 없음
    if (error.response?.status === 403) {
      console.error('접근 권한이 없습니다.')
    }

    return Promise.reject(error)
  }
)

export default api
```

### 9.5 Vite 설정

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: { '@': resolve(__dirname, 'src') },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true },
      '/swagger-ui': { target: 'http://localhost:8080', changeOrigin: true },
      '/v3/api-docs': { target: 'http://localhost:8080', changeOrigin: true },
      '/uploads': { target: 'http://localhost:8080', changeOrigin: true },
    },
  },
  build: {
    outDir: 'dist',
    chunkSizeWarningLimit: 1000,
    rollupOptions: {
      output: {
        manualChunks: {
          'vue-vendor': ['vue', 'vue-router', 'pinia'],
          'ui-vendor': ['axios'],
        },
      },
    },
  },
})
```

### 9.6 TailwindCSS 설정

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{vue,js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff', 100: '#dbeafe', 200: '#bfdbfe',
          300: '#93c5fd', 400: '#60a5fa', 500: '#3b82f6',
          600: '#2563eb', 700: '#1d4ed8', 800: '#1e40af',
          900: '#1e3a8a',
        },
        // 학습 플랫폼 테마 색상
        success: '#22c55e',
        warning: '#f59e0b',
        danger: '#ef4444',
      },
      fontFamily: {
        sans: ['Pretendard', 'system-ui', 'sans-serif'],
      },
    },
  },
  plugins: [],
}
```

---

## 10. 배포 아키텍처

### 10.1 Docker Compose 전체 구성

```yaml
# docker-compose.yml
# Daily Study 전체 서비스 orchestration
# 사용법:
#   docker compose up -d          # 전체 서비스 시작
#   docker compose down           # 전체 서비스 중지
#   docker compose logs -f        # 로그 실시간 확인
#   docker compose restart backend # 백엔드만 재시арт

services:
  # ── MariaDB 11 데이터베이스 ──
  mariadb:
    image: mariadb:11
    container_name: dailystudy-db
    restart: unless-stopped
    environment:
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-rootpass}
      MARIADB_DATABASE: dailystudy
      MARIADB_USER: ${DB_USER:-shaloman}
      MARIADB_PASSWORD: ${DB_PASSWORD:-123321}
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./docker/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks:
      - dailystudy-net
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Redis 7 캐시/세션 ──
  redis:
    image: redis:7-alpine
    container_name: dailystudy-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - dailystudy-net
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Spring Boot 백엔드 ──
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: dailystudy-backend
    restart: unless-stopped
    depends_on:
      mariadb:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "8080:8080"
    networks:
      - dailystudy-net
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mariadb://mariadb:3306/dailystudy
      - SPRING_DATASOURCE_USERNAME=${DB_USER:-shaloman}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD:-123321}
      - SPRING_DATA_REDIS_HOST=redis
      - SPRING_DATA_REDIS_PORT=6379
      - JWT_SECRET=${JWT_SECRET:-dailystudy-secret-key-2026}
      - GLM_API_KEY=${GLM_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID:-}
      - GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET:-}
      - KAKAO_CLIENT_ID=${KAKAO_CLIENT_ID:-}
      - KAKAO_CLIENT_SECRET=${KAKAO_CLIENT_SECRET:-}
    volumes:
      - uploads_data:/opt/dailystudy/uploads

  # ── Nginx 프론트엔드 ──
  frontend:
    build:
      context: ../dailyStudyView
      dockerfile: Dockerfile
    container_name: dailystudy-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "80:80"
      # - "443:443"  # Phase 2: HTTPS
    networks:
      - dailystudy-net
    volumes:
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      # - ./docker/ssl:/etc/nginx/ssl:ro  # Phase 2: SSL 인증서

volumes:
  db_data:
  redis_data:
  uploads_data:

networks:
  dailystudy-net:
    driver: bridge
```

### 10.2 백엔드 Dockerfile

```dockerfile
# Daily Study Backend Dockerfile
# Multi-stage build: Maven build → JRE 17 runtime

# ── Stage 1: Build ──
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /build

# pom.xml 먼저 복사 (의존성 캐싱)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 소스 복사 및 빌드
COPY src ./src
RUN mvn package -q -DskipTests

# ── Stage 2: Runtime ──
FROM eclipse-temurin:17-jre

WORKDIR /app

# 빌드된 JAR 복사
COPY --from=builder /build/target/dailystudy-*-SNAPSHOT.jar app.jar

# application-prod.yml 복사
COPY src/main/resources/application-prod.yml /app/config/application.yml

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar", \
  "--spring.profiles.active=prod", \
  "--spring.config.additional-location=file:/app/config/"]
```

### 10.3 프론트엔드 Dockerfile

```dockerfile
# Daily Study Frontend Dockerfile
# Vue 3 빌드 + Nginx 서빙

# ── Stage 1: Build Vue app ──
FROM node:22-alpine AS builder

WORKDIR /build

COPY package.json package-lock.json* ./
RUN npm ci --silent 2>/dev/null || npm install --silent

COPY . .
RUN npm run build

# ── Stage 2: Nginx serve ──
FROM nginx:alpine

COPY --from=builder /build/dist /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 10.4 Nginx 설정 (Docker용)

```nginx
# docker/nginx.conf
server {
    listen 80;
    server_name _;

    # Vue.js 정적 파일 서빙
    root /usr/share/nginx/html;
    index index.html;

    # 보안 헤더
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # index.html — 캐시 방지
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate" always;
        expires 0;
    }

    # SPA 라우팅
    location / {
        add_header Cache-Control "no-cache, no-store, must-revalidate" always;
        try_files $uri $uri/ /index.html;
    }

    # 정적 자산 — 장기 캐싱
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # API 프록시
    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        # AI 생성 API는 타임아웃 연장
        location /api/ai/ {
            proxy_pass http://backend:8080;
            proxy_read_timeout 120s;
        }
    }

    # Swagger UI 프록시 (dev만)
    location /swagger-ui/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
    }
    location /v3/api-docs {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
    }

    # 업로드 파일 프록시
    location /uploads/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
    }
}
```

### 10.5 환경 변수 (.env)

```bash
# .env (Docker Compose용 — .gitignore에 추가)
DB_ROOT_PASSWORD=strong-root-password
DB_USER=shaloman
DB_PASSWORD=strong-db-password
JWT_SECRET=your-jwt-secret-key-at-least-32-chars-long

# AI API Keys
GLM_API_KEY=your-glm-api-key
OPENAI_API_KEY=your-openai-api-key

# OAuth2 (Phase 2)
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
KAKAO_CLIENT_ID=your-kakao-client-id
KAKAO_CLIENT_SECRET=your-kakao-client-secret
```

### 10.6 배포 디렉토리 구조

```
/opt/dailystudy/                        # 배포 루트
├── docker-compose.yml                  # Docker Compose 설정
├── .env                                # 환경 변수
├── docker/
│   ├── init-db.sql                     # DB 초기화 스크립트
│   ├── nginx.conf                      # Nginx 설정
│   └── ssl/                            # SSL 인증서 (Phase 2)
│       ├── fullchain.pem
│       └── privkey.pem
├── uploads/                            # 업로드 파일
└── logs/                               # 로그
    ├── backend/
    └── nginx/
```

### 10.7 배포 절차

```bash
# 1. 저장소 클론
git clone https://github.com/ban-jh/dailystudy.git
git clone https://github.com/ban-jh/dailyStudyView.git

# 2. 환경 변수 설정
cd dailystudy
cp .env.example .env
vi .env  # API 키, 비밀번호 설정

# 3. DB 초기화 스크립트 생성
mysqldump -u shaloman -p dailystudy --single-transaction --routines --triggers > docker/init-db.sql

# 4. 전체 서비스 빌드 및 시작
docker compose up -d --build

# 5. 서비스 상태 확인
docker compose ps
docker compose logs -f backend

# 6. 헬스체크
curl http://localhost:8080/api/auth/me
curl http://localhost/
```

---

## 부록: 개발 단계별 구현 범위

### Phase 1 — MVP (4~6주)

| 항목 | 상세 |
|------|------|
| 인증 | 이메일 회원가입/로그인, JWT |
| 과목 | 3과목 (수학, 영어, 과학) + 단원 구조 |
| 요약 | AI 요약 생성 (GLM-5.2), 요약 조회 |
| 문제 | 관리자 문제 생성(AI/크롤링/직접) + 승인 시스템, 승인된 문제 풀이, 채점 |
| 대시보드 | 학습 통계, 최근 활동 |
| 프론트엔드 | 로그인, 대시보드, 과목목록, 요약, 문제풀이 |
| 인프라 | Docker Compose (MariaDB + Backend + Nginx) |

### Phase 2 — 확장 (4주)

| 항목 | 상세 |
|------|------|
| 실생활 영어 | 영어 문장 조회, 카테고리별 탐색 |
| TTS | OpenAI TTS / Edge TTS 음성 합성 |
| 오답노트 | 오답 목록, 재풀이 기능 |
| 소셜 로그인 | Google, Kakao OAuth2 |
| 크롤링 | Spring Batch 영어 문장 크롤링 |
| Redis | 캐싱, Rate Limiting 도입 |

### Phase 3 — 고도화 (4주)

| 항목 | 상세 |
|------|------|
| AI 추천 | 학습 패턴 기반 맞춤 추천 |
| 게이미피케이션 | 배지, 포인트, 레벨, 연속 학습일, 리더보드 |
| 관리자 | 사용자 관리, 콘텐츠 승인, 플랫폼 통계 |
| 학습 캘린더 | 일일 학습 여부 시각화 |

### Phase 4 — 상용화 (지속)

| 항목 | 상세 |
|------|------|
| 전 과목 | 중·고 전 교과과정 확장 |
| 성능 최적화 | 쿼리 튜닝, CDN, 이미지 최적화 |
| 모니터링 | Prometheus + Grafana, 로그 수집 |
| HTTPS | Let's Encrypt SSL 적용 |
| 모바일 | PWA 또는 반응형 최적화 |

---

> **문서 끝** — Daily Study 상세 아키텍처 설계서 v1.0  
> 기술 스택 및 패키지 구조는 holylife 프로젝트 (`org.shaloman.pj`) 패턴을 계승합니다.