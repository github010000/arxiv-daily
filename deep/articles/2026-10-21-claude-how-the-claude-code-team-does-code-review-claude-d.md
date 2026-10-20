# How the Claude Code team does code review / claude.dev Blog

- **원문**: [claude.dev](https://claude.dev/blog/solving-code-review-at-anthropic/)
- **저자**: Addy Osmani
- **게시일**: 2026-10-15
- **분석일**: 2026-10-21 08:31 KST
- **분석 모델**: deepseek
- **분석 근거**: 원문 본문 19,064자

---

## 한 줄 요약

Claude Code 팀은 agent가 만들어 내는 대량의 변경을 사람이 줄 단위로 읽는 마지막 gate에서 검사하는 대신, 자동 검증을 연속으로 돌리고 사람은 위험도가 높은 판단과 증거 확인에만 쓰는 다층 구조로 code review를 재설계했다. 그 결과 merged pull request가 2배 이상 늘었는데 revert 비율은 40% 이상 줄었다.

## 배경과 핵심 개념

- **agent가 작성하는 코드의 양과 전통적 code review의 불일치**  
  전통적인 code review는 사람이 diff를 한 줄씩 읽고 승인하는 방식이다. 이 방식은 변경량이 적고 작성자가 모두 사람일 때 성립한다. 그런데 agent가 하루 수천 건에서 수만 건의 변경을 만들면 reviewer가 물리적으로 따라잡을 수 없다. 이 글은 그 불일치를 해결하기 위해 review의 목적을 "사람이 읽었는가"가 아니라 "이 변경에서 무엇이 잘못될 수 있고, 자동 체크와 인간 중 어느 쪽이 잡아야 하는가"로 바꾸었다고 설명한다.

- **Claude Code와 자기 적용**  
  Claude Code는 Anthropic이 만든 coding agent 제품으로, CLI와 desktop, web, mobile app을 포함한다. 이 글의 저자들은 Claude Code를 만드는 팀 내부 사람들이다. 이 팀은 Claude Code를 만들 때 Claude가 자기 코드의 약 90%를 작성한다고 밝혔다. 즉 이 글은 일반적인 code review 이론이 아니라, agent가 자기 제품을 만드는 환경에서 실제로 운영되는 프로세스의 사례다.

- **code review의 두 가지 일**  
  code review는 결함을 배포 전에 찾아 고치는 일과, 팀원들이 codebase 전체를 이해하는 공유 맥락을 만드는 일을 동시에 한다. 전자는 기계로 대체하거나 보조할 수 있지만, 후자는 단순히 자동화하면 사라질 수 있다. 이 글은 전자에 집중하고, 후자는 마지막 부분에서 explainer와 open channel 문화로 대체하는 방식을 다룬다.

- **인간 시간이 희소 자원이고 revert와 incident가 guardrail**  
  목표는 단순히 인간 승인을 줄이는 것이 아니라, 인간이 review에 쓰는 시간을 줄이면서도 잘못된 코드로 인한 사고가 늘지 않도록 하는 것이다. review 시간을 줄였더라도 나쁜 merge가 배포를 망가뜨리면 수습 시간이 늘어나므로 총비용이 줄지 않는다. 그래서 인간 승인 수, 사람이 단 comment 수, pull request가 사람을 기다리는 시간을 지표로 보고, revert와 incident 비율을 상한선으로 둔다.

- **다층 방어와 위험도 기반 승인**  
  하나의 gate가 아니라 여러 layer가 서로 다른 실수를 잡고, 마지막에는 빠른 rollback이 남은 실수를 처리한다. 변경의 위험도는 작성자가 아니라 classifier가 판단해 승인 요건을 달리한다. 예를 들어 문서 수정은 자동 검사만 통과해도 merge되지만 billing이나 auto-updater 같은 민감 코드는 사람 승인을 요구한다.

- **reviewer 자체의 평가와 수렴**  
  agent가 review하면 인간과 달리 사소한 지적에도 자동으로 고치려 하고, reviewer와 작성자가 서로 agent일 때 무한 반복할 수 있다. 이 팀은 reviewer의 finding이 실제 버그인지, 사람이 어떻게 반응했는지, 깨끗한 pull request에 해로운 지적을 하는지 측정한다. 또 review 라운드 수를 제한하고 새 diff만 다시 보게 해서 review가 수렴하도록 만든다.

## 글의 내용

원문은 먼저 이 팀의 전제를 제시한다. Claude Code 팀은 Claude가 Claude Code의 코드 약 90%를 작성한다는 기존 발표를 언급하면서, 이 정도 물량을 어떻게 review하는지 질문한다. 6월부터 9월까지 merged pull request는 두 배 이상 늘었고, 그중 revert된 비율은 40% 이상 줄었다. 이 결과는 review를 없앤 것이 아니라 읽는 일 대부분을 기계로 옮기고 사람의 시간과 주의를 아끼는 방향으로 프로세스를 바꾸고, 실수를 빠르고 쉽게 되돌리는 데 투자한 결과라고 말한다.

### 목적 재정의

원문은 agent가 코드를 쓰면 변경 수가 빠르게 늘어난다고 말한다. Anthropic의 monorepo에서 하루 landing되는 변경 수가 5월 약 2,500건에서 9월 30,000건 이상으로 늘었다. Claude Code 저장소는 monorepo와 분리되어 있지만 같은 압력을 받는다. 이런 속도에서는 사람이 줄 단위로 읽는 review는 사실상 불가능하다. 저자는 "사람이 코드를 읽었나"는 부정확한 proxy라고 본다. 대신 "이 변경에서 무엇이 잘못될 수 있고, 자동 체크와 인간 중 어느 쪽이 잡아야 하는가"를 질문한다. 이 질문은 review의 시점도 바꾸었다. 예전에는 코드를 다 쓴 뒤 pull request를 열고 동료를 기다리는 마지막 gate였지만, 지금은 글을 쓰는 동안 맞춤법 검사처럼 계속 실행된다. 사람이 보기 전에 대부분 변경은 Claude가 최소 한 번 review하고 고친 뒤다.

### 희소 자원: 인간 시간과 주의

목표는 모든 pull request가 최대 한 명의 사람만 통과해도 merge되도록 하는 것이다. 여기서 한 명은 reviewer가 아니라 author일 수도 있다. 단, revert나 incident가 늘어나면 안 된다. 측정 지표는 pull request당 human approval 중앙값과 95번째 백분위수, 사람이 남긴 comment, pull request가 사람을 기다린 시간이다. revert와 incident 비율은 guardrail이다. 이 지표가 오르면 인간 시간을 아낀 것이 아니라 수습 비용으로 옮긴 것뿐이다.

### 변경의 생애와 layer

원문은 일반적인 변경이 author의 노트북이나 Slack의 Claude session에서 시작해 사용자에게 도달하기까지 여러 layer를 거친다고 설명한다. 원문의 그림 A와 B는 대부분 경로가 자동화되어 있고 사람의 주의는 위험이 있는 곳에만 배치된다는 점을 보여준다. 각 layer는 어떤 문제는 잡고 어떤 문제는 놓친다. 모든 layer를 뚫고 나간 실수는 rollback이 처리한다.

**첫 번째 layer: 다른 사람이 보기 전에 검증**

첫 번째 layer는 pull request 밖에서 일어나는 사전 검증이다. 작성자는 Claude에게 테스트, 빌드, 디자인과 비교할 스크린샷처럼 실행 가능한 것을 주고, "완료했다"는 말 대신 결과물을 보여 달라고 한다. 그 다음 subagent가 diff를 새 맥락에서 review한다. 추가로 내부 skill들이 버그를 찾고, diff를 단순화하고, end-to-end로 동작을 확인한다. UI 변경은 디자인 가이드라인과 비교한다. Claude Code를 만든 Boris Cherny는 Claude가 자기 작업을 검증할 방법을 주는 것이 좋은 결과를 얻는 가장 중요한 일이며 최종 결과를 2~3배 좋게 만든다고 추정한다.

자동 reviewer가 같은 것을 반복해서 지적하면 소스에서 고친다. CLAUDE.md는 프로젝트의 스택, 관례, 목표와 목표가 아닌 것을 짧게 담는다. reviewer는 이 파일을 매번 읽고, 코드가 관례를 어기면 지적하며 이 파일의 내용이 outdated되면 알린다. REVIEW.md는 code owner가 작성한다. 좋은 REVIEW.md는 짧고, 해당 repo에서 실패가 어떤 모습인지 말하고, nit 개수를 제한하고, 건너뛸 경로를 나열하고, 이름을 추측한 판단 대신 파일과 줄 번호를 증거로 요구한다. 요약은 개수로 시작하고 고칠 것이 없으면 그렇게 명확히 말하라고 한다.

**두 번째 layer: 모든 pull request에 AI review**

두 번째 layer는 모든 pull request에 달리는 AI review다. 사람이 보기 전에 Claude가 repo 전체와 pull request의 stated intent를 비교해 diff를 읽고, 변경이 문제를 해결하는지, 시스템에 맞는지, 다른 것을 깨뜨리지는 않는지 질문한다. bug fix에서는 근본 원인을 찾는다. 공개 제품인 Code Review for Claude Code가 Team과 Enterprise plan에서 generally available이며 /code-review는 가벼운 버전을 로컬에서 돌린다. research preview 때 substantive review comment를 받는 internal pull request 비율이 16%에서 54%로 올랐다. 1,000줄이 넘는 pull request 대부분은 finding이 나오고, 작은 pull request는 대부분 깨끗하게 통과한다. 관리자는 repo별로 Quick, Standard, Deep을 고른다. Deep은 놓친 버그 하나의 비용이 추가 review 시간보다 클 때 가치가 있다.

finding은 사람이 보기 전에 다른 agent들이 검증한다. 여러 agent가 동시에 버그를 찾고, 별도 그룹이 각 finding에 투표해서 실제라고 동의해야만 게시된다. 스타일, 진단, 정리만 말하는 finding은 버린다. 이 필터는 내부 pull request로 A/B 테스트를 먼저 거쳤다. 남은 finding은 중요도로 정렬된 inline comment로 표시되고 Important, Nit, Pre-existing 태그가 붙는다. pull request가 만들지 않은 기존 버그가 author 잘못처럼 보이지 않게 하려는 것이다. 모든 finding은 답을 해야 하며, 고치거나 이유를 적고 thread를 닫아야 한다. 종종 agent가 밤새 수정을 맡아 author는 이미 한 라운드 더 지난 pull request로 돌아온다.

**세 번째 layer: merge를 막을 수 있는 security review**

세 번째는 merge를 막을 수 있는 security review다. bug review나 security review의 finding이 열려 있으면 pull request는 merge될 수 없다. false positive는 짧은 이유를 적어 thread를 닫아 해소한다. 이후 commit이 문제를 고치면 finding은 자동으로 사라진다. 오픈소스 security review GitHub Action이나 /security-review 명령으로 같은 설정을 쓸 수 있다.

**네 번째 layer: merge를 막는 자동 검사**

네 번째는 lint, type check, 플랫폼별 테스트, 빌드 같은 일반적인 자동 검사다. 일반적인 CLI pull request는 사람이 보기 전에 수십 개의 check를 실행할 수 있다. pull request 템플릿은 변경을 어떻게 테스트했고 어떻게 되돌릴 것인지 묻는다. Claude가 채울 수 있지만 변경을 요청한 사람이 최종 확인한다.

**다섯 번째 layer: 위험도에 따라 달라지는 human approval**

다섯 번째는 위험도에 따라 달라지는 human approval이다. 원문은 표로 위험도를 나눈다. Low는 docs, copy, internal-only feature, flag 뒤 additive change로 automated review와 security check가 green이면 human approval이 필요 없다. Medium은 일반 product change, caching, auto-updater, system prompt 같은 민감 영역으로 한 명이 필요하다. High는 최고 위험 코드와 항상 두 명이 필요한 경로다. 등급은 author가 아니라 classifier가 변경을 읽고 정한다. 두 번째 classifier는 등급이 사람이 필요 없다고 해도 이 특정 변경에는 추가 사람이 필요한지 판단한다. 일부 경로는 항상 사람이 필요하다. owner가 민감하다고 표시한 경로와 모든 CI workflow file이 그렇다. 신뢰할 수 있는 author만 가벼운 경로를 쓸 수 있다. repo의 trusted list에 있는 팀 멤버가 주로 해당된다. 다른 사람은 두 번째 승인이 필요하다. agent의 approval은 결코 사람 몫으로 세지 않는다. 자동 review는 low-risk 변경을 통과시킬 수 있지만 사람이 필요한 자리를 채울 수는 없다. 누가 변경을 요청했든 그 사람이 책임을 진다. agent가 작성한 pull request에서는 agent를 지시한 사람이 필수 reviewer 한 명으로 계산된다. 직접 코드를 친 것과 같다. approval은 특정 commit에만 적용된다. 시스템이 보증할 수 없는 새 commit이 들어오면, 예컨대 agent push나 손으로 해결한 merge가 있으면 approval을 다시 받아야 한다.

**여섯 번째 layer: merge queue, flag, 단계적 release**

여섯 번째는 merge queue와 flag, 단계적 release다. merge queue는 변경을 main에 적용한 상태로 모든 check를 다시 실행한다. 그래서 따로는 통과한 두 pull request가 합쳐져 깨진 main을 만들 수 없다. 새 동작은 보통 Anthropic 직원에게 먼저 flag로 배포된다. 내부에서 많이 쓰면 user보다 먼저 regression이 드러나는 경우가 많다. 공개 release는 latest channel부터 나가고 stable channel은 그 뒤를 따른다.

**일곱 번째 layer: 빠른 탐지와 rollback**

일곱 번째는 빠른 탐지와 rollback이다. 일부 실수는 여전히 통과하므로 마지막 layer는 빠르게 잡는 일이다. flag는 deploy 없이 끌 수 있다. 주간 on-call rotation이 있고 incident review는 어느 layer가 그 문제를 잡았어야 했는지 묻는다. 원문은 나쁜 merge를 알아채는 것이 여전히 author와 on-call, revert와 incident dashboard에 의존한다고 인정한다. revert나 incident가 늘어날 때 approval을 자동으로 강화하는 circuit breaker는 아직 없다. 엔지니어들은 zero-human merge를 더 확대하기 전에 그런 장치를 원한다. 한 동료는 속도를 늦추면 error rate이 낮아지지만 대부분 모든 일을 덜 하기 때문이라고 말했다. 팀은 문제를 알아채고 고치는 속도를 높이는 쪽을 선호한다.

### reviewer를 개선하는 루프

원문은 reviewer를 개선하는 루프를 설명한다. incident는 이제 eval이 된다. 과거 incident를 일으킨 pull request와 fix-forward, revert를 모으고 해당 코드를 아는 엔지니어가 각 버그와 위치를 확인했다. reviewer는 이 세트를 매일 돌린다. eval은 incident마다 커지고 reviewer를 바꿀 때마다 전체에 대해 점수를 매긴다.

noise는 사람이 comment에 어떻게 반응했는지로 측정한다. 버그를 더 많이 잡으려면 아닌 것도 많이 잡으면 되기 때문이다. reviewer의 모든 comment를 고쳤다, 틀렸다고 했다, 의도된 동작이라고 했다, 상관없다고 했다로 분류한다. 깨끗한 pull request에 reviewer를 돌려 해로운 변경을 유도할 comment를 얼마나 다는지도 센다. 우선순위는 진짜 버그를 먼저 잡고 review round를 줄이는 것은 그 다음이다.

noise 비용은 author가 agent일 때 더 크다. 사람은 nit를 몇 초면 무시하지만 agent는 그것을 고치려고 또 commit하고 review round를 추가한다. review는 수렴해야 한다. agent author와 agent reviewer는 서로 끝없이 바쁘게 만들 수 있다. 한 CLI pull request는 author가 멈추기 전까지 밤새 50회가 넘는 review round를 돌았다. git blame을 보니 reviewer가 계속 지적한 것의 약 59%는 reviewer의 첫 pass부터 이미 있던 코드였다. reviewer가 새 버그를 찾는 것이 아니라 매 라운드 오래된 버그를 몇 개 더 찾고 있었다. 팀은 기본적으로 여러 라운드를 돌리는 습관을 만들었다. 각 pass에서 새 무언가를 잡았기 때문이다. 위험한 변경에서는 가치가 있지만 모든 pull request에 그럴 필요는 없다. 이제 recall과 함께 rounds-to-pass를 추적한다. re-review 방식을 바꿨다. 라운드 상한을 두어 3라운드 후에는 author가 접근을 다시 생각하게 하고, 처음 몇 pass 이후에는 새 diff만 다시 보며, 한 줄이 아니라 버그의 class를 지적해 모든 사례를 고치게 하고, 이미 게시한 finding은 반복하지 않는다. review agent는 기존 코드에도 nightly로 돌면서 오래된 버그가 다른 곳에서 나타나기 전에 잡는다.

### 인간이 남는 지점

원문은 senior engineer를 deep work에서 꺼내 routine mistake를 잡게 하는 비용이 자동 review보다 훨씬 크다고 말한다. 그래도 사람이 하는 일은 남는다. 무엇을 만들지 결정하는 일, 방향과 취향과 범위 결정이 그 예다. 한 팀은 review bottleneck이 해결됐다고 선언했지만 외부 팀 변경에는 human review를 유지했다. 누군가는 "이걸 애초에 해야 하는가"를 물어야 한다. billing, auto-updater, CI workflow 같은 위험한 코드 review도 사람 몫이다. 다른 사람이 따라 할 패턴을 정하는 architecture review도 그렇다. 증거를 읽는 일도 중요하다. 테스트 출력, 실행한 명령과 반환값, 스크린샷, reviewer의 finding을 봐야 한다. 체크가 무엇을 놓칠지 말할 수 없다면 빠른 approval보다 실제 review를 요청하라고 원문은 말한다. 마지막으로 결과의 ownership은 사람에게 있다. 누가 코드를 입력했든 모든 변경에 한 사람이 책임을 진다.

### 팀 전체에 맥락 공유하기

code review의 두 번째 일인 맥락 공유는 기계가 대부분 읽으면 어떻게 되느냐는 질문이 남는다. 원문은 formal process는 없다고 말한다. 팀 누구도 Claude Code 전체 mental model을 머리에 유지하지 않으며, 그럴 필요도 없다고 본다. 어떤 부분을 이해하고 싶으면 Claude에게 그 부분만 설명하는 explainer를 작성하게 한다. 몇 분이면 되고 오늘 코드를 반영한다. 일부는 해당 부분을 시각화하는 작은 artifact를 만들어 자신이 좋아하는 학습 방식으로 가르치게 한다. 이 방식은 trivial diff를 줄 단위로 읽는 것보다 낫다. 한 줄 copy change는 배우는 게 거의 없지만, 곧 바꿀 module의 explainer는 많은 것을 가르쳐 준다. open Slack channel에서 기본적으로 일하는 문화도 다른 사람이 무엇을 만드는지 만드는 중에 보게 해 준다.

### "You land it, you own it"

원문은 "You land it, you own it"이라는 팀 규범을 소개한다. 작거나 중간 변경을 merge했는데 문제가 생기면 보통 그 사람이 고친다. 이게 나쁘게 돌아가지 않는 이유는 변경이 작아 검토와 디버그, revert가 쉽고, 모든 것이 flag나 revert, staged release로 되돌릴 수 있고, 폭발 반경이 기준을 정하며 비싼 코드는 더 많은 사람과 check를 받고, telemetry와 내부 사용으로 regression이 빨리 드러나고, blame 문화가 아니라는 점이다. incident review는 시스템을 본다. ownership은 공유되기도 한다. 열린 channel에서 누구나 다른 사람의 작업을 보고 thread를 잡거나 Claude를 gap에 붙일 수 있고, 코드 owner가 아닌 사람들도 feedback을 준다. 이 개방성이 모든 줄에 gate를 두지 않고 품질을 유지하는 큰 부분이다.

### 방향을 바꿀 때

agent는 무언가 시작하는 비용을 낮춰서 일부 작업은 land되지 말아야 한다. 보통 channel에서 누군가 이유를 묻거나 이전 결정을 가리키거나 더 단순한 길을 제안하면 author가 pull request를 닫거나 다시 만든다. 코드를 빨리 썼기 때문에 pull request를 닫아도 손실처럼 느껴지지 않는다. 요령은 다른 방향을 thread 초반에 제기하는 것이다. 완성된 pull request에 blocking comment로 다는 것이 아니다.

### 이 프로세스를 도입하는 방법

원문의 마지막 부분은 다른 팀이 이 프로세스를 도입하는 방법을 단계별로 정리한다. pull request 전에는 agent가 실행할 check를 짧은 CLAUDE.md에 나열하고 formatter와 linter hook을 걸고, 로컬에서 /code-review와 /security-review 또는 fresh-context subagent review를 돌리고, REVIEW.md를 추가하고 반복되는 review comment를 그쪽으로 옮기라고 한다. 모든 pull request에는 CI를 필수로 두고 pull request template에 Testing, Risk, How to revert 항목을 넣고, 한 repo에 AI reviewer를 추가하고 REVIEW.md를 몇 주 튜닝한 뒤 넓히고, 자동 security review를 추가하고 모든 review thread가 resolve되어야 merge되게 하라고 한다. review 노력을 위험도에 맞추는 것도 포함된다. routine change는 quick pass, missed bug가 비싼 곳은 deep이다. 승인 단계에서는 risk tier를 author가 아니라 변경 기준으로 문서화하고 처음에는 엄격하게 시작해 증거로 풀고, 가벼운 approval path는 신뢰하는 author에게만 주고, CODEOWNERS와 branch protection으로 auth, billing, CI와 release workflow, installer는 항상 사람을 요구하라고 한다. bot approval이 필수 human approval을 대체하지 못하게 하고 새 commit이 오면 approval을 dismiss하라고 한다. merge 이후에는 merge queue, feature flag, 내부 사용 우선, staged release, one-step rollback을 쓰라고 한다. incident와 revert를 eval set으로 만들고 reviewer의 prompt, tool, model이 바뀔 때마다 다시 점수를 매기라고 한다. 마지막으로 human time을 측정하되 revert와 incident rate을 guardrail로 두라고 권고한다.

### 앞으로의 방향

앞으로의 방향에 대해 원문은 Code Review 제품 발표 당시 approval은 여전히 사람의 call이고 제품은 finding을 올릴 뿐 approval을 내리지 않는다고 했지만, Claude Code 팀의 repo는 더 나아가 owner가 low-risk 변경을 자동 검사만으로 merge하게 한다고 말한다. 그 경계는 계속 움직인다. review가 무언가를 놓치면 모델 앞에 무엇이 없었는지 묻고 CLAUDE.md, skill, 새 check에 고친다. 목표는 변하지 않았다. 변경당 인간 시간은 줄이고 revert는 예전보다 늘리지 않는 것. agent가 pull request를 열게 하는 팀이라면 어떤 check를 자동화했고 어떤 check에 여전히 사람이 필요한지, 이 글이 잘못 본 것이 무엇인지 알려 달라고 한다.

## 근거와 수치

- **"Claude writes about 90 percent of Claude Code's own code"**  
  Claude Code 팀은 자사 제품 코드의 약 90%를 Claude가 작성한다고 말한다. 이는 agent가 보조 수준이 아니라 거의 주 작성자임을 전제로 한 수치다. 다만 이 90%가 어떤 기준으로 측정됐는지는 원문에 명시되어 있지 않다.

- **"From June to September, merged pull requests to Claude Code's codebase more than doubled while the share that had to be reverted fell by more than 40 percent."**  
  merged pull request 수가 2배 이상 늘었고 revert 비율은 40% 이상 줄었다는 것이다. 이는 변경량이 늘어도 품질이 나빠지지 않았다는 주장의 핵심 근거다. 다만 절대적인 revert 건수나 baseline 비율은 공개되지 않았다.

- **"changes landing per day went from about 2,500 in May to more than 30,000 in September"**  
  Anthropic monorepo에서 하루 landing되는 변경 수가 5월 약 2,500건에서 9월 30,000건 이상으로 늘었다. 단순 계산으로 약 12배 증가한 것이다. 이는 agent 도입 이후 변경 규모가 얼마나 빠르게 커지는지 보여주는 수치다.

- **"the share of internal PRs getting substantive review comments went from 16% to 54%"**  
  research preview에서 substantive review comment를 받는 internal pull request 비율이 16%에서 54%로 올랐다. 38% 포인트 상승이며 3배 이상이다. 이는 자동 review가 사람 review보다 더 많은 pull request에서 실질적인 finding을 남기기 시작했음을 보여준다. 다만 이 finding이 모두 실제 버그인지는 별도 검증 단계를 거친다.

- **"Most PRs over 1,000 lines get findings, and most small ones come back clean."**  
  줄 수가 많은 pull request일수록 finding이 나올 가능성이 높고, 작은 pull request는 대체로 깨끗하게 통과한다. 이는 변경 크기와 review finding 사이의 상관관계를 보여주는 정성적 관찰이다. 정확한 비율은 원문에 없다.

- **"He estimates it makes the final result two to three times better"**  
  Claude Code를 만든 Boris Cherny는 Claude에게 검증 수단을 주는 것이 최종 결과를 2~3배 좋게 만든다고 추정한다. 이는 측정된 실험 결과가 아니라 개인적인 추정이다. 그래도 pre-PR 검증의 가치를 강조하는 근거로 쓰인다.

- **"One CLI PR went through more than fifty rounds of review and fixes overnight before its author stopped it."**  
  특정 CLI pull request가 밤새 50회가 넘는 review round를 돌았다. 이는 agent author와 agent reviewer가 서로 수렴하지 않고 반복할 수 있음을 보여주는 사례다. 50회는 매우 비정상적으로 많은 수치이며, 이 때문에 라운드 상한을 도입했다.

- **"about 59% of it was code that had been there since the reviewer's first pass."**  
  reviewer가 반복해서 지적한 것 중 약 59%는 첫 pass부터 이미 있던 코드였다. 이는 reviewer가 새 버그를 찾기보다는 오래된 finding을 반복해서 지적하고 있었음을 뜻한다. 이 수치는 review가 수렴하지 않을 때 문제의 성격이 무엇인지 보여준다.

- **"After three rounds, the reviewer asks the author to step back and reconsider the approach."**  
  review 라운드 상한을 3회로 두었다는 구체적 기준이다. 이 상한은 무한 반복을 막기 위한 운영 규칙이다. 다만 3이라는 숫자가 실험으로 최적화된 것인지, 경험적으로 정한 것인지는 원문에 명시되어 있지 않다.

## 따져볼 점

이 글은 Anthropic이 자사 제품인 Claude Code와 Code Review for Claude Code를 소개하는 blog다. 저자들은 제품 팀 내부자이고, "우리 프로세스가 잘 작동한다"는 주장과 "이 도구를 쓰라"는 권고가 섞여 있다. Code Review for Claude Code의 generally available 소식, GitHub Action, /code-review, /security-review 등이 구체적으로 언급된다. 따라서 독립적인 평가라기보다는 성공 사례 마케팅 성격을 감안해야 한다.

- **검증되지 않은 수치와 정의의 모호함**  
  "Claude가 90%를 작성한다"는 주장은 어떤 기준으로 측정했는지 명시되지 않았다. 자동 생성 후 사람이 수정한 코드, 단순 복사, 설정 파일 자동 생성 등이 모두 포함될 수 있다. monorepo의 "changes landing per day"도 change의 정의가 명확하지 않다. 작은 설정 변경 하나와 대규모 기능 변경 하나가 같은 change로 집계될 수 있다.

- **revert 비율 40% 감소는 상대값일 뿐 절대 기준이 없다**  
  baseline revert 비율이 원문에 없다. 예를 들어 baseline이 1%였다면 40% 감소는 0.6%로 내려간 것이고, baseline이 10%였다면 6%로 내려간 것이다. 절대적인 revert 건수나 incident 발생률도 명시되어 있지 않다. 따라서 "품질이 나빠지지 않았다"는 결론을 내리기에는 정보가 부족하다.

- **사람 승인 최소화의 위험**  
  원문은 author가 유일한 human in the loop가 될 수 있다고 말한다. agent가 쓴 코드를 지시한 사람이 필수 reviewer 한 명으로 계산된다. 이는 전통적인 독립적 review와 다르다. 문서나 low-risk 변경에서는 문제가 적을 수 있지만, medium 변경에서도 author 자신이 승인자가 될 수 있다면 독립적인 견제가 사라진다. 이 시스템은 빠른 rollback과 flag, telemetry에 크게 의존하는데, rollback이 항상 싸거나 쉬운 환경이 아닐 수 있다.

- **자동 reviewer의 정확도와 classifier 성능 미공개**  
  finding을 다른 agent들이 검증하고 필터를 A/B 테스트했다고는 하지만, false positive rate, false negative rate, risk classifier의 오분류율, security review의 탐지율 같은 구체적 수치는 없다. "대부분의 1,000줄 이상 pull request에서 finding이 나온다"는 말도 finding이 모두 유효하다는 뜻은 아니다.

- **eval set의 한계**  
  reviewer eval은 과거 incident와 revert에서 만든다. 이는 이미 발생한 버그 유형에는 강할 수 있지만 새로운 유형의 버그나 near-miss에는 약할 수 있다. 과거 사고에 overfit될 위험이 있다. eval에 incident가 쌓일수록 개선된다고 하지만, 그 자체로 미래 위험을 예측하는 것은 아니다.

- **circuit breaker 부재는 원문도 인정한다**  
  revert나 incident가 늘어나면 approval을 자동으로 강화하는 장치가 아직 없다. 나쁜 merge를 알아채는 것이 여전히 사람과 dashboard에 의존한다. zero-human merge를 더 확대하려면 이런 안전장치가 먼저 필요하다는 점을 원문 스스로도 지적한다.

- **맥락 공유 대체 방식의 검증 부족**  
  사람이 diff를 읽으며 얻는 codebase 이해를 Claude explainer와 open Slack channel로 대체한다고 한다. 개인 학습에는 효과적일 수 있지만 팀 차원의 공유 이해가 실제로 유지되는지는 검증되지 않았다. formal process가 없다는 점도 팀 문화가 바뀌면 약해질 수 있다.

- **도구 의존성과 문화 의존성**  
  CLAUDE.md, REVIEW.md, skill, trusted author list, open Slack channel 문화는 Claude Code 팀에 맞게 진화한 것이다. 다른 조직에서 그대로 복사하기 어려운 부분이 있다. 특히 규제 산업, 금융, 의료, 보안이 중요한 환경에서는 자동 merge나 author self-approval이 허용되지 않을 수 있다.

## 왜 중요한가

이 글은 code review를 "사람이 읽는지"가 아니라 "어떤 layer가 어떤 실패를 잡는지"로 바꾸는 전환 사례다. agent가 코드의 대부분을 작성하는 팀이라면 기존 gate 방식은 병목과 형식적 승인으로 붕괴한다. 실무자는 먼저 자신의 pull request당 human approval 시간과 revert 및 incident 비율을 동시에 추적해 보는 것이 좋다. review를 줄였는데 revert가 늘면 비용이 줄어든 것이 아니라 수습 쪽으로 이동한 것이다.

작은 것부터 실험하려면 다음을 시도해 볼 수 있다. 첫째, agent가 실행할 수 있는 구체적인 check를 CLAUDE.md나 instruction에 넣고 pre-PR에 자동 review를 돌린다. 둘째, 반복되는 review comment를 REVIEW.md로 옮기고 사람이 판단할 때는 파일과 줄 번호를 증거로 요구한다. 셋째, risk tier를 작성자가 아니라 변경 기준으로 정하고 docs나 flag 뒤 변경부터 자동 merge를 허용하되, billing이나 CI workflow처럼 되돌리기 어렵거나 영향이 큰 경로에는 항상 사람 승인을 남긴다. 넷째, merge queue와 feature flag, 내부 사용 우선, one-step rollback을 먼저 갖춘 뒤에 사람 승인을 줄인다. 다섯째, incident와 revert를 eval set으로 만들어 reviewer의 prompt, tool, model이 바뀔 때마다 재평가한다.

특히 agent author가 등장한 조직에서는 review round가 무한히 늘어나는 문제를 주의해야 한다. 이 글의 50회 review round 사례와 59% 반복 지적 사례는 자동 review가 오히려 노이즈를 만들 수 있음을 보여준다. review round 수와 recall을 함께 추적하고, 3회 같은 라운드 상한과 diff-only re-review를 적용하면 실용적인 출발점이 될 수 있다.

마지막으로 이 글은 product 승인을 자동화하라고 단순히 권하지 않는다. Code Review for Claude Code 제품은 finding을 올릴 뿐 approval을 내리지 않는다는 점을 분명히 한다. 그런데도 Claude Code 팀은 내부적으로 low-risk 변경에 한해 자동 merge 경계를 넓히고 있다. 앞으로 지켜볼 것은 절대적인 incident rate, ris
