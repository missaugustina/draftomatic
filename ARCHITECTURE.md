# Draftomatic Architecture

Draftomatic is a local Python command-line application.

The design is intentionally human-in-the-loop. Draftomatic does not run as a shared bot, silently publish generated content, or replace subject-matter review. It helps a technical writer move from a grounded Jira ticket to a first documentation draft while preserving the writer's identity, judgment, and ability to stop the workflow.

## What Draftomatic does

The workflow covers five stages:

1. Prepare or create a DOCS Jira ticket with an ordered `Recommended updates` section.
2. Find related engineering tickets, linked pull requests, and local source repositories.
3. Check every recommended update against current source code and narrow or remove unsupported claims.
4. Use one focused Codex prompt for each update, run repository checks, and commit the result.
5. Create a draft pull request, pause for human review, then resume explicitly to apply the first feedback pass.

The same Codex thread carries the ticket context through preparation, implementation, the review pause, and the first feedback pass. Each ticket has its own saved workflow and documentation branch. The local application can save several tickets, but only one ticket owns the ordinary documentation checkout at a time.

## System view

~~~mermaid
flowchart LR
    Human[Technical writer] --> CLI[Local CLI]
    CLI --> Actions[Workflow actions]
    Actions --> Jira[Jira adapter]
    Actions --> Source[Source and evidence adapters]
    Actions --> Codex[OpenAI Codex SDK adapter]
    Actions --> Git[Local Git adapter]
    Actions --> GitHub[GitHub adapter]
    Actions --> Store[(SQLite state and events)]
~~~

The application layer coordinates the workflow through small interfaces. Jira, GitHub, Codex, Git, local repositories, skills, identity checks, and SQLite are replaceable adapters behind those interfaces. Tests use fakes in place of external systems, so normal development does not modify real tickets, repositories, pull requests, or Codex threads.

## Workflow

~~~mermaid
sequenceDiagram
    actor Writer
    participant CLI
    participant Actions
    participant Jira
    participant Codex
    participant Git
    participant GitHub

    Writer->>CLI: start DOCS-1234
    CLI->>Actions: start workflow
    Actions->>Git: verify checkout and create branch
    Actions->>Codex: prepare ticket
    Actions->>Jira: save recommended updates
    Actions->>Codex: gather and verify engineering evidence
    Actions->>Jira: save evidence-backed instructions

    loop One recommended update at a time
        Actions->>Codex: implement one focused update
        Actions->>Git: run checks, commit, and push
    end

    Actions->>GitHub: create draft pull request
    Actions->>GitHub: add needs-human-review label
    Actions-->>Writer: stop for review
~~~

On resume, Draftomatic restores the saved branch and Codex thread, removes the review label, applies one review comment or tightly related comment group at a time, runs checks, and pushes the first feedback pass. If the saved thread cannot be resumed, the workflow stops instead of silently starting a new session with weaker context.

## Internal architecture

Draftomatic combines Hexagonal Architecture and Clean Architecture. The core workflow depends on domain values and narrow protocols. It does not depend directly on Jira, GitHub, Codex, a shell, or a database.

~~~text
CLI
  -> Workflow actions
      -> immutable domain models
      -> narrow protocols
          -> Jira adapter
          -> GitHub adapter
          -> OpenAI Codex SDK adapter
          -> Git adapter
          -> local source and skills adapters
          -> identity guard
          -> SQLite workflow store
~~~

The layers have distinct responsibilities:

| Layer | Responsibility |
| --- | --- |
| CLI | Parse commands, display results, and map expected errors to exit codes. |
| Workflow actions | Coordinate one business operation and return a new workflow value. |
| Domain models | Hold validated, immutable values passed between parts of the application. |
| Protocols | Define the smallest external operations the workflow needs. |
| Adapters | Translate protocols to Jira, GitHub, Codex, Git, local files, skills, and SQLite. |
| Tests | Supply safe fakes and verify workflow decisions without live writes. |

This structure adds more small interfaces than a script that calls every service directly. The tradeoff is deliberate: Draftomatic crosses several systems, performs writes as a human user, and must stop safely when it cannot verify identity, state, evidence, or session continuity.

## Codex and context design

The Codex adapter uses the supported openai-codex Python SDK to connect to the writer's local Codex runtime. Draftomatic does not define a separate shared bot identity or require a separate CODEX_API_KEY. Codex runs through the writer's local account, and the writer remains responsible for reviewing the resulting changes.

Codex receives focused context rather than an undifferentiated dump of project data:

- the Jira ticket and its ordered recommended updates;
- evidence from linked engineering work and local source files;
- the target documentation repository and branch;
- the current workflow stage and saved state; and
- the single update or review comment being handled in the current prompt.

This keeps the agent focused and makes each change easier to inspect. A documentation ontology and persona-based audience overlay provide an additional semantic layer for prompt construction, allowing generated instructions to focus the model on relevant topics, audience needs, tone, and delivery format.

## State and persistence

Draftomatic uses a local SQLite database for workflow snapshots, append-only events, archive state, optimistic version checks, and a workspace lease. It stores references and operational state, not secrets or full credentials.

Each workflow records its Jira key, component, stage, execution status, Codex thread ID, documentation path, branch, pull request, current update, timestamps, archive time, and version. The workspace lease prevents two saved workflows from silently using the same checkout. Draftomatic requires a clean working tree and never stashes, discards, resets, or force-pushes changes automatically.

The commands draftomatic start DOCS-1234, resume, list, and archive are designed as explicit lifecycle commands. A saved workflow can be paused and resumed without losing its branch, history, or Codex context.

## Identity and safety boundaries

Draftomatic is bound to one physical human. Before a state-changing operation, it verifies the active Jira, GitHub, Git, operating-system, and Codex identity references that the runtime exposes. An account mismatch stops the workflow before the write.

The application also:

- treats Jira text, attachments, pull requests, review comments, and repository content as untrusted data;
- keeps tokens and credentials out of prompts, logs, SQLite events, and Git;
- preserves unrelated Jira description content and rejects stale writes;
- uses argument lists for Git commands rather than shell strings;
- avoids duplicate pull requests and labels after retries;
- records successful writes and state transitions; and
- never bypasses the explicit human-review gate.

## Testing strategy

The design supports testing at several levels:

1. Pure tests verify routing, prompt construction, naming, validation, and state transitions.
2. Action tests use fakes to verify protocol calls, identity checks, retries, and event recording.
3. Adapter contract tests verify mappings and error handling without live writes.
4. SQLite tests verify migrations, transactions, version checks, archive behavior, and workspace leases.
5. An offline acceptance test exercises all five workflow stages with fake Jira, GitHub, Codex, Git, skills, and repositories.

Using fake external systems, the acceptance test verifies workflow coordination across ticket preparation, evidence gathering, incremental implementation, draft pull request creation, the review pause, and resumption with the saved branch and Codex thread.

## Public project context

The original Draftomatic implementation lives in an internal GitHub repository that is not publicly accessible. This public repository documents a workflow I developed independently. The material shared here contains no confidential information belonging to my employer and describes a general, publicly shareable approach to human-in-the-loop documentation automation.



## Further reading

- [Draftomatic repository](https://github.com/missaugustina/draftomatic)
- [OpenAI Codex SDK documentation](https://developers.openai.com/codex/sdk/)
- [Hexagonal Architecture by Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture)
- [The Clean Architecture by Robert C Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
