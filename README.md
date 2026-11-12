# claude-plugins

Claude Code 플러그인 마켓플레이스. 플러그인은 `plugins/` 아래에 하나씩 두고, 쓰는 것만 골라 설치한다.

| 플러그인 | 하는 일 |
|---|---|
| [`workdoc`](plugins/workdoc) | 작업지시서 흐름 — 지시서 작성 → 검수 → 구현 |

## 설치

두 방법 중 하나로 한다. 결과는 같다.

### 셸에서

마켓플레이스를 등록한다(기기마다 한 번).

```bash
claude plugin marketplace add idas4you/claude-plugins
```

플러그인을 설치한다.

```bash
claude plugin install workdoc@idas4you --scope user
```

### Claude Code 세션 안에서

`claude`를 실행한 뒤 입력창에 차례로 친다.

```
/plugin marketplace add idas4you/claude-plugins
```

```
/plugin install workdoc@idas4you
```

`/plugin`만 치면 목록 화면에서 마켓플레이스와 플러그인을 골라 설치할 수 있다. 설치한 스킬은 새 세션부터 보인다(실행 중인 세션은 `/reload-plugins`).

## 갱신

```bash
claude plugin marketplace update idas4you
```

```bash
claude plugin update workdoc@idas4you --scope user
```

갱신은 다음 세션부터 적용된다(실행 중인 세션은 `/reload-plugins`).

## workdoc

작업을 "지시서 쓰기 → 지시서 검수 → 구현" 세 단계로 나누고, 단계마다 맞는 모델로 돌린다.

| 스킬 | 모델(`model:`) | effort(`effort:`) | 하는 일 |
|---|---|---|---|
| `/idas4you-doc <주제>` | `fable` | `medium` | 작업지시서(design.md·plan.md·tasks.md, 필요하면 ADR)를 쓰거나 고친다. 코드는 읽기만 한다. 다 쓰면 바로 `/idas4you-doc-check`로 검수할지 묻는다 |
| `/idas4you-doc-check <지시서 디렉터리>` | `fable` | `high` | 지시서를 구현 전에 검수한다(사실 대조·설계 동작·실행 가능성). 사실 오류만 바로 고치고 설계 결정은 보고만 한다. 구현에 쓸 모델을 tasks.md 머리의 `구현 모델 권장`에 적는다. 끝나면 판정에 맞춰 다음 걸음(`/idas4you-doc-impl` 등)을 추천 문장으로 남긴다 |
| `/idas4you-doc-impl <지시서 디렉터리>` | `opus` | 세션 설정 | 지시서대로 기능 브랜치에서 구현·시험하고 결과를 지시서에 기록한다. `구현 모델 권장`이 `sonnet`·`fable`이면 그 단계의 코드 구현을 해당 모델 하위 에이전트에 맡긴다. 같은 단계를 두 번 고쳐도 통과하지 못하면 한 단계 위 모델로 올린다(`sonnet` → `opus` → `fable`). 시험을 통과해도 머지하지 않고 묻는다 |

- **필요한 모델**: Fable과 Opus를 쓸 수 있는 계정이어야 의도대로 돈다(구현을 맡길 때는 Sonnet도 쓴다). 스킬의 `model:`·`effort:`는 스킬을 부른 턴에 세션 설정 대신 적용된다. 조직이 쓸 수 있는 모델을 제한했으면 `model:` 지정이 적용되지 않을 수 있다.
- 같은 이름의 다른 스킬이 있으면 `/workdoc:idas4you-doc`처럼 플러그인 이름을 붙여 부른다.
- 저장소별 값(지시서 위치, ADR 위치, 브랜치 이름 규칙, 커밋 작성자, 시험·배포 명령)은 스킬에 박혀 있지 않다. 작업하는 저장소의 `CLAUDE.md`·`AGENTS.md`와 사용자 개인 안내에서 읽고, 없으면 스킬에 적힌 기본값(`docs/work/YYYY-MM-DD-<slug>/` 등)을 쓴다.
- 쓰는 곳: Claude Code CLI, 데스크톱 앱 Code 탭(로컬 세션). claude.ai/code 웹 세션은 플러그인을 로드하지 않는다.

## 플러그인 추가

1. `plugins/<이름>/.claude-plugin/plugin.json`과 `plugins/<이름>/skills/<스킬>/SKILL.md`를 만든다.
2. `.claude-plugin/marketplace.json`의 `plugins`에 항목을 추가한다.
3. 위 표에 한 줄 추가한다. 플러그인을 고칠 때는 `plugin.json`의 `version`을 올린다.

## 라이선스

[MIT](LICENSE)
