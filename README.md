# 📚 Tech Daily Insight

> AI가 매일 커밋합니다. Java/Spring · DB · 네트워크 · 블록체인 · 최신 IT 동향
> Claude AI × GitHub Actions로 자동화된 백엔드 개발자 기술 아카이브

<br/>

## 🤖 어떻게 동작하나요?

매일 오전 9시, 아무도 키보드를 두드리지 않아도 새로운 기술 포스팅이 올라옵니다.

```
매일 09:00 (KST)
      │
      ▼
GitHub Actions 트리거
      │
      ▼
Claude AI가 주제 선택 & 포스팅 생성
      │
      ▼
posts/ 디렉토리에 마크다운 파일 저장
      │
      ▼
자동 커밋 & Push 완료 ✅
```

<br/>

## 🛠️ 기술 스택

| 역할 | 기술 |
|------|------|
| AI 콘텐츠 생성 | [Claude AI](https://anthropic.com) (claude-sonnet-4-6) |
| 자동화 파이프라인 | GitHub Actions |
| 언어 | Python 3.11 |
| 주제 중복 방지 | `.topic-history.json` (카테고리별 균형 선택) |

<br/>

## 📂 구조

```
tech_daily_insight/
├── .github/
│   └── workflows/
│       └── daily-post.yml       # GitHub Actions 워크플로우
├── posts/                       # 생성된 기술 포스팅 모음
│   └── YYYY-MM-DD-title.md
├── scripts/
│   └── generate.py              # Claude AI 포스팅 생성 스크립트
└── .topic-history.json          # 주제 중복 방지 이력
```

<br/>

## 📋 다루는 주제

| 카테고리 | 예시 주제 |
|---------|---------|
| ☕ Java/Spring | Virtual Threads, Spring Boot 3.x, GraalVM, WebFlux |
| 🖥️ 서버/인프라 | Kubernetes, Docker, CI/CD, AWS, 분산 시스템 |
| 🗄️ 데이터베이스 | PostgreSQL 최적화, Redis, MongoDB, Elasticsearch |
| 🌐 네트워크 | TCP/IP, HTTP/3, TLS, OAuth 2.0, Load Balancer |
| ⛓️ 블록체인 | 스마트 컨트랙트, DeFi, Web3.0, ZKP |
| 🚀 최신 IT 동향 | LLM 통합, RAG, GitOps, eBPF, 플랫폼 엔지니어링 |

총 **83개** 주제를 카테고리별 균형 있게 순환하며, 중복 없이 포스팅됩니다.

<br/>

## ⚙️ 자동화 구현 방식

### 1. 주제 선택 로직
- 6개 카테고리에서 가장 적게 사용된 카테고리 우선 선택
- `.topic-history.json`으로 사용된 주제 추적 → 완전 중복 방지
- 83개 주제 소진 시 자동 리셋 후 재시작

### 2. Claude AI 포스팅 생성
- 카테고리와 주제를 프롬프트에 포함하여 고품질 포스팅 생성
- 1,200 ~ 2,000 단어 분량의 실전 예제 포함 마크다운
- 구성: 개요 → 핵심 개념 → 실전 예제 → 주의사항 → 정리

### 3. GitHub Actions 파이프라인
- `cron: '0 0 * * *'` → 매일 UTC 00:00 (KST 09:00) 실행
- `concurrency` 설정으로 중복 실행 방지
- push 실패 시 자동 재시도 (최대 3회)
- `ANTHROPIC_API_KEY` 등 민감 정보는 GitHub Secrets로 안전하게 관리

<br/>

## 📈 Stats

![GitHub commit activity](https://img.shields.io/github/commit-activity/m/kwongyumin/tech_daily_insight)
![GitHub last commit](https://img.shields.io/github/last-commit/kwongyumin/tech_daily_insight)

---

> 포스팅은 Claude AI가 생성하며, 학습 및 참고 목적으로 활용하시기 바랍니다.
