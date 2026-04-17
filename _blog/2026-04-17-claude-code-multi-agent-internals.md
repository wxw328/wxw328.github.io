---
title: "Claude Code Multi-Agent Internals: How Subagents and Swarm Teammates Actually Work"
date: 2026-04-17
excerpt: "A source-level analysis of how Claude Code implements multi-agent behavior, tracing both subagent delegation and long-lived swarm teammate orchestration in the runtime."
tags:
  - Claude Code
  - Multi-Agent Systems
  - LLM Agents
  - Runtime Architecture
---

> A source-level analysis of how Claude Code implements multi-agent behavior, based on tracing the decompiled TypeScript runtime across `AgentTool`, `runAgent`, `query`, swarm teammate infrastructure, mailbox delivery, and task notifications.

---

## Table of Contents

1. [Why Claude Code Actually Has Two Multi-Agent Systems](#1-why-claude-code-actually-has-two-multi-agent-systems)
2. [System Overview](#2-system-overview)
3. [Model A: A Main Agent Launches Multiple Subagents](#3-model-a-a-main-agent-launches-multiple-subagents)
4. [When the Main Agent Decomposes Work and Passes It Down](#4-when-the-main-agent-decomposes-work-and-passes-it-down)
5. [Built-In Specialized Agents](#5-built-in-specialized-agents)
6. [Model B: Spawned Swarm Teammates](#6-model-b-spawned-swarm-teammates)
7. [How Spawned Agents Receive Messages While Working](#7-how-spawned-agents-receive-messages-while-working)
8. [Is Agent Communication Pre-Planned or Decided at Runtime?](#8-is-agent-communication-pre-planned-or-decided-at-runtime)
9. [Architectural Comparison](#9-architectural-comparison)
10. [Key Source Files](#10-key-source-files)
11. [Conclusion](#11-conclusion)
12. [Appendix A: End-to-End Sequence Diagrams](#12-appendix-a-end-to-end-sequence-diagrams)
13. [Appendix B: Function-Level Call Chains](#13-function-level-call-chains)

---

## 1. Why Claude Code Actually Has Two Multi-Agent Systems

One of the easiest ways to misunderstand Claude Code is to assume that “multi-agent” refers to a single implementation path.

It does not.

At the source level, Claude Code contains **two distinct orchestration models**:

### 1. Subagent delegation
- A main agent launches one or more task-focused subagents.
- Entry point: `AgentTool`
- Typical use case: research, code exploration, verification, isolated implementation work
- Lifecycle: usually short-lived, task-scoped

### 2. Swarm / teammate collaboration
- A leader or agent spawns long-lived teammates.
- Entry points: swarm spawning infrastructure and teammate runners
- Typical use case: persistent collaboration, task distribution, long-running teamwork
- Lifecycle: long-lived workers that can go idle and later resume new work

Both paths ultimately reuse the same underlying agent runtime, especially `runAgent() -> query()`, but the **control plane, communication model, and result-delivery semantics are different**.

That distinction is the core architectural fact behind Claude Code multi-agent behavior.

---

## 2. System Overview

At a high level, the relevant runtime layers look like this:

```text
                        Claude Code Main Conversation
                                   |
                                   v
                              query() loop
                                   |
                    assistant emits tool_use blocks
                                   |
              +--------------------+--------------------+
              |                                         |
              v                                         v
        AgentTool path                           Swarm / teammate path
              |                                         |
              v                                         v
         runAgent()                             runInProcessTeammate()
              |                                         |
              v                                         v
           query()                                runAgent() -> query()
              |                                         |
              v                                         v
    result or task-notification              mailbox, idle notification,
                                             SendMessage, team task list
```

The key insight is that Claude Code does **not** implement “multi-agent” as a separate external scheduler. Instead, it layers orchestration on top of the normal query/tool/runtime loop.

---

## 3. Model A: A Main Agent Launches Multiple Subagents

## 3.1 The unified entry point is `AgentTool`

The core entry point for subagent spawning is:

- `src/tools/AgentTool/AgentTool.tsx`

Its input schema includes fields such as:

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`

In gated modes it may also accept:

- `name`
- `team_name`
- `mode`
- `isolation`
- `cwd`

At runtime, `AgentTool.call(...)` branches into three major cases:

1. **Teammate / swarm spawning** when `team_name` and `name` are present
2. **Fork mode** when `subagent_type` is omitted and forking is enabled
3. **Regular specialized subagent mode** when `subagent_type` selects an agent definition

This means `AgentTool` is not merely “the subagent tool.” It is the top-level dispatcher for several agent-launching behaviors.

---

## 3.2 Why the main agent can launch multiple subagents in parallel

One of the most important implementation details is that Claude Code does **not** hardcode a separate parallel-agent scheduler. Instead, parallel subagent launching falls out of the general tool orchestration system.

Relevant files:

- `src/tools/AgentTool/AgentTool.tsx`
- `src/services/tools/toolOrchestration.ts`

The key mechanism is simple:

- `AgentTool.isConcurrencySafe()` returns `true`
- `runTools(...)` partitions tool calls into concurrent-safe and serial batches
- consecutive `Agent` tool calls can therefore run through `runToolsConcurrently(...)`

So if the assistant emits **multiple `Agent` tool uses in a single message**, Claude Code can execute them in parallel.

This is the direct source-level explanation for why the main agent can “call multiple subagents at once.”

---

## 3.3 A subagent is a real sidechain agent loop, not a helper function

Relevant file:

- `src/tools/AgentTool/runAgent.ts`

The most important architectural fact here is that `runAgent(...)` itself calls `query(...)`.

That means a subagent is **not**:

- a synchronous helper
- a templated one-shot LLM call
- a lightweight internal callback

It is instead a **full sidechain agent session** with its own:

- prompt messages
- tool pool
- system prompt
- permission mode
- transcript persistence
- model loop
- streaming behavior

In practice, `runAgent(...)` does the following:

1. Builds initial messages from the subagent prompt and optional inherited context
2. Resolves the selected agent definition
3. Computes system/user context and system prompt
4. Derives tool restrictions and model options
5. Creates a subagent `ToolUseContext`
6. Writes subagent metadata and transcript records
7. Streams messages from `query(...)`
8. Persists the sidechain transcript incrementally

That is why subagents in Claude Code behave like genuine agents rather than synthetic function wrappers.

---

## 3.4 Foreground vs background subagents

Claude Code supports both synchronous and asynchronous subagent execution.

### Foreground subagents
- Run inline with the current turn
- The parent waits for the result
- The result can be summarized back into the parent conversation immediately

### Background subagents
- Represented as task objects
- Continue out-of-band
- Return later via task notification machinery

Relevant files:

- `src/tasks/LocalAgentTask/LocalAgentTask.tsx`
- `src/tools/AgentTool/agentToolUtils.ts`
- `src/query.ts`

Background flow works like this:

1. `runAsyncAgentLifecycle(...)` drives the long-running subagent
2. Completion triggers `enqueueAgentNotification(...)`
3. A `<task-notification>...</task-notification>` payload is queued
4. A later `query()` turn drains the queued notification
5. The notification is converted into attachment/user-style context
6. The parent agent sees the completed result on a subsequent turn

So asynchronous subagents are **not** streamed back directly into the same parent turn. Their completion re-enters the conversation through the task-notification channel.

---

## 3.5 Fork mode is a special subagent path

Relevant file:

- `src/tools/AgentTool/forkSubagent.ts`

Forking is not the same as launching a fresh specialized subagent.

In fork mode:

- `subagent_type` is omitted
- the child inherits the parent conversation context
- the parent assistant message is transformed so tool uses become placeholder tool results
- a directive-style child message is appended

The synthetic fork definition is designed for cache sharing and inherited context. This is why fork prompts are expected to be **directive-style**, not full re-briefings.

This is one of the subtle but crucial distinctions in Claude Code’s multi-agent architecture:

- **Fresh subagent**: starts with zero context, needs a complete brief
- **Fork**: inherits parent context, needs a scoped directive

---

## 4. When the Main Agent Decomposes Work and Passes It Down

This is one of the most important implementation questions.

The short answer is:

**The runtime does not independently decide task decomposition. The model decides to decompose work when it emits one or more `Agent` tool calls. The runtime then executes those decisions.**

---

## 4.1 The decomposition decision lives in prompt guidance, not orchestration code

Relevant files:

- `src/constants/prompts.ts`
- `src/tools/AgentTool/prompt.ts`

Claude Code uses two prompt layers to teach the main agent when to delegate:

### Global/system guidance
`getAgentToolSection()` in `src/constants/prompts.ts` provides higher-level principles such as:

- subagents are useful for parallelizable independent work
- subagents are useful when tool output would pollute the main context window
- exploration should often go to an Explore agent
- certain modes require verification work after implementation

### Tool-level operational guidance
`src/tools/AgentTool/prompt.ts` tells the model:

- what `Agent` is for
- when not to use it
- how to write a good subagent prompt
- when to fork instead of using a fresh agent
- when to launch multiple agents concurrently
- that explicit “run in parallel” requests should produce multiple `Agent` calls in a single assistant message

This is a critical architectural conclusion:

**the runtime executes `Agent` tool calls, but the model is the layer that decides whether to decompose work at all.**

---

## 4.2 The exact moment the task is passed to a subagent

Once the main agent emits something like:

```json
{
  "description": "Explore auth flow",
  "prompt": "Inspect the login and refresh-token flow. Identify where session invalidation happens and whether logout clears all persisted state.",
  "subagent_type": "explore"
}
```

the handoff happens in the following runtime phase:

```text
assistant emits Agent tool_use
  -> tool execution phase begins
  -> AgentTool.call(...) parses the input
  -> runAgent(...) is invoked with promptMessages
  -> subagent enters its own query() loop
```

So the concrete task is transferred **during tool execution**, not during a separate hidden planning stage.

---

## 4.3 What actually carries the task specification

The task specification is encoded in the `Agent` tool input fields themselves:

- `description`
- `prompt`
- `subagent_type`

Those fields are the actual handoff contract.

This means the subagent is not expected to “infer the assignment” by rereading the parent conversation unless it is a fork. For a fresh subagent, the main agent must explicitly brief it.

That is why the tool prompt strongly emphasizes that fresh agents begin with zero context and require a complete task description.

---

## 4.4 Why prompt quality matters so much for fresh subagents

For fresh specialized agents, `runAgent(...)` constructs the child conversation around the provided prompt messages.

So if the parent passes only a terse command like:

```text
check auth bug
```

the child does not automatically know:

- why the issue matters
- what has already been tried
- which files are relevant
- what is in scope or out of scope
- what output format the parent needs

This is not merely prompt-engineering advice. It follows directly from how the subagent runtime is built.

In Claude Code, a fresh subagent is launched as a new agent loop with a new prompt prefix and no inherited problem history unless you explicitly put that history into the prompt.

---

## 5. Built-In Specialized Agents

The built-in specialized agents are assembled from:

- `src/tools/AgentTool/builtInAgents.ts`
- `src/tools/AgentTool/loadAgentsDir.ts`
- `src/tools/AgentTool/built-in/*.ts`

During this session, the following built-in agent definition files were confirmed directly:

- `src/tools/AgentTool/built-in/generalPurposeAgent.ts`
- `src/tools/AgentTool/built-in/exploreAgent.ts`
- `src/tools/AgentTool/built-in/planAgent.ts`
- `src/tools/AgentTool/built-in/verificationAgent.ts`
- `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts`
- `src/tools/AgentTool/built-in/statuslineSetup.ts`

## 5.1 Built-in agents confirmed in the source

### 1. `general-purpose`
- the default general agent
- fallback for broad multi-step work

### 2. `explore`
- specialized for codebase exploration and discovery
- recommended in prompt guidance for broad codebase search and investigation

### 3. `plan`
- specialized for planning and decomposition work

### 4. `verification`
- specialized for checking and validating work
- used as the “second pass” or implementation verification path in certain flows

### 5. `claude-code-guide`
- specialized for Claude Code product/workflow guidance

### 6. `statusline-setup`
- specialized for statusline-related setup or guided configuration work

---

## 5.2 How the agent list is exposed to the model

Relevant file:

- `src/tools/AgentTool/prompt.ts`

Claude Code supports two ways to expose available agent definitions to the model:

### Inline in tool description
The agent list is interpolated directly into the `Agent` tool prompt.

### Injected via message attachments
When `shouldInjectAgentListInMessages()` is enabled, the list is instead delivered through a system-reminder/attachment path.

This is a performance-sensitive design choice: if the tool description changes every time the available agents change, prompt-cache reuse degrades. So Claude Code can move the agent list out of the tool schema and into the conversation stream to stabilize cache behavior.

---

## 6. Model B: Spawned Swarm Teammates

The second major multi-agent system is the swarm/teammate architecture.

Relevant files:

- `src/tools/shared/spawnMultiAgent.ts`
- `src/utils/swarm/spawnInProcess.ts`
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`
- `src/utils/swarm/inProcessRunner.ts`
- `src/tools/SendMessageTool/SendMessageTool.ts`
- `src/utils/teammateMailbox.ts`
- `src/utils/attachments.ts`
- `src/utils/messages.ts`

This system is fundamentally different from subagent delegation.

Instead of saying “go do this task and come back,” it creates **ongoing teammates** that can continue receiving work over time.

---

## 6.1 Two teammate backends: in-process and pane-based

`spawnMultiAgent.ts` supports multiple backend strategies.

### In-process teammates
- run in the same Node.js process
- use `AsyncLocalStorage` for teammate isolation
- are launched via `spawnInProcessTeammate(...)` and `startInProcessTeammate(...)`

### Pane / process teammates
- run in another pane or another CLI process
- use tmux/iTerm2 or related backend infrastructure
- receive startup instructions through command-line flags and mailbox writes

The backend can be auto-selected, with in-process fallback if pane spawning is unavailable.

---

## 6.2 Swarm teammates are long-lived workers

This is the single most important conceptual difference.

Relevant file:

- `src/utils/swarm/inProcessRunner.ts`

The teammate runner is structured as a loop that does not terminate after one assignment:

```text
while (!aborted && !shouldExit) {
  process current prompt via runAgent()
  mark teammate idle
  wait for next prompt or shutdown
}
```

That means a teammate can:

- process a prompt
- go idle
- later receive another prompt
- continue with preserved local context/history

This is not “background subagent, but longer.” It is an entirely different lifecycle model.

---

## 6.3 Teammates still reuse the same agent core

Even though the orchestration model differs, teammates still execute actual work through `runAgent(...)`.

So the underlying execution core is shared:

- subagent path: `AgentTool -> runAgent -> query`
- teammate path: `runInProcessTeammate -> runAgent -> query`

Claude Code achieves multi-agent variety by reusing one agent runtime under multiple supervision layers.

---

## 6.4 Teammates get team-aware prompts and tool injection

Inside `runInProcessTeammate(...)`, Claude Code builds a teammate-specific system prompt from:

- the default system prompt
- `TEAMMATE_SYSTEM_PROMPT_ADDENDUM`
- an optional custom agent prompt

It also forcibly injects team-essential tools even if a custom teammate definition has an explicit allowlist. These tools include:

- `SendMessage`
- `TeamCreate`
- `TeamDelete`
- `TaskCreate`
- `TaskGet`
- `TaskList`
- `TaskUpdate`

This tells us something important about the design philosophy: **teammates are not only workers; they are expected to coordinate.**

---

## 6.5 Teammate output is not automatically surfaced back to the leader

Another subtle but important detail is that teammates do not automatically dump their full response back to the leader.

Relevant file:

- `src/utils/swarm/inProcessRunner.ts`

After a teammate finishes a prompt iteration, the system marks it idle and may send an idle notification, but it does **not** automatically return the full substantive response to the team lead.

Instead, meaningful collaboration is supposed to happen through:

- explicit `SendMessage`
- task list updates
- mailbox delivery
- idle summaries

This makes teammate collaboration closer to human team coordination than to RPC-style subtask return values.

---

## 7. How Spawned Agents Receive Messages While Working

This is one of the most subtle implementation details in the entire system.

The answer is: **not all messages are treated the same way**.

Claude Code effectively uses two communication channels:

1. ordinary work messages
2. control/protocol messages

---

## 7.1 Ordinary messages are usually injected on the next turn

Relevant files:

- `src/tools/SendMessageTool/SendMessageTool.ts`
- `src/utils/teammateMailbox.ts`
- `src/utils/attachments.ts`
- `src/utils/messages.ts`
- `src/query.ts`

Normal inter-agent communication works like this:

```text
Agent A calls SendMessage
  -> writeToMailbox(...)
  -> recipient inbox JSON is updated
  -> later getTeammateMailboxAttachments(...) reads unread messages
  -> builds a teammate_mailbox attachment
  -> normalizeAttachmentForAPI() converts it to a meta user message
  -> the recipient sees it in a subsequent query() turn
```

This is extremely important:

**ordinary teammate communication is not usually inserted into the middle of an already-running streaming query turn.**

Instead, it is surfaced as context for the next turn.

---

## 7.2 Control messages can affect the runtime more directly

Claude Code also defines structured protocol messages such as:

- `shutdown_request`
- `shutdown_response`
- `plan_approval_response`
- mailbox-mediated permission signals

These messages use the same broad infrastructure, but their semantics are more immediate.

Relevant file:

- `src/utils/swarm/inProcessRunner.ts`

Two especially important paths are:

### Permission handling during execution
`createInProcessCanUseTool(...)` can wait for permission-related responses while the teammate is still running.

### Waiting for work or shutdown
`waitForNextPromptOrShutdown(...)` actively polls for:

- `pendingUserMessages`
- unread mailbox messages
- shutdown requests
- messages from the team lead
- messages from peers

So while ordinary business messages are generally consumed on a subsequent turn, certain control-plane signals can influence the agent runtime more directly.

---

## 7.3 What “receiving messages while working” really means in practice

The most precise interpretation is this:

- a teammate is a long-lived worker
- a given `runAgent()` iteration handles one current prompt
- after the iteration ends, the teammate immediately enters a waiting state
- in that waiting state it polls for new work, shutdown, approvals, and mailbox traffic
- then it launches another `runAgent()` iteration with the new input

So the system is best described as:

- **turn-by-turn business message injection**
- **runtime polling for protocol/control signals**
- **persistent worker lifecycle across many turns**

This is more nuanced than saying the agent literally merges arbitrary new business messages into the middle of a single running stream.

---

## 8. Is Agent Communication Pre-Planned or Decided at Runtime?

The cleanest answer is:

**communication capabilities are predefined, but communication behavior is decided at runtime.**

---

## 8.1 What is predefined

Claude Code defines the communication infrastructure in advance:

- mailbox storage layout
- `SendMessageTool`
- structured message protocols
- team context reminders
- idle notification framework
- task list tooling
- attachment-to-message conversion rules

This is the static communication substrate.

---

## 8.2 What is decided dynamically

What is *not* fixed in advance is:

- which agent contacts which other agent
- when the message is sent
- what the message says
- whether a message is direct or broadcast
- whether the leader reassigns work
- whether a teammate creates or claims a task

Those choices are made by agents during runtime, based on the current state of the conversation, task list, mailbox contents, and prompt instructions.

This means Claude Code does not hardcode a static task DAG or fixed communication graph for swarm collaboration.

---

## 8.3 The design philosophy behind that choice

This architecture gives Claude Code a flexible collaboration model:

- the protocol is stable
- the behavior is adaptive

In other words, Claude Code does not pre-plan the entire collaboration graph. It provides a constrained, persistent team operating system and lets the agents decide how to use it during execution.

---

## 9. Architectural Comparison

| Dimension | Subagents | Swarm teammates |
|---|---|---|
| Primary entry point | `AgentTool` | `spawnMultiAgent` / teammate runner |
| Lifetime | task-scoped | long-lived |
| Context model | fresh subagent usually starts with zero context; fork inherits | persistent team identity and mailbox model |
| Result delivery | direct result or later `task-notification` | explicit message passing, idle notifications, task updates |
| Main communication style | parent-child delegation | peer/team coordination |
| Parallelism | via tool concurrency | via multiple active teammates |
| Runtime core | `runAgent() -> query()` | `runAgent() -> query()` |
| Best use case | isolated research, verification, bounded work units | persistent collaboration, evolving division of labor |

---

## 10. Key Source Files

### Subagent delegation
- `src/tools/AgentTool/AgentTool.tsx`
- `src/tools/AgentTool/runAgent.ts`
- `src/tools/AgentTool/forkSubagent.ts`
- `src/tools/AgentTool/agentToolUtils.ts`
- `src/tools/AgentTool/resumeAgent.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/query.ts`

### Built-in agent definitions
- `src/tools/AgentTool/builtInAgents.ts`
- `src/tools/AgentTool/loadAgentsDir.ts`
- `src/tools/AgentTool/built-in/generalPurposeAgent.ts`
- `src/tools/AgentTool/built-in/exploreAgent.ts`
- `src/tools/AgentTool/built-in/planAgent.ts`
- `src/tools/AgentTool/built-in/verificationAgent.ts`
- `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts`
- `src/tools/AgentTool/built-in/statuslineSetup.ts`
- `src/tools/AgentTool/prompt.ts`
- `src/constants/prompts.ts`

### Swarm / teammate system
- `src/tools/shared/spawnMultiAgent.ts`
- `src/utils/swarm/spawnInProcess.ts`
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`
- `src/utils/swarm/inProcessRunner.ts`
- `src/tools/SendMessageTool/SendMessageTool.ts`
- `src/utils/teammateMailbox.ts`
- `src/utils/attachments.ts`
- `src/utils/messages.ts`

---

## 11. Conclusion

Claude Code’s multi-agent implementation is best understood as **one shared agent runtime with two orchestration layers**.

### Layer 1: Delegated subagents
- the main agent decides to delegate by emitting `Agent` tool calls
- the task handoff is encoded in `description`, `prompt`, and `subagent_type`
- multiple subagents can run in parallel because `AgentTool` is concurrency-safe
- asynchronous subagents return through task notifications

### Layer 2: Persistent swarm teammates
- teammates are long-lived workers rather than one-shot helpers
- they are spawned through swarm infrastructure and keep running across tasks
- communication happens through mailbox delivery, task lists, and `SendMessage`
- the protocol is predefined, but actual collaboration patterns are runtime decisions

The result is a system that supports both:

- **tight, bounded delegation** for isolated subtasks
- **persistent teamwork** for open-ended collaborative workflows

That dual architecture is the real story behind Claude Code multi-agent behavior.

---

## 12. Appendix A: End-to-End Sequence Diagrams

## A.1 Main agent launches multiple subagents in parallel

```
User
  -> Main QueryEngine/query()
  -> Model responds with one assistant message containing multiple Agent tool_use blocks
  -> toolOrchestration.runTools()
  -> partitionToolCalls(...)
  -> AgentTool.isConcurrencySafe() == true
  -> runToolsConcurrently(...)

  -> AgentTool.call(subagent #1)
       -> resolve agent definition
       -> runAgent(...)
       -> query(...)
       -> tool execution / model turns
       -> result

  -> AgentTool.call(subagent #2)
       -> resolve agent definition
       -> runAgent(...)
       -> query(...)
       -> tool execution / model turns
       -> result

  -> parent agent receives tool results
  -> next parent turn synthesizes and responds to user
```

## A.2 Background subagent notification path

```text
Main agent
  -> AgentTool.call(run_in_background=true)
  -> runAsyncAgentLifecycle(...)
  -> runAgent(...)
  -> child query() finishes later
  -> enqueueAgentNotification(...)
  -> queue gets <task-notification>
  -> later main query() drains queued commands
  -> getAttachmentMessages(...)
  -> task notification becomes attachment/user-style message
  -> main agent sees child completion in a later turn
```

## A.3 Swarm teammate spawn and first assignment

```text
Leader agent
  -> spawnTeammate(...)

  if in-process:
    -> spawnInProcessTeammate(...)
    -> register task + identity + abort controller
    -> startInProcessTeammate(...)
    -> runInProcessTeammate(...)
    -> wrap initial prompt as teammate-message
    -> runAgent(...)
    -> query(...)

  if pane/process backend:
    -> create pane / process
    -> launch new Claude Code CLI with teammate flags
    -> write initial message to teammate mailbox
```

## A.4 Teammate communication loop

```text
Teammate A
  -> SendMessageTool
  -> writeToMailbox("teammate-b", ...)

Teammate B
  -> finishes current runAgent() iteration
  -> waitForNextPromptOrShutdown(...)
  -> mailbox unread message discovered
  -> message wrapped as teammate-message or mailbox attachment
  -> next runAgent(...)
  -> query(...)
```

## A.5 Idle notification back to leader

```text
Teammate
  -> completes current prompt iteration
  -> mark task idle
  -> sendIdleNotification(...)
  -> leader mailbox receives structured idle message
  -> later attachment pipeline surfaces mailbox update
  -> leader sees teammate status / summary
```

---

## 13. Appendix B: Function-Level Call Chains

## B.1 Main-agent-to-subagent call chain

```text
QueryEngine.submitMessage(...)
  -> query(...)
  -> model emits assistant message with Agent tool_use
  -> services/tools/toolOrchestration.runTools(...)
  -> tools/AgentTool/AgentTool.call(...)
  -> tools/AgentTool/runAgent(...)
  -> query(...)
  -> subagent tool/model loop
  -> tool result returned to parent
```

## B.2 Parallel subagent execution chain

```text
runTools(...)
  -> partitionToolCalls(...)
  -> tool.isConcurrencySafe(...)
  -> runToolsConcurrently(...)
  -> AgentTool.call(...) x N
```

## B.3 Background subagent lifecycle chain

```text
AgentTool.call(...)
  -> decide shouldRunAsync
  -> runAsyncAgentLifecycle(...)
  -> runAgent(...)
  -> child messages streamed and persisted
  -> enqueueAgentNotification(...)
  -> pending notification queue
  -> query() drains queued commands
  -> getAttachmentMessages(...)
  -> parent sees completion on later turn
```

## B.4 Fork subagent chain

```text
AgentTool.call(...) with no subagent_type
  -> forkSubagent path
  -> buildForkedMessages(...)
  -> runAgent(...)
  -> query(...) with inherited parent context
```

## B.5 Swarm in-process spawn chain

```text
spawnMultiAgent.spawnTeammate(...)
  -> handleSpawnInProcess(...)
  -> spawnInProcessTeammate(...)
  -> startInProcessTeammate(...)
  -> runInProcessTeammate(...)
  -> runAgent(...)
  -> query(...)
```

## B.6 Pane/process teammate spawn chain

```text
spawnMultiAgent.spawnTeammate(...)
  -> handleSpawnSplitPane(...) / handleSpawnSeparateWindow(...)
  -> create teammate pane
  -> construct CLI args
  -> send command to pane
  -> write initial prompt to mailbox
  -> spawned CLI process boots its own agent runtime
```

## B.7 Teammate message delivery chain

```text
SendMessageTool.call(...)
  -> writeToMailbox(...)
  -> teammateMailbox JSON inbox updated
  -> attachments.getTeammateMailboxAttachments(...)
  -> messages.normalizeAttachmentForAPI(...)
  -> meta user message injected into next model turn
```

## B.8 Teammate wait-loop chain

```text
runInProcessTeammate(...)
  -> runAgent(...)
  -> current prompt finishes
  -> sendIdleNotification(...)
  -> waitForNextPromptOrShutdown(...)
      -> inspect pendingUserMessages
      -> inspect unread mailbox
      -> inspect shutdown/protocol messages
  -> on new work: construct next prompt
  -> runAgent(...) again
```

## B.9 Communication-planning chain

```text
Static layer:
  SendMessageTool / mailbox / attachment normalization / team_context

Dynamic layer:
  runtime model decides whether to message,
  whom to message,
  when to message,
  and whether to create or claim tasks
```

---

## Final Takeaway

If you only look at `AgentTool`, Claude Code appears to have a straightforward “main agent spawns subagents” implementation.

If you follow the source more deeply, the real picture is richer:

- subagents are real sidechain agents
- forked workers are optimized for inherited context and cache sharing
- background agents re-enter the parent through task notifications
- swarm teammates are persistent collaborative workers
- mailbox delivery is the backbone of long-lived inter-agent coordination
- communication primitives are predefined, but collaboration itself is runtime-driven

That combination is what makes Claude Code’s multi-agent system both practical and unusually flexible.
