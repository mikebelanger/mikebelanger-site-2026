+++
title = "Rails and static typing"
date = "2026-09-12"
tags = ["crystal", "lucky", "rails", "ruby", "web"]
categories = ["general"]
authors = ["mike"]
draft = true
description = "Can the major drawbacks of Rails be mitigated with static typing?"
+++

[Rails](https://rubyonrails.org) got popular for a reason. Between a [CLI](https://guides.rubyonrails.org/command_line.html) that could generate consistent, conventional code, the [console (really, a REPL)](https://www.codynorman.com/ruby/rails_console_deep_dive/) that helped debug runtime errors, getting stuff done was easy. Most of its functionality is hidden away in parent objects and preprocessors. This allowed application developers to focus on business logic, not boilerplate. My first job as a developer was writing Rails apps. I went from knowing nothing about Rails (and [Ruby](https://ruby-lang.org/), really) to bugfixes within weeks, and full features within months.

After a few years of Rails work, I got burned out. Despite all my work and effort, runtime errors kept creeping up. If I'm being honest, they were my mistakes. Because I was a junior developer, I assumed I'd make less of these mistakes as I got more experienced. Even then, I'd watch my senior-level co-workers make the same kind of mistakes.

Looking back, it wasn't Rails itself which led to these mistakes, but its language: [Ruby](https://ruby-lang.org/). Ruby's dynamically-typed nature made it next to impossible to perform any kind of static analysis. The kind of static analysis that could predict nil-reference errors, or improperly spelled property/method names. There were performance issues too. Dynamically-typed languages almost always pay a performance penalty for being interpreted at runtime, and Ruby paid a particularly hefty performance price, [even relative to its other dynamically-typed peers.](https://programming-language-benchmarks.vercel.app/ruby-vs-javascript)

Of course, these issues can be mitigated. [There's a number of guards against nil-references](https://thoughtbot.com/blog/if-you-gaze-into-nil-nil-gazes-also-into-you). Performance issues are a broader topic, and usually require more analysis before mitigation. Analysis could reveal n+1 queries, a slow method, some kind of O(n)**2 process. While you'd be hard-pressed to find any developer that admits to doing this, I've seen many devs just throw a more powerful server at many performance issues. So while the range of workarounds are diverse, they're almost always easier, and [cheaper to mitigate than rewriting in a more performant language.](https://www.speedshop.co/blog/is-ruby-too-slow-for-web-scale/#rewrite-your-entire-application-to-save-1000month)

With all that in mind, why consider a rewrite, and a statically-typed language at that?

Two big reasons: refactoring, and merging.

In my experience, the line between a simple optimization and a refactor is blurry. Sure, in principle most people understand the difference, but it's hard to tell them apart.
