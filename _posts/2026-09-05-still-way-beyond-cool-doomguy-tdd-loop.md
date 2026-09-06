---
layout: post
title: "Still way beyond cool: resurrecting the Doomguy TDD loop"
date: 2026-09-05 09:48:57 -0300
lang: en
categories: [ruby, testing]
tags: [ruby, rspec, tdd, doom, autotest, cruise, gem]
---

It was my first few months with Ruby. I was running through everything I
could find, still figuring out what made this language different. One day I
landed on an old blog post titled *"Way beyond cool: autotest + growl +
Doomguy"*. The date read 2007. The screenshots showed a Doom Marine face —
smiling when the tests passed, bleeding more with every failure.

I followed the script in that post and set up autotest with the Doomguy
notifications, using it through my first Ruby studies. The feedback was
short, productive, even addictive. But that post made something click: a loop.
Type code, save, get feedback. Fix, get feedback again. For the rest of my
career, testing was never just a chore I ran — it was something I wanted to
interact with.

Years later I decided to resurrect that idea. This is how I turned a 2007 hack
into a modern Ruby gem, and why a bleeding Doom Marine is still the best
teaching tool I know.

## The original recipe

The post by **szeryf** (on his blog *require 'brain'*) took John Nunemaker's
[Autotest Growl Pass/Fail Notifications][railstips] idea and gave it a soul.
The setup was Mac-only:

- **autotest** (part of ZenTest) to watch your files and rerun tests,
- **Growl** with `growlnotify` to show desktop notifications,
- a **`~/.autotest`** script that monkey-patched `Autotest::Growl` and picked
  one of six Doom Guy head sprites depending on the results.

If everything passed, a happy Marine popped up. If failures and errors
accumulated, the blood started to flow — `fail10`, `fail20`, `fail30`,
`fail40`, up to `fail50` at which point, as the post put it, "it looks like we
killed him, we bastards!".

The feedback was instant — and it scaled. Two failures looked annoying.
Fifty failures looked like a horror show.

The post was a hit. It got featured on Doomworld under the headline *"The
Doomguy is a pedagogue"*. Carlos Brando built a Linux/Mac port as a gem. Dylan
Egan adapted the regexes to RSpec in the comments. Someone ported it to
Watchr. The Doom head sprites (from Doom's HUD, where a healthy Marine smiles
and a dying one is a bloody mess) became my favorite test runner mascot.

Then the icons' download links died, one RapidShare mirror at a time. autotest
faded into history alongside growlnotify. The memory stayed.

## Why it still matters

The point of that setup was never the notification. It was closing the loop.

TDD without feedback is a discipline on faith: you write a test, then you
*run* it minutes later, then you *remember* what it said. The Doomguy turned
that into a reflex. You save a file, and a face on your screen tells you —
immediately — whether you did a good thing.

That's the difference between *hoping your suite is green* and *watching it go
green*. And short feedback loops are productive: the less distance between "I
wrote this code" and "I know whether it works", the more experiments you try,
the faster you improve. It's the same cycle that made me want to write a test
first, then make it pass — to watch it go green.

## The homage

So I rebuilt it. The 2026 stack:

- **Cruise** — a fast, OS-native file watcher for Ruby (FSEvents on macOS,
  inotify on Linux, no polling, built-in debounce). The spiritual heir of the
  watch loop autotest used to provide, minus the platform pain.
- **Notiffany** — the same notification fan-out that powers Guard: it sends to
  libnotify, TerminalNotifier, Growl, tmux, whatever your desktop speaks.
- **RSpec** — today's lingua franca of Ruby testing (and, just like Dylan Egan
  in 2007, we target it directly).

I first drafted it as a Guard plugin (`guard-rspec-doom`) — never published,
just for my own joy. Then I realized the whole point was *not* to depend on
Guard. The watcher, the runner and the notifier fit together in ~200 lines; a
standalone gem made more sense. Meet **[rspec-doom]**, a file watcher that
runs your RSpec suite and shows you a Doom Guy based on the results.

### Using it

```bash
bundle exec rspec-doom
```

That's it. It watches `app/` and `spec/` by default. When a spec changes, it
runs that spec. When a source file changes, it runs the matching spec — or the
whole suite if there's no match. Useful flags:

```bash
bundle exec rspec-doom lib spec        # extra directories to watch
bundle exec rspec-doom --cmd 'rspec'   # custom RSpec command
bundle exec rspec-doom --debounce 0.5  # calm the watcher down
```

Ctrl-C stops it cleanly.

### The Marine, tiered

This is the part that matters. The more damage you take, the more the Marine
bleeds — just like 2007.

| Image | Failures | The message |
| --- | --- | --- |
| ![Doomguy, full health](/assets/img/rspec-doom/doom1.png) | 0 | All clear! |
| ![Doomguy, light damage](/assets/img/rspec-doom/doom2.png) | 1–2 | Minor casualties... |
| ![Doomguy, taking damage](/assets/img/rspec-doom/doom3.png) | 3–5 | Taking damage! |
| ![Doomguy, badly hurt](/assets/img/rspec-doom/doom4.png) | 6–10 | REQUIRED: MORE DAKKA! |
| ![Doomguy, barely alive](/assets/img/rspec-doom/doom5.png) | 11+ | IMPS EVERYWHERE! |

The original scaled with `fail10` → `fail50`; we scaled with the same instinct
— a happy face, and four degrees of suffering.

## The loop, alive again

Fire up `rspec-doom`, write a failing test, and watch the Marine take damage.
Then implement — not to satisfy a runner or a CI badge, but to make the face
on your screen smile again. The reward is immediate; the habit forms by
itself.

```bash
$ bundle exec rspec-doom
Watching app/ and spec/...
# edit spec/foo_spec.rb → write a failing expectation
# a bloody Marine appears: "Taking damage! (3 failures)"
# edit app/foo.rb → make it pass
# the Marine smiles: "12 tests - ALL CLEAR!"
```

Don't underestimate the playfulness. A Doom Marine judging your test suite
turns a grind into a game — and games are very good at keeping you in the
loop.

## RIP AND TEAR

Nineteen years after szeryf's post, the recipe still works: watch files, run
tests, show a face. The tools have changed (Cruise instead of autotest,
Notiffany instead of growlnotify, RSpec instead of Test::Unit). The feedback
loop didn't.

If you want to try it, the source is at **[rspec-doom]** — and the post that
started it all is still online: *[Way beyond cool: autotest + growl +
Doomguy][original]*. Go read it; it's short, and it may change how you test.
Thanks for the idea, szeryf — it was way beyond cool in 2007, and it still is.
The icons are back, this time in a gem that won't disappear when the file host
does.

[railstips]: https://railstips.org/2007/7/23/autotest-growl-pass-fail-notifications
[original]: https://szeryf.wordpress.com/2007/07/30/way-beyond-cool-autotest-growl-doomguy/
[rspec-doom]: https://github.com/amalrik/rspec-doom
