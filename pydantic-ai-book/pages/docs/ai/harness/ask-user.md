---
type: Web Page
title: Ask User | Pydantic Docs
description: Let the model ask the user multiple-choice questions mid-run and wait
  for the answers; you supply the answerer, so it works from a terminal, a web UI,
  or a test.
resource: https://pydantic.dev/docs/ai/harness/ask-user
timestamp: '2026-09-21T12:25:24.826293+00:00'
---

# Ask User

Let the model ask the user multiple-choice questions mid-run and wait for the answers.

While Pydantic AI Harness is on 0.x releases, the API may change between minor releases; when it does, deprecation warnings and release-note migration guidance tell you (or your agent) exactly how to upgrade. See the [version policy](/docs/ai/harness/#version-policy).

An agent given an ambiguous task either guesses or buries a question in its output and stops. Guessing wastes a run; a question in prose is only found by a person reading the whole answer. The model needs a way to ask a small, structured question and get the answer back as data.

`AskUser` exposes one tool, `ask_user_question`. The model passes one to ten questions, each
with a short `header`, the question text, two to six options (a `label` and an optional
`description`), and `multi_select` when several answers are allowed. The capability validates
the call, hands it to your `answerer`, and returns the picked labels keyed by header. If the
user declines, the model is told so and the run continues.

The capability owns the schema and validation. It never prints, reads stdin, or imports a
terminal library: the `answerer` is whatever puts the questions in front of a person, and you
supply it. There is no default, because a capability that reads stdin is unusable from a server.

```
from pydantic_ai import Agent
from pydantic_ai_harness import AskUser
from pydantic_ai_harness.ask_user import AskUserAnswer, AskUserRequest, AskUserResponse
async def pick_first(request: AskUserRequest) -> AskUserResponse:
    answers = [AskUserAnswer(header=q.header, selected=(q.options[0].label,)) for q in request.questions]
    return AskUserResponse(answers=tuple(answers))
agent = Agent('anthropic:claude-fable-5', capabilities=[AskUser(answerer=pick_first)])
```
`pick_first` stands in for a real UI. [CLAI](https://github.com/pydantic/pydantic-ai-harness/tree/main/pydantic-clai2/)
ships a terminal menu built on the same protocol; a web form would be another.

An `Answerer` is one async callable: it takes an `AskUserRequest` and returns an
`AskUserResponse`. A plain `async def` qualifies; a class with `async def __call__` does too.

- `AskUserRequest.questions` holds the validated`Question` objects;`AskUserRequest.id` distinguishes concurrent or repeated calls so a UI can match its reply to the request.
- A completed `AskUserResponse` carries one`AskUserAnswer` per question, keyed by`header` ,
with the picked option labels in`selected` : exactly one unless the question is`multi_select` , at least one either way, and no label twice.
- When the user declines, return `AskUserResponse(cancelled=True)` with no answers. The tool
result tells the model the user declined; nothing is raised into the run.
- A response that does not fit the request (an unknown header, a label the question did not
offer, several labels on a single-select question, a missing answer) is a bug in the answerer
and raises `ValueError` , which fails the run.`check_response` is exported so an answerer can
validate before returning.

The run waits inside the tool call for the answerer to return, so a web front end that collects the answer on another request should hold the run open until it arrives.

Two `CapabilityEvent`s let anything else in the run observe the exchange:

| Event | When | Fields | 
|---|---|---|
| `AskUserRequestedEvent` | before the answerer is called | `request` | 
| `AskUserAnsweredEvent` | after it returns, before the response is checked or the model sees the result | `request_id` ,`response` | 

Both dispatch immediately, so a listener that shows a “waiting for you” state sees the wait
start and end in step with the run. Subscribe with `@on_event` on a capability or through the
run’s event stream:

```
from pydantic_ai import RunContext
from pydantic_ai.capabilities import AbstractCapability, on_event
from pydantic_ai_harness.ask_user import AskUserAnsweredEvent, AskUserRequestedEvent
class WaitIndicator(AbstractCapability[None]):
    @on_event(AskUserRequestedEvent)
    async def waiting(self, ctx: RunContext[None], event: AskUserRequestedEvent) -> None:
        print(f'waiting on {len(event.request.questions)} question(s)')
    @on_event(AskUserAnsweredEvent)
    async def done(self, ctx: RunContext[None], event: AskUserAnsweredEvent) -> None:
        print('declined' if event.response.cancelled else 'answered')
```
The tool schema mirrors Code Puppy’s `ask_user_question`, so prompts written for it carry
over. Limits: 1 to 10 questions per call, unique headers of at most 25 characters, question text
of at most 500, 2 to 6 options per question with unique labels of at most 50 characters and
descriptions of at most 200; no control characters anywhere (headers and labels are one line;
question text and descriptions may span lines), since these strings are drawn on terminals and
an escape sequence in a prompt-injected call is an attack. A call outside those limits, or
carrying a field the schema does not have, is returned to the model as a validation retry,
not sent to the answerer. Once
validated the questions are frozen: what the answerer sees is what the model asked.

The result is a JSON object mapping each header to the list of picked labels, or the sentence
`The user declined to answer. Continue without the answer, or ask differently if it is essential.`

The capability adds one instruction: ask when the task is ambiguous and the answer is not in the workspace, offer concrete options, batch related questions, and make a stated choice if the user declines.

`AskUser` declares no default `id`. Two on one agent collide on the `ask_user_question` tool
name: two answerers is a conflict, not one configuration stated twice.

`AskUser` emits no spans. Core’s tool-call span already covers the wait, and the two events
above carry what was asked and answered.

`Agent.from_spec` cannot construct `AskUser`: the answerer is a live object a spec has no way
to name.

`check_response`, `TOOL_NAME`, `DECLINED`, and `MAX_QUESTIONS` are also exported from
`pydantic_ai_harness.ask_user`.

**Bases:** `AbstractCapability[AgentDepsT]`

Let the model ask the user multiple-choice questions mid-run.

The capability owns the question schema and validation; the `answerer` owns the person.
Whatever it is, a terminal menu, a web form, or a scripted function in a test, the run waits
for it inside the tool call and the model gets the picked labels back. There is no default:
a capability that reads stdin would be unusable from a server.

```
from pydantic_ai import Agent
from pydantic_ai_harness import AskUser
from pydantic_ai_harness.ask_user import AskUserAnswer, AskUserRequest, AskUserResponse
async def pick_first(request: AskUserRequest) -> AskUserResponse:
    answers = [AskUserAnswer(header=q.header, selected=(q.options[0].label,)) for q in request.questions]
    return AskUserResponse(answers=tuple(answers))
agent = Agent('anthropic:claude-fable-5', capabilities=[AskUser(answerer=pick_first)])
```
Presents each `AskUserRequest` to the user and returns their `AskUserResponse`.

**Type:** `Answerer`

```
def get_instructions() -> AgentInstructions[AgentDepsT] | None
```
Static, cache-stable guidance on when to ask.

`AgentInstructions`[`AgentDepsT`] | `None`

```
def get_toolset() -> AgentToolset[AgentDepsT] | None
```
The `ask_user_question` tool bound to this capability’s answerer.

[`AgentToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AgentToolset)[`AgentDepsT`] | `None`

**Bases:** `Protocol`

Whatever puts the questions in front of a person: a terminal menu, a web form, a scripted test.

A plain `async def` with this signature satisfies it. Return a cancelled response when the user
declines; raise only for failures that should fail the run.

`@async`

```
def __call__(request: AskUserRequest, /) -> AskUserResponse
```
Put `request` to the user and return what they picked.

`AskUserResponse`

One `ask_user_question` call, handed to the `Answerer` and carried by `AskUserRequestedEvent`.

Distinguishes concurrent or repeated calls; a UI matches its reply to it.

**Type:** `str`**Default:** `field(default_factory=(lambda: uuid4().hex))`

What the `Answerer` returns: one answer per question, or `cancelled` when the user declined.

The labels the user picked for one question.

**Bases:** `BaseModel`

One multiple-choice question.

**Bases:** `BaseModel`

One choice the user can pick.

**Bases:** `CapabilityEvent`

The model asked the user something; the answerer is about to be called.

**Bases:** `CapabilityEvent`

The answerer returned; `response.cancelled` says whether the user declined.

Emitted before the response is checked against the request, so a listener waiting since the request event is released even when the answerer misbehaved and the run is about to fail.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/ask-user
