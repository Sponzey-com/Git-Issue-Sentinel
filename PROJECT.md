# Git Issue Sentinel 프로젝트 기획 및 구현 체크리스트

작성일: 2026-06-08  
목표: 가능한 모든 Git 호스팅/forge/issue tracker의 이슈를 한곳에 가져와서 보고, 검색하고, 분류하고, 최종적으로 관리 및 타 프로젝트 관리 도구와 연계하는 제품을 구현한다.

## 1. 프로젝트 정의

## 1.1 한 줄 정의

**Git Issue Sentinel**은 GitHub, GitLab, Gitea, Forgejo, Bitbucket, Azure DevOps, SourceHut 등 여러 Git 기반 협업 시스템의 이슈, 티켓, work item, pull request 관련 작업 항목을 통합 수집하고 정규화하여 하나의 inbox/list/dashboard에서 보고 관리하는 issue intelligence 플랫폼이다.

## 1.2 문제 정의

개발 조직은 저장소와 프로젝트가 늘어날수록 이슈가 여러 시스템에 흩어진다.

- GitHub에는 public OSS issue와 private product issue가 섞여 있다.
- GitLab에는 group/project issue, MR, epic, milestone이 따로 존재한다.
- Gitea/Forgejo는 자체 호스팅 인스턴스마다 URL, 버전, API 권한, 검색 기능이 다르다.
- Bitbucket Cloud는 repository issue tracker가 있으나 Jira와 결합되는 경우가 많고, 일부 issue tracker API는 deprecated 표시가 있다.
- Azure DevOps는 “issue”가 아니라 work item이며, Bug/User Story/Task/Epic 등 process template마다 필드와 상태가 다르다.
- SourceHut은 todo.sr.ht ticket/tracker 모델을 쓴다.
- Gerrit은 일반 이슈 트래커가 아니라 change 중심 code review 시스템이다.
- Redmine, Jira, Linear, Plane, OpenProject 같은 프로젝트 관리 도구는 Git 이슈와 연결되거나 대체 tracker로 쓰인다.

따라서 단순히 “Git issue API 하나”를 붙이는 방식으로는 모든 시스템을 다룰 수 없다. 핵심은 **provider별 차이를 adapter로 감추고, 내부에서는 정규화된 공통 issue 모델로 다루는 것**이다.

## 1.3 제품의 단계별 목표

| 단계  | 목표                                                                     | 제품 형태                                      |
| --- | ---------------------------------------------------------------------- | ------------------------------------------ |
| MVP | 연결된 모든 provider의 모든 issue를 읽어와 단일 리스트로 보여준다.                           | read-only issue inbox                      |
| V1  | 검색, 필터, 저장된 뷰, 상세 보기, 댓글/이력 조회, webhook 기반 동기화를 제공한다.                  | issue observability dashboard              |
| V2  | 라벨/담당자/상태/댓글/닫기/재오픈 등 관리 작업을 provider로 다시 반영한다.                        | multi-provider issue management            |
| V3  | Jira, Linear, Plane, OpenProject, Slack, Notion, 이메일, 고객 피드백 도구와 연계한다. | issue operations hub                       |
| 최종  | AI triage, 중복 탐지, 우선순위 추천, 릴리스/고객/SLA 연결, 자동화 워크플로우를 제공한다.             | issue intelligence and automation platform |

## 2. 범위 명확화

## 2.1 “Git 종류”의 실제 의미

Git 자체는 분산 버전 관리 시스템이며 issue 기능을 제공하지 않는다. 이슈는 GitHub/GitLab/Gitea/Forgejo/Bitbucket/Azure DevOps/SourceHut 같은 Git hosting 또는 forge 제품이 제공하는 협업 기능이다. 따라서 본 프로젝트의 “모든 Git 종류 Issue”는 다음처럼 해석한다.

1. Git 저장소와 같은 제품 안에 있는 issue tracker
2. Git 저장소와 강하게 연결되는 work item/ticket tracker
3. Git commit, branch, PR/MR, release와 참조 관계를 갖는 외부 프로젝트 관리 도구
4. self-hosted Git forge 인스턴스의 issue API
5. provider별 pull request/merge request/change를 issue와 함께 관찰해야 하는 경우

## 2.2 포함 대상

| 분류                           | 포함 여부             | 예시                                                                           |
| ---------------------------- | ----------------- | ---------------------------------------------------------------------------- |
| Git forge issue              | 포함                | GitHub Issues, GitLab Issues, Gitea Issues, Forgejo Issues, Bitbucket Issues |
| Work item/ticket             | 포함                | Azure DevOps Work Items, SourceHut todo.sr.ht Tickets                        |
| Pull request / merge request | MVP에서는 선택, 최종 포함  | GitHub PR, GitLab MR, Gitea/Forgejo PR, Bitbucket PR                         |
| Code review change           | 별도 adapter로 제한 포함 | Gerrit Changes                                                               |
| 외부 PM issue                  | 연계 대상으로 포함        | Jira, Linear, Plane, OpenProject, Redmine                                    |
| Git bare repository          | 직접 issue 없음       | commit message issue reference만 추출 가능                                        |
| Discussions/Q&A              | 최종 확장             | GitHub Discussions, GitLab Discussions/epics 등                               |

## 2.3 제외 또는 후순위 대상

| 대상                       | 판단                                   |
| ------------------------ | ------------------------------------ |
| 순수 Git repository만 있는 서버 | Git protocol에는 issue가 없으므로 MVP 대상 아님 |
| 이메일만 있는 버그 리포트           | 최종 intake 기능에서 처리                    |
| Slack/Discord 메시지        | 최종 외부 피드백 source로 처리                 |
| 고객 지원 티켓                 | Zendesk/Intercom/Freshdesk 연계는 V3 이후 |
| 보안 취약점 private advisory  | 권한/보안 민감도가 높아 별도 보안 설계 후 포함          |

## 3. MVP 정의

## 3.1 MVP 한 줄 목표

**MVP는 여러 Git provider 계정을 연결하고, 선택한 저장소/프로젝트의 모든 이슈를 읽어와 정규화한 뒤, 하나의 리스트 화면에서 검색/필터링 가능한 read-only 목록으로 보여주는 것이다.**

## 3.2 MVP에서 반드시 되는 것

1. Provider 연결
   - GitHub.com
   - GitLab.com
   - self-hosted GitLab
   - Gitea
   - Forgejo/Codeberg
   - Bitbucket Cloud
   - Azure DevOps
   - SourceHut todo.sr.ht
   - provider별로 구현 난이도에 따라 feature flag 가능
2. Credential 등록
   - OAuth/App 기반 연결을 우선한다.
   - MVP에서는 Personal Access Token/API Token 방식도 허용한다.
   - self-hosted provider는 base URL을 직접 입력할 수 있어야 한다.
   - 토큰은 암호화 저장한다.
   - token scope 검증 결과를 사용자에게 보여준다.
3. 저장소/프로젝트 발견
   - 사용자가 접근 가능한 org/group/workspace/project/repository 목록을 가져온다.
   - public/private 구분을 표시한다.
   - issue tracker가 꺼져 있거나 권한이 없는 저장소는 reason을 표시한다.
   - 동기화 대상 저장소를 선택할 수 있다.
4. 이슈 초기 수집
   - open/closed/all 상태를 가져온다.
   - title, body/description, state, author, assignee, labels, milestone, created_at, updated_at, closed_at, comments_count, url을 가져온다.
   - provider 원본 ID와 URL을 보존한다.
   - provider별 pagination을 끝까지 처리한다.
   - rate limit에 걸리면 재시도/대기/부분 완료 상태를 기록한다.
5. 정규화 저장
   - 모든 provider issue를 내부 `Issue` 모델로 변환한다.
   - 원본 payload는 JSON으로 별도 저장한다.
   - provider별 손실 정보는 `provider_extra`에 보존한다.
   - 같은 provider issue를 재수집해도 중복 생성하지 않는다.
6. 단일 리스트 보기
   - 모든 provider의 issue를 한 테이블에 표시한다.
   - provider, org/group, repository/project, number/key, title, status, labels, assignee, updated_at을 표시한다.
   - provider별 아이콘/색상으로 구분한다.
   - 기본 정렬은 `updated_at desc`.
   - 제목/본문/라벨/저장소명 검색을 제공한다.
   - provider, repository, state, assignee, label, updated range 필터를 제공한다.
7. 상세 보기
   - 제목, 본문, 원본 링크, 작성자, 담당자, 라벨, 마일스톤, 상태, 생성/수정/종료 시각 표시
   - 원본 provider로 열기 버튼
   - MVP에서는 댓글과 이벤트 이력은 선택 기능으로 둔다.
8. 동기화 상태 표시
   - integration별 마지막 동기화 시각
   - 성공/실패/부분 성공
   - 가져온 issue 수
   - rate limit 상태
   - 인증 오류/권한 오류/네트워크 오류

## 3.3 MVP에서 하지 않는 것

MVP의 범위를 명확히 줄이기 위해 다음은 제외한다.

- 이슈 생성/수정/삭제
- 라벨/담당자/상태 변경
- 댓글 작성
- 양방향 동기화
- 자동 중복 병합
- AI triage
- 프로젝트 보드/로드맵
- Jira/Linear/Plane/OpenProject로 내보내기
- 브라우저 확장
- 모바일 앱
- 완전한 RBAC/감사 로그

## 3.4 MVP 성공 기준

| 기준   | 성공 조건                                                                                  |
| ---- | -------------------------------------------------------------------------------------- |
| 연결성  | 최소 GitHub, GitLab, Gitea/Forgejo, Bitbucket Cloud, Azure DevOps 중 3개 이상 provider 연결 가능 |
| 수집성  | 각 provider에서 1개 이상 repository/project의 open/closed issue를 끝까지 수집                       |
| 정규화  | 서로 다른 provider issue가 하나의 리스트에서 동일한 컬럼으로 표시                                            |
| 검색성  | 제목, 저장소, 라벨, 상태 기준으로 1초 이내 필터 반응                                                       |
| 안정성  | 동기화 중 일부 provider가 실패해도 전체 앱은 동작                                                       |
| 추적성  | 모든 issue가 원본 URL과 원본 ID로 추적 가능                                                         |
| 재실행성 | 동기화를 반복해도 중복 issue가 생기지 않음                                                             |

## 4. 최종 목표 정의

## 4.1 최종 제품 한 줄 목표

**최종 제품은 모든 Git 기반 issue source를 단일 운영 계층으로 통합하고, 읽기/쓰기/동기화/분석/자동화/외부 프로젝트 관리 도구 연계를 제공하는 multi-provider issue management hub가 된다.**

## 4.2 최종 제품 핵심 기능

1. 모든 issue source 통합
   - GitHub/GitLab/Gitea/Forgejo/Bitbucket/Azure DevOps/SourceHut
   - self-hosted enterprise instance
   - Jira/Linear/Plane/OpenProject/Redmine 같은 외부 tracker
   - OSS public repository monitoring
2. 통합 issue inbox
   - cross-provider unread/updated queue
   - stale issue queue
   - unassigned issue queue
   - customer-impact issue queue
   - release-blocker queue
   - security issue queue
3. 통합 관리 작업
   - assign/unassign
   - label add/remove/set
   - status transition
   - close/reopen
   - comment/reply
   - milestone/version/cycle 연결
   - link/unlink related issue
   - duplicate marking
   - issue transfer/export
4. 양방향 동기화
   - 원본 provider 변경 사항을 webhook/polling으로 반영
   - Sentinel에서 한 변경 사항을 원본 provider에 write-back
   - 충돌 감지와 해결
   - 필드 매핑 정책
   - dry-run write
   - sync audit log
5. 타 프로젝트 관리 도구 연계
   - Jira issue 생성/연결/상태 전이
   - Linear issue 생성/연결/상태 전이
   - Plane work item 생성/연결
   - OpenProject work package 생성/연결
   - Redmine issue 생성/연결
   - Notion/Confluence 문서 연결
   - Slack/Teams 알림
   - 이메일 digest
   - GitHub Projects/GitLab Boards와의 부분 연계
6. AI 기능
   - issue 요약
   - 중복 issue 후보 추천
   - label 추천
   - 담당자 추천
   - 우선순위/심각도 추천
   - 재현 정보 누락 감지
   - 관련 PR/MR/commit/file 추천
   - triage 댓글 초안
   - release note 후보 추출
   - stale issue 정리 제안
7. 보고/분석
   - provider별 open issue 추세
   - repository별 backlog aging
   - label별 처리 시간
   - assignee workload
   - SLA breach
   - duplicate rate
   - stale rate
   - triage latency
   - close rate
   - release blocker list
8. 엔터프라이즈
   - SSO/SAML/OIDC
   - RBAC
   - audit log
   - encrypted secrets
   - private network/self-hosted deployment
   - tenant isolation
   - SCIM
   - backup/restore
   - data retention
   - compliance export

## 5. Provider 지원 범위

## 5.1 Provider 우선순위

| 우선순위 | Provider                         | 이유                                       |
| ---- | -------------------------------- | ---------------------------------------- |
| P0   | GitHub.com                       | 최대 시장, REST/GraphQL/webhook 성숙           |
| P0   | GitLab.com / GitLab Self-Managed | 그룹/프로젝트 issue, enterprise 수요             |
| P0   | Gitea / Forgejo                  | self-hosted OSS forge 핵심                 |
| P1   | Bitbucket Cloud                  | Atlassian 생태계, Jira 연계 수요                |
| P1   | Azure DevOps                     | enterprise work item 수요                  |
| P1   | SourceHut todo.sr.ht             | OSS/해커 문화권 Git hosting                   |
| P2   | Gerrit                           | issue는 아니지만 change review 관찰 수요          |
| P2   | Redmine                          | Git repository browser와 issue tracker 결합 |
| P2   | Jira/Linear/Plane/OpenProject    | 최종 write/export/sync target              |

## 5.2 Provider별 확인 사항

| Provider                 | 원본 객체                         | API 방식                                | MVP 지원 | 최종 지원 | 주요 리스크                                                 |
| ------------------------ | ----------------------------- | ------------------------------------- | ------ | ----- | ------------------------------------------------------ |
| GitHub                   | Issue, PR, Discussion         | REST, GraphQL, Webhook                | 예      | 예     | PR도 issue API에 섞임. GitHub App 권장. secondary rate limit |
| GitHub Enterprise Server | Issue, PR                     | REST, GraphQL, Webhook                | 예      | 예     | 버전별 API 차이, 사내망 접근                                     |
| GitLab                   | Project/Group Issue, MR, Epic | REST, GraphQL 일부, Webhook/System Hook | 예      | 예     | self-managed 버전 차이, group issue/epic tier 차이           |
| Gitea                    | Issue, PR                     | REST/OpenAPI, Webhook                 | 예      | 예     | 인스턴스별 버전/설정 차이, MAX_RESPONSE_ITEMS                     |
| Forgejo/Codeberg         | Issue, PR                     | Gitea-compatible REST, Webhook        | 예      | 예     | Gitea와의 API divergence 가능성                             |
| Bitbucket Cloud          | Issue, PR                     | REST, Webhook                         | 예      | 예     | issue tracker API deprecated 표시, Jira 중심 조직 많음         |
| Azure DevOps             | Work Item                     | REST, WIQL, Service Hooks             | 예      | 예     | issue가 아니라 process-specific work item                  |
| SourceHut                | Ticket                        | GraphQL, Webhook                      | 선택     | 예     | GraphQL cursor, tracker/ticket 권한 모델                   |
| Gerrit                   | Change                        | REST                                  | 아니오    | 제한    | issue가 아니라 code review change                          |
| Redmine                  | Issue                         | REST, Webhook 플러그인 가능                 | 아니오    | 예     | 플러그인/버전별 기능 차이                                         |

## 5.3 공식 문서 기준 핵심 API 메모

- GitHub REST Issues API는 issue, assignee, comment, label, milestone, issue event, dependency, sub-issue 관련 endpoint를 제공한다. 참고: <https://docs.github.com/en/rest/issues>
- GitHub REST API는 primary/secondary rate limit이 있고, secondary limit에는 동시 요청, endpoint별 요청, content-generating 요청 제한이 포함된다. 참고: <https://docs.github.com/rest/overview/rate-limits-for-the-rest-api>
- GitLab Issues API는 project/group issue 생성, 수정, 삭제, assignee, label, milestone, time tracking, cross-reference를 지원한다. 참고: <https://docs.gitlab.com/api/issues/>
- GitLab Webhooks는 issue 변경, comment, merge request 등 이벤트를 외부 애플리케이션으로 보낼 수 있다. 참고: <https://docs.gitlab.com/user/project/integrations/webhooks/>
- Gitea API는 Swagger/OpenAPI 기반이며 token, OAuth2, pagination, `x-total-count`를 제공한다. 참고: <https://docs.gitea.com/development/api-usage/>
- Forgejo는 issue/PR 검색에서 `is:open`, `is:closed`, `author:*`, `assignee:*`, `sort:*`, `modified:*` 같은 검색 문법을 제공한다. 참고: <https://forgejo.org/docs/latest/user/issue-search/>
- Bitbucket Cloud issue tracker REST API는 repository issue list/create/comment/component/milestone/version endpoint를 제공하지만 문서상 여러 issue tracker endpoint에 deprecated 표시가 있다. 참고: <https://developer.atlassian.com/cloud/bitbucket/rest/api-group-issue-tracker/>
- Bitbucket Cloud webhook은 issue event payload를 제공한다. 참고: <https://support.atlassian.com/bitbucket-cloud/docs/manage-webhooks/>
- Azure DevOps Work Items API는 work item list/create/update를 제공하며 batch get은 최대 200개 단위다. 참고: <https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items?view=azure-devops-rest-7.1>
- Azure DevOps Service Hooks는 work item created/updated/deleted/restored/commented events를 제공한다. 참고: <https://learn.microsoft.com/en-us/azure/devops/service-hooks/events?view=azure-devops>
- SourceHut todo.sr.ht는 GraphQL API로 tracker, tickets, labels, assignment, status, webhook mutation을 제공한다. 참고: <https://docs.sourcehut.org/todo.sr.ht/>
- Gerrit Changes REST API는 query changes, get change, abandon, restore, rebase, submit 등을 제공한다. 참고: <https://gerrit-review.googlesource.com/Documentation/rest-api-changes.html>
- Jira Cloud REST API는 issue create/get/edit/assign/transition/comment/search 등을 제공한다. 참고: <https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/>
- Linear는 GraphQL API와 webhook을 제공한다. 참고: <https://linear.app/docs/api-and-webhooks>
- OpenProject Work Packages API는 work package list/view/update/comment/relation/time log 등을 제공한다. 참고: <https://www.openproject.org/docs/api/endpoints/work-packages/>

## 6. Provider Adapter 설계

## 6.1 Adapter 원칙

모든 provider는 내부 core가 직접 호출하지 않고 adapter를 통해서만 호출한다.

Adapter가 책임지는 것:

- provider 인증 방식 차이 흡수
- endpoint URL 구성
- pagination 처리
- rate limit/backoff 처리
- provider-specific state/label/field 변환
- provider 원본 payload 보존
- webhook signature 검증
- capability reporting
- read/write 권한 검증

Core가 책임지는 것:

- 공통 issue 모델 저장
- 검색/필터/뷰
- sync job scheduling
- conflict policy
- user/team/tenant 권한
- UI/API 제공
- audit log

## 6.2 Adapter interface 초안

```ts
interface IssueProviderAdapter {
  providerKey: ProviderKey;
  capabilities(): ProviderCapabilities;

  validateConnection(input: ConnectionInput): Promise<ConnectionCheckResult>;
  refreshToken?(connection: ProviderConnection): Promise<ProviderConnection>;

  listNamespaces(connection: ProviderConnection): Promise<ProviderNamespace[]>;
  listRepositories(connection: ProviderConnection, namespaceId?: string): Promise<ProviderRepository[]>;

  listIssues(
    connection: ProviderConnection,
    repository: ProviderRepositoryRef,
    cursor?: SyncCursor,
    options?: ListIssueOptions
  ): Promise<PaginatedProviderIssues>;

  getIssue(
    connection: ProviderConnection,
    issueRef: ProviderIssueRef
  ): Promise<ProviderIssue>;

  listComments?(
    connection: ProviderConnection,
    issueRef: ProviderIssueRef,
    cursor?: SyncCursor
  ): Promise<PaginatedProviderComments>;

  listEvents?(
    connection: ProviderConnection,
    issueRef: ProviderIssueRef,
    cursor?: SyncCursor
  ): Promise<PaginatedProviderEvents>;

  createWebhook?(
    connection: ProviderConnection,
    repository: ProviderRepositoryRef,
    targetUrl: string
  ): Promise<ProviderWebhook>;

  verifyWebhook?(
    request: WebhookRequest,
    secret: string
  ): Promise<WebhookVerificationResult>;

  normalizeIssue(raw: unknown, context: NormalizeContext): NormalizedIssue;
  normalizeComment?(raw: unknown, context: NormalizeContext): NormalizedComment;
  normalizeEvent?(raw: unknown, context: NormalizeContext): NormalizedIssueEvent;

  createIssue?(
    connection: ProviderConnection,
    repository: ProviderRepositoryRef,
    input: CreateIssueInput
  ): Promise<ProviderIssue>;

  updateIssue?(
    connection: ProviderConnection,
    issueRef: ProviderIssueRef,
    patch: UpdateIssuePatch
  ): Promise<ProviderIssue>;

  addComment?(
    connection: ProviderConnection,
    issueRef: ProviderIssueRef,
    input: AddCommentInput
  ): Promise<ProviderComment>;

  transitionIssue?(
    connection: ProviderConnection,
    issueRef: ProviderIssueRef,
    transition: IssueTransitionInput
  ): Promise<ProviderIssue>;
}
```

## 6.3 Capability model

Provider마다 가능한 작업이 다르므로 기능별 capability를 명시해야 한다.

```ts
type ProviderCapabilities = {
  read: {
    repositories: boolean;
    issues: boolean;
    comments: boolean;
    events: boolean;
    labels: boolean;
    milestones: boolean;
    assignees: boolean;
    attachments: boolean;
    relations: boolean;
    pullRequests: boolean;
  };
  write: {
    createIssue: boolean;
    updateTitle: boolean;
    updateBody: boolean;
    closeIssue: boolean;
    reopenIssue: boolean;
    assign: boolean;
    label: boolean;
    milestone: boolean;
    comment: boolean;
    linkIssue: boolean;
    deleteIssue: boolean;
  };
  sync: {
    webhook: boolean;
    polling: boolean;
    deltaByUpdatedAt: boolean;
    etagOrConditionalRequest: boolean;
    cursorPagination: boolean;
    offsetPagination: boolean;
  };
  auth: {
    oauth: boolean;
    appInstallation: boolean;
    personalToken: boolean;
    apiToken: boolean;
    basicAuth: boolean;
    selfHostedBaseUrl: boolean;
  };
};
```

## 6.4 Provider별 adapter 특이사항

### GitHub adapter

확인할 것:

- GitHub App을 쓸지 OAuth App을 쓸지 결정
- PAT fallback 지원 여부
- GitHub.com과 GitHub Enterprise Server 모두 지원할지 결정
- REST와 GraphQL 중 어떤 것을 기본으로 할지 결정
- PR이 issue endpoint에 포함되는 특성 처리
- `pull_request` 필드가 있는 item은 issue list에서 숨길지 별도 type으로 표시할지 결정
- issue dependencies/sub-issues API 지원 범위 결정
- repo installation permissions: Issues read/write, Metadata read, Pull requests read 등
- webhook events: issues, issue_comment, label, milestone, pull_request, repository
- rate limit: primary limit, secondary limit, abuse detection, retry-after 처리

MVP:

- REST `List repository issues`
- `state=all`, `since` 기반 incremental sync
- `per_page=100`
- `Link` header pagination 처리
- PR 제외 옵션 기본값: 제외

최종:

- GitHub App 기반 multi-tenant 설치
- GraphQL search로 cross-repo issue 검색 최적화
- issue comment/event/timeline 수집
- labels/milestones/assignees write-back
- GitHub Projects v2 연계 검토

### GitLab adapter

확인할 것:

- GitLab.com, self-managed, Dedicated 지원
- group issue와 project issue 중 MVP 기준 결정
- self-managed version별 API 차이
- private project 404 처리
- group/subgroup 구조
- issue `iid`와 global `id` 차이
- labels가 문자열 배열로 오는 점
- assignee 단수/복수 필드 차이
- milestone, iteration, weight, health_status, time_stats 지원 여부
- webhook: issue event, note event, merge request event
- system hook은 self-managed admin 기능이므로 별도 enterprise feature로 분리

MVP:

- project issue list 우선
- `updated_after` 기반 incremental sync
- page/per_page pagination 처리
- group/project/repository discovery

최종:

- group issue 통합
- epic/iteration/board 연계
- state transition, labels, assignees, comments write-back
- GitLab MR과 issue cross-reference 수집

### Gitea adapter

확인할 것:

- Gitea version별 API 스펙
- self-hosted base URL 입력
- `/api/v1` endpoint
- Swagger/OpenAPI 노출 여부
- token scope: `read:issue`, `write:issue`, `read:repository`
- pagination: `page`, `limit`, `Link`, `x-total-count`
- server config `MAX_RESPONSE_ITEMS` 기본값/최대값
- issue와 PR가 같은 number space를 공유하는지 확인
- issue tracker disabled repository 처리
- webhook endpoint/secret 방식

MVP:

- `GET /api/v1/repos/{owner}/{repo}/issues`
- `state=all`
- PR 제외/포함 옵션
- token header `Authorization: token ...`

최종:

- issue 생성/수정/댓글/라벨/마일스톤 write
- webhook 자동 생성
- Gitea Enterprise/Gitea Cloud 연결 검토

### Forgejo adapter

확인할 것:

- Gitea-compatible API로 시작하되 Forgejo version별 divergence 추적
- Codeberg public instance 지원 여부
- Forgejo issue search syntax 활용 여부
- `is:open`, `is:closed`, `modified:>date`, `author:*`, `assignee:*` 검색 지원
- token scope와 OAuth2 provider 지원
- issue/PR linked references 처리

MVP:

- Gitea adapter를 확장해서 Forgejo profile로 동작
- Codeberg base URL preset 제공

최종:

- Forgejo-specific issue search 최적화
- federation/ActivityPub 관련 확장 가능성 조사

### Bitbucket Cloud adapter

확인할 것:

- Bitbucket Cloud issue tracker가 repository별로 활성화되어 있는지
- Atlassian API token/OAuth 2.0 scopes
- issue API 문서의 deprecated 표시가 제품 로드맵상 어떤 의미인지 확인
- Jira issue와 Bitbucket issue의 구분
- workspace/project/repository hierarchy
- issue fields: priority, type, component, version, milestone, state
- webhook issue events: created, updated, commented 등
- rate limit: anonymous 60/h, authenticated resource별 제한, scaled rate limit

MVP:

- issue tracker enabled repository만 수집
- repository issue list
- deprecated API 리스크를 UI/문서에 표시

최종:

- Jira integration을 사실상 Bitbucket의 primary issue management path로 지원
- Bitbucket PR/commit과 Jira issue linkage 수집

### Azure DevOps adapter

확인할 것:

- Azure DevOps는 issue가 아니라 Work Item Tracking 모델
- organization/project/team 구조
- Work Item Type: Bug, Task, User Story, Product Backlog Item, Feature, Epic 등
- process template: Agile, Scrum, CMMI, inherited process
- WIQL로 ID 검색 후 Work Items Batch API로 상세 조회
- batch size 제한
- Area Path, Iteration Path, State, Reason, Tags, AssignedTo
- identity fields 형식
- OAuth/Microsoft Entra ID/PAT 인증 전략
- Service Hooks: workitem.created, workitem.updated 등
- Azure DevOps Server 지원 여부

MVP:

- project별 WIQL로 최근 work items 조회
- Bug/User Story/Task/Epic 모두 `issue_kind`로 정규화
- Work Item Type 필터 제공

최종:

- state transition mapping
- field metadata introspection
- custom process field sync
- GitHub/Azure Boards 연동 관계 수집
- Azure Repos PR와 work item link 수집

### SourceHut todo.sr.ht adapter

확인할 것:

- GraphQL endpoint와 auth token
- tracker list, ticket list, cursor pagination
- ticket status/resolution/authenticity
- label, assignment, subscription, webhook
- externalId/externalUrl 필드 활용 가능성
- SourceHut의 Git 서비스와 todo service가 별도라는 점

MVP:

- tracker/ticket 읽기
- ticket title/body/status/labels/assignees 정규화

최종:

- status update, label, assign, comment
- tracker webhook

### Gerrit adapter

확인할 것:

- Gerrit은 issue tracker가 아니라 code review change 시스템
- Change-Id, project, branch, topic, owner, reviewers, status
- changes query language
- comments/robot comments
- external issue link는 commit message/topic/hashtag에서 추출 가능
- final product에서 “review item”으로 별도 타입 처리

MVP:

- 제외

최종:

- `WorkItemType = change`로 제한 지원
- issue와 change를 link graph로 연결

### Redmine adapter

확인할 것:

- Redmine REST API 활성화 여부
- issue, project, tracker, status, priority, category, version, custom field
- Git repository browser와 changeset reference
- API key 인증
- webhook은 core가 아니라 플러그인/외부 설정 필요 가능성

MVP:

- 제외 또는 read-only optional

최종:

- issue read/write
- Git changeset reference 수집
- legacy enterprise import target

## 7. 정규화 데이터 모델

## 7.1 설계 원칙

정규화 모델은 provider의 모든 필드를 억지로 하나의 좁은 스키마에 넣지 않는다.

원칙:

- 공통 UI/검색에 필요한 필드는 typed column으로 저장한다.
- provider별 고유 필드는 JSONB로 보존한다.
- 원본 객체의 identity는 절대 잃지 않는다.
- write-back 가능한 필드와 read-only 필드를 구분한다.
- provider별 상태/라벨/우선순위는 canonical value와 original value를 모두 저장한다.
- issue, PR, MR, work item, ticket, change를 하나의 `WorkItem` 계열로 모델링하되, MVP UI는 `issue_kind = issue` 중심으로 시작한다.

## 7.2 핵심 엔티티

| 엔티티                 | 설명                                     |
| ------------------- | -------------------------------------- |
| Tenant              | 조직/워크스페이스 단위                           |
| User                | Sentinel 사용자                           |
| Team                | Sentinel 팀                             |
| ProviderConnection  | GitHub/GitLab 등 외부 계정 연결               |
| ProviderNamespace   | org/group/workspace/project collection |
| Repository          | 저장소 또는 project                         |
| Issue               | 정규화된 작업 항목                             |
| IssueComment        | 댓글                                     |
| IssueEvent          | 상태/라벨/담당자 변경 이력                        |
| Label               | provider label/tag                     |
| Milestone           | milestone/version/iteration/cycle      |
| Assignee            | provider user identity                 |
| SyncJob             | 동기화 작업                                 |
| SyncCursor          | incremental sync 커서                    |
| WebhookSubscription | provider webhook 설정                    |
| ExternalLink        | Jira/Linear/Plane/OpenProject 등 연결     |
| AuditLog            | 사용자/시스템 변경 이력                          |
| AutomationRule      | 자동화 규칙                                 |
| SavedView           | 저장된 검색/필터                              |

## 7.3 Issue 테이블 초안

```sql
create table issues (
  id uuid primary key,
  tenant_id uuid not null,
  provider_connection_id uuid not null,
  provider_key text not null,

  provider_namespace_id text,
  provider_repository_id text,
  provider_repository_full_name text,

  provider_issue_id text not null,
  provider_issue_number text,
  provider_issue_iid text,
  provider_global_id text,
  provider_url text not null,

  issue_kind text not null,
  canonical_state text not null,
  provider_state text not null,
  provider_state_reason text,

  title text not null,
  body text,
  body_format text,

  author_provider_id text,
  author_login text,
  author_display_name text,

  assignee_logins text[],
  label_names text[],
  milestone_title text,
  priority text,
  severity text,

  comments_count integer default 0,
  reactions_count integer default 0,

  source_created_at timestamptz,
  source_updated_at timestamptz,
  source_closed_at timestamptz,

  first_seen_at timestamptz not null,
  last_seen_at timestamptz not null,
  last_synced_at timestamptz not null,

  is_deleted_on_source boolean default false,
  is_archived boolean default false,
  is_locked boolean default false,

  provider_extra jsonb not null default '{}',
  raw_payload jsonb not null,

  unique (tenant_id, provider_key, provider_connection_id, provider_repository_id, provider_issue_id)
);
```

## 7.4 Canonical state mapping

| Canonical state | 의미         | GitHub              | GitLab                | Gitea/Forgejo    | Bitbucket          | Azure DevOps        | SourceHut            |
| --------------- | ---------- | ------------------- | --------------------- | ---------------- | ------------------ | ------------------- | -------------------- |
| open            | 작업 전/진행 가능 | open                | opened                | open             | new/open/submitted | New/Active/To Do    | reported             |
| in_progress     | 진행 중       | label/field 기반      | label/weight/board 기반 | label/project 기반 | open/on hold 일부    | Active/In Progress  | confirmed            |
| blocked         | 차단         | label/dependency 기반 | blocked/label 기반      | label 기반         | on hold            | Blocked custom      | custom/label         |
| resolved        | 해결됐으나 종료 전 | 없음/label            | resolved custom       | 없음/label         | resolved           | Resolved            | resolved             |
| closed          | 종료         | closed              | closed                | closed           | closed             | Closed/Done/Removed | resolved/closed      |
| duplicate       | 중복         | state_reason/label  | label/close reason    | label            | duplicate          | custom reason       | resolution duplicate |
| invalid         | 유효하지 않음    | label               | label                 | label            | invalid/wontfix    | Removed/Rejected    | resolution           |
| unknown         | 매핑 불가      | 기타                  | 기타                    | 기타               | 기타                 | 기타                  | 기타                   |

주의:

- provider 상태를 canonical state로 덮어쓰면 안 된다.
- canonical state는 검색/대시보드용이다.
- write-back은 항상 provider-specific transition으로 변환해야 한다.
- Azure DevOps와 Jira는 workflow별 transition이 다르므로 field metadata 조회가 필요하다.

## 7.5 Issue kind

| kind          | 설명                                |
| ------------- | --------------------------------- |
| issue         | 일반 이슈                             |
| bug           | bug type 또는 bug label이 명확한 항목     |
| task          | 작업/task                           |
| feature       | feature/enhancement/request       |
| story         | user story/product backlog item   |
| epic          | epic/feature/parent planning item |
| pull_request  | GitHub/Gitea/Forgejo/Bitbucket PR |
| merge_request | GitLab MR                         |
| change        | Gerrit change                     |
| ticket        | SourceHut/Redmine/Jira ticket     |
| unknown       | 매핑 불가                             |

## 8. 동기화 설계

## 8.1 동기화 방식

| 방식                  | 용도                              | 장점            | 단점                |
| ------------------- | ------------------------------- | ------------- | ----------------- |
| Full sync           | 최초 연결, 복구, 정합성 검증               | 누락 방지         | 느림, rate limit 부담 |
| Incremental polling | `updated_since`/cursor 기반 변경 수집 | 구현 안정적        | 실시간성 낮음           |
| Webhook             | 실시간 변경 반영                       | 빠름, API 호출 절감 | 설정/보안/재시도 필요      |
| Manual sync         | 사용자 재시도                         | 단순            | 자동성 없음            |
| Backfill            | 과거 댓글/이력 보강                     | 데이터 완성도       | 비용 큼              |

MVP는 Full sync + Incremental polling으로 시작한다. Webhook은 V1에서 도입한다.

## 8.2 Sync job 상태

| 상태                   | 설명                        |
| -------------------- | ------------------------- |
| queued               | 실행 대기                     |
| running              | 실행 중                      |
| succeeded            | 완전 성공                     |
| partial              | 일부 repository/provider 실패 |
| failed               | 전체 실패                     |
| canceled             | 사용자/시스템 취소                |
| rate_limited         | rate limit 때문에 중단 또는 지연   |
| auth_failed          | 인증 실패                     |
| permission_denied    | 권한 부족                     |
| provider_unavailable | provider 장애               |

## 8.3 Cursor 저장

Provider별 cursor를 별도 저장한다.

```json
{
  "provider": "github",
  "repository": "owner/repo",
  "mode": "issues",
  "last_updated_at": "2026-06-08T00:00:00Z",
  "last_seen_provider_id": "123456789",
  "etag": "optional",
  "next_page": null,
  "provider_cursor": null
}
```

## 8.4 중복 방지 키

중복 방지 기준:

```text
tenant_id
+ provider_key
+ provider_connection_id
+ provider_repository_id
+ provider_issue_id
```

주의:

- GitHub issue number는 repository 내에서만 unique하다.
- GitLab `iid`는 project 내에서만 unique하고 `id`는 instance/global 성격이다.
- SourceHut ticket id는 tracker 내에서 unique하다.
- Azure DevOps work item id는 organization/project 맥락을 함께 저장해야 한다.
- Bitbucket issue id는 repository/workspace와 함께 저장해야 한다.

## 8.5 삭제/접근 권한 변경 처리

원본에서 사라진 issue가 있을 때 즉시 삭제하지 않는다.

- 404 + 이전에 존재: `is_deleted_on_source = true` 또는 `source_access_lost`
- 403/404 + private repo: 권한 회수 가능성으로 표시
- repository archived/deleted: issue도 archived 상태로 표시
- 삭제된 provider connection: local cache 보존 여부는 retention policy에 따름

## 8.6 Rate limit/backoff 정책

필수:

- provider별 rate limit header 파싱
- `Retry-After` 존중
- exponential backoff + jitter
- provider/repository 단위 queue 분리
- 동시성 제한
- user에게 “일부 데이터가 아직 동기화 중” 표시
- full sync와 incremental sync의 priority 분리

기본값:

- provider connection당 동시 repository sync 2개
- repository issue page fetch 동시성 1개
- write operation은 read보다 낮은 동시성
- 429/secondary rate limit 감지 시 provider connection 단위 cooldown

## 9. 인증/권한/보안

## 9.1 인증 방식

| 방식                    | 사용처                                            | 장점                                          | 단점                 |
| --------------------- | ---------------------------------------------- | ------------------------------------------- | ------------------ |
| GitHub App            | GitHub                                         | fine-grained permission, installation scope | 구현 복잡              |
| OAuth App             | GitHub/GitLab/Gitea/Forgejo/Bitbucket/Linear 등 | 사용자 친화적                                     | refresh/consent 관리 |
| PAT/API Token         | self-hosted, MVP                               | 구현 쉬움                                       | 보안/권한 과다 위험        |
| Microsoft Entra OAuth | Azure DevOps                                   | production 권장                               | 테넌트 설정 복잡          |
| Basic Auth            | 일부 legacy                                      | 쉬움                                          | 피해야 함              |
| API Key               | Redmine 등                                      | 단순                                          | rotation/권한 관리 필요  |

## 9.2 토큰 저장

필수:

- DB에는 평문 토큰 저장 금지
- KMS 또는 application-level envelope encryption 사용
- 토큰 마지막 4~8자리만 표시
- token scope와 만료 시각 저장
- refresh token rotation 대응
- token revoke 감지
- connection 삭제 시 token 즉시 폐기
- audit log에 token 값 기록 금지

## 9.3 최소 권한 원칙

MVP read-only scope:

- repository metadata read
- issue read
- label/milestone/assignee read
- comment read는 상세 보기에서만 필요하면 optional

Write scope는 V2 이후 별도 consent:

- issue write
- comment write
- label write
- project/work item write

권장 UI:

- 연결 시 “읽기 전용 연결”
- 관리 기능 활성화 시 “쓰기 권한 추가”
- provider별 필요한 scope 설명
- 권한 부족 기능은 disabled + reason 표시

## 9.4 Multi-tenant 보안

확인할 것:

- tenant별 데이터 완전 분리
- 사용자가 볼 수 없는 provider connection issue 노출 금지
- provider token owner와 Sentinel user/team 권한 매핑
- shared integration connection을 누가 관리할지
- team-level repository access
- audit log: 누가 어떤 provider에 어떤 변경을 보냈는지 기록
- background job에서 tenant context 누락 방지
- webhook endpoint가 tenant/provider/repository를 안전하게 식별하는지

## 9.5 Webhook 보안

필수:

- provider별 signature 검증
- webhook secret 저장/회전
- replay attack 방지: delivery id + timestamp 저장
- idempotent processing
- payload size limit
- unknown event 무시/로그
- webhook endpoint rate limit
- webhook failure history UI

## 10. 사용자 경험 설계

## 10.1 MVP 화면

MVP 화면은 단순해야 한다.

1. Connections
   - provider 추가
   - base URL 입력
   - token/OAuth 연결
   - 연결 테스트
   - scope 확인
2. Sources
   - org/group/workspace/project 목록
   - repository/project 목록
   - issue tracker enabled 여부
   - sync 대상 선택
3. Sync Status
   - provider별 동기화 상태
   - repository별 issue 수
   - 오류/재시도
   - rate limit/cooldown
4. All Issues
   - 모든 issue 리스트
   - 검색
   - 필터
   - 정렬
   - 원본 링크
   - 상세 패널

## 10.2 All Issues 리스트 컬럼

필수 컬럼:

- Provider
- Source
- Repository/Project
- Number/Key
- Title
- State
- Labels
- Assignees
- Author
- Comments
- Updated
- Created
- Original Link

선택 컬럼:

- Priority
- Severity
- Milestone/Version
- Age
- Last activity
- SLA
- Duplicate candidates
- Linked PR/MR
- Linked external project item

## 10.3 필터

MVP 필터:

- provider
- source/repository
- state
- label
- assignee
- author
- updated date range
- created date range
- has no assignee
- has labels / no labels

V1 필터:

- stale
- inactive days
- comments count
- priority
- severity
- milestone
- external link status
- sync status
- issue kind
- mentions me
- subscribed/watching

최종 필터:

- duplicated
- blocked
- customer impact
- SLA breach
- release blocker
- no reproduction
- needs maintainer response
- AI confidence

## 10.4 저장된 뷰

예시:

- All Open Issues
- Unassigned Issues
- Recently Updated
- Needs Triage
- Stale for 30 Days
- High Priority Bugs
- Release Blockers
- Security Issues
- OSS Contributor Queue
- Customer Escalations
- My Issues Across Providers

## 10.5 상세 화면

필수:

- issue title/body
- provider/source/repository
- 원본 URL
- current state
- labels
- assignees
- author
- created/updated/closed
- raw provider metadata toggle

V1:

- comments
- event timeline
- linked PR/MR/commit
- linked external issue
- activity diff

V2:

- edit title/body
- change state
- add/remove label
- assign/unassign
- comment
- close/reopen
- sync conflict warning

## 11. 외부 프로젝트 연계

## 11.1 연계 대상

| 도구                         | 방향        | 목적                |
| -------------------------- | --------- | ----------------- |
| Jira                       | 양방향       | 기업 PM/스프린트/로드맵    |
| Linear                     | 양방향       | 스타트업 제품/개발 이슈 관리  |
| Plane                      | 양방향       | OSS/오픈코어 프로젝트 관리  |
| OpenProject                | 양방향       | 엔터프라이즈/공공 프로젝트 관리 |
| Redmine                    | 양방향       | 레거시 이슈 트래킹        |
| GitHub Projects            | 부분        | GitHub 내 프로젝트 보드  |
| GitLab Boards/Epics        | 부분        | GitLab planning   |
| Slack/Teams                | 알림/명령     | triage 알림, 담당자 호출 |
| Notion/Confluence          | 링크/요약     | 문서화               |
| Email                      | 알림/intake | digest, 외부 요청     |
| Zendesk/Intercom/Freshdesk | 링크/intake | 고객 지원 -> 개발 이슈    |

## 11.2 연계 방식

| 방식                 | 설명                              |
| ------------------ | ------------------------------- |
| Link only          | 원본 issue URL을 외부 도구에 연결         |
| Mirror             | 외부 도구에 같은 issue를 생성하고 주요 필드 동기화 |
| Export             | 선택 issue를 외부 프로젝트로 내보내기         |
| Import             | 외부 프로젝트 issue를 Sentinel로 가져오기   |
| Two-way sync       | 양쪽 변경 사항을 정책 기반으로 반영            |
| Comment bridge     | 댓글을 양쪽에 중계                      |
| Status bridge      | 상태 변경만 동기화                      |
| Automation trigger | 특정 조건에서 외부 작업 생성                |

## 11.3 필드 매핑 예시

| Sentinel  | GitHub             | GitLab              | Jira              | Linear        | Plane           | OpenProject              |
| --------- | ------------------ | ------------------- | ----------------- | ------------- | --------------- | ------------------------ |
| title     | title              | title               | summary           | title         | name/title      | subject                  |
| body      | body               | description         | description       | description   | description     | description              |
| state     | state/state_reason | state               | status            | state         | state           | status                   |
| labels    | labels             | labels              | labels/components | labels        | labels          | categories/custom fields |
| assignee  | assignees          | assignees           | assignee          | assignee      | assignee        | assignee                 |
| milestone | milestone          | milestone/iteration | fixVersion/sprint | cycle/project | cycle/module    | version                  |
| priority  | label/custom       | label/weight        | priority          | priority      | priority/custom | priority                 |
| relation  | linked issue       | related issue       | issue links       | relations     | relations       | relations                |

## 11.4 충돌 정책

양방향 동기화에서 반드시 결정해야 한다.

| 정책                | 설명                     | 사용 상황       |
| ----------------- | ---------------------- | ----------- |
| Source wins       | 원본 Git provider 변경이 우선 | read-mostly |
| Target wins       | 외부 PM 도구 변경이 우선        | PM 중심 조직    |
| Last write wins   | 최신 변경이 우선              | 단순하지만 위험    |
| Field owner       | 필드별 owner 지정           | 추천          |
| Manual resolution | 충돌 큐에서 사람이 해결          | 중요 이슈       |

추천:

- title/body/status/assignee/labels/milestone별 field owner를 둔다.
- 댓글은 append-only로 처리한다.
- delete/close는 destructive action이므로 confirmation 또는 policy가 필요하다.
- Sentinel이 쓴 변경에는 idempotency key 또는 marker를 남겨 webhook loop를 방지한다.

## 12. AI 기능 설계

## 12.1 AI 도입 시점

AI는 MVP에 넣지 않는다. MVP에서는 원본 데이터를 신뢰성 있게 모으는 것이 우선이다. AI는 V2 이후 정규화 데이터와 이벤트 이력이 쌓인 뒤 도입한다.

## 12.2 AI 기능 목록

| 기능              | 입력                                 | 출력                       | 주의                     |
| --------------- | ---------------------------------- | ------------------------ | ---------------------- |
| Issue 요약        | title/body/comments                | 3줄 요약                    | private data 처리        |
| Label 추천        | title/body/repo labels             | label candidates         | 기존 label vocabulary 사용 |
| 담당자 추천          | issue content, CODEOWNERS, history | assignee candidates      | 편향/부정확성 표시             |
| 우선순위 추천         | severity, customer, frequency      | priority                 | 사람 승인 필요               |
| 중복 탐지           | embeddings/title/body              | duplicate candidates     | 자동 병합 금지               |
| 재현 정보 체크        | bug report                         | missing fields           | 템플릿 기반                 |
| 관련 코드 추천        | issue text + code search           | files/commits            | RAG 비용/보안              |
| 댓글 초안           | issue context                      | response draft           | 자동 게시 금지               |
| Release note 후보 | closed issues                      | grouped notes            | human review           |
| Stale cleanup   | inactivity/history                 | close/comment suggestion | OSS maintainer 정책 반영   |

## 12.3 AI 안전장치

필수:

- AI 결과는 추천으로 표시하고 자동 변경 금지
- confidence score 표시
- 근거 issue/댓글/파일 링크 제공
- provider write-back 전 사용자 승인
- private repository data의 모델 전송 정책 설정
- tenant별 AI opt-in
- prompt/audit log에 민감정보 노출 방지
- hallucination 대비 원문 링크 기반 UI

## 13. 자동화 규칙

## 13.1 Rule engine

예시 DSL:

```yaml
when:
  provider: github
  state: open
  labels_missing: true
  updated_before_days: 14
then:
  - suggest_labels: true
  - add_to_view: needs-triage
  - notify: "#triage"
```

## 13.2 자동화 예시

- 새 issue가 들어오면 `needs-triage` 큐에 추가
- `bug` label이 있고 assignee가 없으면 담당자 추천
- 30일 동안 활동 없는 issue를 stale queue에 추가
- `security` label이 있으면 제한된 팀에만 표시
- release milestone이 있는 open issue를 release blocker로 표시
- customer escalation issue를 Jira/Linear에 자동 생성
- GitHub issue가 closed되면 연결된 Plane work item 상태 변경 제안

## 13.3 자동화 권한

자동화도 사람 사용자와 동일하게 권한을 가져야 한다.

- rule owner
- allowed actions
- provider write scope
- dry-run mode
- last execution log
- failure notification
- rollback 가능 여부

## 14. 백엔드 아키텍처

## 14.1 추천 구조

```text
apps/
  web/                  # 웹 UI
  api/                  # REST/GraphQL API
  worker/               # sync/background jobs
packages/
  core/                 # domain model, use cases
  adapters/
    github/
    gitlab/
    gitea/
    forgejo/
    bitbucket/
    azure-devops/
    sourcehut/
    gerrit/
    jira/
    linear/
    plane/
    openproject/
  db/                   # migrations, schema
  auth/                 # app auth/session
  crypto/               # secret encryption
  search/               # indexing
  queue/                # job queues
  shared/               # types/utils
```

## 14.2 주요 서비스

| 서비스                 | 역할                             |
| ------------------- | ------------------------------ |
| API Server          | UI/API 요청 처리                   |
| Sync Worker         | provider issue 수집              |
| Webhook Receiver    | provider webhook 수신/검증         |
| Normalizer          | raw payload -> canonical model |
| Search Indexer      | full-text/filter index         |
| Rule Engine         | 자동화 실행                         |
| Notification Worker | Slack/email/web notification   |
| AI Worker           | 요약/추천/중복 탐지                    |
| Export/Sync Worker  | 외부 PM 도구 연계                    |

## 14.3 저장소

추천:

- PostgreSQL: primary data store
- JSONB: provider raw payload/extra field
- Redis 또는 PostgreSQL queue: background jobs
- OpenSearch/Meilisearch/Postgres FTS: 검색
- Object storage: attachment/cache export
- KMS/Vault: secret encryption

MVP는 PostgreSQL + job queue + Postgres full-text search로 충분하다.

## 14.4 API 설계

MVP API:

```text
POST   /connections
GET    /connections
POST   /connections/:id/test
GET    /connections/:id/namespaces
GET    /connections/:id/repositories
POST   /sources
GET    /sources
POST   /sync-jobs
GET    /sync-jobs/:id
GET    /issues
GET    /issues/:id
GET    /issues/facets
```

V2 API:

```text
POST   /issues/:id/comments
PATCH  /issues/:id
POST   /issues/:id/transition
POST   /issues/:id/labels
DELETE /issues/:id/labels/:label
POST   /issues/:id/assignees
DELETE /issues/:id/assignees/:assignee
POST   /issues/:id/external-links
POST   /automation-rules
```

## 15. 프론트엔드 요구사항

## 15.1 MVP UI 원칙

- 첫 화면은 All Issues 리스트다.
- 마케팅/랜딩 페이지가 아니라 실제 도구 화면을 보여준다.
- 이슈가 많아도 스캔하기 쉬운 밀도 높은 테이블을 사용한다.
- provider와 repository 구분이 즉시 보여야 한다.
- sync 상태와 데이터 신선도를 항상 노출한다.
- 실패한 provider가 있어도 나머지 issue는 계속 볼 수 있어야 한다.

## 15.2 핵심 컴포넌트

| 컴포넌트                   | 설명                                 |
| ---------------------- | ---------------------------------- |
| ProviderConnectionForm | provider 선택, base URL, token/OAuth |
| RepositoryPicker       | org/group/workspace/repository 선택  |
| SyncStatusPanel        | sync 진행/오류/rate limit              |
| IssueTable             | 가상 스크롤/정렬/필터                       |
| IssueFilterBar         | provider/state/label/assignee/date |
| IssueDetailDrawer      | 상세 보기                              |
| RawPayloadViewer       | provider raw JSON                  |
| SavedViewTabs          | 저장된 뷰                              |
| EmptyState             | 연결 전/검색 결과 없음                      |
| ErrorBanner            | provider별 오류                       |

## 15.3 UX 세부 요구

- provider별 로고/색상은 보조 정보로만 사용한다.
- 긴 제목은 2줄 clamp 또는 tooltip.
- 라벨은 색상 swatch와 텍스트를 함께 표시.
- 날짜는 상대 시간과 absolute tooltip 모두 제공.
- 원본 링크는 새 탭으로 열기.
- keyboard shortcut은 MVP에서 필수 아님.
- 테이블 컬럼 선택/저장은 V1.
- issue 상세는 side drawer로 열고 리스트 context를 유지한다.

## 16. 검색/필터/인덱싱

## 16.1 MVP 검색

대상:

- title
- body
- repository full name
- label names
- author login
- assignee logins
- provider issue number/key

PostgreSQL FTS 또는 trigram을 우선한다.

## 16.2 최종 검색

필요 기능:

- full-text search
- faceted search
- saved query
- provider native query passthrough
- semantic search
- duplicate similarity search
- advanced query language

예시 query:

```text
provider:github repo:owner/repo state:open label:bug assignee:none updated:<30d
```

## 17. 운영/배포

## 17.1 배포 형태

| 형태                      | 대상    | 설명                                     |
| ----------------------- | ----- | -------------------------------------- |
| Local dev               | 개발자   | Docker Compose                         |
| Self-hosted single node | 소규모 팀 | API/Web/Worker/Postgres/Redis          |
| Self-hosted enterprise  | 기업    | Kubernetes, external DB, Vault         |
| SaaS                    | 일반 고객 | multi-tenant hosted                    |
| Air-gapped              | 보안 조직 | offline install, manual license/update |

## 17.2 필수 설정

- app base URL
- database URL
- queue backend
- encryption key/KMS
- OAuth callback URLs
- webhook public URL
- provider app credentials
- allowed self-hosted domains
- outbound proxy
- log level
- retention policy

## 17.3 관측성

Metrics:

- sync job count/duration/failure
- issues fetched per provider
- API request count per provider
- rate limit remaining
- webhook delivery success/failure
- queue depth
- normalization errors
- DB query latency
- search latency

Logs:

- provider request id
- sync job id
- tenant id
- repository id
- issue id
- error code
- retry count

Tracing:

- sync job -> provider API call -> normalization -> DB upsert -> index
- webhook receiver -> dedupe -> issue update

## 18. 품질/테스트 전략

## 18.1 테스트 종류

| 테스트              | 대상                                                   |
| ---------------- | ---------------------------------------------------- |
| Unit test        | normalizer, state mapping, cursor parsing            |
| Contract test    | adapter API response fixture                         |
| Integration test | provider sandbox/API mock                            |
| E2E test         | connection -> sync -> list -> detail                 |
| Migration test   | DB schema migration                                  |
| Security test    | token encryption, webhook signature                  |
| Load test        | 10만~100만 issue list/search                           |
| Failure test     | rate limit, expired token, 403, 404, network timeout |

## 18.2 Fixture 전략

Provider API는 실제 호출 없이도 테스트 가능해야 한다.

- GitHub issue open/closed/PR mixed fixture
- GitLab project issue/group issue fixture
- Gitea/Forgejo issue fixture
- Bitbucket issue fixture
- Azure Work Item fixture
- SourceHut ticket fixture
- malformed payload fixture
- missing optional field fixture
- pagination fixture
- rate limit fixture
- permission denied fixture

## 18.3 수용 테스트

MVP 수용 기준:

1. GitHub repository 1개를 연결한다.
2. open/closed issue를 모두 가져온다.
3. 같은 repository를 다시 sync해도 issue 수가 중복 증가하지 않는다.
4. GitLab project 1개를 연결한다.
5. GitHub와 GitLab issue가 같은 리스트에 표시된다.
6. `state=open`, `provider=github`, `label=bug` 필터가 동작한다.
7. provider 인증 오류가 나도 기존 issue list는 유지된다.
8. issue 원본 URL로 이동할 수 있다.

## 19. 구현 로드맵

## 19.1 Phase 0: 기반 설계

산출물:

- provider adapter interface
- canonical issue model
- database schema
- sync job lifecycle
- security/token storage design
- MVP UI wireframe

확인 사항:

- 어떤 언어/프레임워크를 사용할지
- SaaS와 self-host 둘 다 고려할지
- OAuth/App 등록을 어디까지 할지
- read-only MVP로 확정할지

## 19.2 Phase 1: MVP

기능:

- GitHub adapter
- GitLab adapter
- Gitea/Forgejo adapter
- provider connection 관리
- repository picker
- initial sync
- issue upsert
- all issues list
- basic search/filter
- issue detail drawer
- sync status

가능하면 추가:

- Bitbucket Cloud adapter
- Azure DevOps adapter

## 19.3 Phase 2: 안정화/V1

기능:

- webhook receiver
- comments/events sync
- saved views
- advanced filters
- sync retry/cooldown UI
- provider capability UI
- audit log 기초
- SourceHut adapter
- better search index

## 19.4 Phase 3: 관리 기능/V2

기능:

- issue comment write-back
- label/assignee/status write-back
- close/reopen
- conflict detection
- provider write permission upgrade
- bulk action
- rule engine dry-run

## 19.5 Phase 4: 외부 연계/V3

기능:

- Jira export/sync
- Linear export/sync
- Plane export/sync
- OpenProject export/sync
- Slack/Teams notification
- Notion/Confluence links
- customer support integration PoC

## 19.6 Phase 5: AI/자동화/엔터프라이즈

기능:

- issue summary
- duplicate detection
- label/assignee/priority suggestion
- stale issue automation
- release blocker dashboard
- SSO/RBAC/audit log
- self-host enterprise deployment
- compliance export

## 20. 핵심 의사결정 목록

구현 전에 반드시 결정해야 한다.

1. 제품 이름: Git Issue Sentinel, Sponzey Issue Sentinel 등
2. MVP provider 범위: GitHub/GitLab/Gitea/Forgejo만 먼저 할지, Bitbucket/Azure까지 포함할지
3. MVP 인증 방식: PAT 우선인지 OAuth/App 우선인지
4. SaaS 우선인지 self-host 우선인지
5. read-only MVP 원칙을 지킬지
6. PR/MR을 MVP 리스트에 포함할지
7. closed issue를 기본 수집할지 open만 기본 수집할지
8. comments를 MVP에서 수집할지 상세 클릭 시 lazy load할지
9. raw payload 보존 기간
10. provider token 암호화 방식
11. 검색 엔진: Postgres FTS로 시작할지 별도 검색엔진을 쓸지
12. job queue: Postgres 기반인지 Redis 기반인지
13. webhook을 V1로 미룰지 MVP에 넣을지
14. write-back 기능을 언제 열지
15. AI 기능을 SaaS 전용으로 할지 self-host도 제공할지
16. 외부 프로젝트 연계의 1순위: Jira, Linear, Plane, OpenProject 중 무엇인지
17. 라이선스/오픈소스 전략
18. 고객 데이터 보존/삭제 정책
19. enterprise 기능의 유료화 경계
20. 감사 로그와 compliance 요구 수준

## 21. 리스크와 대응

| 리스크                            | 영향         | 대응                                       |
| ------------------------------ | ---------- | ---------------------------------------- |
| Provider API 변경                | sync 실패    | adapter contract test, versioned adapter |
| Rate limit                     | 대량 sync 지연 | incremental sync, webhook, backoff       |
| Token 권한 부족                    | 일부 기능 비활성  | scope validation UI                      |
| Self-hosted version 차이         | API 오류     | version detection, capability model      |
| Bitbucket issue API deprecated | 장기 지원 불확실  | Jira path 병행                             |
| Azure work item 모델 복잡도         | 정규화 어려움    | field metadata + custom mapping          |
| 양방향 sync 충돌                    | 데이터 훼손     | field owner policy, audit, dry-run       |
| AI 오추천                         | 잘못된 triage | human approval                           |
| 민감정보 유출                        | 보안 사고      | encryption, tenant isolation, AI opt-in  |
| 대량 데이터 성능                      | UI/검색 저하   | indexing, pagination, virtualization     |
| Webhook loop                   | 무한 동기화     | idempotency marker, delivery dedupe      |
| 삭제/권한 회수                       | 데이터 불일치    | soft delete/source_access_lost           |

## 22. MVP 작업 분해

## 22.1 백엔드 작업

- 프로젝트 스캐폴딩
- DB schema/migration
- secret encryption utility
- provider connection CRUD
- adapter interface
- GitHub adapter
- GitLab adapter
- Gitea adapter
- Forgejo profile
- repository discovery
- issue list sync
- pagination handler
- rate limit handler
- issue normalizer
- issue upsert
- sync job queue
- sync status API
- issue list API
- issue detail API
- filter/facet API

## 22.2 프론트엔드 작업

- app shell
- sidebar/navigation
- connection list
- add connection flow
- repository picker
- sync status panel
- all issues table
- filter bar
- search input
- issue detail drawer
- raw metadata viewer
- error states
- loading/skeleton states
- empty states

## 22.3 테스트 작업

- adapter fixtures
- normalizer unit tests
- state mapping tests
- pagination tests
- expired token test
- rate limit test
- issue upsert idempotency test
- e2e: connection -> sync -> list
- UI visual checks

## 23. 데이터 개인정보/컴플라이언스 확인 사항

필수 확인:

- issue body/comment에 개인정보/고객정보/API key가 들어올 수 있음
- private repository issue를 외부 AI API로 보낼 수 있는지
- attachment 저장 여부
- raw payload 저장 기간
- 사용자 삭제 시 provider token 삭제
- tenant 삭제 시 데이터 완전 삭제
- audit log 보존 기간
- export/delete request 대응
- self-host 고객의 outbound network 제한
- webhook payload 로그 마스킹

## 24. 가격/사업 모델 검토 항목

가능한 과금 기준:

- connected provider 수
- synced repositories 수
- synced issues 수
- users/seats
- automation rule 수
- AI usage
- retention 기간
- self-host enterprise license
- SSO/audit/RBAC enterprise add-on

무료 플랜 가능 경계:

- 1~2 provider
- 5 repositories
- 5,000 issues
- daily sync
- read-only

Pro/Team:

- unlimited repositories 또는 높은 한도
- webhook sync
- saved views
- Slack/email
- write-back
- Jira/Linear export

Enterprise:

- self-host
- SSO/SAML
- audit log
- SCIM
- custom retention
- private AI/no AI mode
- support/SLA
- custom provider adapter

## 25. 참고 문서

- 기존 시장 조사: [RESEARCH.md](./RESEARCH.md)
- GitHub REST Issues API: <https://docs.github.com/en/rest/issues>
- GitHub REST API rate limits: <https://docs.github.com/rest/overview/rate-limits-for-the-rest-api>
- GitHub webhook events: <https://docs.github.com/en/webhooks/webhook-events-and-payloads>
- GitLab Issues API: <https://docs.gitlab.com/api/issues/>
- GitLab Webhooks: <https://docs.gitlab.com/user/project/integrations/webhooks/>
- Gitea API Usage: <https://docs.gitea.com/development/api-usage/>
- Forgejo Issue Search: <https://forgejo.org/docs/latest/user/issue-search/>
- Bitbucket Cloud Issue Tracker API: <https://developer.atlassian.com/cloud/bitbucket/rest/api-group-issue-tracker/>
- Bitbucket Cloud API request limits: <https://support.atlassian.com/bitbucket-cloud/docs/api-request-limits/>
- Bitbucket Cloud Webhooks: <https://support.atlassian.com/bitbucket-cloud/docs/manage-webhooks/>
- Azure DevOps Work Items API: <https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items?view=azure-devops-rest-7.1>
- Azure DevOps Service Hook Events: <https://learn.microsoft.com/en-us/azure/devops/service-hooks/events?view=azure-devops>
- Azure DevOps rate and usage limits: <https://learn.microsoft.com/vsts/reference/rate-limits>
- SourceHut todo.sr.ht GraphQL API: <https://docs.sourcehut.org/todo.sr.ht/>
- Gerrit Changes REST API: <https://gerrit-review.googlesource.com/Documentation/rest-api-changes.html>
- Jira Cloud REST Issues API: <https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/>
- Linear API and Webhooks: <https://linear.app/docs/api-and-webhooks>
- OpenProject Work Packages API: <https://www.openproject.org/docs/api/endpoints/work-packages/>