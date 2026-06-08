# Git Issue 관리 지원 OSS 솔루션 리서치

조사일: 2026-06-05  
범위: Git 저장소와 직접 결합되거나 GitHub/GitLab/Gitea/Forgejo 이슈를 가져와 정리, 분류, 우선순위화, 보드화, 로드맵화, 협업 관리하는 오픈소스 또는 오픈코어 솔루션

## 1. 요약

Git 이슈 관리 OSS 시장은 크게 두 갈래로 나뉜다.

1. **Git forge형**: GitLab, Gitea, Forgejo, OneDev처럼 Git 저장소, PR/MR, 이슈, 보드, CI/CD를 한 제품 안에서 제공한다.
2. **프로젝트 관리/이슈 관리형**: Plane, OpenProject, Redmine, Taiga, Huly처럼 GitHub/GitLab 등 외부 Git 플랫폼과 연동하거나 별도 프로젝트 관리 계층을 제공한다.

현 시점에서 “Git의 Issue 관련 정리/관리”를 지원하는 제품을 만들거나 도입한다면, 가장 강한 후보군은 다음과 같다.

| 우선순위 | 솔루션                                  | 추천 포지션                                    | 핵심 판단                                                                             |
| ---- | ------------------------------------ | ----------------------------------------- | --------------------------------------------------------------------------------- |
| 1    | **Plane**                            | GitHub/GitLab 이슈 동기화 + 현대적 Jira/Linear 대체 | GitHub/GitLab 양방향 동기화, 보드/사이클/모듈/위키/인테이크가 강함. 오픈소스 인지도도 매우 높음.                    |
| 2    | **GitLab Self-Managed / GitLab.com** | Git 저장소까지 통합한 DevSecOps 플랫폼               | 이슈, 보드, 에픽, 마일스톤, CI/CD, 보안까지 올인원. 다만 가격과 운영 복잡도가 큼.                              |
| 3    | **OpenProject**                      | 프로젝트 관리/포트폴리오/공공·엔터프라이즈형                  | GitHub/GitLab 연동, 워크패키지, Gantt, 비용/시간 추적, Jira 마이그레이션 강점. 개발팀 전용보다는 조직 전체 PM에 강함. |
| 4    | **Gitea / Forgejo**                  | 가볍고 자체 호스팅 가능한 GitHub 대체                  | 자체 Git 서버를 운영하며 이슈/PR/보드까지 처리할 때 적합. 고급 PM 기능은 Plane/OpenProject보다 약함.            |
| 5    | **Redmine**                          | 오래된 안정형 이슈 트래커                            | Git 연동, 커스텀 워크플로우, 플러그인 생태계가 강하지만 UX는 오래됨.                                        |
| 6    | **OneDev**                           | GitLab보다 가벼운 올인원 DevOps                   | 이슈 커스텀, 보드, 서비스 데스크, CI/CD, 패키지까지 강력. 다만 시장 인지도는 상위권보다 낮음.                        |
| 7    | **Taiga**                            | Scrum/Kanban 중심 애자일 관리                    | GitHub/GitLab 연동 가능. 제품 성숙도는 있으나 최근 시장 주목도는 Plane/Huly보다 낮음.                      |
| 8    | **Huly**                             | Linear + Notion + Slack 대체형 통합 협업         | GitHub 양방향 연동을 전면에 내세움. 잠재력은 크지만 상대적으로 신생이며 가격/상용 구조가 덜 명확함.                      |

## 2. 시장성 분석

### 2.1 수요 배경

GitHub Octoverse 2025에 따르면 GitHub에는 **1억 8천만 명 이상 개발자**가 있고, 2025년에만 **3,600만 명 이상**이 새로 가입했다. 또한 2025년에는 **분당 230개 이상**의 신규 저장소가 생성됐고, 공개/오픈소스 저장소는 전체 저장소의 **63%**를 차지한다. 같은 보고서는 2025년 7월 한 달에 공개/비공개 프로젝트에서 **550만 개 이슈가 종료**됐다고 밝힌다.  
출처: [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)

이 수치는 Git 이슈 관리가 단순한 부가기능이 아니라 개발 협업의 핵심 데이터 계층임을 보여준다. 특히 AI 코딩/에이전트 사용 증가로 저장소, PR, 이슈, 커밋 수가 늘어나면서 다음 문제가 커지고 있다.

- 이슈 중복, 라벨 누락, 우선순위 미정, 담당자 미정
- GitHub/GitLab 안에서는 개발팀 단위 관리는 쉽지만 제품/CS/운영/경영 레벨 로드맵 연결은 부족
- Jira/Linear/Asana/ClickUp 같은 상용 SaaS 비용 부담
- 보안, 데이터 주권, 내부망 운영, 규제 산업에서 self-host 요구 증가
- OSS 프로젝트의 public issue triage, good first issue 관리, contributor onboarding 수요 증가

### 2.2 경쟁 구도

상용 경쟁 제품은 Jira, Linear, GitHub Issues/Projects, GitLab, ClickUp, Asana, Monday 등이 중심이다. OSS 제품은 상용 SaaS 대비 다음 가치를 제공한다.

- **비용 절감**: 자체 호스팅 시 라이선스 비용을 줄이고 인프라 비용 중심으로 운영 가능
- **데이터 통제**: 이슈, 첨부파일, 내부 논의, 보안 취약점 정보를 사내망에 보관 가능
- **커스터마이징**: 워크플로우, 필드, 라벨, 자동화, 통합을 직접 수정 가능
- **벤더 락인 완화**: GitHub/GitLab/Jira 등 외부 시스템과 동기화하거나 대체 가능

다만 OSS 도입에는 운영 인력, 백업/업그레이드, 보안 패치, 플러그인 호환성, SSO/감사 로그 같은 엔터프라이즈 기능의 유료화 여부를 반드시 확인해야 한다.

### 2.3 제품화 기회

이 시장에서 새로운 제품 또는 내부 솔루션을 만든다면 단순 이슈 트래커 자체보다 다음 영역이 더 유망하다.

| 기회                             | 설명                                                          | 경쟁 강도 | 시장성   |
| ------------------------------ | ----------------------------------------------------------- | ----- | ----- |
| GitHub/GitLab/Gitea 이슈 통합 대시보드 | 여러 저장소의 이슈를 제품/고객/릴리스 기준으로 재분류                              | 중간    | 높음    |
| AI triage                      | 중복 이슈 탐지, 라벨/우선순위/담당자 추천, 요약                                | 높음    | 높음    |
| OSS maintainer cockpit         | good first issue, stale issue, PR 대기, contributor 응답 SLA 관리 | 중간    | 중간~높음 |
| 고객 피드백 -> Git issue 변환         | CS/세일즈/폼/메일을 이슈로 연결하고 상태 동기화                                | 높음    | 높음    |
| self-hosted compliance PM      | 내부망, 감사 로그, RBAC, 보안 취약점 이슈 관리                              | 중간    | 높음    |
| 한국어/국내 조직 특화                   | Jira/Notion/GitHub 혼용 조직을 위한 한글 UX, 보고서, 결재 흐름              | 낮음~중간 | 중간    |

## 3. 평가 기준

| 기준                             | 중요도   | 이유                                  |
| ------------------------------ | ----- | ----------------------------------- |
| GitHub/GitLab/Gitea/Forgejo 연동 | 매우 높음 | Git 이슈와 커밋/PR/MR 추적이 핵심             |
| 이슈 정리 기능                       | 매우 높음 | 라벨, 상태, 우선순위, 담당자, 중복, 검색, 필터       |
| 보드/로드맵                         | 높음    | Kanban/Scrum/마일스톤/사이클/에픽 관리         |
| 커스텀 워크플로우/필드                   | 높음    | 조직별 프로세스 대응                         |
| 가격/라이선스                        | 높음    | OSS 도입의 주요 이유                       |
| self-host 난이도                  | 높음    | 내부망/보안 요구 대응                        |
| 시장 신뢰도                         | 높음    | GitHub stars, 릴리스, 커뮤니티, 고객 로고, 생태계 |
| 확장성/API                        | 중간~높음 | 자체 자동화/AI 연동에 필요                    |
| 엔터프라이즈 기능                      | 중간    | SSO, LDAP, SAML, 감사 로그, SLA, 지원     |

## 4. 주요 솔루션 상세

## 4.1 Plane

공식 사이트: <https://plane.so/>  
GitHub: <https://github.com/makeplane/plane>  
라이선스: AGPL-3.0  
유형: 오픈소스 프로젝트 관리, Jira/Linear/ClickUp 대체

### 개요

Plane은 이슈, 사이클, 모듈, 프로젝트, 위키, 인테이크, 대시보드, GitHub/GitLab 연동을 제공하는 현대적 프로젝트 관리 도구다. 공식 GitHub 저장소는 2026-06-05 조사 기준 **50.3k stars**, **4.4k forks**로 OSS 프로젝트 관리 도구 중 최상위 인지도를 보인다.  
출처: [Plane GitHub repository](https://github.com/makeplane/plane)

### Git 이슈 관리 기능

Plane 가격/기능 페이지는 GitHub 및 GitLab 동기화를 명시한다.

- GitHub Sync: Plane work item과 GitHub issue/state를 동기화하고 양쪽 활동을 서로 업데이트
- GitHub Enterprise: 온프레미스 GitHub Enterprise Server 연결
- GitLab Sync: Plane work item과 GitLab issue/state 동기화
- GitLab Enterprise: self-managed GitLab Enterprise 연결

출처: [Plane Pricing - Integrations](https://plane.so/pricing)

핵심 기능:

- Work Items: 이슈/작업 단위 생성, 속성, 댓글, 첨부
- Cycles: 스프린트/반복 주기 관리
- Modules: 기능 묶음, 제품 영역별 관리
- Views: List, Board, Calendar, Gantt, Spreadsheet 레이아웃
- Intake: 외부 요청을 triage 가능한 work item으로 전환
- Pages/Wiki: 문서와 이슈 연결
- Estimates, milestones, dependencies, recurring work items
- Pro 이상에서 time tracking, work logs, dashboards, custom properties

### 가격

공식 가격 페이지 기준:

| 플랜              | 가격             | 주요 내용                                                                                                          |
| --------------- | -------------- | -------------------------------------------------------------------------------------------------------------- |
| Free            | $0/seat/month  | 프로젝트, work item, cycles, modules, views, intake, estimates, pages. 좌석 제한 12 users                              |
| Pro             | $6/seat/month  | work item types/properties, workspace wiki, time tracking, templates, dashboards, teamspaces, integrations     |
| Business        | $13/seat/month | project templates, recurring work items, intake email/forms, customers, advanced dashboards, single workflow   |
| Enterprise Grid | 견적             | private/managed deployments, granular access control, multiple workflows, LDAP, audit logs, migration services |

출처: [Plane Pricing](https://plane.so/pricing)

### 장점

- GitHub/GitLab 양방향 동기화가 제품 핵심 기능으로 들어가 있음
- Jira/Linear 대체 UX를 지향해 현대적인 화면과 워크플로우가 강함
- self-host 가능하며 AGPL 기반
- GitHub stars가 높아 시장 관심도와 커뮤니티 신호가 강함
- 제품/개발/운영 이슈를 하나의 work item 체계로 묶기 좋음

### 단점/리스크

- 고급 기능 상당수가 Pro/Business/Enterprise에 배치되어 있음
- AGPL이므로 상용 재배포/수정 SaaS 모델에서는 라이선스 검토 필요
- Git 저장소 자체 호스팅 기능은 없음. GitHub/GitLab/Gitea/Forgejo 같은 forge와 함께 써야 함
- 대기업 규제 환경에서는 SAML/OIDC/LDAP/audit log 등 플랜 확인 필요

### 적합한 사용처

- GitHub/GitLab을 이미 쓰면서 Jira/Linear를 대체하고 싶은 팀
- 이슈 triage, 제품 로드맵, 스프린트, 고객 요청을 한곳에서 관리하려는 SaaS/제품 조직
- 새로운 “Git Issue Sentinel”류 제품이 벤치마크하기 좋은 1순위 OSS

## 4.2 GitLab

공식 사이트: <https://about.gitlab.com/>  
문서: <https://docs.gitlab.com/>  
라이선스/유형: 오픈코어 DevSecOps 플랫폼

### 개요

GitLab은 Git 저장소, 이슈, 보드, 에픽, CI/CD, 패키지, 보안, 컴플라이언스까지 포함하는 올인원 DevSecOps 플랫폼이다. Git 이슈 관리 관점에서는 저장소와 이슈/PR/MR/CI가 가장 깊게 결합된 후보 중 하나다.

### Git 이슈 관리 기능

GitLab 공식 문서는 Issues가 feature proposal, task, support request, bug report를 추적하고 assignee, due date, health status, labels, epics, boards, templates, external tools를 지원한다고 설명한다.  
출처: [GitLab Issues Docs](https://docs.gitlab.com/user/project/issues/)

Issue Boards는 Free/Premium/Ultimate 모두에서 사용할 수 있고, label, milestone, assignee, status 등으로 리스트를 구성하며 Kanban/Scrum 워크플로우를 지원한다. Free에서는 group issue board 1개 제한이 있고, Premium/Ultimate에서 group board와 configurable board 기능이 확장된다.  
출처: [GitLab Issue Boards Docs](https://docs.gitlab.com/user/project/issue_board/)

핵심 기능:

- 프로젝트/그룹 이슈
- 라벨, 마일스톤, iteration, weight, due date
- issue board, group issue board, multiple boards
- 에픽, 태스크, 하위 태스크, issue promotion
- MR/commit과 issue 자동 링크 및 close
- CI/CD, package registry, security scan과 연결
- 외부 이슈 트래커/Jira/email 연동

### 가격

공식 가격 페이지 기준:

| 플랜                               | 가격                             | 주요 내용                                                                                                                                           |
| -------------------------------- | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Free                             | $0/user/month                  | SCM, CI/CD, GitLab.com Free private top-level group은 5 licensed users, 400 compute minutes/month, 10GiB storage. self-managed Free는 별도 조건 확인 필요 |
| Premium                          | $29/user/month, annual billing | unlimited licensed users, 10,000 compute minutes, advanced CI/CD, Team Project Management, SLA Management, Priority Support                     |
| Ultimate                         | Custom pricing                 | security/compliance, strategic portfolio management, value stream management, 50,000 compute minutes                                            |
| Enterprise Agile Planning add-on | $15/user/month, annual billing | Ultimate 고객 대상 planning workflow 확장                                                                                                             |

출처: [GitLab Pricing](https://about.gitlab.com/pricing/)

### 장점

- Git 저장소와 이슈/보드/MR/CI/CD가 한 플랫폼에 결합되어 추적성이 매우 좋음
- 개발, 보안, 배포, 컴플라이언스까지 확장 가능
- self-managed 지원
- 이슈 관리가 단순 task board를 넘어 DevSecOps 흐름에 자연스럽게 연결됨

### 단점/리스크

- 가격이 높고 플랜 구조가 복잡함
- 이슈 관리만 필요할 경우 기능과 운영 부담이 과함
- Free tier의 group board 등 일부 기능 제한
- self-managed 운영 시 백업, 업그레이드, 러너, 보안 패치 책임이 큼

### 적합한 사용처

- GitHub를 대체하거나 내부망 Git DevOps 플랫폼을 구축하려는 조직
- CI/CD, 보안, MR, 이슈, 배포까지 한 플랫폼에서 운영하려는 팀
- 이슈 관리보다 전체 소프트웨어 전달 체계가 핵심인 엔터프라이즈

## 4.3 OpenProject

공식 사이트: <https://www.openproject.org/>  
GitHub: <https://github.com/opf/openproject>  
라이선스: GPL-3.0  
유형: 오픈소스 프로젝트 관리/이슈 트래킹

### 개요

OpenProject는 프로젝트 관리, work package, Gantt/timeline, agile boards, time tracking, cost tracking, budgeting, wiki, forum, meeting 등을 제공한다. GitHub 저장소는 2026-06-05 조사 기준 **15.2k stars**, **3.3k forks** 수준이다.  
출처: [OpenProject GitHub repository](https://github.com/opf/openproject)

### Git 이슈 관리 기능

OpenProject는 GitHub/GitLab 통합을 제공한다.

- GitHub integration: GitHub pull request와 issue를 OpenProject work package에 연결
- GitLab integration: GitLab merge request와 issue를 OpenProject work package에 연결
- OpenProject work package에서 GitLab MR 상태, CI action 상태 등을 확인
- `OP#388` 같은 참조 코드를 issue/MR title 또는 description에 넣어 연결 가능

출처:

- [OpenProject GitHub integration docs](https://www.openproject.org/docs/system-admin-guide/integrations/github-integration/)
- [OpenProject GitLab integration docs](https://www.openproject.org/docs/system-admin-guide/integrations/gitlab-integration/)

핵심 기능:

- Work packages: 이슈/작업/버그/요구사항 단위
- Agile boards, Kanban boards, Scrum backlog/sprint board
- Gantt charts/timelines
- time tracking, cost tracking, budgeting, reports
- custom fields, custom workflows, permissions
- Jira migrator
- Nextcloud/OneDrive/SharePoint 문서 연동

### 가격

공식 가격 페이지 기준:

| 플랜           | 가격                | 주요 내용                                                                   |
| ------------ | ----------------- | ----------------------------------------------------------------------- |
| Community    | Free              | self-managed, community features/support                                |
| Basic        | €5.95/user/month  | 25 users minimum, basic enterprise add-ons, email support               |
| Professional | €10.95/user/month | 25 users minimum, phone support, professional add-ons                   |
| Premium      | €15.95/user/month | 100 users minimum, remote hands, install/upgrade assistance             |
| Corporate    | 견적                | 1000 users minimum, custom plugin support, dedicated support engineer 등 |

출처: [OpenProject Pricing](https://www.openproject.org/pricing/)

### 장점

- 개발 이슈뿐 아니라 조직/프로젝트/포트폴리오 관리 기능이 강함
- Gantt, 비용, 시간, 보고서, 회의록 등 PMO 요구에 강함
- GitHub/GitLab 이슈와 work package 연결 가능
- 공공, 연구, 제조, 건설 등 비개발 조직까지 확장 가능

### 단점/리스크

- Git forge가 아니므로 저장소 자체 관리는 GitHub/GitLab 등 외부 시스템 필요
- 개발팀이 원하는 빠르고 단순한 Linear/Jira식 UX와는 결이 다를 수 있음
- 최소 사용자 수와 유로 단위 과금 구조 확인 필요
- 이슈 자동 triage/AI 분류 같은 최신 기능은 Plane/Huly보다 약한 편

### 적합한 사용처

- 개발 이슈를 전체 프로젝트 계획, 일정, 비용, 산출물과 묶어 관리해야 하는 조직
- Jira에서 OSS 대체재로 이전하려는 공공/엔터프라이즈
- GitHub/GitLab은 코드 저장소로 유지하고 PM 계층만 오픈소스로 두려는 경우

## 4.4 Gitea

공식 사이트: <https://about.gitea.com/>  
GitHub: <https://github.com/go-gitea/gitea>  
라이선스: MIT  
유형: self-hosted Git forge

### 개요

Gitea는 GitHub/GitLab보다 가벼운 자체 호스팅 Git 서비스다. Git 저장소, 이슈, PR, 프로젝트, Actions CI/CD, package registry를 제공한다. 공식 가격 페이지는 Open Source self-managed 플랜이 “unlimited users and repositories”를 제공한다고 설명한다.  
출처: [Gitea Pricing](https://about.gitea.com/pricing?preview=true)

### Git 이슈 관리 기능

공식 비교 문서상 Gitea는 다음 이슈 트래커 기능을 지원한다.

- Issue tracker
- Issue templates
- Labels
- Time tracking
- Multiple assignees
- Comment reactions
- Lock discussion
- Batch issue handling
- Projects
- Issue search, global issue search
- Issue dependency

출처: [Gitea Comparison - Issue Tracker](https://docs.gitea.com/installation/comparison)

또한 Tea CLI는 issues, pull requests, releases, Gitea servers를 터미널에서 다룰 수 있다.  
출처: [Gitea Pricing page navigation](https://about.gitea.com/pricing?preview=true)

### 가격

| 플랜          | 가격                  | 주요 내용                                                                                                                   |
| ----------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Open Source | Free                | MIT license, self-hosted, unlimited users/repositories, issue tracking, PR, project management, Gitea Actions, packages |
| Enterprise  | $9.5~$19/user/month | SAML SSO, audit logs, Kubernetes autoscaling runners, priority support/SLA, 설치/업그레이드 지원                                 |

출처: [Gitea Pricing](https://about.gitea.com/pricing?preview=true)

### 장점

- MIT 라이선스라 상용 활용 부담이 상대적으로 낮음
- GitHub 스타일 UX를 가볍게 자체 호스팅 가능
- 이슈, 프로젝트, PR, CI/CD까지 기본 내장
- 소규모 팀, 내부 도구, 사내 Git 서버에 적합

### 단점/리스크

- Plane/OpenProject 같은 고급 PM/로드맵/포트폴리오 기능은 약함
- Gitea Ltd/CommitGo 이후 거버넌스 논쟁으로 Forgejo와 생태계가 분기됨
- GitHub/GitLab 대비 marketplace, 앱, 자동화 생태계는 작음

### 적합한 사용처

- 자체 Git 서버와 이슈 트래킹을 가볍게 운영하려는 팀
- GitHub/GitLab보다 낮은 운영비, 낮은 자원 사용량이 중요한 경우
- MIT 기반으로 커스터마이징/상용 내재화 가능성이 필요한 경우

## 4.5 Forgejo

공식 사이트: <https://forgejo.org/>  
코드 저장소: <https://codeberg.org/forgejo/forgejo>  
유형: Gitea fork, community-driven Git forge

### 개요

Forgejo는 Gitea에서 fork된 community-driven Git forge다. Codeberg가 Forgejo 기반으로 운영되고 있어, 공개 OSS hosting 대체재로서 의미가 크다.

### Git 이슈 관리 기능

Forgejo 공식 문서는 issues가 bug report뿐 아니라 enhancement, feature request, project direction discussion, question 등에 쓰인다고 설명한다. Issue tracker는 browsable/filterable list, labels, open/closed 상태, milestones, new issue 생성을 지원한다.  
출처: [Forgejo Issue Tracking Basics](https://forgejo.org/docs/v1.19/user/issue-tracking-basics/)

Codeberg 문서는 저장소가 Git source code를 담는 공간일 뿐 아니라 issue tracking과 wiki를 제공한다고 설명한다.  
출처: [Codeberg First Repository Docs](https://docs.codeberg.org/getting-started/first-repository/)

핵심 기능:

- Git 저장소, issue, pull request, wiki
- labels, milestones, issue filtering/search
- Forgejo Actions CI
- linked references: issue, PR, commit message 자동 참조
- Codeberg 같은 public forge 운영 사례

### 가격

Forgejo 자체는 self-hosted OSS로 별도 라이선스 비용 없이 운영할 수 있다. 공식적으로 Gitea Enterprise처럼 표준 상용 가격표를 제공하는 구조는 아니다. 운영 비용은 서버, 백업, 보안, 관리 인력 비용 중심이다. Codeberg는 별도 공용 호스팅 서비스이지만, 기업용 SLA SaaS 가격표 기반 제품이라기보다 비영리/커뮤니티 성격이 강하다.

### 장점

- 커뮤니티 주도 거버넌스에 민감한 OSS 프로젝트에 적합
- GitHub/GitLab 대체 self-host forge로 가볍고 실용적
- Codeberg 생태계를 통해 공개 OSS 프로젝트 사용 사례가 존재
- 이슈/PR/wiki/Actions 등 기본 forge 기능 제공

### 단점/리스크

- Gitea Enterprise 같은 상용 지원/가격 체계가 상대적으로 덜 명확
- 고급 PM 기능은 Plane/OpenProject/GitLab보다 약함
- 조직 규모가 커질수록 운영/보안/SLA를 자체 해결해야 함

### 적합한 사용처

- GitHub 의존도를 줄이고 싶은 OSS 프로젝트
- community governance와 data sovereignty를 중시하는 조직
- Gitea와 유사한 기능을 원하지만 Forgejo/Codeberg 생태계를 선호하는 경우

## 4.6 Redmine

공식 사이트: <https://www.redmine.org/>  
GitHub mirror: <https://github.com/redmine/redmine>  
라이선스: GPL v2  
유형: 전통적 프로젝트 관리/이슈 트래커

### 개요

Redmine은 Ruby on Rails 기반의 오래된 오픈소스 프로젝트 관리 웹 애플리케이션이다. 공식 사이트는 Redmine을 flexible project management web application이라고 설명하고, GPL v2로 공개되어 있다고 명시한다.  
출처: [Redmine Overview](https://www.redmine.org/)

### Git 이슈 관리 기능

공식 기능:

- Multiple projects/subprojects
- Role-based access control
- Flexible issue tracking system
- Gantt chart/calendar
- Time tracking
- Custom fields
- Repository browser and diff viewer
- Supported SCM: Subversion, CVS, Mercurial, Bazaar, Git
- Feeds/email notifications
- LDAP
- Multiple databases

출처:

- [Redmine Overview](https://www.redmine.org/)
- [Redmine Features](https://www.redmine.org/projects/redmine/wiki/Features)

Git 연동 측면에서는 기존 Git repository를 프로젝트에 attach하고 contents, changesets, diff, annotate/blame view를 볼 수 있다.

### 가격

Redmine 자체는 GPL v2 오픈소스로 무료다. 공식 다운로드 페이지는 최신 릴리스를 tar.gz/zip으로 제공하며, 2026-03-17 기준 6.1.2가 최신 stable 계열로 확인된다.  
출처: [Redmine Download](https://www.redmine.org/projects/redmine/wiki/Download)

비용 구조:

- 소프트웨어 라이선스: 무료
- 서버/DB/백업/운영 인력: 자체 부담
- 플러그인/테마/상용 호스팅/컨설팅: 벤더별 별도

### 장점

- 매우 오래 검증된 이슈 트래커
- 커스텀 필드/워크플로우/권한이 강함
- Git뿐 아니라 여러 SCM을 지원
- 플러그인 생태계가 넓고 기존 기업 도입 사례가 많음
- 무료 코어에 기능 제한이 거의 없음

### 단점/리스크

- UI/UX가 오래되어 현대적 제품팀에는 진입 장벽이 있음
- GitHub/GitLab 이슈 양방향 동기화 같은 최신 네이티브 기능은 기본 제공이 약함
- 플러그인 의존도가 높아 업그레이드 호환성 관리가 필요
- AI triage, 현대적 로드맵/사이클 관리 경험은 부족

### 적합한 사용처

- 안정성, 커스터마이징, 비용 절감이 UX보다 중요한 조직
- 이미 Redmine 플러그인/프로세스가 있는 기업
- Git 저장소와 이슈 추적을 전통적인 프로젝트 관리 방식으로 결합하려는 경우

## 4.7 OneDev

공식 사이트: <https://onedev.io/>  
GitHub: <https://github.com/theonedev/onedev>  
유형: self-hosted all-in-one DevOps platform

### 개요

OneDev는 Git server, CI/CD, Kanban, package registry를 통합한 self-hosted DevOps 플랫폼이다. 공식 가격 페이지는 Community Edition에서 built-in CI/CD, package registry, code search, issue customization, service desk 등을 제공한다고 설명한다.  
출처: [OneDev Pricing](https://onedev.io/pricing)

### Git 이슈 관리 기능

공식 기능:

- Issue state/field/link customization
- issue workflow를 이벤트에 따라 자동 전환
- service desk: 이메일로 이슈 생성/토론
- issue boards와 iterations로 Scrum/Kanban 지원
- command palette로 projects/files/issues/PR/builds/packages 이동
- project hierarchy와 setting inheritance
- MCP server: AI agent에서 issues, pull requests, builds 관리

출처:

- [OneDev Pricing](https://onedev.io/pricing)
- [OneDev Boards and Iterations Docs](https://docs.onedev.io/tutorials/issue/board-and-iteration)

### 가격

| 플랜                 | 가격            | 주요 내용                                                                                                                                      |
| ------------------ | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Community Edition  | Free          | CI/CD, package registry, code search, issue customization, service desk, LDAP/AD, SSO/OIDC, AI capabilities, MCP server, community support |
| Enterprise Edition | $6/user/month | HA/scalability 등 enterprise 기능 추가                                                                                                          |

출처: [OneDev Pricing](https://onedev.io/pricing)

### 장점

- GitLab보다 가볍고 단순한 올인원 대체재로 포지셔닝
- 이슈 상태/필드/링크 커스터마이징이 강함
- 이메일 기반 service desk가 내장되어 외부 요청을 이슈로 전환하기 좋음
- MCP server를 제공해 AI agent 기반 이슈 관리와 연결 가능성이 큼
- 가격이 GitLab Premium보다 낮음

### 단점/리스크

- GitLab/Gitea/Plane보다 시장 인지도와 커뮤니티 규모가 작음
- Java 기반 운영, 자체 CI/CD 개념 등 팀 선호도 확인 필요
- 문서/생태계/외부 연동 앱 수는 대형 플랫폼보다 제한적

### 적합한 사용처

- GitLab은 과하고 Gitea는 PM/CI 기능이 부족하다고 느끼는 팀
- 이슈 워크플로우 커스터마이징과 service desk가 중요한 내부 개발 조직
- AI agent와 issue/build 관리 연동을 실험하려는 팀

## 4.8 Taiga

공식 사이트: <https://taiga.io/>  
GitHub 조직: <https://github.com/taigaio>  
유형: open-source agile project management

### 개요

Taiga는 Scrum/Kanban 중심의 오픈소스 애자일 프로젝트 관리 도구다. GitHub 조직 설명은 “Your Agile, Free and Open Source Project Management Tool”로 소개한다.  
출처: [Taiga GitHub organization](https://github.com/taigaio)

### Git 이슈 관리 기능

Taiga의 GitHub integration 문서는 GitHub repository와 Taiga project를 연결해 다음을 지원한다고 설명한다.

- commit message로 epic/user story/issue/task 상태 변경
- commit을 epic/user story/issue/task에 attach
- GitHub issue 생성 시 Taiga issue 생성

출처: [Taiga GitHub Integration](https://docs.taiga.io/integrations-github.html)

커뮤니티 문서/논의에서는 GitHub, GitLab, Gitea/Gogs 연동이 언급된다. GitHub 연동이 가장 포괄적이라는 평가가 있다.  
출처: [Taiga Community - Git integration](https://community.taiga.io/t/best-opensource-git-integration/1532)

### 가격

공식 deployment/pricing 페이지 기준:

| 플랜                         | 가격        | 주요 내용                                                                 |
| -------------------------- | --------- | --------------------------------------------------------------------- |
| Enthusiast                 | €5/month  | 5 public projects, 5 private projects, 100MB storage, unlimited users |
| Basic                      | €20/month | unlimited projects, 500MB storage, unlimited users                    |
| Premium                    | €60/month | unlimited projects, 3GB storage, unlimited users                      |
| Private Cloud / On premise | 견적 또는 문의  | 전용/온프레미스 옵션                                                           |

출처: [Taiga Deployment & Pricing Options](https://taiga.io/deployment-pricing-options/)

### 장점

- Scrum/Kanban, user story, sprint 중심 팀에 적합
- 사용자 수 무제한 가격 구조는 소규모 조직에 매력적
- GitHub issue/commit과 Taiga issue/story 연결 가능
- 오래된 OSS 애자일 관리 도구로 성숙도 있음

### 단점/리스크

- GitHub/GitLab 양방향 동기화 깊이는 Plane보다 약함
- 제품 시장의 최신 관심도는 Plane/Huly보다 낮아 보임
- 저장소 자체 호스팅 도구가 아니므로 Git forge는 별도 필요

### 적합한 사용처

- Scrum/Kanban과 user story 중심으로 관리하는 소규모/중형 팀
- Jira보다 단순하고 저렴한 애자일 보드가 필요한 조직
- GitHub issue를 프로젝트 관리 이슈로 부분 연동하려는 경우

## 4.9 Huly

공식 사이트: <https://v1.huly.io/>  
문서: <https://docs.huly.io/>  
GitHub: <https://github.com/hcengineering/platform>  
유형: all-in-one open source project management/collaboration

### 개요

Huly는 Linear, Jira, Slack, Notion, Motion 대체를 지향하는 오픈소스 all-in-one 프로젝트 관리 플랫폼이다. 공식 사이트는 “2-way GitHub integration”과 GitHub repositories 동기화를 전면에 내세운다.  
출처: [Huly official site](https://v1.huly.io/)

### Git 이슈 관리 기능

공식 사이트/문서 기준:

- 2-way GitHub integration
- GitHub 경험을 보강하는 advanced front end로 포지셔닝
- project management, inbox/chat, team planning, documents, calendar, spotlight
- workflow management, collaborative editing, video conferencing

출처:

- [Huly official site](https://v1.huly.io/)
- [Huly Docs - What is Huly?](https://docs.huly.io/getting-started/introduction-huly/)

### 가격

2026-06-05 조사 기준 공식 사이트에서 GitHub/Plane/GitLab처럼 명확한 seat-based 가격표를 쉽게 확인하기 어렵다. 따라서 도입 검토 시 다음을 추가 확인해야 한다.

- self-host 기능의 안정성
- 상용 cloud/enterprise 가격
- GitHub integration이 무료인지, OSS 프로젝트/비공개 프로젝트별 제한이 있는지
- SSO, audit log, RBAC, support SLA 제공 여부

### 장점

- GitHub 이슈를 더 넓은 협업/문서/일정/채팅 맥락에 연결하려는 방향성이 강함
- Linear + Notion + Slack 대체를 지향해 제품 확장성이 큼
- 신생 OSS로 성장 여지가 있음

### 단점/리스크

- 상대적으로 신생 제품이라 장기 안정성/운영 경험 검증 필요
- 가격/엔터프라이즈 기능 공개 수준이 Plane/GitLab/OpenProject보다 덜 명확
- 범위가 넓어 “이슈 관리만” 원하는 팀에는 과할 수 있음

### 적합한 사용처

- GitHub 이슈를 문서/채팅/일정/팀 플래닝까지 한 workspace에서 다루고 싶은 팀
- 최신 협업 UX와 OSS self-host 가능성을 모두 보는 스타트업
- 장기적으로 Jira/Linear/Notion/Slack 일부를 통합하고 싶은 조직

## 5. 기능 비교표

| 기능                    | Plane          | GitLab                     | OpenProject | Gitea               | Forgejo   | Redmine                | OneDev           | Taiga | Huly  |
| --------------------- | -------------- | -------------------------- | ----------- | ------------------- | --------- | ---------------------- | ---------------- | ----- | ----- |
| 자체 Git 저장소            | 아니오            | 예                          | 아니오         | 예                   | 예         | 제한적 repository browser | 예                | 아니오   | 아니오   |
| GitHub 이슈 연동          | 예              | import/external 가능         | 예           | 마이그레이션/외부 연동 중심     | 제한적/외부 연동 | 플러그인 중심                | import 가능성 확인 필요 | 예     | 예     |
| GitLab 이슈 연동          | 예              | 자체                         | 예           | 비교/마이그레이션 중심        | 제한적       | 플러그인 중심                | import 가능성 확인 필요 | 예     | 확인 필요 |
| 이슈/작업 관리              | 강함             | 강함                         | 강함          | 중간                  | 중간        | 강함                     | 강함               | 중간    | 강함    |
| Kanban/Scrum          | 강함             | 강함                         | 강함          | 중간                  | 중간        | 플러그인/기본 제한             | 강함               | 강함    | 강함    |
| 로드맵/에픽                | 중간~강함          | 강함                         | 강함          | 약함~중간               | 약함~중간     | 중간                     | 중간               | 중간    | 중간    |
| 커스텀 필드/워크플로우          | Pro 이상 강함      | Premium/Ultimate 강함        | 강함          | 중간                  | 중간        | 강함                     | 강함               | 중간    | 확인 필요 |
| Time tracking         | Pro 이상         | 예                          | 예           | 예                   | 예/유사      | 예                      | 확인/강함            | 제한적   | 확인 필요 |
| Service desk / intake | Business 이상 강함 | 일부/Service Desk는 GitLab 기능 | 제한적         | create via email 일부 | 제한적       | email issue            | 강함               | 제한적   | 확인 필요 |
| Wiki/docs             | 예              | 예                          | 예           | 예                   | 예         | 예                      | 예                | 제한적   | 강함    |
| CI/CD                 | 아니오            | 예                          | 아니오         | 예                   | 예         | 아니오                    | 예                | 아니오   | 아니오   |
| Self-host             | 예              | 예                          | 예           | 예                   | 예         | 예                      | 예                | 예     | 예     |
| OSS 라이선스              | AGPL-3.0       | 오픈코어                       | GPL-3.0     | MIT                 | OSS       | GPL v2                 | OSS/상용 혼합        | OSS   | OSS   |
| 현대적 UX                | 높음             | 중간~높음                      | 중간          | 중간                  | 중간        | 낮음~중간                  | 중간               | 중간    | 높음    |

## 6. 가격 비교

| 솔루션         | 무료 self-host | 유료 시작가                             | 가격 관련 주의                                                                                    |
| ----------- | ------------ | ---------------------------------- | ------------------------------------------------------------------------------------------- |
| Plane       | 예            | Pro $6/seat/month                  | Free는 12 users 제한, 고급 PM/통합 기능은 Pro 이상 확인                                                   |
| GitLab      | 예/Free tier  | Premium $29/user/month             | GitLab.com Free private group은 사용자/스토리지/CI 제한, self-managed Free는 제한 조건 별도 확인, Ultimate는 견적 |
| OpenProject | 예            | €5.95/user/month, 25 users minimum | Basic 이상 최소 사용자 수 존재                                                                        |
| Gitea       | 예            | Enterprise $9.5~$19/user/month     | Enterprise는 1년 약정 조건 표시                                                                     |
| Forgejo     | 예            | 공식 표준 seat 가격 없음                   | 상용 SLA/지원은 별도 검토                                                                            |
| Redmine     | 예            | 공식 seat 가격 없음                      | 플러그인/호스팅/컨설팅 비용 별도                                                                          |
| OneDev      | 예            | Enterprise $6/user/month           | Community에도 기능이 많으나 enterprise 차이 확인 필요                                                     |
| Taiga       | 예/온프레미스 문의   | SaaS €5/month                      | 저장공간/프로젝트 수 기준, users unlimited                                                             |
| Huly        | 예로 홍보        | 명확한 공식 가격표 추가 확인 필요                | 상용 제한/지원/통합 가격 확인 필요                                                                        |

## 7. 시장성/사업성 평가

### 7.1 도입 시장

| 고객군            | 니즈                                                         | 적합 솔루션                                         |
| -------------- | ---------------------------------------------------------- | ---------------------------------------------- |
| OSS maintainer | 이슈 triage, label, good first issue, PR 대기열, contributor 응답 | Plane, GitHub Projects, Forgejo/Codeberg, Huly |
| 스타트업 제품팀       | GitHub/GitLab 이슈와 로드맵/스프린트 연결                              | Plane, Huly, Taiga                             |
| 내부 개발 플랫폼팀     | 자체 Git, CI/CD, 이슈, 보안 통합                                   | GitLab, Gitea, Forgejo, OneDev                 |
| 공공/규제 산업       | self-host, 데이터 주권, 감사, 프로젝트 보고                             | OpenProject, GitLab, Redmine                   |
| SI/제조/건설/엔지니어링 | 일정, 비용, 문서, 이슈, Gantt                                      | OpenProject, Redmine                           |
| 소규모 개발팀        | 저비용 GitHub 대체, 자체 이슈 관리                                    | Gitea, Forgejo, OneDev                         |

### 7.2 새로운 OSS/상용 제품 관점의 차별화 포인트

기존 솔루션은 “이슈를 저장하고 보드화”하는 기능은 충분하다. 차별화는 다음에서 나온다.

1. **AI 기반 이슈 정리**
   - 중복 이슈 자동 병합 추천
   - 라벨/컴포넌트/심각도/우선순위 추천
   - 재현 단계/로그/스크린샷 누락 감지
   - stale issue 정리와 maintainer 응답 초안
2. **다중 forge 통합**
   - GitHub, GitLab, Gitea, Forgejo, Bitbucket, Jira를 한 화면에서 관리
   - 같은 제품 기능과 연결된 이슈를 저장소 경계를 넘어 묶기
3. **제품/고객 관점 재구성**
   - Git issue를 고객, 릴리스, 기능, SLA, 수익 영향도와 연결
   - “코드 저장소 이슈”를 “비즈니스 이슈”로 번역
4. **국내/엔터프라이즈 특화**
   - 한국어 UX
   - 온프레미스/폐쇄망 설치
   - LDAP/AD/SAML/OIDC
   - 감사 로그, 결재, 보고서, 전자문서 연동
5. **개발자 에이전트 연동**
   - 이슈에서 바로 브랜치/PR 생성
   - agent가 이슈 분석, 원인 후보, 관련 커밋/파일 추천
   - MCP를 통한 Git issue management 자동화

### 7.3 제품 기회 평가

| 아이디어                                              | 경쟁 제품                               | 가능성   | 비고                                |
| ------------------------------------------------- | ----------------------------------- | ----- | --------------------------------- |
| Git issue AI triage SaaS/self-host                | GitHub Copilot, Linear AI, Plane AI | 높음    | 단독 제품보다 기존 forge 연동 앱으로 시작하기 좋음   |
| OSS maintainer issue cockpit                      | GitHub Projects, Plane              | 중간~높음 | 오픈소스 프로젝트 운영자 대상. 무료/후원/Pro 모델 가능 |
| Enterprise self-host issue intelligence           | GitLab, OpenProject, Redmine        | 높음    | 보안/폐쇄망/감사 로그가 핵심                  |
| Gitea/Forgejo issue enhancer                      | Gitea/Forgejo 자체 기능                 | 중간    | Forgejo/Codeberg 생태계 성장에 베팅       |
| GitHub/GitLab -> Plane/OpenProject migration tool | 각 제품 importer                       | 중간    | 도구성 제품이라 시장 규모 제한                 |

## 8. 도입 추천 시나리오

### 시나리오 A: GitHub/GitLab은 유지하고 이슈 정리만 고도화

추천: **Plane**

이유:

- GitHub/GitLab sync가 명시적이고 현대적
- Jira/Linear 대체를 표방해 팀 도입 저항이 낮음
- Free/Pro로 시작 가능
- work item, cycles, modules, views, pages, intake까지 균형이 좋음

대안:

- OpenProject: PMO/엔터프라이즈 계획 관리가 중요할 때
- Huly: chat/docs/calendar까지 통합하려는 경우

### 시나리오 B: GitHub/GitLab 자체를 대체하고 자체 Git 서버 운영

추천: **Gitea 또는 Forgejo**

이유:

- Git 저장소와 이슈/PR/wiki/project를 자체 호스팅
- 운영 비용과 복잡도가 GitLab보다 낮음
- Gitea는 MIT와 Enterprise support가 강점
- Forgejo는 community governance와 Codeberg 생태계가 강점

대안:

- GitLab: CI/CD, 보안, 컴플라이언스까지 통합해야 할 때
- OneDev: GitLab보다 가벼운 all-in-one이 필요할 때

### 시나리오 C: 프로젝트/일정/비용/보고 중심

추천: **OpenProject**

이유:

- GitHub/GitLab issue/MR 연동 가능
- work package, Gantt, 비용, 시간, 보고서, 회의록 기능이 강함
- 공공/엔터프라이즈/비개발 부서와 함께 쓰기 좋음

대안:

- Redmine: 비용 최소화와 커스텀 워크플로우가 중요할 때
- GitLab Ultimate: DevSecOps와 portfolio management를 한 플랫폼에 두고 싶을 때

### 시나리오 D: 레거시/저비용/커스터마이징 우선

추천: **Redmine**

이유:

- 무료 OSS 코어
- 오래된 검증과 플러그인 생태계
- Git repository browser/diff/blame 연동
- 커스텀 필드/워크플로우/권한이 강함

주의:

- UX 개선과 플러그인 관리가 필요
- GitHub/GitLab native sync는 별도 플러그인/개발 검토 필요

## 9. 최종 결론

### 도입 관점

1. **가장 균형 잡힌 현대적 선택: Plane**
   - GitHub/GitLab 이슈를 제품/프로젝트 관리 계층으로 끌어올리는 목적에 가장 적합하다.
   - 새로운 “Git Issue Sentinel”류 제품의 직접 경쟁/벤치마크 대상으로 봐야 한다.
2. **가장 강력한 올인원 DevOps: GitLab**
   - 이슈 관리만이 아니라 SCM, CI/CD, security, compliance까지 묶을 때 강하다.
   - 비용과 운영 복잡도 때문에 모든 팀에 맞지는 않는다.
3. **가장 강한 조직형 PM 대체재: OpenProject**
   - 개발 이슈를 일정, 비용, 보고, 포트폴리오 관리와 연결할 때 강하다.
4. **가벼운 자체 Git forge: Gitea/Forgejo**
   - GitHub/GitLab 대체 자체 호스팅에는 좋지만, 고급 이슈 정리/제품 로드맵 기능은 보강이 필요하다.
5. **전통적 안정형: Redmine**
   - 무료, 커스터마이징, 검증된 운영을 중시한다면 여전히 유효하다.

### 사업화 관점

단순한 OSS 이슈 트래커를 새로 만드는 것은 경쟁력이 낮다. 이미 GitLab/Gitea/Forgejo/Redmine이 기본 이슈 트래킹을 제공하고, Plane/OpenProject/Huly가 프로젝트 관리 계층을 제공한다. 더 유망한 방향은 다음이다.

- GitHub/GitLab/Gitea/Forgejo의 이슈를 모두 가져오는 **통합 issue intelligence layer**
- AI 기반 **triage, duplicate detection, label/priority/owner 추천**
- 제품/고객/릴리스/SLA 기준으로 Git issue를 재구성하는 **product operations layer**
- 폐쇄망 설치, SSO, 감사 로그, 한국어 보고서까지 포함한 **enterprise self-hosted issue management**

따라서 “Git Issue Sentinel” 같은 제품을 기획한다면, 1차 MVP는 다음 기능 조합이 가장 시장성이 높다.

1. GitHub/GitLab 이슈 import/sync
2. 저장소별 이슈를 cross-repository inbox로 통합
3. AI 라벨/우선순위/담당자 추천
4. 중복 이슈/관련 PR/관련 커밋 추천
5. stale issue, blocked issue, unassigned issue 자동 큐레이션
6. Plane/OpenProject/GitLab로 export 또는 양방향 sync
7. self-host 배포 옵션과 audit log 제공

## 10. 참고 링크

- GitHub Octoverse 2025: <https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/>
- Plane Pricing: <https://plane.so/pricing>
- Plane GitHub repository: <https://github.com/makeplane/plane>
- GitLab Issues Docs: <https://docs.gitlab.com/user/project/issues/>
- GitLab Issue Boards Docs: <https://docs.gitlab.com/user/project/issue_board/>
- GitLab Pricing: <https://about.gitlab.com/pricing/>
- OpenProject Pricing: <https://www.openproject.org/pricing/>
- OpenProject GitHub repository: <https://github.com/opf/openproject>
- OpenProject GitHub integration: <https://www.openproject.org/docs/system-admin-guide/integrations/github-integration/>
- OpenProject GitLab integration: <https://www.openproject.org/docs/system-admin-guide/integrations/gitlab-integration/>
- Gitea Pricing: <https://about.gitea.com/pricing?preview=true>
- Gitea feature comparison: <https://docs.gitea.com/installation/comparison>
- Forgejo issue tracking basics: <https://forgejo.org/docs/v1.19/user/issue-tracking-basics/>
- Codeberg first repository docs: <https://docs.codeberg.org/getting-started/first-repository/>
- Redmine Overview: <https://www.redmine.org/>
- Redmine Features: <https://www.redmine.org/projects/redmine/wiki/Features>
- Redmine Download: <https://www.redmine.org/projects/redmine/wiki/Download>
- OneDev Pricing: <https://onedev.io/pricing>
- OneDev Boards and Iterations: <https://docs.onedev.io/tutorials/issue/board-and-iteration>
- Taiga GitHub Integration: <https://docs.taiga.io/integrations-github.html>
- Taiga Pricing: <https://taiga.io/deployment-pricing-options/>
- Huly official site: <https://v1.huly.io/>
- Huly Docs: <https://docs.huly.io/getting-started/introduction-huly/>