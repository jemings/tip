# OpenAI Agents API 조사 노트

- 기준일: **2026-09-13**
- 대상: OpenAI Agents API **public beta**. 2026-09-10에 공개됐습니다.
- 성격: 발표 자료와 별개로, 조사하고 검증한 사실만 모은 참고 문서입니다.

> **읽는 법**
> - 인용부호 안 영어 문장은 출처 원문입니다.
> - **(추론)** 은 문서에 직접 쓰여 있지 않고 여러 사실을 조합해 내린 결론입니다.
> - **(미확인)** 은 공개 자료로 확인하지 못한 항목입니다.
> - 베타 API라서 필드 이름과 동작은 바뀔 수 있습니다.

**조사 방법**
- **1차 자료:** developers.openai.com 가이드와 API reference(`.md` 사본), openai.com 블로그, GitHub의 `openai/codex`, `openai/openai-cookbook`, `openai/openai-python`, `agentclientprotocol/*`.
- **openai.com 본문:** WebFetch가 403으로 막혀 r.jina.ai 프록시와 Wayback 스냅숏 두 경로로 읽었고, 두 사본이 일치했습니다.
- **2차 자료:** 일부(Zed·JetBrains 블로그, InfoWorld 등)는 요약기를 거쳐 읽었습니다. 원문 인용이 필요하면 다시 확인하세요.
- **검증:** 모든 항목을 다른 에이전트가 반박하는 방식으로 교차 검증했습니다. 이 문서에는 수정 판정을 반영한 문장만 남겼습니다.

---

## 1. 한 줄 정의

> "The Agents API gives your application access to the Codex harness through an OpenAI-managed API. OpenAI manages sessions, orchestration, context compaction, and recovery while your application provides tools and chooses its execution environment."
> — Agents API overview

- **블로그 부제:** "Build and run cloud agents with the Codex harness, fully managed by OpenAI."
- **문서 부제:** "Build durable cloud agents with a managed Codex harness."
- **블로그 설명:** Codex를 구동하는 바로 "that same harness and infrastructure that powers Codex"를 API로 연다고 설명합니다.
- **요금:** "There are no additional fees for using the Agents API – you simply pay for the tokens and tools your agents use."

**정리하면** Codex의 에이전트 루프(모델 호출 → 도구 실행 → 결과 반영 → 반복)와 세션 보관, 컨텍스트 압축, 복구를 OpenAI 서버가 맡습니다. 내 애플리케이션은 일을 맡기고 결과를 받고, 내 도구를 제공하고, 명령이 실행될 장소를 고릅니다.

---

## 2. 구성 요소

### 2.1 런타임을 이루는 세 부분 (Architecture 가이드)

| 부분 | 운영 주체 | 역할 (원문 요지) |
|---|---|---|
| **Harness** | OpenAI | "The OpenAI-hosted Codex instance that runs the model and tool loop and maintains the agent's session." |
| **Environment** | OpenAI 또는 나 | 명령이 실행되고 파일이 있는 곳. 원격 샌드박스, 노트북, Docker 컨테이너, AWS Lambda 함수가 모두 될 수 있습니다. |
| **Application server** | 나 | "It submits tasks, receives events, and handles function tools. When you provide the environment, your code also manages its lifecycle." |

> "OpenAI runs the agent harness. Your application sends it work and receives results. Add an environment when the agent needs compute or files."

### 2.2 핵심 개념 4가지 (Overview)

| 개념 | 설명 |
|---|---|
| **Agent** | 재사용 가능한 설정입니다: `model`, `instructions`, `tools`, `multi_agent`, `reasoning.effort`, `text`, `service_tier`, `name`, `metadata`. 세션을 만들 때 인라인으로 넣거나, 저장해 두고 `agent_id`로 참조합니다. |
| **Environment** | "An optional sandbox or computer where the agent accesses files, loads skills, and runs commands." |
| **Session** | "A durable instance of an agent that works on tasks and responds to input." |
| **Events / items** | 세션이 흘려보내는 진행 이벤트와, 저장되는 결과 단위(메시지, 도구 호출, 명령 실행 등)입니다. |

세션은 다음 순서로 흘러갑니다.
1. 세션을 생성합니다.
2. 작업을 입력합니다.
3. 스트림이나 webhook으로 진행을 받습니다.
4. 이어서 입력하거나, 작업 중에 방향을 바꿉니다(steer).

### 2.3 관리형 harness가 제공하는 기능 (Overview 목록)

1. 샌드박스에서 명령과 코드 실행
2. skills와 instructions 적용
3. tools와 MCP로 외부 데이터 접근
4. 작업 중 방향 전환(steering)
5. 지난 작업을 요약해 컨텍스트 관리(compaction)
6. subagent에게 위임
7. 멈춘 지점에서 세션 재개

이 목록에는 **사람 승인 기능이 없습니다.** 자세한 내용은 §7에 있습니다.

---

## 3. API 표면

### 3.1 공통

- **형태:** HTTPS REST이고, 기본 경로는 `https://api.openai.com/v1/agents/...`입니다. vault 엔드포인트만 예외로 `/vaults` 아래에 있습니다.
- **필수 헤더:** `OpenAI-Beta: agents=v1`. SDK는 자동으로 붙이고, cURL은 직접 넣어야 합니다.
- **SDK 네임스페이스:** `client.beta.agents`. Python, JavaScript, Go, Java, Ruby 예제가 공식 문서에 있습니다.
- **API 키 권한:**
  - 세션 조작: `api.agents.read`, `api.agents.write`
  - 모델 추론: `api.responses.write`
  - vault 관리: `api.vaults.read`, `api.vaults.write`
- **진행 수신 방법:** 두 가지입니다.
  - 이벤트 스트림: `GET /v1/agents/sessions/{id}/events?stream=true`에 `Accept: text/event-stream`. 세션 생성 때 `stream: true`로 받을 수도 있습니다.
  - **webhook**

### 3.2 엔드포인트

| 리소스 | 메서드 |
|---|---|
| Agents | `POST /agents`, `GET /agents`, `GET · POST · DELETE /agents/{agent_id}` |
| Sessions | `POST /agents/sessions`, `GET /agents/sessions`, `GET · DELETE /agents/sessions/{session_id}`. `POST /agents/sessions/{id}`는 메타데이터만 갱신합니다. |
| Events | `POST /agents/sessions/{id}/events`(입력 제출), `GET /agents/sessions/{id}/events`(스트림) |
| 조회 | `GET .../items`, `GET .../turns`, `GET .../turns/{turn_id}`, `GET .../subagents` |
| Artifacts | `GET .../artifacts`, `GET .../artifacts/{id}/content`, `DELETE .../artifacts/{id}` |
| Environments | `GET /agents/environments/{id}`, 그 아래 `files`, environment template CRUD |
| Vaults | `/vaults`, `/vaults/{vault_id}/credentials` (SDK에서는 `client.beta.agents.vaults`) |

### 3.3 입력 이벤트는 딱 3종

| 이벤트 | 의미 |
|---|---|
| `agent.session.input.message` | 사용자 메시지(`input_text`, `input_image`). 세션이 idle이면 새 turn을 시작하고, 작업 중이면 진행 중인 turn을 **steer**합니다. |
| `agent.session.input.cancel` | 진행 중인 turn을 취소합니다. "The session and its previous work remain available." |
| `agent.session.input.tool_result` | function tool 결과를 제출합니다. `turn_id`, `call_id`가 필요하고, 성공이면 `success: true` + `output`, 실패면 `success: false` + `error`를 보냅니다. |

선택 사항인 idempotency key를 쓰면 재시도가 안전해집니다.

### 3.4 상태 값

- **Session `status`:** `idle` · `in_progress` · `requires_action` · `failed`
  - `idle`: "The session has no turn in progress and is ready for input. A hosted environment may still be provisioning."
- **`required_actions`:** 두 종류뿐입니다.
  - `function_call`: `arguments`, `call_id`, `name`, `turn_id`
  - `environment_connection`: `environment_id`
- **Turn `status`:** `queued` · `in_progress` · `waiting` · `completed` · `failed` · `cancelled`
- **Turn 오류 코드 17종:**
  - 한도·요금: `context_length_exceeded`, `session_budget_exceeded`, `usage_limit_exceeded`, `credit_balance_exhausted`, `rate_limit_exceeded`
  - 서버·연결: `server_overloaded`, `connection_failed`, `server_error`, `internal_error`, `request_timeout`
  - 요청·인증: `invalid_request`, `authentication_error`, `resource_not_found`, `active_turn_not_steerable`
  - 정책: `cyber_policy`
  - 실행 환경: `sandbox_error`, `executor_version_incompatible`

### 3.5 이벤트 스트림과 webhook

- **스트림 이벤트 범주:**
  - 세션 상태: created, in_progress, idle, requires_action, failed
  - 환경: pending, ready, connected, disconnected, failed
  - subagent: created, active, closed
  - turn item: added, done
  - 출력 텍스트 delta, reasoning summary, 명령 실행 출력 delta
  - turn 결과, error
- **turn 결과 판정:** `agent.session.turn.completed` / `turn.failed` / `turn.cancelled`를 봅니다.
  - "`agent.session.idle` alone does not mean success."
  - completed turn 안에도 실패한 도구 호출이 있을 수 있습니다.
- **webhook:** 스트림을 열어 두지 않고 상태 변화를 받는 방법입니다. 서명된 HTTP POST로 옵니다.
  - 이벤트: `agent.session.created`, `agent.session.action_required`, `agent.session.in_progress`, `agent.session.idle`, `agent.session.failed`
  - payload에는 액션 종류만 들어 있어서, handler가 세션을 다시 조회해야 합니다.
  - 세션 삭제에는 webhook이 오지 않습니다.
  - 이름 주의: 스트림 이벤트는 `requires_action`, webhook은 `action_required`로 문서에 표기됩니다.

---

## 4. 세션 수명주기

### 4.1 생성: `POST /agents/sessions`

- **필수:** `environment`. `type`은 `none` · `openai_hosted` · `self_hosted` 중 하나입니다.
- **선택:** `agent`, `agent_id`, `input`, `metadata`, `stream`, `vault_ids`
- **`agent`와 `agent_id`를 함께 보내면:** 그 세션에 한해 저장된 설정을 덮어씁니다. 병합하지 않고 필드 전체를 교체합니다("Supplied objects and arrays replace the entire field rather than merging with the saved value.").
- `agent_id` 없이 만들면 `model`이 필수입니다.
- `environment.type: "none"`이면 초기 `input`이 필수입니다.
- `stream: true`면 첫 turn의 이벤트가 같은 응답으로 흘러옵니다.
- **Session 리소스 필드:** `id`, `agent`, `created_at`, `environment`, `error`, `last_active_at`, `metadata`, `object`, `required_actions`, `status`, `usage`, `vault_ids`
- 문서 권고: "Store the `session_id` with your application's conversation state."

**최소 예제** (Quickstart, Python, 원문 그대로):

```python
from openai import OpenAI

with OpenAI() as client:
    with client.beta.agents.sessions.create(
        agent={
            "model": "gpt-6-astra",
            "instructions": "Write clean code, run it, and report the actual output.",
        },
        environment={"type": "openai_hosted"},
        input="Create tree.py, a Python script that prints a readable tree of the files in the current directory. Run it and show me the output.",
        stream=True,
    ) as events:
        for event in events:
            print(event.to_json(indent=None), flush=True)
```

**이어서 입력하기** (Sessions 가이드):

```python
def send_message(client: OpenAI, session_id: str, text: str) -> None:
    client.beta.agents.sessions.events.create(
        session_id,
        events=[
            {
                "type": "agent.session.input.message",
                "input": [
                    {
                        "role": "user",
                        "content": [{"type": "input_text", "text": text}],
                    }
                ],
            }
        ],
    )
```

### 4.2 Turn, steer, cancel

- **turn:** 일 한 사이클입니다.
  - idle 세션에 메시지를 보내면 새 turn이 시작됩니다.
  - 작업 중에 보내면 진행 중인 turn을 **steer**합니다("A message sent during an active turn steers that turn.").
  - turn은 비동기로 돕니다.
- 초반 이벤트를 놓치지 않으려면 **메시지를 보내기 전에 스트림부터 구독**하라고 권합니다.
- **cancel:** 멈춤이지 일시정지가 아닙니다. 세션과 이전 작업은 남습니다.
- **pause/resume 엔드포인트는 없습니다.** 이어서 하려면 같은 `session_id`에 메시지를 다시 보냅니다.

### 4.3 연결이 끊겼을 때

- "Closing an event stream does not cancel the task." 작업은 OpenAI 쪽에서 계속됩니다.
- "Streams do not replay missed events." 놓친 이벤트는 다시 오지 않습니다.
- **복구 절차(문서):**
  1. 새 스트림을 열고 이벤트를 버퍼링합니다.
  2. 스트림을 연 채로 세션과 저장된 items를 조회합니다.
  3. item ID 기준으로 로컬 상태를 복원합니다.
  4. 버퍼에 쌓인 업데이트를 `item_id` 기준으로 적용합니다.
  5. 실시간 처리를 재개합니다.
- "After a restart or stream disconnect, retrieve the session to find pending actions."

### 4.4 보관과 삭제

- 세션 상태는 **삭제할 때까지 보관**됩니다. 데이터 컨트롤 표의 `/v1/agents` 행은 다음과 같습니다.
  - 학습 사용: No
  - abuse monitoring 보관: 30일
  - application state: "Until deleted"
  - ZDR 적격: No
  - Eyes Off / Safety Retention 적격: No
- **hosted 세션 삭제:**
  - 샌드박스 정리를 요청합니다.
  - 설정이나 실행이 마무리 중이면 `409`가 오니 재시도합니다.
  - 물리 정리는 비동기로 이어질 수 있습니다.
- **self-hosted 세션 삭제:** 내 컴퓨트를 멈추지 않고, webhook도 보내지 않습니다.

---

## 5. 도구

### 5.1 `agent.tools`에 넣을 수 있는 타입은 정확히 5종

| 타입 | 설명 |
|---|---|
| `function` | 내 애플리케이션이 실행하는 함수입니다. §5.2를 보세요. |
| `mcp` | MCP 서버입니다. §5.3을 보세요. |
| `web_search` | 모드는 `live`(기본) · `cached` · `disabled`이고, `allowed_domains`는 최대 100개입니다. 요금은 1k 호출당 $10에 검색 콘텐츠 토큰(모델 요금)이 더해집니다. |
| `tool_search` | Agents API에서는 `defer_loading: true`로 표시한 function 정의의 로딩을 미룹니다. |
| `programmatic_tool_calling` | 기본으로 켜져 있습니다. 에이전트에게 JavaScript로 다른 도구들을 부르는 `exec` 도구를 줍니다. "Orchestrating a tool in JavaScript doesn't change where the tool runs." |

- 문서가 built-in으로 부르는 **Bash**와 **apply-patch**는 environment가 있을 때만 존재합니다.
  > "Without an environment, the built-in Bash and apply-patch tools, workspace files, and executor MCPs are unavailable."

### 5.2 Function tool: 콜백 왕복

**동작 순서:**
1. harness가 turn을 멈추고 `agent.session.requires_action`을 냅니다.
2. 내 코드가 `required_actions`를 읽어 함수를 실행합니다.
3. 같은 `turn_id` · `call_id`로 `agent.session.input.tool_result`를 보냅니다.
4. harness가 **같은 turn을 이어서** 진행합니다.

**주의할 점:**
- "Attaching an environment to a session does not automatically run function tools there." function tool은 샌드박스가 아니라 내 앱에서 실행됩니다.
- "If that handler is unavailable, the agent can remain waiting for a result."
- 부작용이 있는 함수는 결과를 저장해 두고, 재시작 후에는 같은 `turn_id` · `call_id`로 저장된 결과를 다시 제출합니다.
- subagent는 function tool을 쓸 수 없습니다.

정의 (Functions 가이드 원문):

```json
{
  "type": "function",
  "name": "get_customer",
  "description": "Look up a customer by ID.",
  "parameters": {
    "type": "object",
    "properties": { "customer_id": { "type": "string" } },
    "required": ["customer_id"],
    "additionalProperties": false
  }
}
```

결과 제출 (Functions 가이드 원문):

```python
import json

action = action.to_dict()

result = {
    "type": "agent.session.input.tool_result",
    "turn_id": action["turn_id"],
    "call_id": action["call_id"],
}

output = get_customer(action["arguments"])
result.update(success=True, output=json.dumps(output))

client.beta.agents.sessions.events.create(session_id, events=[result])
```

### 5.3 MCP와 Vault

> "An MCP server publishes tool definitions and runs tool calls. The Agents API discovers the tools, calls the server, and returns results to the agent. Your application does not need to handle each call."

**연결 방식 3가지:**

| 연결 | 누가 연결하나 | environment 필요 |
|---|---|---|
| HTTP, `connection_origin: "service"`(기본) | OpenAI 서비스 | 아니요 |
| HTTP, `connection_origin: "environment"` | 세션의 environment(여기서 localhost는 environment를 뜻함) | 예 |
| stdio | environment 안의 프로세스 | 예 |

- **설정 필드:** `server_label`, `transport`(`http` + `server_url` / `stdio` + `command` · `args` · `cwd` · `env_vars`), `connection_origin`, `allowed_tools`, `required`, `credential_id`
- **`allowed_tools`:** 생략하면 서버의 모든 도구가 허용됩니다.
- **`required: true`:** 서버 초기화에 실패하면 turn도 실패합니다.
- **인라인 `authorization` / `headers`:** 암호화되고, 응답에서는 빠집니다.
- **stdio MCP의 `env_vars`:** environment 안 코드가 읽을 수 있습니다.
- **MCP 도구 설정에는 `require_approval` 필드가 없습니다.** Responses API의 MCP 도구에는 있습니다.

**Vault:**
- MCP 자격증명(정적 bearer token, 또는 refresh가 되는 OAuth)을 담는 곳입니다.
- "use authenticated tools without receiving the secret values". 에이전트는 비밀 값을 보지 못합니다.
- **OpenAI 서비스 쪽에서 맺는 MCP 연결에만 적용됩니다.** environment 쪽 연결에는 쓸 수 없습니다.
- "Deleting stored credentials does not revoke the original tokens with their providers or stop a running session."

### 5.4 Skills, Plugins, Templates

- **Skills:** `capability_directories`에 절대경로를 최대 32개 넣으면, harness가 그 안의 `SKILL.md`를 찾아 이름과 설명을 컨텍스트에 넣습니다.
- **Plugins:** `.codex-plugin/plugin.json`으로 skills와 MCP 설정을 묶습니다.
  - hosted: base64 ZIP으로 넣습니다.
  - self-hosted: capability directory로 넣습니다.
- "Existing sessions do not reload the tools."
- **Environment template:** hosted 설정을 재사용용으로 저장합니다. "Templates save configuration, not a running workspace."

---

## 6. 실행 환경 (Environment)

### 6.1 세 가지 타입

| `type` | 파일·명령이 있는 곳 | 핵심 특징 |
|---|---|---|
| `none` | 없음 | Bash, apply-patch, workspace, executor MCP를 쓸 수 없습니다. function tool, service MCP, web_search만 됩니다. 초기 input이 필수입니다. |
| `openai_hosted` | OpenAI 샌드박스의 `/workspace` | Linux이고 Python, Node.js, CLI 도구가 있습니다. OpenAI가 프로비저닝합니다. 1시간 무활동 시 삭제될 수 있습니다. |
| `self_hosted` | 내 노트북, 컨테이너, 파트너 샌드박스 | 그 안에서 `codex exec-server`가 OpenAI로 **outbound** 연결합니다. 수명주기는 내 책임입니다. |

**어느 경우든 모델·도구 루프(harness)는 OpenAI에서 돕니다.** self-hosted에서 내 쪽으로 오는 것은 명령 실행, 파일 읽기·쓰기, environment 쪽 MCP뿐입니다.

### 6.2 `openai_hosted`

- **설정:** `packages`, `setup_commands`, `files`, `env`, `skills`, `plugins`, `capability_directories`, `environment_template_id`, `network`
  - `setup_commands`가 0이 아닌 코드로 끝나면 에이전트가 시작되지 않습니다.
  - `env`에서 `PATH`, `CODEX_*`, `OPENAI_API_KEY` 같은 예약 이름은 거부됩니다.
- **네트워크:**
  - `enabled`: 기본값입니다.
  - `disabled`
  - `restricted`: 정확한 호스트명을 1~100개 넣습니다. 와일드카드, 프로토콜, 경로, 포트는 넣을 수 없습니다.
- **상태:** 환경 조회 결과는 `pending` · `connected` · `disconnected` · `expired` · `failed`입니다.
- **수명:**
  > "Connected sandboxes receive keep-alives, including between turns. If activity and keep-alives stop for an hour, the sandbox can be deleted. This timeout isn't configurable."
  - 샌드박스가 살아 있는 동안 파일은 turn을 넘어 유지됩니다.
  - `/workspace/outputs` 아래 파일은 turn이 끝날 때 **불변 artifact**로 게시되고, 샌드박스가 사라진 뒤에도 내려받을 수 있습니다.
  - "An agent session can outlive its environment."
  - 샌드박스가 만료되면 문서의 해법은 새 세션을 만들고 입력을 다시 넣는 것입니다.
- **파일 한도:**
  - 생성 요청당 파일 50개
  - 인라인 업로드는 파일당 5 MiB, 합계 10 MiB
  - Files API 사본 50 MiB
  - artifact 200 MiB
  - 한 번에 게시되는 outputs 500 MiB
  - 일괄 다운로드는 없고, artifact는 업로드하거나 수정할 수 없습니다.
- 샌드박스 인프라는 "the same sandboxing infrastructure that powers Codex and ChatGPT"를 씁니다(블로그).

### 6.3 `self_hosted`

- **설치:** `npm install -g @openai/codex@alpha`
- **실행:** `codex exec-server --remote "<session.environment.remote_url>" --environment-id "<session.environment.id>"`
  - `CODEX_API_KEY`에는 environment 연결만 허용하는 **제한 키**를 넣습니다. Agents 탭에서 만듭니다.
- **나가는 연결:**
  - 대상은 `api.openai.com`과 `wss://codex-cloud-environments.chatgpt.com`입니다.
  - "All connections are outbound. The executor reconnects if the connection drops."
- **세션과 executor:** "Each session has its own environment ID and needs its own executor."
- **내 책임:** "You own provisioning, reconnection, shutdown, and any files you need to preserve."
- **executor가 오프라인일 때 입력이 들어오면:**
  - `environment_connection` required action이 생기고, webhook으로도 옵니다.
  - 최대 5분 기다린 뒤 제출이 실패합니다.
  - 입력은 영속 큐에 쌓이지 않습니다. "The API does not guarantee recovery of pending input after a process crash."
- **turn 중간에 끊기면:**
  - "A mid-turn disconnect can fail a tool even if the turn completes."
  - 자동 재연결 요청이나 죽은 명령의 재시작은 없습니다.
- 같은 environment ID를 재사용해도 파일은 복원되지 않습니다.
- self-hosted 파일은 artifact로 게시되지 않습니다. `/workspace/outputs`도 마찬가지이니, 내 파일시스템이나 공급자 API로 가져옵니다.
- **self-hosted environment 필드:** `id`, `capability_directories`, `remote_url`, `type`, `workspace_directory`
- **`codex exec-server` 내부** (openai/codex README와 코드):
  - 하위 프로세스를 띄우고 제어하는 작은 JSON-RPC 서버입니다.
  - remote 모드에서는 environment registry에 등록하고, rendezvous WebSocket에 붙습니다.
  - 기본 전송은 **Noise relay**(암호화된 binary protobuf 프레임)이고, AWS SigV4를 쓰는 Direct 전송도 있습니다.
  - WebSocket이 닫히면 그 연결의 관리 프로세스를 종료합니다.

### 6.4 샌드박스 파트너 9곳

- **목록:** Blaxel, Cloudflare, Daytona, DigitalOcean, E2B, Modal, Oracle(OCI), Runloop, Vercel. 모두 self-hosted environment로 `codex exec-server`를 띄우는 방식입니다.
- 이 중 7곳은 2026년 4월 Agents SDK에 이미 들어가 있었고, DigitalOcean과 Oracle이 새로 추가됐습니다(두 블로그의 목록 비교).
- **파트너 기능 자체도 초기 단계인 곳이 있습니다.**
  - DigitalOcean M.A.R.S.: 초대제 private preview
  - OCI GenAI Sandboxes: beta
  - Modal 앱 관리형 예제: 샌드박스 최대 수명 15분(webhook 관리형 handler는 30분)
  - webhook 관리형 예제의 워커: 기본 30분이며, "can interrupt an active turn"
- **파트너 통합 사례:**
  - Vercel: OpenAI가 harness, 추론 루프, 세션 상태를 맡고 Vercel이 UI와 control plane을 맡습니다. 세션마다 public ingress 없는 Vercel Sandbox를 씁니다.
  - Cloudflare: Worker가 서명된 webhook을 받습니다. 세션별 Durable Object가 `codex exec-server`를 돌리는 세션 전용 Container를 관리합니다.
  - E2B: 채팅마다 Firecracker microVM 샌드박스를 붙이고, pause와 fork를 지원합니다. 요약기를 거쳐 읽었습니다.

### 6.5 보안 가이드 요지

- "Agent-generated code can access the files, credentials, and network available to its environment."
- "Run workloads in isolated compute, such as virtual machines. Use separate environments for users or workloads that must not share data."
- "Keep third-party credentials outside the environment. Where possible, route requests through a credential broker."

### 6.6 self-hosted executor의 명령 격리 (미확인 영역)

- **공개 문서:** hosted harness가 self-hosted executor에 **어떤 샌드박스·권한 프로필을 요청하는지** 적혀 있지 않습니다.
- **오픈소스 코드:**
  - `exec-server`는 오케스트레이터가 `process/start`에 sandbox context를 넣었을 때만 격리합니다. 넣지 않으면 `SandboxType::None`으로 실행합니다.
  - `codex exec-server`에는 샌드박스나 승인을 조정하는 CLI 플래그가 없습니다.
- **조사 중 로컬 실험** (WSL2, codex-cli 0.155.0-alpha.3.10, 직접 작성한 JSON-RPC 클라이언트):
  - context 없음: `sandboxType: none`으로 OS 사용자 권한 그대로 실행됐습니다. 작업 폴더 밖에 쓰기, `~/.ssh` 조회, 외부 HTTP가 모두 됐습니다.
  - restricted context: `linuxSeccomp`가 적용돼 폴더 밖 쓰기와 네트워크가 막혔습니다.
  - 이 실험은 **exec-server 단독** 동작만 보여줍니다. hosted harness가 실제로 무엇을 보내는지는 확인하지 못했습니다.
- **(추론)** OpenAI가 문서화하기 전까지는, 노트북에 띄운 executor가 해당 OS 사용자가 접근할 수 있는 전부에 닿는다고 가정하는 편이 안전합니다. 전용 VM·컨테이너와 저권한 사용자로 돌리세요.

---

## 7. 사람 승인 (Human-in-the-loop)

- **내장된 승인이나 명령 단위 권한 기능은 문서에 없습니다.**
  - `required_actions`는 `function_call`과 `environment_connection` 두 종류뿐입니다.
  - MCP 도구 설정에 `require_approval`이 없습니다.
  - `command_execution` item에도 승인 필드가 없습니다.
  - agent나 environment 파라미터에 `approval_policy`나 `sandbox_mode`도 없습니다.
- OpenAI의 런타임 비교 문서는 승인 제어를 **Agents SDK** 쪽 장점으로 적습니다.
  > "The Agents SDK gives your application control over deployment, storage, approvals, and runtime integration."
- **문서화된 패턴은 function tool로 승인을 직접 만드는 것**입니다. 공식 `sev_bot` 예제가 이렇게 합니다.
  1. 핸들러를 등록하지 않은 function tool `propose_rollback`을 둡니다. 모델이 호출하면 pending 상태로 남습니다.
     > "Do not register an automatic handler for propose_rollback: the function call must remain pending until a person decides."
  2. `agent.session.action_required` webhook이 오면, 앱이 세션에서 pending call을 조회합니다.
  3. Slack에 승인/거절 버튼을 올립니다.
  4. 사람이 누른 결과를 `agent.session.input.tool_result`로 보냅니다.
  5. 에이전트는 **같은 turn**을 이어갑니다.
  - "Approval is not execution: the example never contacts a deployment system."
- **대기 시간:**
  - pending `function_call`, session, turn 어디에도 만료 필드가 없습니다. 대기 중 turn 상태는 `waiting`입니다.
  - 문서에 최대 대기 시간이 없어서, 밤새 기다렸다가 승인하는 흐름을 막는 명시적 제한은 없습니다.
  - **(미확인)** hosted 샌드박스가 대기 중에도 keep-alive를 받는지(1시간 규칙), 그 시간이 과금되는지는 문서에 없습니다.
- `agent.session.input.cancel`은 turn을 멈추는 것이지 일시정지가 아닙니다.

---

## 8. 멀티 에이전트 (Subagents)

- **켜는 법:** `multi_agent.enabled: true`
- **도구:** harness가 subagent를 만들고, 메시지를 보내고, 기다리고, 중단하는 도구를 제공합니다.
- **item 타입:** 생성, 입력 전송, 재개, 대기, 중단, 종료에 해당하는 subagent 호출 item이 있습니다. 예: `create_subagent_call`, `send_subagent_input_call`, `wait_for_subagents_call`, `interrupt_subagent_call`.
- **동시 실행 수:** `max_concurrent_subagents`의 기본값은 **6**이고, coordinator는 세지 않습니다.
- **환경 공유:** "Creating a subagent does not create another environment." coordinator와 subagent가 한 파일시스템을 씁니다.
- **도구 상속:** subagent는 function tool을 쓸 수 없고, 설정된 MCP 도구, 자격증명, 허용 도구, web search 설정은 물려받습니다.

---

## 9. 가격 · 모델 · 베타 제약

### 9.1 과금 구조

- **Agents API 자체 요금은 없습니다.** 모델 토큰(API 요금), OpenAI 도구(표준 요금), hosted 샌드박스 컨테이너 시간을 냅니다.
- 한 작업이 모델을 여러 번 호출할 수 있고, subagent 호출도 포함됩니다. reasoning 토큰은 output으로 과금됩니다.
- **usage 필드:** `input_tokens`(cached 포함), `output_tokens`(reasoning 포함), `total_tokens`
  - best-effort이고 null일 수 있습니다. "These counts are not a final bill."
  - cache-write 개수가 따로 노출되지 않아서, cache-write 요금이 있는 모델의 정확한 비용은 계산할 수 없습니다.
- **인증:** Platform **API 키**만 문서화돼 있습니다. **(추론)** ChatGPT 구독으로 쓰는 방법은 문서에 없고, HN 댓글도 같은 점을 지적했습니다.

### 9.2 요금표 (pricing 페이지, 1M 토큰당, standard)

| 모델 | 입력 | 캐시 입력 | 캐시 쓰기 | 출력 | 긴 컨텍스트 (입력 / 캐시 입력 / 캐시 쓰기 / 출력) |
|---|---|---|---|---|---|
| gpt-6-astra | $10.00 | $1.00 | $12.50 | $50.00 | $20 / $2 / $25 / $75 |
| gpt-5.6-sol | $4.00 | $0.40 | $5.00 | $20.00 | $8 / $0.80 / $10 / $30 |
| gpt-5.6-terra | $2.00 | $0.20 | $2.50 | $12.00 | — |

- 문서와 발표 예제는 `gpt-6-astra`를 쓰고, cookbook은 `gpt-5.6-sol`과 `gpt-5.6-luna`를 씁니다. **공식 지원 모델 목록은 찾지 못했습니다.**
- **컨테이너(hosted shell / code interpreter 행):**
  - 1 GB $0.03 · 4 GB $0.12 · 16 GB $0.48 · 64 GB $1.92, 컨테이너 20분 세션당
  - "Eligible container sessions will be billed by the minute, with a 5-minute minimum per session." 2026-06-02부터 적용됐습니다.
  - Agents API 문서는 "standard container rates"라고만 적습니다.
  - **(미확인)** hosted 샌드박스가 이 행과 "eligible"에 해당하는지, 메모리 등급이 무엇인지, idle keep-alive 시간이 과금되는지

### 9.3 비용 감 잡기 (문서의 trace 예시)

- **trace 예시:** Tracing 가이드의 한 turn입니다. subagent 2개, 도구 호출 10회, 1분 37초, **총 252,468 토큰**.
  - root agent: 입력 126,390 / 출력 1,567
  - subagent A: 입력 34,075 / 출력 465
  - subagent B: 입력 89,304 / 출력 667
  - 모델은 명시돼 있지 않습니다.
- **(추론)** gpt-6-astra 표준 요금으로 계산하면 약 **$0.39**(입력 전부 캐시)~**$2.63**(캐시 없음)입니다. cache-write 비용은 따로입니다.

### 9.4 베타 제약

- tracing 설정이나 외부 trace exporter가 없습니다.
- Agents API 전용 rate limit이 문서화돼 있지 않습니다.
- **데이터 레지던시는 미국만** 지원하고, **ZDR은 지원하지 않습니다.**
  > "Choosing a self-hosted sandbox does not make the Agents API ZDR-eligible."
- 로드맵은 "iterate quickly ... as we work toward general availability" 정도이고, 날짜는 없습니다.

### 9.5 관측

- **대시보드:** Platform `Logs → Agents`(`platform.openai.com/logs?api=agents`)에서 세션 ID로 turn, 도구 호출, subagent, trace 타임라인을 봅니다.
  - 조회용 화면입니다. 거기서 대화하거나 승인하는 기능은 문서에 없습니다.
- tracing은 기본으로 켜져 있습니다.

---

## 10. Codex harness와의 관계, 그리고 배경

### 10.1 "같은 Codex"의 범위

- **블로그 설명:**
  - "The Agents API is powered by the open-source Codex harness ... OpenAI operates and maintains that harness while developers can inspect and learn from its public codebase."
  - "The Agents API provides versioned access to these capabilities with each model launch."
- **OpenAI 서버의 Codex 버전은 API로도 문서로도 알 수 없고, 고정할 수도 없습니다.**
  - 참고로 2026-09-13 npm 기준 self-hosted executor용 `@openai/codex@alpha`는 `0.155.0-alpha.3.10`, 안정판은 `0.154.0`입니다.
  - codex-acp 1.11.0은 `@openai/codex ^0.153.4`를 번들합니다.
  - openai/codex 저장소에는 버전이 다른 app-server와 exec-server의 호환 테스트가 있습니다.
- **로컬 Codex와 다른 점** (문서와 코드 기준):
  - **지시문:** Agents API 문서에 `AGENTS.md`, `config.toml`, `CODEX_HOME`이 한 번도 나오지 않습니다. 지시는 `agent.instructions`(기본 지시문 뒤에 붙음)와 skills로만 문서화돼 있습니다.
  - **(추론)** 오픈소스 코드에는 environment 파일시스템에서 `AGENTS.md`를 읽는 경로와 그 테스트가 있습니다. hosted에서 켜져 있는지는 미확인입니다.
  - **설정 주입:** hosted에서는 `CODEX_*` 환경변수가 거부되니, 내 Codex 설정을 넣을 수 없습니다.
  - **self-hosted 설정:** 코드상 harness가 executor 설정에서 읽는 것은 `mcp_servers`뿐입니다. model, approval, sandbox 설정은 읽지 않습니다.
  - **이벤트 종류:** Agents API item 타입은 메시지, reasoning, function_call, mcp_call, web_search_call, command_execution, subagent 호출 정도입니다. 로컬 app-server에 있는 plan, file change, review mode, compaction item은 없습니다.
  - **로컬 기능:** 슬래시 커맨드(`/review`, `/compact` 등)와 ChatGPT 로그인에 대응하는 것이 없습니다. compaction은 자동입니다.

### 10.2 Codex App Server: Agents API 이전의 "harness 개방"

출처: OpenAI 블로그 "Unlocking the Codex harness"(2026-02-04)와 Codex app-server 문서.

- **출발점:** VS Code 확장입니다.
  - 처음에는 Codex를 MCP 서버로 노출해 보았습니다(PR #2264, 2025-08-14 머지).
  - "maintaining MCP semantics in a way that made sense for VS Code proved difficult"
  - 그래서 TUI 루프를 본뜬 JSON-RPC 프로토콜을 만들었습니다. PR #4471(2025-09-30)에서 `codex mcp-server`와 `codex app-server`로 분리됐습니다.
- **정체:** JSON-RPC 프로토콜이자, Codex core thread를 담는 오래 사는 프로세스입니다.
- **기본 단위:** **Item / Turn / Thread**. thread는 생성, 재개, 포크, 보관할 수 있고 기록이 저장됩니다.
- **양방향 호출:**
  - 서버가 `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`로 클라이언트에 승인을 요청합니다.
  - 결정 값은 `accept` / `acceptForSession` / `decline` / `cancel`입니다.
- **전송:**
  - 기본은 stdio JSONL(`"jsonrpc":"2.0"` 헤더 생략)입니다.
  - WebSocket 리스너(실험적, 미지원)와 Unix socket도 있습니다.
- **클라이언트:**
  - VS Code 확장과 Desktop 앱이 App Server를 번들합니다. Xcode는 파트너로 언급됩니다.
  - Codex Web은 컨테이너 안에서 App Server를 띄우고, 브라우저는 백엔드와 HTTP/SSE로 통신합니다. "work continues even if the tab disappears"
- **위상이 엇갈립니다:**
  - 블로그: "the first-class integration method we maintain moving forward"
  - 현재 문서: "experimental and isn't supported for production workloads"
- 블로그는 교차 공급자 harness 프로토콜이 "converge on the common subset of capabilities"한다고 비판합니다. ACP를 직접 거명하지는 않습니다.

### 10.3 연표

| 날짜 | 사건 |
|---|---|
| 2025-08-14 | Codex MCP 서버에 JSON-RPC 요청 추가(PR #2264). App Server의 전신입니다. |
| 2025-08-27 | Zed가 ACP(Agent Client Protocol)를 발표했습니다. |
| 2025-09-30 | `codex app-server`가 분리됐습니다(PR #4471). |
| 2026-02-04 | "Unlocking the Codex harness"(App Server 설명) 게시 |
| 2026-02-10 | Responses API에 Hosted Shell 도구와 컨테이너 네트워킹이 출시됐습니다(changelog). |
| 2026-02-11 | "Harness engineering" 게시: "Humans steer. Agents execute.", 단일 Codex 실행 6시간 이상 |
| 2026-02-27 | "Stateful Runtime Environment for Agents in Amazon Bedrock" 발표 |
| 2026-04-15 | Agents SDK에 sandbox harness가 추가됐습니다("The next evolution of the Agents SDK"). |
| 2026-04-28 | Amazon Bedrock Managed Agents powered by OpenAI(limited preview) |
| 2026-09-10 | **Agents API public beta** |

Agents SDK 글(2026-04)이 이미 설계 방향을 밝혔습니다.
- "Separating harness and compute helps keep credentials out of environments where model-generated code executes."
- "When the agent's state is externalized, losing a sandbox container does not mean losing the run."
- 반면 관리형 agent API는 "constrain where agents run"이라는 트레이드오프도 함께 적었습니다.

---

## 11. 원시 API · Agents SDK와의 위치

### 11.1 OpenAI 공식 런타임 비교표 (Agents 가이드 원문)

|                          | Agents API | Agents SDK | Responses API |
|---|---|---|---|
| **Use for** | Long-running tasks where OpenAI manages the agent and saves its progress | Building agents with custom tools and workflows in your application | Calling models directly or building an agent from scratch |
| Where the agent runs | OpenAI runs a managed Codex harness | The SDK runs inside your application | Your application, with optional hosted orchestration |
| Agent integration effort | Low | Medium | High |
| State between tasks | Saved session configuration, turns, and items | Your storage and SDK sessions, or Responses conversation state | Manual history, response chaining, or Conversations |
| Tool execution | Service-connected tools, application function handlers, and an optional sandbox | Tools and integrations configured in your application | Hosted tools and tools your application runs |
| Execution environment | OpenAI hosted sandbox, self-hosted sandbox, or no sandbox | Your runtime and sandbox provider integrations | Your own execution environment |

- "The Agents API runs the Codex harness and manages the underlying agent infrastructure so you can focus on what your agents do. It includes automatic context compaction, multi-agent orchestration, programmatic tool calling, and support for MCP servers."
- "The Agents SDK gives your application control over deployment, storage, approvals, and runtime integration. Its runner handles the agent loop and handoffs."
- "An Agents API session, an SDK session, a Responses conversation, and a sandbox are different resources."

### 11.2 Chat Completions → Responses API → Agents API

**한 줄 요약:** 단계가 올라갈 때마다 OpenAI가 맡는 몫이 늘었습니다. 다만 내 함수를 실행하는 일은 끝까지 내 몫입니다.

1. **Chat Completions:** 대화 기록, 도구 루프, 도구 구현이 모두 내 몫입니다.
2. **Responses API:** 내장 도구, 서버 쪽 상태 옵션, background 모드, 켜야 동작하는 압축이 더해졌습니다. 그래도 내 함수가 끼면 루프는 여전히 내 코드가 돕니다.
3. **Agents API:** 루프, 세션, 자동 압축, 복구, subagent, 그리고 필요하면 실행 환경까지 OpenAI가 맡습니다.

**OpenAI의 설명 (원문)**
- **Responses API 출시 (2025-03-11)**
  - "The Responses API is our new API primitive for leveraging OpenAI's built-in tools to build agents. It combines the simplicity of Chat Completions with the tool-use capabilities of the Assistants API."
  - "the Responses API is a superset of Chat Completions with the same great performance, so for new integrations, we recommend starting with the Responses API."
- **Responses API의 남은 한계 (2026-03-11 엔지니어링 글)**
  - "When used with custom tools, the Responses API yields control back to the client, and the client requires its own harness for running the tools."
- **Codex harness와 Responses API (2026-01-23 "Unrolling the Codex agent loop")**
  - "The Codex CLI sends HTTP requests to the Responses API to run model inference."
  - **(추론)** Agents API가 Responses 위에 있다는 공식 문장은 없습니다. 이 인용과 `api.responses.write` 권한, "as in the Responses API"라는 가격 설명에서 추정한 것입니다.
- **Agents API 공개 (2026-09-10 changelog)**
  - "Released the Agents API in public beta. Build agents with a managed Codex harness while OpenAI handles session orchestration, context compaction, and recovery."

**현재 상태 (2026-09-13)**
- **Chat Completions:** 지원 중이고 deprecation 항목도 없습니다.
  - "While Chat Completions remains supported, Responses is recommended for all new projects."
  - GPT-6 Astra(2026-09-03)는 Chat Completions에도 제공되지만 "Tool calling requires the Responses API."입니다.
  - GPT-5.4부터 Chat Completions의 도구 호출은 `reasoning_effort` none일 때만 됩니다.
  - OpenAI 내장 도구는 쓸 수 없습니다("you cannot use OpenAI-hosted tools natively"). 웹 검색은 `gpt-5-search-api` 같은 검색 모델로만 가능합니다.
- **Responses API:** 권장 API이고 "an evolution of Chat Completions"로 소개됩니다. multi-agent 기능은 beta입니다.
- **Assistants API:** 2026-08-26에 종료됐습니다(§11.4).
- **Agents API:** 2026-09-10부터 public beta입니다.

**세 단계 비교**

| 차원 | Chat Completions | Responses API | Agents API |
|---|---|---|---|
| 주고받는 단위 | 보냄: `messages[]`(system·developer·user·assistant·tool 역할)<br>받음: `choices[].message`(`n`으로 여러 개) | 보냄: `input`(문자열이나 Items) + 최상위 `instructions`<br>받음: output Items(message, reasoning, function_call…)<br>`n` 없음 | 보냄: 세션 이벤트(`input.message`, `input.tool_result`)<br>받음: 이벤트 스트림<br>items·turns는 나중에 조회 |
| 대화 기록 | 내 앱이 매번 누적 `messages`를 보냄<br>`store`는 이어 쓰기용이 아니라 distillation·evals용 **(추론)** | 셋 중 선택: output Items 직접 재전송 / `previous_response_id`(기본 30일 이상 보관) / Conversations(삭제할 때까지) | OpenAI 세션이 설정·turn·items를 삭제할 때까지 보관<br>내 앱은 `session_id`만 저장하고 새 메시지만 보냄 |
| 도구 루프 | 내 앱: `message.tool_calls` 읽기 → 실행 → `role: "tool"` + `tool_call_id` 추가 → 다시 호출 | 둘로 나뉨: 내장 도구는 한 요청 안의 "agentic loop"<br>일반 function call은 모델 턴을 멈추고, 새 요청으로 `function_call_output`을 보냄<br>async 도구(2026-09-03)도 실행은 내 앱 | OpenAI의 Codex harness가 루프를 돎<br>`requires_action`에서 내 결과(`tool_result`)를 받으면 같은 turn을 이어감 |
| 함수 도구 실행 위치 | 내 앱 | 내 앱 | 여전히 내 앱<br>environment를 붙여도 거기서 자동 실행되지 않음<br>subagent는 함수 도구 불가 |
| 내장 도구 | 없음 | web search, file search, code interpreter, image generation, remote MCP, hosted shell<br>computer use 동작은 내 앱이 실행 | function, MCP(기본은 OpenAI에서 연결), web_search, tool_search, programmatic tool calling<br>environment가 있으면 Bash·apply-patch |
| 코드 실행 환경 | 없음 (내가 실행) | OpenAI 컨테이너(20분 미사용 시 만료, 복구 불가) 또는 내 런타임 | `none` / `openai_hosted`(1시간 규칙) / `self_hosted`(수명주기는 내 몫) |
| 긴 작업·끊김 | 호출 하나가 요청 하나(스트리밍 가능)<br>background·재개 옵션 없음 **(추론)** | `background: true`로 응답 **하나**를 비동기로 처리: 폴링, 취소, `starting_after`로 재개<br>여러 호출로 된 루프는 내 프로세스 **(추론)** | 영속 세션: "Closing an event stream does not cancel the task"<br>놓친 이벤트는 세션·items 조회로 복구 |
| 컨텍스트 관리 | 직접 자르거나 요약 (관련 파라미터 없음) **(추론)** | 켜야 동작: `context_management`(compact_threshold) 또는 `/responses/compact`<br>기본 `truncation`은 disabled라 넘치면 400 | 자동: "automatically compacts earlier context as a session approaches its context limit"<br>설정 필드 없음 |
| 여러 에이전트 | 내장 없음 **(추론)** | multi-agent beta(GPT-5.6, 기본 3개 동시)<br>함수 호출은 여전히 내 앱 | `multi_agent.enabled`, 기본 6개 동시, environment 공유 |
| 진행 전달 | 전체 응답 또는 `stream: true` SSE delta<br>폴링·webhook 없음 **(추론)** | 전체 응답, 타입 있는 SSE, background 폴링, webhook(`response.*`, 최대 72시간 재시도), WebSocket 모드 | 세션 이벤트 스트림, 세션·items·turns 조회, webhook(`agent.session.*`)<br>idle은 성공이 아님 |
| 과금 | 모델 요금의 토큰(`n`이면 모든 choice의 생성 토큰) | 토큰 + 내장 도구 요금<br>`previous_response_id` 체인 입력은 다시 과금 | 추가 요금 없음: 작업 안의 모든 모델 호출(subagent 포함) + 도구 + 컨테이너 |
| 내가 여전히 짜는 코드 | 히스토리 배열, 도구 분기 루프, 모든 도구 구현, 컨텍스트 자르기, 재시도 **(추론)** | 함수 호출 루프와 핸들러, 상태 전략, 압축 설정, 폴링이나 webhook 수신, 컨테이너 재사용<br>"the client requires its own harness" | 함수 핸들러와 `tool_result`(session·turn·call ID별 결과 저장), 이벤트·webhook 소비, `session_id` 매핑과 삭제, environment 선택(self-hosted면 수명주기) |
| OpenAI 런타임 표 | 표에 없음 | "Your application, with optional hosted orchestration" / 통합 난이도 **High** | "OpenAI runs a managed Codex harness" / 통합 난이도 **Low** |
| 데이터 통제 | 애플리케이션 상태 없음(예외 있음)<br>ZDR 적격(제한 있음) | ZDR 적격(제한 있음), 응답 30일 이상 보관<br>Conversations는 ZDR 불가 | 삭제할 때까지 보관, ZDR 불가, 레지던시는 미국만 |

**연표**

| 날짜 | 사건 | 개발자에게 달라진 것 |
|---|---|---|
| 2023-03-01 | Chat Completions API (gpt-3.5-turbo) | 텍스트 프롬프트 대신 역할이 붙은 `messages` 배열을 보냅니다. 상태가 없어 기록을 매번 다시 보냅니다. "ChatGPT models instead consume a sequence of messages together with metadata." |
| 2023-06-13 | Chat Completions에 function calling 추가 | 모델이 함수 인자(JSON)를 제안합니다. 실행하고 결과를 다시 보내는 것은 앱의 몫입니다. |
| 2023-11-06 | Assistants API beta (옆 갈래) | 처음으로 서버가 스레드를 들고, Code Interpreter와 Retrieval을 호스팅했습니다. 같은 날 `functions`는 `tools`로 대체됐습니다. |
| 2025-03-11 | Responses API, 내장 도구, Agents SDK | 타입 있는 Items, 선택형 서버 상태, 한 호출 안의 web search·file search·computer use가 생겼습니다. |
| 2025-05-20 | Responses에 remote MCP와 Code Interpreter | 블로그(5/21)에는 background 모드도 소개됐습니다. |
| 2025-06-24 | Webhooks | 연결을 열어 두거나 폴링하지 않고 결과를 받을 수 있게 됐습니다. |
| 2025-08-20 | Conversations API | Responses용 서버 쪽 대화 객체가 생겼습니다. Assistants 스레드를 대체합니다. |
| 2025-08-26 | Assistants API 종료 공지 | 1년 뒤 종료하고, Responses와 Conversations로 옮기라고 알렸습니다. |
| 2025-12-11 | `/responses/compact` | 개발자가 호출하는 방식의 압축이 생겼습니다. |
| 2026-01-23 | "Unrolling the Codex agent loop" | Codex harness가 Responses API로 루프를 돈다고 밝혔습니다. |
| 2026-02-10 | Responses에 서버 쪽 compaction과 Hosted Shell | 임계값을 정하면 자동으로 압축됩니다. 네트워크가 되는 호스팅 셸 컨테이너도 추가됐습니다. |
| 2026-03-11 | Responses 컴퓨터 환경 엔지니어링 글 | 사용자 정의 도구를 쓰면 제어가 클라이언트로 돌아가고, "the client requires its own harness"라고 적었습니다. |
| 2026-08-26 | Assistants API 종료 | "The Assistants API shut down on August 26, 2026." |
| 2026-09-03 | GPT-6 Astra 출시 | Chat Completions에도 올라왔지만, 도구 호출은 Responses API가 필요합니다. |
| 2026-09-10 | **Agents API public beta** | OpenAI가 Codex harness를 운영합니다: 루프, 영속 세션, 압축, 복구, subagent, 선택형 hosted 샌드박스. |

**아이 눈높이 세 줄**
1. **Chat Completions:** 내가 지난 대화를 매번 다 보내야 하고, AI가 도구를 쓰자고 하면 내가 직접 실행해서 결과를 알려 줘요.
2. **Responses API:** 검색 같은 OpenAI 도구는 OpenAI가 대신 쓰고 대화도 기억해 줄 수 있어요. 하지만 내 함수는 여전히 내가 돌리고 다시 물어봐야 해요.
3. **Agents API:** OpenAI 서버의 에이전트가 반복 작업, 기억, 긴 일 이어 가기를 맡아요. 나는 내 함수 결과만 돌려주면 돼요.

**같은 함수 호출: Chat Completions 원문** (function calling 가이드의 chat 탭, Python, 67줄)

같은 `get_horoscope` 예제의 Responses API 버전은 §11.5에 있습니다. 모델이 `gpt-5.6`인 이유는 "GPT-6 Astra requires the Responses API for tool calling."이기 때문입니다.

```python
from openai import OpenAI
import json

client = OpenAI()

# 1. Define a list of callable tools for the model
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_horoscope",
            "description": "Get today's horoscope for an astrological sign.",
            "parameters": {
                "type": "object",
                "properties": {
                    "sign": {
                        "type": "string",
                        "description": "An astrological sign like Taurus or Aquarius",
                    },
                },
                "required": ["sign"],
                "additionalProperties": False,
            },
            "strict": True,
        },
    },
]


def get_horoscope(sign):
    return f"{sign}: Next Tuesday you will befriend a baby otter."


messages = [{"role": "user", "content": "What is my horoscope? I am an Aquarius."}]

# 2. Prompt the model with tools defined
response = client.chat.completions.create(
    model="gpt-5.6",
    messages=messages,
    tools=tools,
)

messages.append(response.choices[0].message)

for tool_call in response.choices[0].message.tool_calls or []:
    if tool_call.function.name == "get_horoscope":
        # 3. Execute the function logic for get_horoscope
        args = json.loads(tool_call.function.arguments)
        horoscope = get_horoscope(args["sign"])

        # 4. Provide function call results to the model
        messages.append(
            {
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps({"horoscope": horoscope}),
            }
        )

response = client.chat.completions.create(
    model="gpt-5.6",
    messages=messages,
    tools=tools,
)

# 5. The model should be able to give a response!
print(response.choices[0].message.content)
```

**Chat Completions → Responses 필드 대응**

| Chat Completions | Responses API |
|---|---|
| `messages` | input Items |
| `message.tool_calls` | `function_call` item |
| `role: "tool"` + `tool_call_id` | `function_call_output` + `call_id` |
| `{"type":"function","function":{…}}`로 감싼 도구 정의 | 최상위에 바로 쓰는 도구 정의 |

**Agents API에서 세션에 함수 도구 붙이기** (tool search 가이드, 원문 그대로)

docs에서 `agent.tools`에 함수를 넣어 `sessions.create`를 호출하는 Python 예시는 이것뿐입니다. `tool_search`와 `defer_loading`은 이 예시에 따라 붙은 설정입니다.

```python
from openai import OpenAI

client = OpenAI()

result = client.beta.agents.sessions.create(
    agent={
        "model": "gpt-6-astra",
        "tools": [
            {"type": "tool_search"},
            {
                "type": "function",
                "name": "lookup_account",
                "description": "Find an account by its account number.",
                "parameters": {
                    "type": "object",
                    "properties": {"account_id": {"type": "string"}},
                    "required": ["account_id"],
                    "additionalProperties": False,
                },
                "defer_loading": True,
            },
        ],
    },
    environment={"type": "none"},
    input=[
        {
            "role": "user",
            "content": [{"type": "input_text", "text": "Look up account 42."}],
        }
    ],
)
print(result.id)
```

- **결과 돌려주기:** §5.2의 `tool_result` 코드를 쓰면 됩니다. 기록을 다시 보내거나 모델을 다시 부르는 코드가 없습니다. "The harness continues the turn after it receives the required results."
- **처음부터 끝까지 이어진 예시:** cookbook `apps/data_analyst/agent.py`에 세션 생성 스트림, `requires_action`, `tool_result`, 후속 입력이 모두 들어 있습니다.

**참고할 점**
- 같은 과제를 세 API로 모두 구현한 공식 예제는 없습니다. Chat Completions와 Responses는 `get_horoscope`를 공유하지만, Agents API 문서는 `lookup_account`·`get_customer`를 씁니다.
- Chat Completions 출시일 2023-03-01은 Wayback 스냅숏의 byline과 openai-python v0.27.0 릴리스 시각에서 확인했습니다. RSS pubDate(2024-04-24)는 페이지가 나중에 갱신된 날짜입니다.
- Chat Completions의 긴 작업, 컨텍스트, 여러 에이전트, 진행 전달 칸은 레퍼런스에 해당 파라미터가 **없다는 것**에 근거한 **(추론)**입니다.

### 11.3 관심사별로 누가 맡나: Responses API · Agents SDK · Agents API

같은 관심사를 세 런타임이 어떻게 나눠 맡는지 적었습니다. 모든 행은 공식 문서로 반박 검증했고, 틀린 항목은 수정본만 남겼습니다.

#### ① 에이전트 루프

- **Responses API:** 내 코드가 루프를 돕니다.
  - built-in 도구(web search, file search, hosted shell)는 한 요청 안에서 끝납니다.
  - 일반 function call은 모델의 턴을 멈춥니다. 결과는 새 요청에 `function_call_output`으로 보냅니다.
  - 원격 MCP도 기본적으로 승인을 요청(`mcp_approval_request`)하고, 답은 새 요청으로 보냅니다.
  - 공식 문서는 이 흐름을 "for as many tool calls as the task requires" 이어가라고 적습니다.
  - `max_tool_calls`는 응답 하나에서 built-in 도구 호출 수만 제한합니다.
  - async 도구(`async: true`)를 쓰면 모델이 계속 일하지만, 결과는 여전히 이후 요청으로 보냅니다.
- **Agents SDK:** 내 프로세스 안의 Runner가 model → tools → handoffs를 최종 출력까지 반복합니다.
  - `max_turns`의 기본값은 10이고, 넘기면 `MaxTurnsExceeded`가 납니다. `None`으로 두면 제한이 없습니다.
- **Agents API:** OpenAI가 루프를 돕니다. "The OpenAI-hosted Codex instance that runs the model and tool loop and maintains the agent's session."
- **Agents API에서도 내 몫:** 입력 이벤트를 보내고, 이벤트와 required action에 반응합니다. 루프 코드는 쓰지 않습니다.

#### ② 함수 도구 실행

- **Responses API:** 내가 실행합니다.
  - `function_call` item을 읽고 실행한 뒤, 같은 `call_id`로 `function_call_output`을 보냅니다.
  - async 도구도 마찬가지입니다. "Async tools don't move execution to OpenAI."
- **Agents SDK:** Runner가 `@function_tool` 함수를 프로세스 안에서 자동으로 호출합니다.
- **Agents API:** 여전히 내가 실행합니다.
  - 흐름: `requires_action` → `agent.session.input.tool_result`(`turn_id`, `call_id`)
  - "Attaching an environment to a session does not automatically run function tools there."
  - openai-python 3.13.0의 `sessions.stream(tool_handlers=...)`가 결과 제출을 대신해 줍니다. cookbook 앱이 쓰지만 가이드에는 문서화돼 있지 않습니다.
- **Agents API에서도 내 몫:**
  - 핸들러를 호스팅합니다.
  - 부작용 있는 함수는 session·turn·call ID별로 결과를 저장합니다.
  - subagent는 함수 도구를 쓸 수 없습니다.

#### ③ 대화 상태와 보관 기간

- **Responses API:** 셋 중 하나를 고릅니다.
  - 수동 히스토리(`store=false`와 함께 쓸 수 있음)
  - `previous_response_id`
  - Conversations
  - **보관 기간:** response 객체는 기본 30일, conversation은 삭제할 때까지 남습니다.
  - **체인 과금:** `previous_response_id` 체인에서는 이전 입력 토큰이 다시 과금됩니다.
  - **reasoning 모델:** 수동 히스토리로 쓰면 암호화된 reasoning item을 포함해 output item을 모두 다시 넣습니다.
- **Agents SDK:** 대화마다 하나를 고릅니다.
  - 로컬 히스토리(`to_input_list`)
  - 내 저장소에 두는 SDK 세션(SQLite, Redis, SQLAlchemy, MongoDB, Dapr 등)
  - OpenAI Conversations(`OpenAIConversationsSession`)
  - `previous_response_id`
- **Agents API:** 서버 세션(설정·turn·items)이 "Until deleted" 보관됩니다. 후속 입력은 같은 `session_id`에 message를 보내면 되고, 히스토리를 다시 보내지 않습니다.
- **Agents API에서도 내 몫:**
  - 사용자와 `session_id`의 매핑을 저장합니다.
  - 필요 없어진 세션을 삭제합니다("Delete a session when your application no longer needs it").
  - 저장된 items로 UI를 다시 그립니다.

#### ④ 컨텍스트 압축

- **Responses API:** 켜야 동작합니다(opt-in).
  - 방법 1: `context_management: [{type: "compaction", compact_threshold}]`. 같은 요청 안에서 서버가 압축합니다.
  - 방법 2: 별도로 `POST /responses/compact`를 호출합니다.
  - 기본 `truncation`은 `disabled`라서, 컨텍스트가 넘치면 400으로 실패합니다.
  - Responses multi-agent beta에서는 압축이 암묵적으로 켜지고 `/responses/compact`는 지원하지 않습니다.
- **Agents SDK:** 연결은 내 프로세스가 하고 계산은 OpenAI가 합니다. `OpenAIResponsesCompactionSession`, SandboxAgent 기본 capability의 `Compaction()`이 있습니다.
- **Agents API:** 자동입니다. "includes automatic context compaction"이고, 블로그는 "automatically compacts earlier context as a session approaches its context limit"라고 설명합니다. 세션 생성 파라미터에 압축 설정이 없습니다.
- **Agents API에서도 내 몫:** 연결할 것이 없습니다. 조정할 수도 없는 것으로 보입니다 **(추론)**.

#### ⑤ 긴 작업 · 끊김 · 복구

- **Responses API:** 복구는 요청 하나 단위입니다 **(추론)**.
  - **background 모드:** `background=true`면 `retrieve`로 폴링하고 `cancel`할 수 있습니다. `stream=true`로 만든 background 응답만 `starting_after`로 스트림을 이어 받습니다.
  - **시간 제한:** openai-python 기본 타임아웃은 10분, WebSocket 연결은 최대 60분입니다.
  - **자동 삭제:** `store`를 생략했거나 false인 background 응답은 약 10분 뒤 삭제됩니다.
  - **재시도:** Assistants 마이그레이션 가이드는 재시도를 앱의 몫으로 적습니다.
- **Agents SDK:** 루프는 내 프로세스와 함께 삽니다.
  - RunState는 직렬화할 수 있습니다.
  - 재시작을 넘어 계속되는 실행은 Dapr, Temporal, Restate, DBOS 통합으로 합니다.
  - sandbox agent는 세션 상태나 스냅숏에서 재개합니다.
- **Agents API:** "recovery"를 OpenAI가 관리합니다.
  - "Closing an event stream does not cancel the task."
  - 놓친 이벤트는 다시 오지 않으니 세션과 items를 조회합니다.
  - 블로그에는 "infrastructure that keeps them running reliably for days"라는 표현이 있지만, 기간을 보장한다는 문서는 없습니다.
- **Agents API에서도 내 몫:**
  - 다시 붙고, items로 상태를 복원합니다.
  - 저장한 도구 결과는 같은 `turn_id`·`call_id`로 다시 보냅니다.
  - self-hosted면 새 입력이 executor를 최대 5분 기다립니다. turn 중간에 끊기면 도구가 실패할 수 있고, 크래시 후 입력 복구는 보장되지 않습니다.

#### ⑥ 샌드박스 · 코드 실행 환경

- **Responses API:** hosted shell(`container_auto`)이나 Code Interpreter 컨테이너를 씁니다.
  - **수명:** 20분 동안 쓰지 않으면 만료되고, 다시 활성화할 수 없습니다.
  - **네트워크:** 기본은 **아웃바운드 없음**입니다. 켜려면 조직 allow list와 `network_policy`가 필요합니다.
  - **내가 실행하는 도구:** local shell, apply_patch, computer use.
- **Agents SDK:** SandboxAgent(beta)에 클라이언트를 골라 붙이고, harness는 내 앱이 돌립니다.
  - 로컬 클라이언트: UnixLocal, Docker
  - 호스팅 클라이언트: Blaxel, Cloudflare, Daytona, E2B, Modal, Runloop, Vercel
- **Agents API:** `none` · `openai_hosted` · `self_hosted` 중 고릅니다.
  - **openai_hosted:** `/workspace`에서 harness가 명령을 직접 실행합니다.
    - 네트워크 기본값은 **enabled**입니다.
    - 활동과 keep-alive가 1시간 끊기면 삭제될 수 있습니다.
    - outputs는 artifact로 남습니다.
  - **self_hosted:** `codex exec-server`가 명령을 실행합니다.
- **Agents API에서도 내 몫:**
  - self-hosted는 "provisioning, reconnection, shutdown, and any files you need to preserve"를 맡습니다.
  - hosted는 필요하면 네트워크를 제한합니다. Responses 컨테이너와 기본값이 반대입니다.
  - 필요한 outputs는 저장합니다.

#### ⑦ 내장 도구 · MCP · 자격증명

- **Responses API:**
  - **도구 목록:** function calling, web search, remote MCP, skills, shell, computer use, image generation, file search, tool search, programmatic tool calling. Code Interpreter와 apply_patch는 별도 가이드에 있습니다.
  - **원격 MCP:** API가 서버를 직접 부르고, 요청마다 `authorization`에 OAuth 토큰을 넣습니다. 승인이 기본으로 켜져 있습니다.
- **Agents SDK:**
  - **Hosted 도구:** WebSearchTool, FileSearchTool, CodeInterpreterTool, HostedMCPTool, ImageGenerationTool, ToolSearchTool, 그리고 hosted 컨테이너의 ShellTool.
  - **MCP:** HostedMCPTool은 Responses API가 연결을 맡습니다. 그 외에는 내 프로세스가 stdio, SSE, Streamable HTTP로 연결합니다.
- **Agents API:**
  - **도구 5종:** `function`, `mcp`, `web_search`, `tool_search`, `programmatic_tool_calling`(기본 켜짐).
  - **MCP 연결 방식:** service, environment HTTP, stdio.
  - **자격증명:** 인라인 `transport.authorization`/`headers`(암호화)로 넣거나 vault("without receiving the secret values", 서비스 연결 전용)에 둡니다.
- **Agents API에서도 내 몫:**
  - 공급자 OAuth 동의 흐름
  - vault 관리(`api.vaults.*`)
  - stdio 자격증명은 environment 안의 코드가 읽을 수 있다는 점

#### ⑧ 사람 승인

- **Responses API:** MCP `require_approval`(기본) → `mcp_approval_request` → 새 요청에 `mcp_approval_response`. 함수는 실행 시점을 내가 정하는 것으로 승인합니다 **(추론)**.
- **Agents SDK:** 승인이 **내장**돼 있습니다.
  - `needs_approval=True`로 두면 `interruptions`와 재개 가능한 RunState가 나옵니다.
  - 승인이나 거절 후 같은 run을 이어갑니다.
  - 상태는 직렬화할 수 있습니다.
- **Agents API:** 승인 프리미티브가 문서에 없습니다 **(추론)**.
  - MCP에 `require_approval`이 없습니다.
  - `required_actions`는 `function_call`과 `environment_connection`뿐입니다.
  - 문서화된 패턴은 핸들러 없는 함수 도구에 `action_required` webhook을 결합하는 방식입니다(§7).
- **Agents API에서도 내 몫:** 승인 UX 전체(Slack 버튼 등)와 pending call 보존. shell 명령은 사람에게 묻는 대신 샌드박스·네트워크 정책으로 제한합니다.

#### ⑨ 서브에이전트 · 병렬

- **Responses API:** Multi-agent beta(`OpenAI-Beta: responses_multi_agent=v1`)가 있습니다.
  - 오케스트레이션은 호스팅되고, `max_concurrent_subagents` 기본값은 3입니다.
  - 어느 에이전트의 함수 호출이든 실행은 내 몫입니다.
  - `/responses/compact`, `reasoning.summary`, `max_tool_calls`는 지원하지 않습니다.
- **Agents SDK:** handoff와 agents-as-tools를 코드로 연결합니다. 실험 기능 `OpenAIHostedMultiAgentModel`도 있습니다.
- **Agents API:** `multi_agent.enabled`. 기본 6개가 동시에 돌고, environment를 공유하며, 함수 도구는 쓸 수 없습니다.
- **Agents API에서도 내 몫:** 세션 여러 개로 나누는 일은 직접 합니다. Agents API 전용 rate limit은 문서에 없습니다.

#### ⑩ 진행 전달

- **Responses API:**
  - `stream=true` SSE, background 폴링
  - webhook `response.completed` / `failed` / `cancelled` / `incomplete`. 최대 72시간 재시도합니다.
  - WebSocket 모드에서는 turn 중 `response.steer`가 있습니다(gpt-6-astra).
- **Agents SDK:** 프로세스 안에서 `Runner.run_streamed()` 이벤트를 받습니다. 사용자에게 전달하는 일은 앱의 몫입니다 **(추론)**.
- **Agents API:**
  - 이벤트 스트림과 세션·items·turns 조회
  - webhook `agent.session.*`
  - 작업 중에 message를 보내면 steer가 됩니다.
- **Agents API에서도 내 몫:** 검증된 webhook 수신기, idle을 성공으로 오해하지 않기, 사용자 UI.

#### ⑪ 가격

- **Responses API:** 토큰에 도구 요금이 더해집니다.
  - web search: 1k당 $10 + 콘텐츠 토큰
  - 컨테이너: 1 GB $0.03 ~ 64 GB $1.92(20분 세션당). eligible이면 분 단위, 5분 최소
  - file search: 1k당 $2.50 + 저장 GB·일당 $0.10
  - `previous_response_id` 체인 입력은 다시 과금됩니다.
- **Agents SDK:** SDK 자체는 MIT로 무료입니다. 새 기능은 "standard API pricing, based on tokens and tool use"이고, 내 컴퓨트와 샌드박스 공급자 비용이 따로 듭니다 **(추론)**.
- **Agents API:** 추가 요금이 없습니다. 한 작업의 여러 모델 호출은 각각 "as in the Responses API" 가격·캐싱 규칙을 따릅니다.
- **Agents API에서도 내 몫:** subagent·재시도·도구·샌드박스·외부 비용을 모두 합쳐 추정합니다. usage는 best-effort이고 cache-write 수가 없습니다.

#### ⑫ 제어 · 유연성

- **Responses API:**
  - 요청 필드(model, reasoning, tools, truncation, store)는 전부 내가 정합니다.
  - OpenAI 모델만 쓸 수 있습니다 **(추론)**.
  - Amazon Bedrock에 OpenAI 호환 Responses 엔드포인트가 있습니다(2026-06-01 changelog). 지원 모델과 기능은 리전마다 다릅니다.
- **Agents SDK:** 가장 이식성이 높고, 내 코드가 도는 어디서나 돕니다.
  - **비OpenAI 모델 연결:** `set_default_openai_client`, 사용자 정의 `ModelProvider`, `Agent.model`, Any-LLM·LiteLLM 어댑터(beta)
  - **기본 모델:** `gpt-5.6-luna`
- **Agents API:**
  - model·instructions·tools·reasoning·text·multi_agent를 세션마다 인라인으로 주거나, 저장한 agent를 `agent_id`로 재사용합니다.
  - harness는 OpenAI가 운영합니다.
  - environment는 hosted, self-hosted, 파트너 중에서 고릅니다.
- **Agents API에서도 내 몫:** 컴퓨트 위치를 고르고, 문서화된 설정 밖의 harness 동작은 받아들입니다 **(추론)**.

#### ⑬ 데이터 통제

- **Responses API:**
  - `/v1/responses`는 ZDR 적격입니다. ZDR에서는 `store`가 false로 강제되는 등 제한이 있습니다.
  - `/v1/conversations`는 삭제할 때까지 보관되고, ZDR이 불가합니다.
  - 데이터 레지던시는 여러 지역을 지원합니다.
- **Agents SDK:** MIT 코드이고, 데이터 통제는 고른 백엔드를 따릅니다 **(추론)**.
- **Agents API:**
  - `/v1/agents`는 삭제할 때까지 보관되고, abuse monitoring은 30일입니다.
  - ZDR이 불가합니다.
  - 데이터 레지던시는 미국만 지원합니다.
  - `/v1/assistants`·`/v1/threads`와 같은 행입니다.
- **Agents API에서도 내 몫:** ZDR 대상이거나 비미국 레지던시가 필요한 데이터는 넣지 않습니다. 세션을 삭제하고, 출구 계획을 세웁니다(세션·vault·이벤트 형식이 OpenAI 전용) **(추론)**.

#### ⑭ 성숙도

- **Responses API:** 플래그십입니다("Always start with the Responses API."). multi-agent는 beta입니다.
- **Agents SDK:** 0.x(Python v0.22.2)이고 "still evolving rapidly"라고 적혀 있습니다. sandbox agents는 beta입니다(Python 2026-04-15, TypeScript 2026-05-06).
- **Agents API:** 2026-09-10 공개 베타이고, trace exporter가 없습니다.

### 11.4 Assistants API: 서버가 상태를 쥐었다가 사라진 API

**구조와 도구 호출 방식**
- **개념:** Assistant(model, instructions, tools), Thread(대화 상태), Run(Thread에 대한 비동기 실행), Run Step.
- **도구 호출:** Run이 `requires_action`이 되면 `required_action.submit_tool_outputs.tool_calls`의 결과를 모아 `submit_tool_outputs`로 **한 번에** 제출했습니다.
  - 레거시 문서: "all `tool_calls` need to be submitted at the same time"
  - "runs expire ten minutes after creation"

**연혁**
- **2024-12-18:** v1 beta(`OpenAI-Beta: assistants=v1`) 종료
- **2025-03-11:** Responses API 출시. Assistants 기능을 Responses로 옮기고 2026년에 종료한다고 예고했습니다("after achieving full feature parity").
- **2025-08-26:** 개발자에게 종료 공지. deprecations 페이지 제목에는 2025-08-20으로 표기돼 있습니다.
- **2026-08-26:** "The Assistants API was officially sunset on August 26, 2026, and is no longer available."

**마이그레이션 매핑**

| Assistants | 옮겨 간 곳 | 비고 |
|---|---|---|
| Assistants | Prompts | |
| Threads | Conversations | |
| Runs | Responses | "tool call loops are explicitly managed" |
| Run steps | Items | |

- "Your application code now handles orchestration (history pruning, tool loop, retries)."
- Prompts 쪽도 정리 중입니다. `v1/prompts`와 재사용 prompt 객체는 2026-11-30 종료 예정입니다.

**Agents API와 닮은 점 · 다른 점**
- **닮은 점:** 서버 쪽 상태, `requires_action` 상태, 결과를 서비스에 제출하는 모양.
- **다른 점:**
  - 결과를 `turn_id`/`call_id`로 식별합니다.
  - `agent.session.input.tool_result` 이벤트에 success/error를 명시합니다.
  - 루프를 호스팅된 Codex harness가 돕니다.
- 데이터 컨트롤 표에서 `/v1/agents`는 `/v1/assistants`·`/v1/threads`와 같은 행입니다(30일, Until deleted, ZDR 불가).

### 11.5 코드로 보는 차이 (공식 원문)

**Responses API — 함수 호출 한 번 왕복** (function calling 가이드, 원문 그대로)

공식 예제에는 반복문이 없습니다. 도구 호출이 여러 번이면 이 왕복을 반복하고, 재시도와 압축을 챙기는 것은 개발자 몫입니다.

```python
from openai import OpenAI
import json

client = OpenAI()

# 1. Define a list of callable tools for the model
tools = [
    {
        "type": "function",
        "name": "get_horoscope",
        "description": "Get today's horoscope for an astrological sign.",
        "parameters": {
            "type": "object",
            "properties": {
                "sign": {
                    "type": "string",
                    "description": "An astrological sign like Taurus or Aquarius",
                },
            },
            "required": ["sign"],
        },
    },
]


def get_horoscope(sign):
    return f"{sign}: Next Tuesday you will befriend a baby otter."


# Create a running input list we will add to over time
input_list = [{"role": "user", "content": "What is my horoscope? I am an Aquarius."}]

# 2. Prompt the model with tools defined
response = client.responses.create(
    model="gpt-6-astra",
    tools=tools,
    input=input_list,
)

# Save function call outputs for subsequent requests
input_list += response.output

for item in response.output:
    if item.type == "function_call":
        if item.name == "get_horoscope":
            # 3. Execute the function logic for get_horoscope
            sign = json.loads(item.arguments)["sign"]
            horoscope = get_horoscope(sign)

            # 4. Provide function call results to the model
            input_list.append(
                {
                    "type": "function_call_output",
                    "call_id": item.call_id,
                    "output": horoscope,
                }
            )

print("Final input:")
print(input_list)

response = client.responses.create(
    model="gpt-6-astra",
    instructions="Respond only with a horoscope generated by a tool.",
    tools=tools,
    input=input_list,
)

# 5. The model should be able to give a response!
print("Final output:")
print(response.model_dump_json(indent=2))
print("\n" + response.output_text)
```

**Responses API — 켜야 동작하는 압축을 내 루프 안에서** (compaction 가이드, 원문 그대로)

`keep_going`과 `get_next_user_input()`은 원문에도 정의되지 않은 자리표시자입니다.

```python
conversation = [
    {
        "type": "message",
        "role": "user",
        "content": "Let's begin a long coding task.",
    }
]

while keep_going:
    response = client.responses.create(
        model="gpt-5.3-codex",
        input=conversation,
        store=False,
        context_management=[{"type": "compaction", "compact_threshold": 200000}],
    )

    conversation.extend(response.output)

    conversation.append(
        {
            "type": "message",
            "role": "user",
            "content": get_next_user_input(),
        }
    )
```

**Responses API — background 모드 폴링** (background 가이드, 원문 그대로)

긴 응답 하나가 타임아웃을 견디게 해 줄 뿐이고, 여러 단계로 된 영속 실행을 만들어 주지는 않습니다.

```python
from openai import OpenAI
from time import sleep

client = OpenAI()

resp = client.responses.create(
    model="gpt-6-astra",
    input="Write a very long novel about otters in space.",
    background=True,
)

while resp.status in {"queued", "in_progress"}:
    print(f"Current status: {resp.status}")
    sleep(2)
    resp = client.responses.retrieve(resp.id)

print(f"Final status: {resp.status}\nOutput:\n{resp.output_text}")
```

**Agents SDK — Runner가 루프를 가진다** (openai-agents-python quickstart, 원문 그대로)

```python
import asyncio
from agents import Agent, Runner
from agents.decorators import tool


@tool
def history_fun_fact() -> str:
    """Return a short history fact."""
    return "Sharks are older than trees."


agent = Agent(
    name="History Tutor",
    instructions="Answer history questions clearly. Use history_fun_fact when it helps.",
    tools=[history_fun_fact],
)


async def main():
    result = await Runner.run(
        agent,
        "Tell me something surprising about ancient life on Earth.",
    )
    print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

**Agents SDK — 내장 승인** (guardrails-approvals 가이드, 원문 그대로)

```python
import asyncio

from agents import Agent, Runner, function_tool


@function_tool(needs_approval=True)
async def cancel_order(order_id: int) -> str:
    return f"Cancelled order {order_id}"


agent = Agent(
    name="Support agent",
    instructions="Handle support requests and ask for approval when needed.",
    tools=[cancel_order],
)


async def main() -> None:
    result = await Runner.run(agent, "Cancel order 123.")

    if result.interruptions:
        state = result.to_state()
        for interruption in result.interruptions:
            state.approve(interruption)
        result = await Runner.run(agent, state)

    print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

Agents API 쪽 코드는 §4.1(세션 생성·이어 입력)과 §5.2(`tool_result`)를 보세요.

**주의할 점**
- 공식 예제는 런타임마다 과제가 다릅니다(`get_horoscope`, `history_fun_fact`, `get_customer`). 같은 과제를 세 방식으로 구현한 공식 자료는 없습니다.
- **(추론)** Agents API harness가 내부적으로 `/v1/responses`를 호출한다는 문서는 없습니다. 모델 추론에 `api.responses.write` 권한이 필요하고, 가격 설명에 "as in the Responses API"라는 표현이 있어 그렇게 추정할 뿐입니다.

---

## 12. 참고: ACP로 Codex를 쓰는 경우와의 비교

같은 Codex를 **ACP로 에디터에서 쓸 때**(Zed·JetBrains → `codex-acp` 어댑터)와 **Agents API로 쓸 때**를 비교합니다.

### 12.1 ACP × Codex의 실제 구조

- **ACP란:** "standardizes communication between code editors/IDEs and coding agents". 사용자가 주로 에디터에 있다고 가정합니다.
- **전송:**
  - 표준 전송은 **JSON-RPC 2.0 over stdio**(한 줄에 메시지 하나)뿐입니다. 클라이언트가 에이전트를 **하위 프로세스로 띄웁니다**.
  - Streamable HTTP/WebSocket 원격 전송 RFD는 2026-04-22 Draft를 거쳐 2026-07-02 Active 단계(작업 중)에 올라 있습니다. Preview·Completed 단계는 아닙니다. 커뮤니티 브리지(acp_rpc_bridge, ACP Remote, Runmote 등)가 빈자리를 메웁니다.
- **양방향 호출:** 클라이언트는 `session/prompt`를 보내고, 에이전트는 `session/update`를 스트리밍하며 `session/request_permission`으로 **클라이언트에 승인을 요청**합니다.
- **Codex 연결:**
  - OpenAI의 `codex` CLI에는 내장 ACP 모드가 없습니다(issue #30052 열림).
  - 대신 `@agentclientprotocol/codex-acp`(1.11.0, 작성자 OpenAI · JetBrains · Zed)가 **`codex app-server`를 자식 프로세스로 띄우고 ACP ↔ App Server를 번역**합니다.
  - 체인: 에디터 →(ACP stdio) codex-acp →(App Server JSON-RPC stdio) codex app-server → 모델 API
  - 명령 실행, 파일 편집, 샌드박스, 세션 파일은 **어댑터를 띄운 머신**에 있습니다. 보통은 노트북이고, Zed SSH 원격 프로젝트면 원격 호스트입니다.
- **승인 모드:** codex-acp의 세션 모드는 승인 정책과 샌드박스 조합의 프리셋입니다.
  - 기본값 "Approve for me": `auto_review`. 리뷰어 에이전트가 판단하고, 잠재적으로 위험하다고 감지된 것만 사람에게 묻습니다.
  - "Ask for approval": 샌드박스 경계를 넘을 때마다 사람에게 묻습니다.
  - "Full access": 묻지 않습니다.
  - 워크스페이스 쓰기 샌드박스는 기본적으로 네트워크가 꺼져 있습니다.
- **에디터를 닫으면:**
  - stdin이 닫히고, codex-acp가 app-server의 stdin을 닫은 뒤 **2초 후에도 살아 있으면 종료**시킵니다.
  - 진행 중 작업은 멈추고, 미완료 백그라운드 작업은 failed로 보고됩니다.
  - 기록은 `~/.codex/sessions/.../rollout-*.jsonl`에 남아 `session/load`나 `session/resume`으로 이어갈 수 있습니다.
- **헤드리스 사용:** 가능합니다.
  - ACP SDK의 클라이언트 쪽이나 `acpx` 같은 CLI로 에디터 없이 구동할 수 있습니다.
  - 다만 **어댑터를 띄운 그 머신이 계속 켜져 있어야** 하고, 네트워크 너머에서 부를 표준 엔드포인트는 없습니다.
- **인증과 과금:** ChatGPT 로그인(플랜의 Codex 사용량), API 키(API 요금), 사용자 지정 게이트웨이 중에서 고릅니다. Zed는 외부 에이전트에 요금을 받지 않습니다.

### 12.2 비교표

| 차원 | Codex × ACP | Codex × Agents API |
|---|---|---|
| 누가 일을 시키나 | 보통 에디터 속 사람입니다. 헤드리스 클라이언트도 가능하지만 어댑터를 직접 띄워야 합니다. | 프로그램(앱 서버, webhook handler, cron, Slack·GitHub 이벤트)이 API 키로 HTTPS 호출합니다. 사람이 없어도 됩니다. |
| 에이전트 루프가 도는 곳 | 어댑터를 띄운 머신 | OpenAI 클라우드. self-hosted여도 루프는 OpenAI에 있습니다. |
| 파일·명령 위치 | 그 머신의 작업 폴더 | `none` / `openai_hosted`(`/workspace`) / `self_hosted`(내 인프라) 중 선택 |
| 호출자가 떠나면 | 멈춥니다(2초 뒤 종료). 기록만 남아 나중에 resume합니다. | 계속됩니다. 단, hosted 샌드박스는 1시간 무활동 시 삭제될 수 있고, self-hosted executor가 끊기면 도구가 실패할 수 있습니다. |
| 세션 보관 | 그 머신 디스크의 JSONL | OpenAI 서버에 삭제할 때까지 보관됩니다. 어느 기기에서든 `session_id`로 이어갑니다. |
| 동시 실행 | 로컬 자원, 작업 트리, 플랜 한도가 한계입니다. 문서화된 상한은 없습니다. | 세션을 여러 개 생성합니다(고객 사례는 "hundreds", "thousands"). 세션 안 subagent는 기본 6개입니다. |
| 사람 승인 | 프로토콜에 내장돼 있습니다(`session/request_permission` → 에디터 대화상자). | 내장돼 있지 않습니다. function tool + `action_required` webhook + 내 UI로 직접 만듭니다. |
| 도구 추가 | 에디터가 `session/new`로 MCP 서버를 넘기고, Codex 로컬 설정과 skills도 적용됩니다. | `agent.tools`로 넣습니다. service MCP는 vault로 비밀을 숨기고, function은 내 앱이 실행하며, web_search가 내장돼 있습니다. |
| 인터페이스 | JSON-RPC over stdio(양방향) | HTTPS REST + 이벤트 스트림 + webhook. self-hosted는 outbound WebSocket이 추가됩니다. |
| 화면(UI) | 에디터가 제공합니다(채팅, diff, 도구 카드, 승인 창). | 없습니다. 직접 만듭니다(웹, Slack, GitHub 댓글 등). 개발자용 조회 대시보드만 있습니다. |
| 인증·과금 | ChatGPT 플랜 사용량 또는 API 키 | API 키 종량제(토큰 + 도구 + 컨테이너) |
| 준비 부담 | 개인은 설치하고 로그인하면 끝입니다. 서버가 필요 없습니다. | hosted면 harness나 VM 운영은 없습니다. 대신 `session_id` 저장, 끊김 복구, 멱등 도구 결과를 챙기고, webhook을 쓰면 HTTPS 엔드포인트가 필요합니다. |
| 표준성 | Apache-2.0 공개 프로토콜(Zed · JetBrains 공동 운영, 재단 이관 추진). 레지스트리에 41개 에이전트가 있습니다. | OpenAI 독점 베타 API. harness 코드만 오픈소스입니다. |
| 데이터 위치 | 파일과 로그는 그 머신에 있고, 프롬프트와 읽은 파일 내용은 모델 공급자로 갑니다. | 세션 상태는 OpenAI에 보관되고, 미국 레지던시만 지원하며 ZDR은 불가합니다. |
| 성숙도 | ACP v1 안정(SDK 1.0은 2026-06, v2는 Draft). codex-acp가 기대는 app-server는 문서상 "experimental"입니다. | 2026-09-10 public beta |

---

## 13. 공식 예제와 사례

### 13.1 OpenAI cookbook 예제 앱 5종 (`openai-cookbook/examples/agents_api/apps`)

| 앱 | 흐름 요지 |
|---|---|
| `sev_bot` (장애 대응) | 알림 webhook → `#oncall` Slack 스레드 → 사고마다 세션 하나. self-hosted Docker 샌드박스에서 `codex exec-server`를 돌리고, GitHub MCP(읽기 전용 endpoint, 도구 5개 허용)와 AWS DevOps Agent MCP는 선택입니다. `propose_rollback` 승인을 Slack 버튼으로 받고, 승인은 기록만 하며 배포는 하지 않습니다. 대기 중인 승인과 세션 ID는 앱 메모리에만 있습니다. |
| `github_issues` (이슈 조사) | 서명된 GitHub `issues` webhook(opened/edited/reopened) → 앱이 호스트에서 저장소를 얕게 clone → Docker 샌드박스 `/workspace`에 마운트 → self-hosted 세션이 조사하고 가능하면 재현한 뒤 `investigation.md`를 작성 → **앱이 이슈 댓글로 게시**하고 세션과 샌드박스를 삭제합니다. PR은 열지 않습니다. 토큰은 컨테이너에 들어가지 않습니다. |
| `slack_bot` (팀 봇) | Slack 스레드마다 self-hosted 세션과 격리된 workspace를 둡니다. Notion, Google Drive, GitHub MCP 연결을 Slack 워크스페이스 단위 vault 하나로 공유합니다. |
| `data_analyst` | **샌드박스 없음**(`none`). 앱이 통제하는 function tool(최대 100행 읽기 전용 SQL)로만 답합니다. 코딩 외 용도 예시입니다. |
| `document_review` | coordinator가 문서마다 subagent에게 정책 skill을 적용시킵니다. 승인은 전부 사람이 합니다. |

- **발표 글의 대표 예제:** 5xx 에러 급증을 조사하는 운영 에이전트입니다. `openai_hosted`, 관측 MCP, `max_concurrent_subagents: 3`, `vault_ids`를 쓰고 결과를 `/workspace/outputs`에 씁니다.
- **주의:** showcase 페이지의 sev_bot 코드는 `OPENAI_API_KEY`를 `CODEX_API_KEY`로 넘깁니다. 가이드와 cookbook은 제한된 executor 키를 요구하니, 코드를 그대로 옮기지 마세요.

### 13.2 출시 고객 인용 (벤더 발표 수치)

- **Ciridae:** 평가 점수 0.71 → 0.85, subagent로 지연 4배 감소
- **SafetyKit:** 건당 비용 60% 절감
- **Hypha(금융):** "by separating the agent harness from the sandbox" 실패 응답 86% 감소
- **Nash.ai:** 장시간 물류 에이전트 수천 개. "OpenAI's Agents API gives us the durable session and orchestration layer we need ... while Nash provides the tools and execution environment"
- **Dwelly:** "We could fan out work across hundreds of agents, run them asynchronously, and collect the results later, without keeping infrastructure idle between peaks."
- **Long Lake:** "Agents API supplies the harness; the environment, context, and UX stay ours."
- **WithCoverage:** "now we can use agents directly in our code much like how Codex works on your laptop."
- **deepsense.ai:** 인용 고객 목록에 있습니다.

### 13.3 비판과 우려

- **HN 스레드**(news.ycombinator.com/item?id=49649213):
  - **샌드박스 운영 대행:** "To save yourself the hassle of running your own sandboxed VM."(simonw)
  - **락인:** Assistants API 은퇴 경험(z2), "If you want to have control over your data, you have to have control over your harness."(hobofan)
  - **구독 불가:** "you can't use your subscription with this"(krashidov)
  - **책상 앞 에이전트 vs 서비스형 에이전트:**
    - "I'd much rather the inverse of this: let me run the agent local but provide secure remote hosted sandboxes."(zmmmmm)
    - "The end goal is not you watching what the agent is doing, verifying, then accepting its changes ... Kind of slack-button-click-to-fix-something workflow."(tokioyoyo)
  - **로컬 한계:**
    - "my VPS can handle maybe 10 parallel sessions max"(druskacik)
    - "Codex doesn't survive a reboot by default or a laptop going to sleep."(windexh8er)
  - **과금 혼란:** 환경 요금 구조를 두고 혼란이 있었습니다(simonw의 질문).
- **InfoWorld 인용 애널리스트**(요약기 경유): "Lock-in is the biggest concern." ZDR이 없어 규제 산업에는 제약이 있다고 봅니다. 경쟁 제품으로 Claude Managed Agents, AWS Bedrock AgentCore, Microsoft Foundry Agent Service를 꼽았습니다.

---

## 14. 아직 확인되지 않은 것

- **hosted harness의 샌드박스·권한:** self-hosted와 hosted executor에 실제로 요청하는 샌드박스·권한 프로필(§6.6)
- **승인 대기:** `requires_action` 대기 중 hosted 샌드박스의 keep-alive 여부, 대기 시간 과금, 대기 중 샌드박스가 만료되면 pending call이 어떻게 되는지
- **컨테이너 과금:** hosted 샌드박스의 메모리 등급, per-minute "eligible" 해당 여부, idle과 삭제 시점의 과금 경계
- **AGENTS.md:** hosted harness가 `/workspace/AGENTS.md`를 실제로 따르는지(코드 경로는 있고 실측은 없음)
- **PR 흐름:** "버그 수정 → PR" 흐름의 공식 경로. 샌드박스 안 `git push`인지 서비스 쪽 GitHub MCP(`push_files`, `create_pull_request`)인지. 공식 예제 중 PR까지 여는 것을 실측한 사례는 없습니다.
- **모델과 한도:** 공식 지원 모델 목록, Agents API 전용 rate limit과 동시 세션 상한
- **ACP 연계:** ACP 클라이언트(에디터)가 Agents API 클라우드 세션에 붙는 공식 브리지. 현재 문서화된 것은 없습니다.
- **Responses API 호출 여부:** harness가 내부적으로 `/v1/responses`를 호출하는지. 권한 이름과 가격 설명으로 추정만 가능합니다.
- **압축 설정:** Agents API 컨텍스트 압축을 조정할 수 있는지. 설정 필드가 보이지 않을 뿐입니다.
- **`tool_handlers` 헬퍼:** openai-python 3.13.0의 `sessions.stream(tool_handlers=...)`. cookbook이 사용하지만 가이드에는 문서화돼 있지 않고, idle 세션에서만 동작합니다.
- **Bedrock과의 관계:** Amazon Bedrock에 올라온 OpenAI 호환 Responses 엔드포인트, Bedrock Managed Agents와 Agents API가 어떤 관계인지.

---

## 15. 출처

**원시 API · Agents SDK · Assistants API**
- Function calling: https://developers.openai.com/api/docs/guides/function-calling
- Async tool calling: https://developers.openai.com/api/docs/guides/async-tool-calling
- Conversation state: https://developers.openai.com/api/docs/guides/conversation-state
- Compaction: https://developers.openai.com/api/docs/guides/compaction
- Background mode: https://developers.openai.com/api/docs/guides/background
- WebSocket mode: https://developers.openai.com/api/docs/guides/websocket-mode
- Shell tool / Code Interpreter: https://developers.openai.com/api/docs/guides/tools-shell, https://developers.openai.com/api/docs/guides/tools-code-interpreter
- Remote MCP (Responses): https://developers.openai.com/api/docs/guides/tools-connectors-mcp
- Responses multi-agent: https://developers.openai.com/api/docs/guides/responses-multi-agent
- Assistants 마이그레이션: https://developers.openai.com/api/docs/assistants/migration
- Deprecations: https://developers.openai.com/api/docs/deprecations
- Agents SDK 문서: https://github.com/openai/openai-agents-python (docs/quickstart.md, running_agents.md, sessions/, sandbox/), https://developers.openai.com/api/docs/guides/agents/guardrails-approvals

**Agents API 공식 문서** (developers.openai.com, 경로 끝에 `.md`를 붙이면 마크다운)
- Overview: https://developers.openai.com/api/docs/guides/agents-api/overview
- Quickstart: https://developers.openai.com/api/docs/guides/agents-api/quickstart
- Architecture: https://developers.openai.com/api/docs/guides/agents-api/architecture
- Configuration: https://developers.openai.com/api/docs/guides/agents-api/configuration
- Sessions / Manage / Events / Webhooks: https://developers.openai.com/api/docs/guides/agents-api/sessions (하위 `/manage`, `/events`, `/webhooks`)
- Environments — OpenAI-hosted / Self-hosted / Lifecycle / Files / Security: https://developers.openai.com/api/docs/guides/agents-api/environments/openai-hosted (하위 동일 경로)
- Tools — Functions / MCP / Vaults / Web search / Plugins: https://developers.openai.com/api/docs/guides/agents-api/tools/functions (하위 동일 경로)
- Multi-agent: https://developers.openai.com/api/docs/guides/agents-api/multi-agent
- Observability / Tracing: https://developers.openai.com/api/docs/guides/agents-api/observability, https://developers.openai.com/api/docs/guides/agents-api/tracing
- Agents(런타임 비교): https://developers.openai.com/api/docs/guides/agents
- Skills: https://developers.openai.com/api/docs/guides/tools-skills
- Programmatic tool calling: https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling
- Pricing: https://developers.openai.com/api/docs/pricing
- Your data: https://developers.openai.com/api/docs/guides/your-data
- Changelog: https://developers.openai.com/api/docs/changelog
- API reference(sessions): https://developers.openai.com/api/reference/typescript/resources/beta/subresources/agents/subresources/sessions

**OpenAI 블로그**
- Introducing the Agents API (2026-09-10): https://openai.com/index/introducing-the-agents-api/
- Unlocking the Codex harness: https://openai.com/index/unlocking-the-codex-harness/
- Harness engineering: https://openai.com/index/harness-engineering/
- The next evolution of the Agents SDK: https://openai.com/index/the-next-evolution-of-the-agents-sdk/

**Codex**
- Codex app-server 문서: https://learn.chatgpt.com/docs/app-server.md
- 인증 / 요금 / 승인·보안: https://learn.chatgpt.com/docs/auth.md, https://learn.chatgpt.com/docs/pricing.md, https://learn.chatgpt.com/docs/agent-approvals-security
- openai/codex(exec-server 등): https://github.com/openai/codex

**예제와 SDK**
- Cookbook Agents API 예제: https://github.com/openai/openai-cookbook/tree/main/examples/agents_api
- openai-python beta agents 타입: https://github.com/openai/openai-python/tree/main/src/openai/types/beta
- Vercel 샘플: https://github.com/vercel-labs/openai-agents-api-vercel
- Cloudflare 샘플: https://github.com/cloudflare/sandbox-sdk/tree/main/openai/agents-api
- E2B 샘플: https://github.com/e2b-dev/e2b-cookbook/tree/main/examples/openai-agents-api-python-sdk

**ACP**
- 사양: https://agentclientprotocol.com (introduction, protocol/v1/*, rfds/*, community/governance)
- codex-acp: https://github.com/agentclientprotocol/codex-acp
- (구) Zed codex-acp: https://github.com/zed-industries/codex-acp
- Zed External Agents 문서: https://github.com/zed-industries/zed/blob/main/docs/src/ai/external-agents.md
- acpx: https://github.com/openclaw/acpx

**반응**
- Hacker News: https://news.ycombinator.com/item?id=49649213
- InfoWorld: https://www.infoworld.com/article/4221163/openai-launches-managed-agents-api-to-simplify-enterprise-ai-agent-development.html
- MarkTechPost(2026-09-10): https://www.marktechpost.com/2026/09/10/openai-launches-the-agents-api-in-public-beta-putting-the-codex-harness-behind-one-api-call/
