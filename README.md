# Draftomatic

Draftomatic is a local Python command-line application I built to automate the path from a Jira documentation request to a reviewable documentation pull request. After a writer starts a run, it uses the OpenAI Codex SDK to gather engineering evidence, generate grounded update instructions, draft changes, run repository checks, and prepare the work for human review.

The original implementation lives in an internal GitHub repository that is not publicly accessible. This public write-up describes the workflow and architecture I developed independently. It contains no confidential information belonging to my employer.

## What it does

A documentation ticket often describes a requested change without giving a writer enough context to document it accurately. The supporting evidence may be spread across engineering tickets, pull requests, and source code. Draftomatic brings that evidence into a repeatable workflow:

1. **Start with a Jira ticket.** Read the request and identify the documentation updates it calls for.
2. **Gather engineering evidence.** Follow linked engineering pull requests and inspect relevant source code to establish what the implementation supports.
3. **Generate grounded instructions.** Turn the evidence into focused documentation updates, narrow unsupported claims, and identify questions that need human input.
4. **Produce a first draft.** Give Codex one focused update at a time, apply changes in the documentation repository, run checks, commit the work, and open a draft pull request.
5. **Pause for review.** Leave the result with the accountable writer and subject-matter experts for judgment, correction, and approval.

The workflow automates the preparation and execution around drafting. The writer initiates the run and remains responsible for the resulting documentation.

## How it works

~~~mermaid
flowchart TD
    Writer[Technical writer] --> CLI[Local Python CLI]
    Jira[Jira ticket] --> CLI
    Evidence[Linked engineering PRs and source code] --> CLI
    Context[Ontology and persona-weighted audience context] --> CLI
    CLI --> Codex[OpenAI Codex SDK]
    Codex --> Draft[Grounded instructions and documentation changes]
    Draft --> Checks[Repository checks and commits]
    Checks --> PR[Draft GitHub pull request]
    PR --> Review[Writer and subject-matter review]
~~~

The CLI coordinates the workflow through adapters for Jira, GitHub, local Git repositories, and Codex. Codex runs through the human user's local Codex account.

I created a documentation ontology that provides a semantic layer for organizing relevant concepts and their relationships. A persona-weighted audience overlay biases that ontology toward what a target reader is most likely to need and adds guidance for tone, level of detail, and delivery formats such as flowcharts or quick-reference tables. Together, these structures focus the model's context, reduce unnecessary token use, and improve the relevance of generated instructions and drafts.

For each update, the application supplies focused context: the ticket, supporting engineering evidence, audience guidance, and target repository. Repository checks provide feedback on the resulting changes. Human review addresses accuracy, clarity, and suitability for publication.

## What I built with AI

I built the orchestration that connects evidence gathering, audience-aware instruction generation, agent-driven repository changes, validation, and the review handoff. Codex acts as an implementation agent inside that workflow, while the Python application controls its inputs, preserves workflow state, and coordinates the surrounding tools.

Draftomatic demonstrates how an AI agent can carry a documentation task across several systems while keeping its output grounded in source evidence and subject to accountable human review.

[Read the detailed architecture](ARCHITECTURE.md)
