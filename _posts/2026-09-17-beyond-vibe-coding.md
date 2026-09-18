---
layout: post
title: "Beyond Vibe Coding: Spec-Driven AI Frameworks for Tech Debt"
author: Serge Aleynikov
date: 2026-09-17
tags: [ai, bmad, speckit, openspec, opengap, erlang, elixir, programming-languages]
---
![Beyond Vibe Coding]({{ '/assets/beyond-vibe-coding.jpg' | relative_url }})

The first time I asked an AI agent to build a feature from scratch I felt like a wizard. Typed a messy paragraph, hit enter, watched two hundred lines of code emitted out of thin air. High-fived the monitor. Felt ridiculous for two days afterward, like my decades-long code-writing experience expired worthless.  But still I was tempted to go back and try more.

Six weeks later, same agent, live multi-tier production app. I wanted to throw the laptop through the window.

It hallucinated a field that had never existed in my data structures, used coding style inconsistent with my codebase, quietly deleted a section of code that was critical to the correctness of the component, and left the test suite looking like it had been run over by a truck. That is the real edge of "vibe coding." The model is still uncanny at 0-to-1 work in an empty folder. Give it a codebase with actual weight and the prompts start rotting context, inventing breaking changes, and scattering silent bugs. "Pretty please don’t break master" is not a CI strategy. I checked.

If these things are going to act like engineers we have to stop treating them like oracles. Blueprints. Contracts. Checks that actually run.

Spec-Driven Development is the unglamorous answer. You write structured specs the model can be held to. It then works inside real constraints instead of improvising. Difference between a sticky note that says "make login better" and a ticket that has acceptance criteria and a definition of done.

## SDD and TDD Are Not the Same Thing

People keep collapsing these two. They shouldn’t.

TDD lives at the unit level. Failing test, minimal code, refactor. It answers one tight question: does this small piece do what I just claimed? Still one of the best design tools we have.

SDD sits higher. You write the actual specification-requirements, constraints, non-goals, acceptance criteria-before anyone (or any model) starts implementing. Different question entirely: did we build the thing we said we were going to build?

| Aspect              | Traditional TDD                          | Spec-Driven Development (SDD)                     |
|---------------------|------------------------------------------|---------------------------------------------------|
| Primary artifact    | Failing unit test                        | Structured specification                          |
| Scope               | One small unit or behavior               | Whole feature or system slice                     |
| Main question       | Does this unit do what I claimed?        | Did we build what we agreed?                      |
| Feedback loop       | Seconds (red-green-refactor)             | Per feature, with review gates                    |
| Best at             | Unit correctness and design feedback     | Aligning intent, especially with AI               |

They fit together cleanly. SDD points at the destination. TDD keeps the individual steps from falling apart. Skip the first and you can end up with beautifully tested code that solves the wrong problem. Skip the second and the elegant spec still ships bugs.

I spent some time putting five of these frameworks through real, messy codebases: OpenSpec, OpenGap, BMAD, GitHub SpecKit, Graphify. Also poked at a couple of lighter options. What follows is less a neat taxonomy and more what actually held up when things got ugly.

## Landscape Snapshot

| Framework       | Core Focus & Paradigm                        | Primary Strengths                              | Ideal Target Use Case                          | Relationship to TDD                              |
|-----------------|----------------------------------------------|------------------------------------------------|------------------------------------------------|--------------------------------------------------|
| **OpenSpec**    | Change-centric diffs & brownfield isolation  | Token-efficient, fast, lightweight             | Legacy codebases, refactoring, rapid iterations| Specs define the change; TDD verifies the units  |
| **OpenGap**     | Compliance, linting & behavioral gap analysis| Prevents scope creep, strict contract checks   | Auditing AI code, security-first, regulated apps| Contracts often feed or become acceptance tests |
| **BMAD**        | Multi-agent team simulations                 | Parallel streams, role-driven delegation       | Complex enterprise platforms, multi-tier systems| QA agent naturally drives TDD-style validation  |
| **GitHub SpecKit** | Gated 4-phase SDD & team standards        | Persistent "constitution" rules, review gates  | Medium-to-large teams, greenfield development  | Explicitly supports tests inside Tasks/Implement|
| **Graphify**    | Visual & graph-based architectural specs     | Deep topology mapping, zero-token local index  | Microservices, polyglot codebases, system mapping| Shows which surfaces actually need new tests   |

## OpenSpec

Brownfield work is where most of these tools fall over. OpenSpec doesn’t. It refuses to document the entire system and instead focuses on the delta. You drop small change specs into `/openspec/changes/` and leave the main specs alone. The model only gets the context it needs for that specific job. Token bills stay reasonable.

You still write the unit tests yourself, the normal way. OpenSpec’s job is mostly to keep the AI from wandering off and "improving" the authentication layer while it’s supposed to be adding a billing endpoint.

I used it to add a subscription tier to an Express app that had grown teeth. Defined only the new `/checkout` contract and the User model changes. Then ordinary TDD for the billing logic. The model stayed in its lane. That alone felt like progress.

## GitHub SpecKit

This one is heavier. Four-stage pipeline-Specify, Plan, Tasks, Implement-and you are not allowed to jump ahead. The Constitution file is where you write the non-negotiables. Short-lived JWTs. Rate limiting on every endpoint. That kind of thing.

It is the cleanest combination of SDD and TDD I saw. High-level spec sets the destination; inside the Tasks phase you can drop straight into red-green-refactor for each piece. Some teams even put "all new code must be TDD-driven" in the Constitution and mean it.

I ran a multi-tenant OAuth service through it. The Constitution killed three half-baked ideas before any code was written. TDD handled the individual pieces. Fewer embarrassing pull requests. The overhead is real, though. Solo work or tiny changes feel like wearing a suit to the grocery store.

## BMAD

When the feature is simply too big for one context window, BMAD is the multi-agent approach that didn’t completely fall apart on me.

It spins up specialized agents-PM, Architect, Dev, QA-and keeps their contexts separate. The PM breaks things down, the Architect maps data flow, the Dev writes code, the QA agent validates. It is a full standup without the video-call performance.

The QA agent is where TDD lives most naturally. While the others work from the high-level spec, QA can generate failing tests first and force the Dev agent to make them pass. I threw a messy monolith-to-microservices split at it. Architect produced the gRPC contracts, Dev implemented the handlers, QA drove the cross-service tests in proper TDD style. Still needed a human watching the handoffs, but it was the only framework that didn’t just collapse under the size of the problem.

## OpenGap

This one is less about generating code and more about refusing to let the model quietly break things three folders away.

It treats your requirements as a contract and keeps checking the generated code against it. Invisible drift-those moments when the AI "fixes" a bug by deleting your edge-case validation-gets flagged. In regulated work this is almost non-negotiable. Spec says every transaction must hit the audit log. TDD makes sure the logging functions themselves work. OpenGap blocks any PR that optimizes the audit trail out of existence. Same pattern for PII scrubbing.

It sits downstream of both SDD and TDD and acts as the last behavioral gate. Not glamorous. Extremely useful.

## Graphify

Most retrieval approaches still treat code as a bag of text. Graphify builds an actual directional graph with local Tree-sitter parsers. Zero LLM tokens for the parsing step.

Its practical value is making TDD less random. Change a core utility used by fourteen services and the graph shows the real blast radius so you know which tests actually need attention. You still write the tests. You just stop testing the wrong surfaces.

To illustrate its work I used it on an Elixir app that implements a distributed order processing engine which saved orders to Postgres. The Graphify's local AST parse mapped the full runtime layout of the application. Here is a sample output:

```
┌──────────────────────────────────────────────────────────────────┐
│                   ORDER PROCESSING SYSTEM GRAPH                  │
└──────────────────────────────────────────────────────────────────┘

                 [OrderSystem.Application]
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
  [OrderSystem.OrderRegistry]  [OrderSystem.OrderSupervisor]
                                          │ (spawns dynamically)
                                          ▼
                             [OrderSystem.Pipeline.Processor]
                                    │               │
                            (uses)  │               │ (dispatches)
                                    ▼               ▼
                       [Core.Order Schema]  [PaymentBehaviour]
                                                    ▲
                                                    │ (implements)
                                                    │
                                        [Pipeline.StripeAdapter]
```

## Two Lighter Options

Aider plus a strict conventions file works surprisingly well for solo work. The conventions act as a lightweight spec; you can still drive individual changes with normal TDD.

There are also harnesses that treat the test suite itself as the specification. The agent is not finished until every generated test passes. Pure TDD raised to system level. Feels extreme until you try it on something that cannot afford to be wrong.

## Cost and Overhead Reality Check

| Framework       | Token Efficiency / LLM Cost                          | Setup Overhead                                      | Maintenance Effort                                      | TDD Integration Cost                  |
|-----------------|------------------------------------------------------|-----------------------------------------------------|---------------------------------------------------------|---------------------------------------|
| **OpenSpec**    | Low - only active diffs are fed to the model         | Minimal (under 5 minutes)                           | Low - specs evolve with normal commits                  | Low (tests stay local)                |
| **GitHub SpecKit** | Moderate - refinement loops add some overhead     | Medium - needs team agreement on the Constitution   | Medium - reviews required at each gate                  | Medium (tests often written in Tasks) |
| **BMAD**        | Higher - multi-agent runs mean parallel model calls  | High - roles and handoff rules need defining        | High - someone still has to supervise the virtual team  | Medium-High (QA agent orchestration)  |
| **OpenGap**     | Moderate - automated verification on PRs             | Medium - needs solid upfront contracts              | Low/Automated - fits into normal CI                     | Low (contracts can feed tests)        |
| **Graphify**    | Lowest LLM cost - local AST work uses zero tokens    | Low - one CLI command to index                      | Low - incremental re-indexing via hooks                 | Low (improves TDD targeting)          |

## What I’m Actually Using

Vibe coding was a fun phase. Production code is not. The combinations that keep working for me:

- Messy active codebases: Graphify for the map, OpenSpec for the change, ordinary TDD on the surfaces that matter.
- Team or greenfield work: SpecKit. Let the Constitution and the gates handle the SDD layer; nest TDD inside implementation.
- High-stakes or genuinely large systems: OpenGap for the contract enforcement, BMAD when one agent is clearly not enough. Keep TDD as the unit-level truth serum.

None of these tools are magic. They just force a discipline most of us already knew and kept skipping: decide what "done" actually means before the model starts inventing it. Everything after that is just making sure the pieces hold when someone else has to live with them.
