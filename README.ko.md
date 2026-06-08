# Git Issue Sentinel

[English](./README.md)

Git Issue Sentinel은 Git 기반 협업 시스템의 이슈를 한곳에 모아 보고, 검색하고, 분류하고, 장기적으로는 관리와 외부 프로젝트 연계까지 제공하는 self-host 우선 issue intelligence 플랫폼입니다.

현재 프로젝트는 기획 및 설계 단계입니다. 구현은 `PROJECT.md`, `AGENTS.md`, `RESEARCH.md`의 결정을 기준으로 진행합니다.

## 핵심 방향

| 항목 | 결정 |
|---|---|
| 제품 우선순위 | Self-host 우선, SaaS는 후순위 |
| MVP Provider | GitHub |
| MVP 제품 범위 | GitHub Issues read-only 수집, 정규화, 단일 리스트 보기 |
| 프론트엔드 | Vue.js 최신 안정 버전, Vue 3 안정 라인 기준 |
| 빌드/개발 도구 | Vite + TypeScript + Vue SFC + Composition API 기준 |
| 이슈 리스트/보드 | `@tanstack/vue-table` 기반 headless issue table |
| 칸반 보드 | `SortableJS` 기반 드래그앤드롭, Vue에서는 안정적인 Vue 3 wrapper 또는 직접 composable로 적용 |
| 아키텍처 | Layered Architecture, Clean Architecture |
| 개발 방식 | Tidy First, TDD |
| 설정 원칙 | 외부 설정 파일 최소화, 환경값은 프로세스 시작 시점에만 수용 |
| 로그 | `product`, `debug`, `dev` 3가지 profile |

## 프로젝트가 해결하려는 문제

개발팀의 이슈는 GitHub, GitLab, Gitea, Forgejo, Bitbucket, Azure DevOps, SourceHut, Jira, Linear, Plane, OpenProject 등 여러 시스템에 흩어지기 쉽습니다. 저장소와 팀이 늘어나면 다음 문제가 커집니다.

- 여러 저장소의 이슈를 한 화면에서 볼 수 없음
- 라벨, 담당자, 상태, 우선순위가 provider마다 다름
- stale issue, unassigned issue, release blocker를 찾기 어려움
- Git issue와 외부 프로젝트 관리 도구의 연결이 약함
- self-host, 폐쇄망, 감사 로그, 데이터 보존 요구가 상용 SaaS와 맞지 않을 수 있음

Git Issue Sentinel은 provider별 API 차이를 adapter로 감추고, 내부에서는 정규화된 issue 모델로 다루는 방식으로 이 문제를 해결합니다.

## MVP

MVP는 범위를 좁게 잡습니다.

**목표:** GitHub Issues를 가져와 하나의 리스트에서 검색하고 필터링할 수 있는 read-only issue inbox를 만든다.

MVP에서 반드시 제공할 기능:

- GitHub 연결
- GitHub repository 선택
- open/closed/all issue 수집
- pagination 처리
- rate limit 상태 처리
- issue 원본 ID, number, URL, repository, state, labels, assignees, author, timestamps 보존
- raw payload 저장
- 정규화된 issue 저장
- 반복 sync 시 중복 생성 방지
- All Issues 리스트
- 기본 검색과 필터
- issue 상세 보기
- sync 상태 표시

MVP에서 제외할 기능:

- issue 생성/수정/삭제
- 댓글 작성
- 라벨/담당자/상태 변경
- 양방향 동기화
- webhook 기반 실시간 sync
- AI triage
- Jira/Linear/Plane/OpenProject 연계
- SaaS 운영 기능

## 최종 목표

최종 제품은 모든 Git 기반 issue source를 통합하는 운영 허브입니다.

장기 목표:

- GitHub, GitLab, Gitea, Forgejo, Bitbucket, Azure DevOps, SourceHut 지원
- self-hosted enterprise instance 지원
- 통합 issue inbox
- issue board와 kanban board
- provider write-back
- webhook/polling 기반 양방향 동기화
- Jira, Linear, Plane, OpenProject, Redmine 연계
- Slack, Teams, email 알림
- AI 기반 요약, 중복 탐지, 라벨/담당자/우선순위 추천
- 자동화 규칙
- SSO, RBAC, audit log, token encryption
- self-host single-node, enterprise, air-gapped 배포
- SaaS는 self-host 제품이 안정화된 이후 검토

## 기술 선택

## Frontend

Vue.js 최신 안정 버전을 사용합니다. 2026-06-08 기준 공식 문서는 Vue 3를 현재 문서 기준으로 제공하고, 대규모 앱에는 Single-File Component와 Composition API 조합을 권장합니다. 실제 스캐폴딩 시점에는 `vue@latest`와 `create-vue@latest`의 stable tag를 사용하고 lockfile로 고정합니다.

권장 구성:

- Vue 3 stable
- TypeScript
- Vite
- Vue Router
- Pinia
- Vue SFC
- Composition API
- Vitest
- Playwright 또는 Cypress

## Issue List / Board

이슈 리스트와 리스트형 보드는 `@tanstack/vue-table`을 기준으로 합니다.

선택 이유:

- Vue adapter 제공
- headless table이라 프로젝트 UI와 분리 가능
- sorting, filtering, pagination, column visibility, row selection 등을 직접 제어 가능
- 대량 issue 목록을 다루는 MVP와 잘 맞음
- Clean Architecture 관점에서 table state를 UI layer에 제한하기 좋음

규칙:

- TanStack Table은 frontend presentation/state 도구로만 사용합니다.
- domain model이나 sync logic은 TanStack Table에 의존하지 않습니다.
- 서버 필터/정렬이 필요한 경우 table state를 API query object로 변환합니다.

## Kanban Board

칸반 보드는 `SortableJS` 기반으로 구현합니다. Vue에서는 안정적인 Vue 3 wrapper를 선택하거나, wrapper의 유지보수 상태가 불충분하면 SortableJS를 직접 감싼 composable을 작성합니다.

선택 이유:

- SortableJS는 오래 검증된 drag-and-drop list 라이브러리입니다.
- GitHub 조직 기준 Vue wrapper 생태계가 존재합니다.
- 칸반의 핵심인 column 간 card 이동, 같은 column 내 정렬, drag handle, touch device 대응에 적합합니다.
- 보드의 도메인 정책과 drag-and-drop UI 동작을 분리하기 쉽습니다.

규칙:

- SortableJS는 UI interaction layer에서만 사용합니다.
- 이슈 상태 변경, 라벨 변경, provider write-back은 application use case가 담당합니다.
- drag 결과는 `MoveIssueOnBoardCommand` 같은 명시적 command로 변환합니다.
- MVP에서는 칸반 보드를 구현하지 않고, V1/V2 이후에 도입합니다.

## Backend 방향

구체적인 백엔드 언어/프레임워크는 아직 확정하지 않았습니다. 단, 구조는 다음 원칙을 따릅니다.

```text
apps/
  web/
  api/
  worker/
packages/
  core/
  adapters/
    github/
    gitlab/
    gitea/
    forgejo/
    bitbucket/
    azure-devops/
    sourcehut/
  db/
  auth/
  crypto/
  search/
  queue/
  shared/
```

MVP backend 필수 요소:

- GitHub provider adapter
- provider connection 관리
- repository discovery
- issue sync job
- pagination/rate-limit 처리
- normalized issue model
- issue upsert
- issue list API
- issue detail API
- sync status API

## Architecture Rules

이 프로젝트는 `AGENTS.md`의 규칙을 따른다.

필수:

- Layered Architecture
- Clean Architecture
- Tidy First
- TDD
- provider adapter boundary
- domain과 infrastructure 분리
- 환경값의 composition root 수용
- 명시적 인자/의존성 전달
- token/secret 로그 금지

금지:

- domain layer에서 provider API 직접 호출
- use case에서 `process.env` 직접 접근
- runtime 중간에 환경값 삽입 또는 변경
- provider token을 global singleton에 저장
- UI에서 provider API 직접 호출
- MVP 범위에 write-back 기능을 몰래 추가

## 설정 원칙

외부 설정 파일은 최소화합니다.

허용되는 설정:

- build/test tool 설정
- DB migration
- Docker Compose 또는 deployment manifest
- OAuth app 등록에 필요한 문서
- 테스트 fixture

환경값은 프로세스 시작 시점에만 읽습니다.

허용:

```ts
const runtimeConfig = loadRuntimeConfig(process.env);
const app = createApp({ config: runtimeConfig, adapters, db, queue, logger });
```

금지:

```ts
process.env.GITHUB_TOKEN = token;
process.env.LOG_LEVEL = "debug";
```

사용자별 provider token, repository별 sync option, tenant별 권한, sync cursor는 환경변수가 아니라 DB record, command input, job payload, 명시적 dependency로 전달합니다.

## 로그 정책

로그 profile은 3가지입니다.

| Profile | 목적 | 사용 환경 |
|---|---|---|
| `product` | 프로덕트용 최소 로그 | production/self-host 기본 |
| `debug` | 현장 확인용 디버그 로그 | 고객 현장 장애 분석 |
| `dev` | 개발 및 테스트 확인용 로그 | local/test |

공통 금지:

- token
- secret
- OAuth code
- Authorization header
- issue body/comment 전문
- raw payload 전체
- 개인정보

`log profile`은 프로세스 시작 시점에 결정하며, 실행 중 환경변수 변경으로 바꾸지 않습니다.

## Self-host 우선 전략

초기 제품은 self-host를 기준으로 설계합니다.

Self-host MVP 기준:

- 단일 서버 또는 Docker Compose로 실행 가능
- local PostgreSQL 사용 가능
- GitHub token 또는 GitHub App 연결 가능
- 외부 SaaS 의존 최소화
- private repository issue가 외부로 나가지 않도록 설계
- log/profile/config 정책이 명확해야 함

SaaS는 다음 조건 이후 검토합니다.

- self-host MVP 안정화
- token encryption과 tenant isolation 검증
- sync job, rate limit, retry 안정화
- audit log와 retention policy 기반 확보
- GitHub 외 provider 확장성 검증

## Provider Roadmap

| Phase | Provider | 범위 |
|---|---|---|
| MVP | GitHub | read-only issue sync/list/detail |
| V1 | GitHub | comments/events, saved views, webhook 검토 |
| V1/V2 | GitLab | read-only issue sync |
| V1/V2 | Gitea/Forgejo | self-host forge read-only sync |
| V2 | GitHub | write-back |
| V3 | Jira/Linear/Plane/OpenProject | export/link/sync |
| Later | Bitbucket, Azure DevOps, SourceHut | 확장 provider |

## 개발 시작 전 결정해야 할 것

- 백엔드 언어/프레임워크
- DB migration 도구
- job queue 방식
- GitHub 인증 방식: GitHub App 우선인지 PAT MVP인지
- raw payload 보존 기간
- token 암호화 방식
- local self-host 배포 방식
- issue list API의 filter/query 형식
- 초기 UI 디자인 시스템
- Kanban 도입 phase

## 문서

- [PROJECT.md](./PROJECT.md): 프로젝트 범위, 기능, provider, 데이터 모델, 로드맵
- [AGENTS.md](./AGENTS.md): 구현 규칙, 아키텍처, TDD, 설정/로그 원칙
- [RESEARCH.md](./RESEARCH.md): OSS 시장 조사, 경쟁 제품, 가격/기능 분석

## 참고

- Vue 공식 문서: <https://vuejs.org/guide/introduction.html>
- TanStack Table Vue 문서: <https://tanstack.com/table/latest/docs/framework/vue>
- TanStack Table v8 소개: <https://tanstack.com/table/v8/docs/introduction>
- SortableJS GitHub: <https://github.com/SortableJS>
- SortableJS 데모/문서: <https://sortablejs.github.io/Sortable/>
