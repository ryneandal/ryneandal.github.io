---
layout: post
title: "Coding Agents Are Opening Up New Opportunities in Legacy Codebases"
date: 2026-09-18
author: Ryne Andal
categories:
  - ai
  - software-engineering
tags:
  - ai-agents
  - software-modernization
  - technical-debt
  - migrations
  - github-copilot
excerpt: |
  Coding agents can change which engineering projects make economic sense.
  GitHub's Copilot runtime migration is a useful starting point for looking
  at the migrations, upgrades, and modernization work we keep deferring.
description: |
  How coding agents can make deferred migrations and modernization projects
  worth doing, with existing behavior and tests helping verify the result.
image: "https://images.unsplash.com/photo-1506399558188-acca6f8cbf41?auto=format&fit=crop&w=1800&h=720&q=85"
image_alt: "Rows of black server racks in a data center"
---

This morning I read through GitHub's recent blog post about migrating the [GitHub Copilot runtime to Rust](https://github.blog/ai-and-ml/generative-ai/migrating-the-github-copilot-runtime-to-rust-using-copilot/) and it brought a few things to mind:

1. Real engineering work shows what coding agents can actually do.
2. Existing behavior and tests make migrations strong candidates for coding agents.
3. Coding agents can make updates worth doing by reducing development cost.


## Real engineering work shows what coding agents can actually do.

GitHub Copilot's app, CLI, and SDK are all wired into the Copilot agent runtime, the harness they embed into various applications and products. It was written in TypeScript on Node.js. The blog post goes on to say that they had to ship quickly to keep up with the rapidly evolving AI software tooling industry, so they built on that initial TypeScript layer, resulting in a bundled Node.js runtime being shipped with any SDK, regardless of language. So no matter what language was being used, the Copilot agent layer was beholden to all of the drawbacks of Node.js and that layer of abstraction. I do like this line from the post, explicitly stating they aren't just drinking the "Rust Kool-Aid" and choosing it because it is currently en vogue:

> "Our requirements emphasized embedding through a C ABI, low startup and steady-state overhead, and predictable resource use. Rust made those goals possible, at the expense of other complications, e.g. we had to represent lifetimes and shared state explicitly (the lifecycle regressions discussed later highlight the implications of that). The right target language legitimately varies from application to application."

They provide high-level architectural diagrams showing exactly why a full rewrite was justifiable: rapid iteration ended up with an SDK that was entirely coupled with Copilot CLI:

![Architecture Diagram of the GitHub Copilot Runtime Migration]({{ site.baseurl }}/assets/img/github-copilot-sdk-migration-diagram.png)
*GitHub Copilot runtime architecture before/after migration to Rust. Source: [GitHub Blog](https://github.blog/ai-and-ml/generative-ai/migrating-the-github-copilot-runtime-to-rust-using-copilot/) &copy; GitHub.*


They opted to replace one piece at a time, atomically, with both Rust and TypeScript components providing full interoperability during the transition. This is an excellent approach if you have the engineering bandwidth because it allows you to incrementally migrate the codebase without disrupting the existing functionality _and_ benefiting from the robust testing and stress testing that the production environment provides. All while benefiting from the drastic performance improvements and memory safety that Rust provides, until one day there are no more TypeScript components left to replace. When all was said and done, GitHub ended up with agents writing the majority of the resulting 800k lines of code over 128 PRs instead of a single bulk rewrite, allowing for much more manageable reviews.

IMO, of the rewrite approaches they describe in the post, 1a (stop the world) is probably the most feasible for codebases without an engineering team on the scale of GitHub's. This would be my preferred approach: freeze the current codebase on `main` and all work is shifted to the rewrite of `main`. This gives explicit targets for each slice handed off to an agent. You can say "look at this feature and what it does, what it outputs and recreate it in this language, following these new patterns and paradigms but ensure the function is retained." This provides a narrowly scoped task for an agent and reduces the chance of an agent getting lost on its way to the defined goal and iteratively causing bloat. Approaching a migration in this way also makes the project management much simpler, since you can simply take an inventory of current features and have your tasks right there.

## Existing behavior and tests make migrations strong candidates for coding agents.

Using large real-world projects gives far more insight into a model's (and its harness's) strengths and weaknesses. You'll learn quickly where the weaknesses are in the output and can revise prompts or context to resolve them going forward. Production migrations expose integration and operational constraints and edge cases that benchmarks can miss and the evaluation loop becomes part of the SDLC.

Thoughtworks' [Legacy Modernization meets GenAI](https://martinfowler.com/articles/legacy-modernization-gen-ai.html) describes another workflow, using AI to understand existing code and recover the requirements buried in it. A company’s codebase can contain years of valuable domain knowledge. Recovering and documenting potentially missing knowledge can help guide future business decisions. That understanding is part of what makes a migration possible in the first place.

Existing interfaces, tests, observability metrics, and production behavior give you a useful baseline for verifying the model's output. Once you have your processes dialed in so generated code is passing these additional verifiers, you build confidence in the behavior those tests cover. Instead of simply "does static analysis/testing/integration pass?" you can check metrics and logs and compare these results from dev branch to main to understand whether the behavior you depend on is preserved. This makes practical testing much more manageable and reliable. You can update environments with the updated code and check whether the application continues to work as expected.

GitHub's [Scientist](https://github.blog/developer-skills/application-development/scientist/) is one example of this approach for read-only code paths without side effects. If you can run the existing and refactored code against the same inputs, then you can compare results and timings, and proactively mitigate potential issues before migrating over.

##  Coding agents can make updates worth doing by reducing development cost.

Software engineers are often unhappy with the state of production codebases. We're often perfectionists who want to go through and resolve identified technical debt, like a mechanic hearing an aged, well-worn belt squeaking. We know it is structurally sound now and probably will be for some time, but we still want our completed work to be in perfect working order.

This eye for detail is part of why software engineers are good at decomposing complex requirements into their constituent parts, then turning those parts into code paths that adhere to the appropriate business rules. It is also why I and many colleagues have proposed refactors or full rewrites after years of accumulated technical debt, or as soon as a deprecation notice gives us 18 months of warning. These projects often get rejected because the effort makes them fiscally infeasible, even when everybody agrees that the migration itself would be valuable. There is always feature work with a more immediate return competing for the same budget.

Coding agents don't remove the difficult parts of a migration. The planning, specification, design, and architecture still need to be correct, and the resulting implementation still needs to be QA'd and validated. They can reduce the implementation cost enough to get approval on projects that historically have been non-starters. For example, if an upgrade or refactor could be completed at 25% of the original cost, it could become a new contract, a new project that would not have been approved in the past.


Amazon also [reported](https://press.aboutamazon.com/2024/12/new-amazon-q-developer-capabilities-accelerate-large-scale-transformations-of-legacy-workloads) migrating tens of thousands of production applications to Java 17 using Amazon Q Developer, estimating more than 4,500 developer-years saved and $260 million in annual performance-related savings. Take Amazon's own estimates with a healthy dose of salt, but they provide another example of the potential financial incentive of AI-assisted modernization.

## Coding agents enable more than just faster feature development.

Most discussions about coding agents focus on how much faster developers can ship the work they would be doing regardless. I have started to look for engineering work that hasn't been done at all because it costs too much.

There is an enormous backlog of this work across all industries:

- dependency modernization
- language migrations
- framework upgrades
- test expansion
- type adoption
- API migrations
- mechanical refactoring
- compatibility cleanup
- security hardening

Engineering organizations should still ask where agents can make their developers faster. They should also ask a second question:

> **What valuable engineering work have we denied because the return did not justify the cost?**

If you’ve been putting off a migration because of the cost, but coding agents have you reconsidering, I’d be happy to help you assess or tackle it.

Contact details: {% include contact-links.html %}.
