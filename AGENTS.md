# AGENTS.md

이 문서는 Git Issue Sentinel 프로젝트에서 작업하는 모든 에이전트와 개발자가 반드시 따라야 할 구현 규칙이다. `PROJECT.md`와 `RESEARCH.md`를 바탕으로 하며, 실제 코드 작성 시 이 파일의 규칙을 우선 적용한다.

## 1. 최상위 원칙

이 프로젝트는 여러 Git provider의 issue/work item/ticket/change를 통합 수집하고 정규화하여, MVP에서는 read-only 리스트를 제공하고 최종적으로는 양방향 관리, 외부 프로젝트 연계, AI/자동화까지 확장하는 제품이다.

모든 구현은 다음 원칙을 기반으로 한다.

1. **Layered Architecture, Clean Architecture, Tidy First, TDD를 반드시 적용한다.**
2. **외부 파일에 설정되는 값은 최소화한다.**
3. **프로세스 실행 중 환경 설정을 삽입하거나 변경하는 방식은 반드시 거부한다.**
4. **외부 환경 상수는 최초 구성 시점에만 받아들이고, 이후에는 프로그램 전역 상수가 아니라 명시적 인자 또는 의존성으로 전달한다.**
5. **로그는 제품용 최소, 현장 확인용 디버그, 개발/테스트용 개발 로그 3가지 수준으로 나눈다.**

## 2. 프로젝트 목표 기준

## 2.1 MVP 기준

MVP는 “여러 provider의 issue를 가져와 하나의 리스트로 보는 read-only issue inbox”이다.

MVP 구현 시 반드시 지킬 것:

- GitHub, GitLab, Gitea/Forgejo를 우선 provider로 본다.
- Bitbucket Cloud, Azure DevOps, SourceHut은 확장 provider로 설계한다.
- 이슈 생성, 수정, 삭제, 댓글 작성, 상태 변경, 라벨 변경, 양방향 동기화는 MVP 범위가 아니다.
- MVP는 Full sync + Incremental polling 기반으로 시작한다.
- Webhook, write-back, AI, 외부 프로젝트 양방향 연계는 MVP 이후 단계로 둔다.
- 모든 원본 issue는 원본 provider ID, repository/project ID, URL, raw payload를 보존해야 한다.
- 동기화를 반복해도 중복 issue가 생기면 안 된다.

## 2.2 최종 목표 기준

최종 제품은 multi-provider issue management hub이다.

최종 목표에 포함되는 것:

- provider issue source 통합
- 통합 issue inbox
- provider write-back
- 양방향 동기화
- Jira, Linear, Plane, OpenProject, Redmine 연계
- Slack/Teams/email 알림
- AI triage, 중복 탐지, 라벨/담당자/우선순위 추천
- 자동화 규칙
- SSO/RBAC/audit log/self-host/compliance

최종 목표를 고려하되, 현재 phase의 범위를 넘는 구현을 미리 넣지 않는다. 확장 가능성은 port/interface/capability로 남기고, 실제 기능은 phase에 맞게 작게 구현한다.

## 3. 아키텍처 규칙

## 3.1 Layered Architecture

권장 레이어:

```text
Interface Layer
  Web UI, API controllers, CLI, Webhook receiver

Application Layer
  Use cases, commands, queries, transaction orchestration

Domain Layer
  Entities, value objects, domain services, policies, state mapping

Port Layer
  Repository ports, provider ports, queue ports, clock/id/logger ports

Adapter Layer
  GitHub/GitLab/Gitea/Forgejo adapters, DB repositories, HTTP clients, queue implementations

Infrastructure Layer
  Process startup, dependency composition, config loading, database, job worker runtime
```

레이어 의존 방향:

- Interface는 Application을 호출한다.
- Application은 Domain과 Port에 의존한다.
- Domain은 외부 framework, DB, HTTP, provider SDK에 의존하지 않는다.
- Adapter는 Port를 구현한다.
- Infrastructure는 의존성을 조립한다.
- 내부 레이어가 외부 레이어를 import하면 안 된다.

## 3.2 Clean Architecture

핵심 규칙:

- business rule은 provider API shape에 종속되면 안 된다.
- provider별 차이는 adapter와 normalizer에서 끝나야 한다.
- core use case는 GitHub/GitLab/Gitea 같은 구체 provider class를 직접 알아서는 안 된다.
- DB schema는 domain model을 오염시키지 않는다.
- UI는 raw provider payload에 직접 의존하지 않는다.
- provider capability는 명시적으로 조회한다.
- read/write 가능 여부는 runtime 조건문이 아니라 capability와 permission policy로 판정한다.

금지:

- domain layer에서 `fetch`, HTTP client, ORM, SQL client 직접 사용
- use case에서 `process.env` 직접 사용
- UI component에서 provider API 직접 호출
- provider adapter에서 UI용 label/color formatting 처리
- DB repository에서 provider pagination/rate limit 처리

## 3.3 Tidy First

구조 변경과 동작 변경을 분리한다.

작업 순서:

1. 코드가 읽히지 않거나 테스트가 어려우면 먼저 작은 구조 정리를 한다.
2. 구조 정리는 observable behavior를 바꾸지 않는다.
3. 그 다음 기능 변경을 한다.
4. 구조 변경과 기능 변경을 같은 패치에 섞지 않는다. 불가피하면 변경 이유를 명확히 남긴다.

Tidy 작업 예시:

- 이름 정리
- 중복 제거
- 작은 함수 추출
- adapter interface 정리
- test fixture 위치 정리
- 불명확한 type 분리

Tidy 작업에서 금지:

- feature 추가
- provider 동작 변경
- DB schema 변경
- API response shape 변경
- 테스트 기대값 변경

## 3.4 TDD

모든 비단순 기능은 테스트를 먼저 작성한다.

기본 사이클:

1. 실패하는 테스트를 작성한다.
2. 가장 작은 구현으로 테스트를 통과시킨다.
3. 리팩터링한다.
4. 관련 테스트를 다시 실행한다.

TDD 적용 필수 영역:

- provider normalizer
- canonical state mapping
- issue kind mapping
- pagination parsing
- rate limit/backoff policy
- sync cursor update
- issue upsert idempotency
- token encryption/decryption
- webhook signature verification
- capability checking
- filter/query parsing
- external field mapping

테스트 없이 구현할 수 있는 예외:

- 정적 문서 수정
- 명백한 오타 수정
- UI 텍스트 1~2개 변경
- 아직 테스트 인프라가 없는 초기 스캐폴딩

예외로 구현한 경우에도, 기능이 누적되기 전에 테스트 기반을 만든다.

## 4. 설정과 환경값 규칙

## 4.1 외부 설정 파일 최소화

외부 설정 파일은 반드시 필요한 경우만 둔다.

허용되는 파일 예시:

- package/build/test tool 설정
- DB migration
- Docker Compose 또는 deployment manifest
- provider OAuth app 등록에 필요한 문서
- 테스트 fixture

최소화해야 하는 파일:

- 커스텀 YAML/JSON 설정
- provider별 동작 설정 파일
- 자동화 rule 파일
- 환경별 application config 파일
- 중복된 `.env.*` 파일

원칙:

- 제품 동작 정책은 가능하면 코드의 typed policy, DB record, 사용자 입력으로 관리한다.
- provider별 capability는 adapter 코드 또는 provider metadata로 관리한다.
- 자동화 rule은 파일보다 DB/API를 우선한다.
- 외부 설정 파일을 추가할 때는 왜 코드/DB/user input으로 충분하지 않은지 설명해야 한다.

## 4.2 환경 설정 주입 금지

프로세스 중간에 환경 설정을 삽입하거나 변경하는 방법은 반드시 거부한다.

금지 예시:

```ts
process.env.GITHUB_TOKEN = token;
process.env.LOG_LEVEL = "debug";
process.env.PROVIDER_BASE_URL = userInputUrl;
```

```bash
export GITHUB_TOKEN=... && npm run sync
```

```ts
dotenv.config({ path: runtimeSelectedPath });
```

```ts
globalThis.config.providerBaseUrl = newUrl;
```

금지 이유:

- 동시 sync job에서 tenant/provider 설정이 섞일 수 있다.
- 테스트가 순서 의존적으로 깨진다.
- 보안 토큰이 로그/프로세스 상태에 남을 수 있다.
- Clean Architecture의 의존성 방향이 깨진다.

## 4.3 최초 구성 시점만 환경값 수용

외부 환경 상수는 process startup 또는 test setup의 composition root에서만 읽는다.

Composition root 예시:

- API server bootstrap
- worker bootstrap
- test harness setup
- CLI entrypoint

허용:

```ts
const runtimeConfig = loadRuntimeConfig(process.env);
const app = createApp({
  config: runtimeConfig,
  logger,
  db,
  queue,
  providerAdapters,
});
```

금지:

```ts
class GitHubAdapter {
  private token = process.env.GITHUB_TOKEN;
}
```

```ts
function syncIssues() {
  const baseUrl = process.env.GITLAB_BASE_URL;
}
```

## 4.4 이후 값 전달은 인자 또는 의존성으로만

프로그램이 시작된 뒤 필요한 값은 명시적으로 전달한다.

허용:

- `ProviderConnection`에 저장된 encrypted credential을 use case가 repository를 통해 불러온다.
- adapter 호출 시 `connection`, `repository`, `cursor`, `options`를 인자로 전달한다.
- user request에서 받은 filter를 query object로 전달한다.
- worker job payload에 `tenantId`, `connectionId`, `sourceId`를 전달한다.

금지:

- provider token을 전역 변수에 저장
- current tenant를 global context에 저장
- selected provider를 singleton mutable config에 저장
- log profile을 runtime 중 환경변수 변경으로 바꾸기

## 4.5 RuntimeConfig 규칙

`RuntimeConfig`는 프로세스 수준의 변하지 않는 구성만 담는다.

포함 가능:

- app base URL
- database connection information
- queue backend
- encryption key reference 또는 KMS reference
- OAuth client id/secret
- webhook public base URL
- default log profile
- allowed self-hosted domain policy
- retention default

포함 금지:

- 사용자별 provider token
- repository별 sync option
- tenant별 권한
- 자동화 rule
- 현재 실행 중인 job state
- provider별 cursor
- issue filter

## 5. 로그 규칙

## 5.1 로그 수준은 3가지 profile로 나눈다

이 프로젝트의 로그는 severity만으로 설계하지 않는다. 운영 목적에 따라 3가지 log profile을 둔다.

| Profile   | 목적              | 사용 환경                        | 기본 정책                                             |
| --------- | --------------- | ---------------------------- | ------------------------------------------------- |
| `product` | 프로덕트용 최소 로그     | SaaS/production/self-host 기본 | 최소 이벤트, 오류 중심, 민감정보 금지                            |
| `debug`   | 현장 확인용 디버그 로그   | 고객 현장 장애 분석, 제한 시간 진단        | provider request id, sync job, rate limit 등 진단 정보 |
| `dev`     | 개발 및 테스트 확인용 로그 | local dev, test, fixture 검증  | 상세 내부 흐름, fixture/raw shape 검증. 운영 금지             |

`log profile`은 프로세스 시작 시 결정한다. 실행 중 환경변수 변경으로 전환하지 않는다. 현장 확인이 필요하면 debug profile로 새 프로세스를 시작하거나, 인증된 운영 설정 변경 절차를 통해 다음 실행부터 적용한다.

## 5.2 Product 로그

허용:

- server started/stopped
- sync job started/completed/failed
- provider connection validation failed
- webhook verification failed
- DB migration failed
- security-relevant event
- fatal error

금지:

- provider token
- OAuth code/refresh token
- issue body/comment 전문
- raw payload
- private repository 이름을 무분별하게 출력
- user email 등 개인정보
- authorization header

Product 로그는 사용자가 실제 제품을 운영할 수 있는 최소 정보만 제공한다.

## 5.3 Debug 로그

현장 확인용 debug 로그는 문제 해결에 필요한 맥락을 제공한다.

허용:

- tenant id 또는 masked tenant id
- provider key
- connection id
- repository id/full name은 정책에 따라 masked 가능
- sync job id
- provider request id
- endpoint category
- pagination cursor summary
- rate limit remaining/reset
- retry count
- error code
- elapsed time

금지:

- token/secret/password
- authorization header
- issue body/comment 전문
- raw payload 전체
- webhook secret
- unmasked customer personal data

Debug 로그는 민감정보 없이 장애를 재현하고 원인을 좁히기 위한 로그다.

## 5.4 Dev 로그

Dev 로그는 개발/테스트 중에만 사용한다.

허용:

- normalizer input/output shape
- fixture name
- test job flow
- provider raw payload 일부
- query plan/slow query detail

조건:

- local/test 환경에서만 활성화한다.
- 운영 배포 artifact에서 dev profile이 기본값이면 안 된다.
- raw payload를 출력해야 하면 fixture 또는 sanitize된 payload를 우선한다.
- dev 로그에도 token/secret은 절대 출력하지 않는다.

## 5.5 Logger 설계

Logger는 port로 둔다.

```ts
interface Logger {
  product(event: ProductLogEvent): void;
  debug(event: DebugLogEvent): void;
  dev(event: DevLogEvent): void;
}
```

규칙:

- domain layer는 logger에 직접 의존하지 않는 것을 우선한다.
- application layer에서 필요한 경우 logger port를 주입한다.
- adapter는 provider 요청/응답 진단을 logger port로 남긴다.
- logger 구현체가 profile에 따라 출력 여부를 결정한다.
- 로그 이벤트는 문자열보다 구조화된 object를 우선한다.
- 로그에 포함되는 값은 작성 시점에 sanitize한다.

## 6. Provider Adapter 규칙

## 6.1 Adapter contract

Provider 호출은 반드시 adapter를 통해 수행한다.

Adapter 책임:

- 인증 방식 차이 흡수
- endpoint URL 구성
- pagination 처리
- rate limit/backoff 처리
- provider-specific field 변환
- raw payload 보존
- webhook signature 검증
- capability reporting
- read/write 권한 검증

Core 책임:

- 공통 issue 저장
- 검색/필터/뷰
- sync job scheduling
- conflict policy
- tenant/user/team 권한
- audit log

## 6.2 Provider별 구현 규칙

GitHub:

- PR이 issue endpoint에 섞이는 점을 반드시 테스트한다.
- MVP 기본값은 PR 제외다.
- GitHub App 또는 OAuth/PAT 선택을 capability와 connection type으로 표현한다.
- secondary rate limit을 별도 오류로 다룬다.

GitLab:

- project issue와 group issue를 구분한다.
- `id`와 `iid`를 혼동하지 않는다.
- self-managed version 차이를 capability로 표현한다.

Gitea/Forgejo:

- Gitea-compatible adapter를 공유하되 provider profile로 분기한다.
- base URL은 connection 인자로 전달한다.
- instance version/capability 확인을 별도 단계로 둔다.

Bitbucket:

- issue tracker API의 장기 지원 리스크를 숨기지 않는다.
- Jira 연계 경로를 최종 목표에 포함한다.

Azure DevOps:

- issue가 아니라 work item임을 모델에 반영한다.
- Work Item Type과 process template을 provider_extra에 보존한다.
- WIQL과 batch fetch 제한을 테스트한다.

SourceHut:

- GraphQL cursor pagination을 adapter가 책임진다.
- Git service와 todo service가 분리된 점을 모델에 반영한다.

Gerrit:

- MVP 대상이 아니다.
- 최종에서는 issue가 아니라 `change` kind로 제한한다.

Redmine:

- MVP 대상이 아니다.
- 최종에서는 legacy issue tracker adapter로 둔다.

## 6.3 Normalization 규칙

정규화는 손실을 최소화해야 한다.

필수:

- 원본 provider ID 보존
- 원본 URL 보존
- repository/project/provider context 보존
- canonical state와 provider state 동시 보존
- canonical issue kind와 provider-specific type 동시 보존
- typed column과 raw payload 동시 저장
- 누락 필드는 실패가 아니라 unknown/null로 처리

금지:

- provider state를 canonical state로 덮어쓰기
- provider issue number를 global unique key로 사용
- raw payload를 버리기
- Azure DevOps work item을 무조건 issue로 단순화하기
- PR/MR/change를 일반 issue와 구분 없이 저장하기

## 7. 동기화 규칙

## 7.1 MVP sync

MVP는 Full sync + Incremental polling이다.

필수:

- provider/repository별 cursor 저장
- pagination 끝까지 처리
- rate limit header 파싱
- `Retry-After` 존중
- exponential backoff + jitter
- partial failure 지원
- 동일 sync 재실행 시 idempotent upsert
- 403/404/삭제/권한 회수 soft handling

## 7.2 Webhook

Webhook은 V1 이후에 도입한다.

도입 시 필수:

- provider별 signature verification
- delivery id dedupe
- replay 방지
- payload size limit
- idempotent update
- webhook loop 방지 marker
- failure history UI

## 7.3 Write-back

Write-back은 V2 이후 별도 권한 승인을 받은 경우에만 구현한다.

필수:

- read-only connection과 write-enabled connection 구분
- provider capability 확인
- field-level conflict policy
- dry-run option
- audit log
- destructive action confirmation
- idempotency marker

## 8. 데이터와 보안 규칙

## 8.1 Token/Secret

필수:

- 평문 token 저장 금지
- envelope encryption 또는 KMS/Vault 사용
- token 마지막 일부만 표시
- audit log에 token 기록 금지
- token revoke/expiry 감지
- connection 삭제 시 token 폐기

금지:

- token을 env var로 runtime 주입
- token을 global singleton에 저장
- token을 logger에 전달
- raw HTTP request를 무조건 로그로 남기기

## 8.2 Multi-tenant

필수:

- 모든 query에 tenant scope 적용
- background job에도 tenant id 명시
- provider connection 접근 권한 확인
- shared connection 관리 권한 분리
- tenant간 issue/search/cache 격리
- tenant 삭제/retention 정책 준수

## 8.3 개인정보와 raw payload

Issue body/comment/raw payload에는 개인정보와 secret이 들어갈 수 있다.

규칙:

- raw payload 저장 여부와 보존 기간을 명시한다.
- 로그에는 raw payload를 출력하지 않는다.
- AI 기능에는 tenant별 opt-in이 필요하다.
- private repository data는 외부 AI API로 보내기 전에 정책 확인이 필요하다.
- webhook payload 로그는 반드시 sanitize한다.

## 9. 데이터 모델 규칙

## 9.1 Identity

중복 방지 키는 다음 조합을 기준으로 한다.

```text
tenant_id
+ provider_key
+ provider_connection_id
+ provider_repository_id
+ provider_issue_id
```

주의:

- GitHub issue number는 repository 내에서만 unique하다.
- GitLab `iid`는 project 내에서만 unique하다.
- Azure DevOps work item은 organization/project context를 함께 저장한다.
- SourceHut ticket은 tracker context를 함께 저장한다.

## 9.2 Raw payload

Raw payload는 provider 변경과 디버깅을 위해 저장하되, UI와 domain rule은 raw payload에 직접 의존하지 않는다.

허용:

- adapter contract test fixture 생성
- provider_extra 보완
- dev/test 진단

금지:

- UI table column을 raw payload path에서 직접 읽기
- use case가 raw payload를 파싱해 business decision하기
- raw payload를 product/debug 로그에 그대로 출력하기

## 10. 프론트엔드 규칙

MVP 첫 화면은 All Issues 리스트다.

필수:

- provider/source/repository 구분이 즉시 보여야 한다.
- sync 상태와 데이터 신선도를 노출한다.
- 일부 provider sync 실패 시 나머지 issue list는 계속 보여준다.
- table은 대량 issue에 맞게 pagination 또는 virtualization을 고려한다.
- issue 상세는 리스트 context를 유지하는 drawer/panel을 우선한다.
- 원본 provider URL을 제공한다.

금지:

- 마케팅 랜딩 페이지를 첫 화면으로 만들기
- UI에서 provider API 직접 호출
- 실패한 provider 때문에 전체 리스트를 비우기
- provider별 raw payload에 직접 의존하는 렌더링

## 11. 테스트 규칙

## 11.1 필수 테스트

다음 기능은 테스트 없이는 완료로 보지 않는다.

- normalizer
- canonical state mapping
- issue kind mapping
- pagination
- rate limit/backoff
- cursor update
- issue upsert idempotency
- token encryption
- webhook signature verification
- provider capability
- search/filter query
- permission failure handling

## 11.2 Fixture

Provider API는 fixture 기반 contract test를 가져야 한다.

필수 fixture:

- GitHub open issue
- GitHub closed issue
- GitHub PR mixed in issue list
- GitLab project issue
- GitLab group issue 또는 group issue 제외 확인 fixture
- Gitea issue
- Forgejo issue
- Bitbucket issue
- Azure DevOps work item
- SourceHut ticket
- malformed payload
- missing optional field
- pagination
- rate limit
- permission denied

## 11.3 E2E 수용 기준

MVP E2E는 최소 다음 흐름을 검증한다.

1. provider connection 생성
2. repository/source 선택
3. sync job 실행
4. issue list 표시
5. provider/state/label 검색 필터
6. issue detail 표시
7. 원본 URL 이동 가능
8. 같은 source 재동기화 시 중복 없음
9. 한 provider 실패 시 기존 리스트 유지

## 12. 작업 방식

## 12.1 구현 전 확인

작업 전 확인할 것:

- 현재 phase가 MVP/V1/V2/V3/최종 중 어디인지
- 변경이 read-only 범위를 넘는지
- provider adapter boundary를 지키는지
- 새로운 외부 설정 파일이 정말 필요한지
- 환경값을 runtime 중간에 주입하려는 구조가 아닌지
- 테스트를 먼저 작성할 수 있는지
- token/secret/logging 보안 이슈가 없는지

## 12.2 변경 중 금지 사항

- 관련 없는 리팩터링
- 대규모 구조 변경과 기능 변경 동시 수행
- 테스트 없이 provider 동작 변경
- runtime `process.env` 수정
- global mutable config 추가
- token/secret 로그 출력
- raw payload product/debug 로그 출력
- provider별 특수 처리를 core use case에 직접 삽입
- MVP에서 write-back 기능을 몰래 추가

## 12.3 완료 기준

완료 보고 전 확인:

- 관련 테스트 통과
- TDD 또는 테스트 보강 여부 명시
- 환경값/설정 파일 추가 여부 명시
- 로그 profile 규칙 준수
- provider adapter boundary 준수
- 민감정보 로그/저장 위험 점검
- MVP/최종 목표 중 어느 범위의 변경인지 명확함

## 13. 코드 스타일 방향

아직 구체적인 언어/프레임워크가 확정되지 않았으므로, 다음 원칙을 우선한다.

- type-safe한 domain model을 만든다.
- provider raw DTO와 domain model을 분리한다.
- use case input/output을 명시한다.
- null/unknown/provider-specific 값을 type으로 표현한다.
- side effect는 adapter/infrastructure에 가둔다.
- clock, id generator, logger, queue, provider client는 주입한다.
- 테스트 가능한 작은 함수와 class를 우선한다.
- 긴 조건문보다 policy/capability/state mapping table을 우선한다.

## 14. 의사결정 보류 항목

다음은 구현 전에 명시적으로 결정해야 한다.

- 제품명
- 언어/프레임워크
- SaaS 우선인지 self-host 우선인지
- MVP provider 범위
- OAuth/App/PAT 우선순위
- PR/MR을 MVP에 포함할지
- closed issue 기본 수집 여부
- comments lazy load 여부
- raw payload 보존 기간
- token 암호화 방식
- 검색 엔진
- job queue
- webhook 도입 시점
- write-back 도입 시점
- 외부 연계 1순위
- AI 기능 제공 방식
- 라이선스/오픈소스 전략

결정되지 않은 항목은 임의로 넓게 구현하지 않는다. 작은 기본값을 선택하고, adapter/capability/interface로 확장 여지를 남긴다.
