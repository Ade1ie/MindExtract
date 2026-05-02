# 데이터 모델 및 API 설계

## 데이터베이스: PostgreSQL

### 선택 근거
- LangGraph `PostgresSaver` 체크포인터와 네이티브 연동
- 맥북 로컬에서 Homebrew/Postgres.app/Docker로 간편 설치
- JSON/JSONB 타입으로 명세서/모듈 데이터의 유연한 저장

---

## ER 다이어그램 개요

```
┌────────────┐      ┌─────────────────┐      ┌──────────────────┐
│  Client     │──1:N─│    Project       │──1:N─│ InterviewSession  │
└────────────┘      └─────────────────┘      └──────────────────┘
                           │                         │
                           │ 1:N                     │ 1:N
                           ▼                         ▼
                    ┌──────────────┐         ┌────────────────┐
                    │   Estimate    │         │InterviewMessage │
                    └──────────────┘         └────────────────┘
                           │
                    ┌──────────────┐
                    │  SpecVersion  │──────────────────────────┐
                    └──────────────┘                          │
                                                               │ N:M
┌──────────────────┐                               ┌──────────────────────┐
│ FrameworkModule   │──────────────────N:M──────────│ ProjectModuleMapping  │
└──────────────────┘                               └──────────────────────┘
                    
┌──────────────────┐
│  InviteLink       │ (선택 — ngrok 사용 시에만 필요)
└──────────────────┘
```

---

## 테이블 상세 정의

### 1. Client (클라이언트 관리)

```sql
CREATE TABLE clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,          -- 클라이언트 이름/회사명
    contact_name    VARCHAR(100),                   -- 담당자 이름
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(20),
    company_type    VARCHAR(30),                    -- startup | sme | agency | individual
    notes           TEXT,                           -- 메모
    status          VARCHAR(20) DEFAULT 'active',   -- active | inactive | blacklist
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 2. Project (프로젝트)

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID REFERENCES clients(id) ON DELETE CASCADE,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    domain          VARCHAR(50),                    -- marketplace | social | fintech | etc.
    product_type    VARCHAR(50),                    -- web_app | mobile_app | api | etc.
    
    -- 프로젝트 상태
    status          VARCHAR(30) DEFAULT 'inquiry',  -- inquiry | spec_done | estimated | contracted | in_progress | delivered | closed
    
    -- 수익 관리
    estimated_cost  DECIMAL(12,0),                  -- 견적 금액 (원)
    contracted_cost DECIMAL(12,0),                  -- 계약 금액 (원)
    actual_hours    DECIMAL(6,1),                   -- 실제 투입 시간
    
    -- 일정
    started_at      TIMESTAMPTZ,
    deadline        TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_projects_client ON projects(client_id);
CREATE INDEX idx_projects_status ON projects(status);
```

### 3. InterviewSession (인터뷰 세션)

```sql
CREATE TABLE interview_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    initial_prompt  TEXT NOT NULL,
    
    -- Analyzer 결과
    domain          VARCHAR(50),
    product_type    VARCHAR(50),
    gap_severity    DECIMAL(3,2),
    functional_gaps JSONB,
    non_functional_gaps JSONB,
    
    -- 세션 상태
    status          VARCHAR(20) DEFAULT 'active',   -- active | completed | abandoned
    current_round   INTEGER DEFAULT 0,
    total_rounds    INTEGER,
    
    -- LangGraph State
    state_snapshot  JSONB,
    conversation_summary TEXT,
    
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 4. InterviewMessage (인터뷰 메시지)

```sql
CREATE TABLE interview_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES interview_sessions(id) ON DELETE CASCADE,
    role            VARCHAR(10) NOT NULL,            -- 'ai' | 'user' | 'system'
    content         TEXT NOT NULL,
    message_type    VARCHAR(20),                     -- 'question' | 'answer' | 'summary'
    metadata        JSONB,                           -- 질문 옵션, 선택된 답변 등
    token_count     INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_session ON interview_messages(session_id, created_at);
```

### 5. SpecVersion (명세서 버전 관리)

```sql
CREATE TABLE spec_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES interview_sessions(id),
    
    -- 명세서 내용
    spec_data       JSONB NOT NULL,                  -- 정형화된 명세서 (전체 구조)
    prd_markdown    TEXT NOT NULL,                    -- 마크다운 PRD
    system_prompt_json JSONB,                        -- AI 코딩 도구용 JSON
    
    -- 버전 관리
    version         INTEGER NOT NULL,
    spec_hash       VARCHAR(64),                     -- SHA-256, diff 추적용
    change_summary  TEXT,                            -- 변경 사항 요약
    
    -- 품질
    quality_score   DECIMAL(3,2),
    
    -- 최신 여부
    is_latest       BOOLEAN DEFAULT TRUE,
    
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_spec_project ON spec_versions(project_id, version);
CREATE UNIQUE INDEX idx_spec_latest ON spec_versions(project_id) WHERE is_latest = TRUE;
```

### 6. FrameworkModule (프레임워크 모듈 카탈로그)

```sql
CREATE TABLE framework_modules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- 모듈 정보
    name            VARCHAR(100) NOT NULL,           -- 예: "소셜 로그인 모듈"
    slug            VARCHAR(50) UNIQUE NOT NULL,      -- 예: "auth_social_v2"
    category        VARCHAR(50) NOT NULL,             -- auth | payment | chat | search | etc.
    description     TEXT,
    
    -- 기술 정보
    tech_stack      JSONB,                           -- {"language": "python", "framework": "fastapi"}
    dependencies    JSONB,                           -- 의존하는 다른 모듈 목록
    source_path     VARCHAR(500),                    -- 소스 코드 경로/리포지토리
    
    -- 성숙도 관리
    maturity        VARCHAR(20) DEFAULT 'poc',       -- poc | stable | battle_tested
    version         VARCHAR(20),                     -- SemVer
    
    -- 시간 추정
    base_dev_hours  DECIMAL(5,1),                    -- 이 모듈을 처음부터 만드는 데 걸리는 시간
    avg_custom_hours DECIMAL(5,1),                   -- 평균 커스터마이징 시간
    
    -- 재사용 통계
    usage_count     INTEGER DEFAULT 0,               -- 사용된 프로젝트 수
    
    -- 매핑용 키워드 (AI가 명세서와 매칭할 때 참조)
    matching_keywords JSONB,                         -- ["로그인", "인증", "OAuth", "소셜"]
    
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_modules_category ON framework_modules(category);
```

### 7. ProjectModuleMapping (프로젝트 ↔ 모듈 매핑)

```sql
CREATE TABLE project_module_mappings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    module_id       UUID REFERENCES framework_modules(id),
    
    -- 매핑 상태
    mapping_status  VARCHAR(20) NOT NULL,            -- available | partial | unavailable
    feature_id      VARCHAR(20),                     -- 명세서 기능 ID (예: "F001")
    feature_name    VARCHAR(200),
    
    -- 커스터마이징 정보
    customization_notes TEXT,                        -- 필요한 커스터마이징 내용
    estimated_custom_hours DECIMAL(5,1),
    estimated_hours_saved DECIMAL(5,1),              -- 이 모듈 덕분에 절약되는 시간
    
    -- 실제 기록 (프로젝트 완료 후 기입)
    actual_custom_hours DECIMAL(5,1),
    
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_mapping_project ON project_module_mappings(project_id);
```

### 8. Estimate (견적서)

```sql
CREATE TABLE estimates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    
    -- 견적 데이터
    breakdown       JSONB NOT NULL,                  -- 항목별 시간/비용 상세
    total_hours     DECIMAL(6,1) NOT NULL,
    hourly_rate     DECIMAL(10,0) NOT NULL,          -- 시급 (원)
    total_cost      DECIMAL(12,0) NOT NULL,
    estimated_weeks DECIMAL(3,1),
    
    -- 프레임워크 효과
    hours_without_framework DECIMAL(6,1),            -- 프레임워크 없이 개발 시 예상 시간
    framework_savings_pct DECIMAL(4,1),              -- 절약률 (%)
    
    -- 버전 관리
    version         INTEGER DEFAULT 1,
    is_latest       BOOLEAN DEFAULT TRUE,
    notes           TEXT,
    
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 9. InviteLink (선택 — ngrok/터널 사용 시에만)

> 내부 앱은 로컬 전용이므로 이 테이블은 선택사항입니다.
> 추후 ngrok/Cloudflare Tunnel로 외부 접근이 필요해지면 구현합니다.

```sql
-- 선택: 외부 인터뷰 링크가 필요할 때만 생성
CREATE TABLE invite_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    link_code       VARCHAR(20) UNIQUE NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 10. ProjectTimeline (프로젝트 진행 이력)

```sql
CREATE TABLE project_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    event_type      VARCHAR(30) NOT NULL,            -- spec_created | estimated | contracted | milestone | delivered | spec_changed
    title           VARCHAR(200) NOT NULL,
    description     TEXT,
    metadata        JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_timeline_project ON project_timeline(project_id, created_at);
```

---

## REST API 엔드포인트 (localhost:8000)

> 로컬 전용이므로 인증 불필요. 모든 API는 인증 없이 접근.

### 클라이언트 관리

| Method | Endpoint | 설명 |
|--------|----------|------|
| `POST` | `/api/clients` | 클라이언트 등록 |
| `GET` | `/api/clients` | 목록 (필터/검색) |
| `GET` | `/api/clients/{id}` | 상세 |
| `PATCH` | `/api/clients/{id}` | 수정 |

### 프로젝트 관리

| Method | Endpoint | 설명 |
|--------|----------|------|
| `POST` | `/api/projects` | 생성 (클라이언트 연결) |
| `GET` | `/api/projects` | 전체 목록 (상태별 필터) |
| `GET` | `/api/projects/{id}` | 상세 (명세서+견적+모듈 매핑 포함) |
| `PATCH` | `/api/projects/{id}` | 수정 |
| `PATCH` | `/api/projects/{id}/status` | 상태 전이 |

### 인터뷰 세션

| Method | Endpoint | 설명 |
|--------|----------|------|
| `POST` | `/api/interview/start` | 시작 (프로젝트 연결) |
| `POST` | `/api/interview/{session_id}/respond` | 응답 |
| `GET` | `/api/interview/{session_id}/resume` | 복원 |
| `POST` | `/api/interview/{session_id}/compile` | 전체 파이프라인 실행 |

### 명세서

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/api/specs/{project_id}` | 최신 명세서 |
| `GET` | `/api/specs/{project_id}/versions` | 버전 이력 |
| `GET` | `/api/specs/{project_id}/diff/{v1}/{v2}` | 버전 간 차이 |
| `GET` | `/api/specs/{project_id}/export?format=md\|json` | 내보내기 |

### 프레임워크 모듈

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/api/framework/modules` | 모듈 카탈로그 |
| `POST` | `/api/framework/modules` | 모듈 등록 |
| `PATCH` | `/api/framework/modules/{id}` | 모듈 업데이트 |
| `GET` | `/api/framework/modules/{id}/usage` | 사용 이력 |
| `GET` | `/api/framework/mapping/{project_id}` | 프로젝트 모듈 매핑 결과 |

### 견적

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/api/estimates/{project_id}` | 견적서 |
| `PATCH` | `/api/estimates/{project_id}` | 수동 조정 |
| `GET` | `/api/estimates/{project_id}/export` | 내보내기 |

### 클라이언트 인터뷰 링크 (선택 — ngrok 사용 시)

| Method | Endpoint | 설명 |
|--------|----------|------|
| `POST` | `/api/links/create` | 초대 링크 생성 |
| `GET` | `/api/s/{link_code}` | 클라이언트 접근 (ngrok 터널 필요) |

### 대시보드 (개인)

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/api/dashboard/overview` | 전체 현황 |
| `GET` | `/api/dashboard/stats` | 통계 (수익, 모듈 재사용률 등) |

---

## API 요청/응답 예시

### POST `/api/interview/start`

**Request:**
```json
{
    "project_id": "550e8400-...",
    "initial_prompt": "당근마켓 같은 중고거래 앱을 만들고 싶어요"
}
```

**Response:**
```json
{
    "session_id": "660e9500-...",
    "analysis": {
        "domain": "marketplace",
        "product_type": "mobile_app",
        "gap_severity": 0.85
    },
    "first_question": {
        "question": "타겟 사용자의 연령대는 어떻게 됩니까?",
        "type": "multiple_choice",
        "options": [
            {"label": "10~20대 (MZ세대)", "description": "트렌디한 UX, SNS 연동 강조"},
            {"label": "20~40대 (직장인)", "description": "실용성, 거래 안전성 강조"},
            {"label": "전 연령대", "description": "범용적 UI, 접근성 우선"}
        ],
        "recommendation": "20~40대 (직장인)",
        "recommendation_reason": "중고거래 플랫폼의 주 사용층이며 구매력이 높습니다"
    },
    "progress": 0.1
}
```

---

## 데이터 마이그레이션

### 도구
- **Alembic** (SQLAlchemy 마이그레이션)
- 로컬 PostgreSQL 백업: `pg_dump`

### 워크플로우
```
스키마 변경 → alembic revision --autogenerate
  → alembic upgrade head (로컬 DB에 즉시 적용)
  → git commit (마이그레이션 파일 버전 관리)
```
