---
layout: post
title: "Threads, Ractors and Fibers in Ruby 4: what I learned by measuring before building a mail server"
date: 2026-10-14 13:48:20 -0300
lang: en
permalink: /posts/ruby-4-threads-ractors-fibers/
categories: [ruby, concurrency]
tags: [ruby, ractors, fibers, threads, async, benchmark]
---

I'm starting a personal project: **Mailractor**, a mail server written in Ruby. Before writing a
single line of SMTP, I wanted to answer the question that shapes the whole architecture: **which of
modern Ruby's concurrency models can handle this kind of load?**

Ruby 4 has three answers on the table:

- **Threads**: the classic model, limited by the GVL (only one thread runs Ruby code at a time).
- **Ractors**: real parallelism with memory isolation, behind an API that changed quite a bit in
  Ruby 4.0.
- **Fibers** with a scheduler (`Fiber::Scheduler`, here through the
  [`async`](https://github.com/socketry/async) gem): cooperative concurrency, one event loop per
  thread.

Instead of picking one by gut feeling, I built a benchmark POC. This post sums up what I measured,
what surprised me and the **Ruby bugs** I hit along the way (one of them became
[Bug #22439](https://bugs.ruby-lang.org/issues/22439)). All the code, raw results and the full
report are at
[github.com/amalrik/ruby-ractors-study](https://github.com/amalrik/ruby-ractors-study).

## How I measured

A few rules I followed so the numbers wouldn't fool me:

- **Same work and same validation in every mode.** Prime counting is checked against a sieve of
  Eratosthenes. Every network session is validated reply by reply.
- **Each case runs in a fresh Ruby process**, with a warm-up round and 3 to 5 repetitions. I report
  the median and min/max, never just the best run.
- **Startup, processing and shutdown measured separately**, along with CPU, memory (peak RSS) and
  p50/p95/p99 latency.
- **The environment is recorded on every run**: CPU, Ruby version and, in the cloud, *steal* (CPU
  time the hypervisor hands to other virtual machines).

The machines: a laptop with a Ryzen 7 5700U (8 cores/16 threads) and, later, a VM on Hetzner
(CPX42, 8 AMD EPYC vCPUs). I ran everything on **Ruby 4.0.6** and repeated it on **4.0.7** and
**4.1.0-preview1**. The version made all the difference.

## 1. CPU: Ractors deliver, threads don't

The first test is the most predictable one: counting primes among 2 million numbers, split into
independent blocks.

| Mode (Ruby 4.0.6, 512 blocks) | Time | Speedup |
|---|---|---|
| sequential | 3.17 s | 1.0× |
| 8 threads | 3.10 s | 1.02× |
| 8 Ractors | 0.59 s | **5.4×** |
| 16 Ractors | 0.59 s | 5.4× |

![Speedup vs. workers: Ractors reach 5.4× at 8 workers, threads stay at 1.0×](/assets/img/ruby-4-threads-ractors-fibers/cpu-speedup-en-light.svg){: .light }
![Speedup vs. workers: Ractors reach 5.4× at 8 workers, threads stay at 1.0×](/assets/img/ruby-4-threads-ractors-fibers/cpu-speedup-en-dark.svg){: .dark }

**Threads don't help at all** with pure Ruby: the GVL serializes execution. **Ractors scale up to
the number of physical cores.** Creating a Ractor costs ~45 µs, so startup cost is negligible.

Two interesting details:

- **16 Ractors are no better than 8.** On 4.0, the scheduler uses at most `RUBY_MAX_CPU` native
  threads (8 by default). Raising it to 16 made things *slower* (0.70 s): the SMT logical cores
  don't pay off for this kind of loop.
- **Granularity matters.** With ~0.4 ms tasks, message passing already costs 8% even with a single
  Ractor. With tasks of a few milliseconds, that cost disappears.

## 2. I/O: the "receive before I/O" trap

For I/O, I simulated operations that wait on the network: each operation opens a connection to a
local server that answers after 10 ms.

Threads did great: with 256 threads, ~8,200 operations/s. Ractors, on the other hand, **got stuck at
~730 operations/s, with either 8 or 256 workers**. The server never saw more than ~15 concurrent
connections.

I went looking for a mistake on my side. The Ractor pool had a central dispatcher, and I suspected
it, then `Enumerator#next`. It was neither. The clue came from counting the **process's native
threads**:

| Ruby 4.0.7, 64 Ractors, 400 operations | Time | Native threads |
|---|---|---|
| Ractor goes straight to I/O with a timeout | 0.12 s | ~67 |
| Ractor waits on `Ractor.receive` and **then** does I/O | **0.59 s** | **10** |

In Ruby 4.0's M:N scheduler, a Ractor that has already blocked waiting for a message (exactly what
every pool worker does) **doesn't get a dedicated native thread**. From then on, each
`wait_readable(timeout)` holds one of the ~8 shared threads. The result is an invisible cap of ~8
concurrent I/O operations. Nothing warns you: the program just gets slow.

The good news: **it's fixed in 4.1.0-preview1** (0.104 s, with ~18 native threads). The reproducer
is in
[`io_bound/bin/repro_ractor_receive_first.rb`](https://github.com/amalrik/ruby-ractors-study/blob/main/io_bound/bin/repro_ractor_receive_first.rb).

![I/O throughput vs. concurrent workers: Ractors on 4.0.6 stay flat at ~730 ops/s while threads, Fibers and Ractors on 4.1 climb past 7,000](/assets/img/ruby-4-threads-ractors-fibers/io-throughput-en-light.svg){: .light }
![I/O throughput vs. concurrent workers: Ractors on 4.0.6 stay flat at ~730 ops/s while threads, Fibers and Ractors on 4.1 climb past 7,000](/assets/img/ruby-4-threads-ractors-fibers/io-throughput-en-dark.svg){: .dark }

## 3. SMTP-like sessions: where things get serious

One request per connection doesn't represent a mail server. So I built **sessions with 7 round
trips** (banner, EHLO, MAIL, RCPT, DATA with 4 KB, QUIT), with a 10 ms wait before each reply and a
timeout on every read, as a real SMTP server needs.

Up to ~1,000 concurrent sessions, threads and Fibers tie. Beyond that, the bottleneck changes
nature. With 10,000 sessions on Ruby 4.0.6, Fibers delivered 745 sessions/s, and the session p50 was
12.9 s. That's **Little's law** at work:

> average latency ≈ concurrency ÷ throughput = 10,000 ÷ 745 ≈ 13.4 s

In other words, latency turned into queueing. A single Ruby process saturates one CPU core, and
sessions wait their turn. Measuring CPU alongside latency is what revealed it; without `cpu_util`,
I'd have blamed the network.

Other numbers that matter for a server with thousands of connections:

- **Threads:** with 10,000 sessions, 646 MB of memory and **~6 s just to create the threads**.
- **Fibers:** 357 MB and 0.1 s to create.
- On **4.1**, Fibers got much better under high concurrency: 10,000 sessions in 10.0 s, against
  40.4 s on 4.0.6.

## 4. The test that matters: the server

So far I had been comparing *client* models. But Mailractor's question is about the **server**. So
I made the server pluggable and kept the load generator fixed:

- **threads**: one thread per connection;
- **fibers**: one process, one Fiber per connection;
- **prefork + fibers**: 4 processes (fork) accepting on the same socket, each with its own event
  loop;
- **Ractors + fibers**: 1 process, 4 Ractors, each with **its own listening socket**
  (`SO_REUSEPORT`, the kernel spreads connections) and its own `async` reactor.

I ran it on a throwaway Hetzner VM (it cost cents), with 4 vCPUs for the server and 4 for the
generator. Steal stayed at 0% in every measurement. Results with **10,000 concurrent sessions**
(Ruby 4.1.0-preview1):

| Server | Sessions/s | p99 | CPU (case) | Memory |
|---|---|---|---|---|
| threads | 1,460 | 7.2 s | 124 s | 100 MB |
| fibers (1 process) | 3,749 | 2.7 s | 32 s | 92 MB |
| **prefork + fibers** | **12,430** | **0.44 s** | 37 s | 317 MB* |
| **Ractors + fibers** | 9,702 | 1.0 s | 45 s | **145 MB** |

\* sum of the 4 processes, which counts pages shared through copy-on-write more than once.

![Server throughput at 1,000, 5,000 and 10,000 sessions: prefork + fibers leads, Ractors + fibers second](/assets/img/ruby-4-threads-ractors-fibers/server-throughput-en-light.svg){: .light }
![Server throughput at 1,000, 5,000 and 10,000 sessions: prefork + fibers leads, Ractors + fibers second](/assets/img/ruby-4-threads-ractors-fibers/server-throughput-en-dark.svg){: .dark }

![Server memory at 10,000 sessions: Ractors + fibers 145 MB vs. prefork + fibers 317 MB](/assets/img/ruby-4-threads-ractors-fibers/server-memory-en-light.svg){: .light }
![Server memory at 10,000 sessions: Ractors + fibers 145 MB vs. prefork + fibers 317 MB](/assets/img/ruby-4-threads-ractors-fibers/server-memory-en-dark.svg){: .dark }

A second run confirmed the ranking. My reading:

- **Thread per connection doesn't work for inbound SMTP.** With 10,000 clients, the threads server
  never went past ~1,000 concurrent connections and burned 3–4× more CPU per session.
- **Fibers in a single process** are great up to ~1,000 sessions, then get stuck on one core.
- **Prefork + Fibers was the fastest**, the most stable, and it works on any Ruby version. It's the
  established model in the ecosystem (Falcon, Puma in cluster mode).
- **Ractors + Fibers reached ~80% of prefork with half the memory**, in a single process.

## 5. Ractors + Fibers: real parallelism inside one process

That last point is the one that intrigued me most. In the same process, with plain Fibers, the
limit is one core (~3,700 sessions/s). With 4 Ractors, each running its own event loop, it reached
~9,700/s. On the client side, the process used **4.6 cores at the same time**.

It works because:

1. **Each Ractor has its own execution lock.** The GVL is no longer global *across Ractors*, so
   threads in different Ractors run on different cores.
2. **The Fiber scheduler is per thread.** Each Ractor installs its own: N independent event loops,
   one per core, like a server with one loop per core, but without fork.
3. **No shared socket.** Each Ractor opens its own listener with `SO_REUSEPORT`, which respects the
   isolation.

So why does prefork still win? My hypothesis, not measured yet: Ractors **share the same heap and
the same garbage collector**. GC and some internal caches need synchronization between them.
Separate processes coordinate nothing. That matches the ~20% higher CPU cost per session. If GC and
those caches become more independent across Ractors in future versions, the gap should shrink.

## 6. The bugs I found along the way

A study like this ends up, unintentionally, being a stress test of the language itself.

**Ruby 4.0.6 and 4.0.7: hangs and memory corruption.** Session clients running in Ractors **hung in
6 out of 20 runs** on 4.0.6 and in 4 out of 20 on 4.0.7. Under `gdb`, the threads were sitting in
`poll` *with* a timeout, but the timeout never fired (with `IO#timeout = 2`, the process sat still
for 60 s). Network queues were empty on both ends. One run ended in `double free or corruption`
**inside the GC sweep**. On 4.1.0-preview1: **0 out of 40**.

**Ruby 4.1: a regression.** Running `async` in several Ractors, a process died with a segfault. I
kept shrinking it until I got ~15 lines, with no gems at all:

```ruby
rs = Array.new(2) do |i|
  Ractor.new(i) do |id|
    500.times do |k|
      begin
        raise ArgumentError, "failure #{id}-#{k}"
      rescue => e
        e.detailed_message(highlight: false)
      end
    end
    :ok
  end
end
p rs.map(&:value) # 4.0.x: [:ok, :ok] — 4.1: [BUG] Segmentation fault
```

On 4.1, `error_highlight` (that `^^^^` under the failing line) started using a new method,
`Thread::Backtrace::Location#syntax_tree`. Called from two Ractors at the same time, it brings the VM
down. With a single Ractor it doesn't crash, but the highlighted snippet silently disappears. I
reported it as [Bug #22439](https://bugs.ruby-lang.org/issues/22439), with reproducers and a
per-version table. It matters because **any library that logs exceptions from inside Ractors can
crash the process** on 4.1.

## 7. Method mistakes I made (and fixed)

I'm writing these down because benchmarks lie easily:

- **Contaminated measurements.** Once, another benchmark ran in parallel with mine and competed for
  CPU. I had to throw away and redo part of the matrix. Lesson: make sure the machine is idle before
  measuring.
- **I blamed my home network too early.** From Brazil to the VM in Finland, sessions had connection
  timeouts. At first I thought it was the load generator. Then I tried opening 400 connections at
  once: zero failures. The problem only showed up with *sustained* rates of new connections, and the
  server's kernel counters showed **zero drops**. SYNs were getting lost on the way (router or ISP).
  In other words, testing from home was measuring my network, not the server.
- **A pretty hypothesis that didn't hold.** I suspected the order in which connections were closed
  (TIME_WAIT on the server + NAT reusing ports) was causing the losses. I tested both orders: same
  symptom. Hypothesis discarded.

## Conclusions for Mailractor

- **Default model for receiving mail: processes (prefork) + Fibers.** More throughput, lower tail
  latency, works on any Ruby.
- **Ractors + Fibers is a real alternative on Ruby 4.1**, when memory and shared state in a single
  process weigh more than ~20% of throughput. It's also the path that produces the most useful
  feedback for the language.
- **Ractors for pure CPU work** (MIME parsing, signatures, filters) already work well, even on 4.0.
- **Ractors with sockets on 4.0.x: avoid.** Concurrency cap, hangs and memory corruption.
- **Thread per connection: no.**

What I still want to measure: generator and server on separate machines, TLS (STARTTLS changes the
CPU cost per session), larger messages and the full pipeline (reception → durable spool →
workers). That's for the next posts.

If you work with Ractors or `async`, the repository is open:
[github.com/amalrik/ruby-ractors-study](https://github.com/amalrik/ruby-ractors-study). Criticism of
the methodology is very welcome.
