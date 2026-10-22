# Qwen3.6-35B-A3B and the 2+2 split (2026-10-21 and 10-22)

Tests of running two models side by side on the four cards: Qwen3.6-35B-A3B (a mixture-of-experts model, 3B
active parameters) on cards 0 and 1, and Qwen3.8-27B `UD-Q6_K_XL` (production's model) on cards 2 and 3, both at
the full 262,144-token context. Measurement only. Production stayed on four cards running Qwen3.8 and was restarted
after each test.

**Summary**

- Qwen3.6 fits on two cards at full context as `Q4_K_XL` and is the faster model: 90.9 tokens/s on short chats and
  61.0 after a 64,000-token prompt, against 37.7 and 35.7 for Qwen3.8 on the other two cards.
- The two models barely slow each other: with both busy at once, Qwen3.8 lost 1% on short requests and 5% deep in a
  64,000-token prompt; Qwen3.6 lost under 1%.
- Qwen3.8 on two cards against four: short requests 37.7 against 44.7 tokens/s (-16%), generation after a
  64,000-token prompt 35.7 against 40.2 (-11%), the prompt itself 343 against 512 tokens/s (-33%). A 250,000-token
  prompt still fits, with about 490 MiB free per card.
- With Qwen3.8 as the main agent, sending its subagents to Qwen3.6 finished agent tasks 14-23% sooner than keeping
  them on Qwen3.8, and passed 12 of 12 tasks (11 of 12 on Qwen3.8).
- Qwen3.8 on two cards passed 24 of 27 tasks of the standard agent battery (26 of 27 on four cards). With one battery
  each, that difference is not established.
- Tuning found nothing better than the starting settings on either model.

## Setup

4x Tesla P100 16 GB at 150 W, `GGML_CUDA_P2P=1`, fans at full during runs, cool-down to 48 °C before each run, 1 Hz
telemetry and a watchdog (stop over 80 °C, on a hang, on a low clock, a slow fan or a new log error).

| | Qwen3.6-35B-A3B | Qwen3.8-27B |
|---|---|---|
| File | `Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL` | `Qwen3.8-27B-UD-Q6_K_XL` |
| Cards | 0, 1 | 2, 3 |
| Build | shinbunbun 29-patches v0.4.0 | Kmic-68 NCCL build (production's) |
| Split | layer | tensor, `-ts 1/1` |
| Context cache | f16 | q4_0 |
| MTP | n-max 3, p-min 0.75 | n-max 3, p-min 0.0 |
| Batch | `-ub 512` | `-b 2048 -ub 2048` |

Speed client: 12 x 384-token short requests, one 64,000-token prompt, 3 x 256 tokens generated on top of it.

## Choosing the Qwen3.6 file (2026-10-21)

Q8_0 on four cards, then Q4_K_XL and Q5_K_XL on two (tokens/s):

| | Q4_K_XL, 2 cards | Q5_K_XL, 2 cards | Q8_0, 4 cards |
|---|---:|---:|---:|
| Short chats, MTP | **90.9** | 83.0 | 85.4 |
| After a 64,000-token prompt, MTP | **61.0** | 39.6 | 55.1 |
| 64,000-token prompt, no MTP (Kmic build) | 846 | 795 | **1208** |
| After a 250,000-token prompt, no MTP | **41.6** | 22.0 | 38.9 |

- Q5_K_XL only fits at 262,144 context with a q8_0 context cache and `-ub 256`, leaving about 145 MiB free on card 1.
  It is much slower deep in context. The q8_0 cache is probably most of that loss (it was also the slowest cache
  type for Qwen3.8), but no run separates the two.
- On this model MTP halves prompt speed (64,000 tokens: 421 tokens/s with it, 952 without, 29-patches on four
  cards) and adds nothing to generation after 250,000 tokens. It helps short chats, so it stays on.
- Tensor split is slower than layer split for this model (about 50 against 70 tokens/s generation on four cards).

## Both models at once (2026-10-21)

| | Qwen3.8 alone | Qwen3.8, Qwen3.6 busy | Qwen3.6 alone | Qwen3.6, Qwen3.8 busy |
|---|---:|---:|---:|---:|
| Short, tokens/s | 37.67 | 37.26 | 90.85 | 90.37 |
| After a 64,000-token prompt | 35.72 | 33.81 | 60.91 | 60.61 |
| 64,000-token prompt | 342.6 | 344.6 | 404.0 | 402.2 |

Peak 69-70 °C on the Qwen3.8 cards (61-66 °C on four cards), 15.9 GB per card.

## Agent batteries (2026-10-22)

Hermes agent harness (copies of the bake-off harness and eval home; nothing in the live Hermes setup changed), task
timeout 1,200 s for the subagent battery and 600 s for the standard one.

**Subagent battery.** Four new tasks that each require `delegate_task` with one subagent per unit of work (summarise
three changelogs, triage three services' logs, fix three modules, extract facts from three documents), with
checked answers. Qwen3.8 is the main agent; its subagents run either on Qwen3.6 (through the `delegation:` block of
the agent config) or on Qwen3.8 itself. Three reps per arm, arms interleaved. Every run delegated three subagents to
the intended model.

| Task | Subagents on Qwen3.6 | Subagents on Qwen3.8 |
|---|---|---|
| Changelogs | 417 s, 3/3 | 504 s, 3/3 |
| Log triage | 293 s, 3/3 | 568 s, 2/3 (one 1,200 s timeout) |
| Fix modules | 474 s, 3/3 | 419 s, 3/3 |
| Document facts | 362 s, 3/3 | 462 s, 3/3 |
| **All 12 runs** | **12/12, mean 427 s** | **11/12, mean 552 s (493 s without the timeout)** |

Median per task shown. Subagents on Qwen3.6 read about twice as many input tokens and still finished sooner. The main
agent read about 20-24k input tokens when its subagents ran on Qwen3.6, and 39-65k when they ran on Qwen3.8; why was not
investigated. In one-shot `hermes chat --query` mode the CLI exits before background subagents return, so the main agent
ran under a small wrapper that waits for them and hands their results back as further turns.

**Standard battery** (9 tasks x 3) on Qwen3.8 with two cards: 24 of 27 (89%), mean 159 s per task. On four cards
(2026-10-12, same file): 26 of 27, 120 s. Two of the three misses were `err_big_file_read` timeouts, a task that also
fails on four cards. On the tasks that passed, two cards took about 1.3-1.5x as long.

No error lines in either server log. Peak 68 °C.

## Tuning each service (2026-10-22)

Full speed client per setting; the starting setting ran first and last as the baseline.

| Qwen3.8, cards 2, 3 | Short | After 64k | Accepted short / deep |
|---|---:|---:|---|
| MTP 3 / 0.0 (production), first | 38.30 | 34.89 | 53 / 66% |
| MTP 4 / 0.2 (the Kmic fork's two-card recommendation) | 35.35 | 32.63 | 44 / 58% |
| MTP 3 / 0.2 | 37.03 | 33.68 | 52 / 65% |
| MTP 4 / 0.0 | 35.08 | 33.57 | 42 / 60% |
| MTP 3 / 0.0 (production), last | 37.69 | 35.81 | 52 / 68% |

| Qwen3.6, cards 0, 1 | Short | After 64k | Accepted short / deep |
|---|---:|---:|---|
| MTP 3 / 0.75, `-ub 512`, first | 90.82 | 61.02 | 91 / 89% |
| MTP 4 | 90.73 | 60.47 | 91 / 87% |
| MTP 2 | 88.83 | 59.94 | 92 / 91% |
| `-ub 1024` | did not load (out of memory on card 1) | | |
| p-min 0.5 | 91.59 | 60.15 | 75 / 70% |
| MTP 3 / 0.75, `-ub 512`, last | 90.84 | 60.93 | 91 / 89% |

The two Qwen3.8 baselines differ by 1.6% and 2.6%, so differences under about 3% are noise. Prompt speed was the same in
every arm (343-344 and 404-405 tokens/s).

## Not measured

Answer quality of Qwen3.6 against Qwen3.8 beyond the agent batteries, hours of mixed use, and the temperature-driven
fan service under the 2+2 load.
