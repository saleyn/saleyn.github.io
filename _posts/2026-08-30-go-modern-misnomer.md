---
layout: post
title: "Go's Modern Label Is a Misnomer"
date: 2026-08-30
tags: [go, erlang, rust, elixir, programming-languages]
---
Every few months some blog post or job ad still calls Go a "modern programming language." And every time I see it I just sit there thinking: which codebases are these people looking at?

The "modern" part was always mostly about the tooling. Back in 2012, `gofmt` shipping with the language itself felt almost unfair. One style, no arguments, done. That was genuinely nice. But it's 2026. Everyone else caught up a long time ago. C++ has clang-format and clang-tidy, Rust has cargo, Elixir has mix format. A decent build tool and formatter isn't special anymore - it's table stakes. Strip that away and what's left is a language whose design choices actively fight against a lot of what the rest of the field learned in the last thirty years.

Concurrency is the usual example people reach for. Go went all-in on CSP (Communicating Sequential Processes) - Tony Hoare's model from 1978. Channels are fine for a lot of I/O-bound web stuff. They're not free, though. Throw them at high-frequency, low-work tasks (incrementing counters, chewing through small arrays) and the synchronization overhead starts eating your throughput. You have to keep managing that balance yourself. And the scheduler still doesn't guarantee fairness. A heavy CPU-bound goroutine can just sit on a thread and starve everything else. Erlang (and Elixir after it) had process isolation and reduction-count scheduling sorted out in the 1980s. Go's story starts looking pretty old once you put them side by side.

Language evolution is even more telling. C++ spent the last decade going from C++14 to C++26 and added a ton of complexity on purpose - RAII, smart pointers, monadic types, ranges, reflection, structured concurrency. It's messy, but it's aimed at people who want the power. Go went the other way. No real metaprogramming. No pattern matching. No algebraic data types. Generics finally showed up in 1.18 in 2022, more than a decade late, and in a deliberately limited form. Interface-bound type sets, GC shape stenciling, no method-level type parameters, no real dynamic introspection. It feels like they were still trying to keep the language simple for the lowest common denominator long after that stopped being the interesting problem.

Error handling is the part that still drives me crazy. `if err != nil` twenty times in a file is not simplicity. It's just noise. When Ian Lance Taylor floated a lightweight `?` operator in 2025 the team killed it. The consistent stance seems to be that if a feature makes the code shorter it must be bad. That attitude has aged especially poorly now that we're all working with LLMs. Token count is actual context-window real estate. If your coding agent burns 80% of its budget just reading the same error-checking boilerplate over and over, it has less room left to think about the rest of the system. Google's usual defense - that the repetitive syntax makes LLM output more predictable - feels like a cop-out. You're sacrificing capacity so the model doesn't invent complicated abstractions. Great.

Even basic package layout suffers. The hard ban on circular imports sounds clean in a talk. In practice it just creates friction. Get the domain boundaries slightly wrong and you're suddenly inventing dummy `common` packages or sprinkling single-method interfaces everywhere to keep the compiler happy.

So where does that leave Go?

Python is still the fastest way to get a prototype out the door, even if the dynamic typing eventually bites you. Elixir is better on fault tolerance, observability, and concurrency - BEAM's per-process GC and the built-in tracing tools give you real system visibility without bolting on OpenTelemetry and hoping the global stop-the-world collector doesn't spike latency. Rust makes you fight the borrow checker up front, but it eliminates whole classes of bugs and keeps the code explicit.

Go sits in the middle. Easier than Rust, less dynamic than Python, less complex than modern C++. That's a reasonable tradeoff if you're a big company trying to onboard thousands of developers quickly and keep the codebases looking the same. It's not a bad tool for a lot of microservices work.

But calling it "modern" is just wrong. It's a deliberately conservative late-2000s language built on 1970s primitives plus a solid build tool. Treating the refusal to evolve as some kind of technical leadership is the part that keeps confusing me.
