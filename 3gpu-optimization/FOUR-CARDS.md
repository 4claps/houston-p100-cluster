# Four cards (2026-10-06 onward)

A fourth Tesla P100 was added to `houston` on 2026-10-06. This page records the test of three
cards against four on the production configuration, the switch of production to four cards,
the agent battery run on the four-card service, and three later tests on four cards (Q8_0, KV
cache type, MTP settings). The rest of this folder describes the
three-card machine.

**Summary**

- Four cards are faster on everything measured: a 64,000-token prompt processes at 514.5
  against 388.5 tokens/s (+32%), and generation with MTP runs at 42.1 against 37.4 (+13%).
- Memory per card drops from about 11 GB to 8.5 GB.
- Output on four cards differs from three by about as much as turning NCCL on does, and two
  four-card runs were byte-identical.
- The agent battery on the four-card service passed 26 of 27 tasks, its best result so far.
- Later tests on four cards (2026-10-11 and 10-12, [below](#later-tests-on-four-cards-2026-10-11-and-10-12))
  found nothing better than the current settings: Q8_0 instead of `UD-Q6_K_XL`, a q8_0 or f16
  context cache instead of q4_0, and seven MTP settings were each slower or within run-to-run noise.
- Not measured on four cards: output beyond 65,536 tokens, a long soak, and temperatures under the
  temperature-driven fan service.

## Detail

Configuration B (NCCL build, `NCCL_P2P_LEVEL=SYS`, graphs `=3`, `-lm none`), UD-Q6_K_XL, 150 W cap on all four cards, fans at full (`fans-full.service` started, `gpu-fan-control.service` stopped, at the owner's choice). Four Tesla P100 16 GB, each PCIe Gen3 x8, all pairs PHB, P2P OK on every pair, no kernel GPU errors. Production was already stopped when the test began. Cool-down to 48 °C, discarded warmup, 1 Hz telemetry, watchdog; no source change.

**Probe.** With all four cards visible, `-ts 1/1/1` stayed on three cards (card 3 peak 445 MiB), so the 3-card arm was run that way, as the serving file would.

**llama-bench, `-r 3`, order alternated per test:**

| Test | 3 cards | 4 cards | Change |
|---|---:|---:|---:|
| tg512 | 33.07 ± 0.17 | 39.24 ± 0.37 | +18.7% |
| pp2048, depth 0 | 505.11 ± 0.60 | 629.00 ± 1.28 | +24.5% |
| pp2048, depth 16384 | 450.96 ± 0.66 | 584.54 ± 1.46 | +29.6% |

**Server with the serving flags** (`-b 2048 -ub 2048`, `-c 262144`, MTP 3 / 0.0, one slot), two servers per card count in the order 3, 4, 4, 3:

| | 3 cards | 4 cards | Change |
|---|---:|---:|---:|
| 64,000-token prompt, t/s (4 runs each) | 391.3, 387.5, 389.8, 385.6 (mean 388.5) | 516.6, 512.8, 516.2, 512.2 (mean 514.5) | +32.4% |
| 64,000-token prompt, total time | 164.6–166.9 s | 124.6–125.9 s | −25% |
| 512-token generation with MTP, t/s (6 runs each) | 35.1, 37.5, 34.5, 39.4, 38.8, 39.3 (mean 37.4) | 42.6, 41.6, 40.9, 43.2, 42.0, 42.2 (mean 42.1) | +12.5% |
| Peak memory per card | 10.9 / 10.9 / 11.5 GB | 8.5 GB on each | |
| Hottest card | 66 °C | 66 °C | |
| GPU power, all cards, mean while busy / peak | 395 W / 528 W | 475 W / 647 W | |

**Output check** at `-c 4096` (wikitext-2 raw test, 8 chunks, 16,376 tokens scored), saved logits compared with kldpos:

| Pair | Mean KLD | Same top token |
|---|---:|---:|
| NCCL 3 cards vs non-NCCL 3 cards | 0.005337 | 98.400% |
| non-NCCL 4 cards vs non-NCCL 3 cards | 0.004896 | 98.284% |
| NCCL 4 cards vs non-NCCL 3 cards | 0.005158 | 98.333% |
| NCCL 4 cards vs non-NCCL 4 cards | 0.005724 | 98.321% |
| NCCL 4 cards, run 1 vs run 2 | 0 (0 of 16,376 records differ) | 100% |

Perplexity: 6.4930 (non-NCCL 3), 6.4899 (NCCL 3), 6.4835 (non-NCCL 4), 6.4767 (NCCL 4, both runs), each ± 0.129. Going from 3 to 4 cards changes the output by about as much as turning NCCL on does. Five 8.1 GB files are left in the scratch disk (not deleted).

**Rule, fixed before the runs, and the outcome:**

| Condition | Result |
|---|---|
| At least 3% faster on the server's 64k prefill or on generation, with no loss over 3% on the other | +32.4% and +12.5%: met |
| 4-card NCCL output no further from the 3-card reference than max(0.005, 2 x the 3-card NCCL difference = 0.0107) mean KLD, and at least 97.5% same top token | 0.005158 and 98.333%: met |
| Two 4-card NCCL runs identical | 0 records differ: met |

**Production change (18:21 EDT).** One value in the `qwen38` service command: `-ts 1/1/1` to `-ts 1/1/1/1`. The previous file was backed up first. `docker compose config` passed; only `qwen38` started; healthy after about 155 s. Startup: MTP draft context created, the usual warnings, no error, `/metrics` answers 200, 8,163 MiB loaded per card. Requests from the client machine:

| Request | Prefill t/s | Decode t/s |
|---|---:|---:|
| Short chat | — | 55.7 |
| Second request right after | — | 50.0 |
| 64,000-token prompt | 517.7 | 38.2 |
| MTP code request | — | 55.2 |

Peak during those requests: 57 / 54 / 57 / 59 °C; 167 / 178 / 167 / 168 W instantaneous; 8,531 MiB per card. With an estimated 60 W for the CPU, the peak draw of about 707 W is 71% of the 1000 W supply.

**Agent bake-off on the 4-card production service (2026-10-06 18:26–19:20 EDT).** Same harness, nine tasks, 600 s limit, 3 repetitions, run against the production service itself (`-b 2048`, 150 W, `-c 262144`, MTP 3 / 0.0).

| | 4 cards | 3 cards with NCCL (round 4, first 3 reps, 125 W, `-b 32768`) | 3 cards without NCCL (same round) |
|---|---:|---:|---:|
| Tasks passed | 26 of 27 (96%) | 24 of 27 (89%) | 23 of 27 (85%) |
| Mean wall time, all 27 runs | 120 s | 124 s | 183 s |
| Mean wall time, eight tasks without `err_big_file_read` | 69 s | 84 s | 131 s |
| Runs killed at the 600 s limit | 0 | 2 | 3 |
| Server prefill over the battery | 453.9 t/s | 377.9 t/s | 242.2 t/s |
| Server decode with MTP over the battery | 53.9 t/s | 48.9 t/s | 48.1 t/s |
| Draft acceptance | 71.4% | 73.3% | 71.2% |

| Task | Passed | Mean wall |
|---|---:|---:|
| err_python_env | 3 of 3 | 67 s |
| err_replay_patch | 3 of 3 | 65 s |
| err_ambiguous_edit | 3 of 3 | 70 s |
| err_case_search | 3 of 3 | 71 s |
| err_hidden_search | 3 of 3 | 63 s |
| err_big_output | 3 of 3 | 64 s |
| err_multi_dir | 3 of 3 | 61 s |
| err_inline_script | 3 of 3 | 88 s |
| err_big_file_read | 2 of 3 | 529 s (453, 535, 599 s) |

- The comparison columns differ from the 4-card run in more than card count (125 W against 150 W, `-b 32768` against `-b 2048`, and a day apart), so the gain is not all from the fourth card.
- `err_big_file_read` passed twice (535 s and 599 s, the second one second inside the limit). Its one failure was not a timeout: the agent's conversation reached 278,695 tokens and the server refused it for exceeding the 262,144-token context; the agent stopped with "Context overflow and auto-compaction is disabled". That refusal is the only error line in the server log.
- The overall mean wall time barely moves (120 s against 124 s) because `err_big_file_read` now runs to the end, for about 9 minutes, where it used to be cut off or finish early.
- After the battery: production healthy, slot idle, 55–61 °C, 8,531 MiB per card, 150 W.

## Later tests on four cards (2026-10-11 and 10-12)

Three more tests on the four-card production configuration, all measurement only: no source or
Compose change, production stopped for each (idle first) and restored after. Same method as above:
NCCL build with `NCCL_P2P_LEVEL=SYS`, graphs `=3`, `-lm none -fit off`, `-ts 1/1/1/1`, 150 W cap, fans
at full during the test, cool-down to 48 °C or below, discarded warmup, 1 Hz telemetry, watchdog. No
safety stop tripped; the hottest card reached 71 °C. **Result: production stays as it is.**

| Question | Answer |
|---|---|
| Is Q8_0 better than `UD-Q6_K_XL`? | No. Battery 26 of 27 for both; Q8_0 generates 4.6% slower and uses 0.8 GB more per card. |
| Is a q8_0 or f16 context cache better than q4_0? | No. q8_0 is the slowest option (−8.7% generation in llama-bench). f16 is about 3% faster on short requests, 2.5% slower on a 64,000-token prompt, and costs 2.9 GB more per card. |
| Does the q4_0 cache hurt output? | Very little: same top token 98.0% of the time against f16, no drift out to 65,536 tokens. |
| Is any other MTP setting better? | No. `n-max` 2 and 5 are 7% and 10% slower; the other settings are within noise. |
| What does MTP cost and buy? | +15% generation on short requests, +11% deep, −4.5% on a 64,000-token prompt. |

### Q8_0 against UD-Q6_K_XL (2026-10-11)

Production was idle before it was stopped (no busy slot over 30 s, zero tokens served since boot). The
Q8_0 battery ran against a copy of the production service with only the model file, mount and container
name changed.

**1. llama-bench (`-r 3`, `-b 32768 -ub 2048`)**

| Test | XL t/s | Q8_0 t/s | Q8_0 / XL | Peak MiB per card XL / Q8_0 |
|---|---:|---:|---:|---:|
| tg512 | 39.22 ± 0.38 | 37.43 ± 0.34 | 0.954 | 6719 / 7513 |
| pp2048 @0 | 629.59 ± 1.24 | 633.33 ± 0.67 | 1.006 | 8393 / 9189 |
| pp2048 @16384 | 583.22 ± 1.43 | 588.02 ± 1.56 | 1.008 | 8517 / 9313 |

**2. Server with the serving flags (`-c 262144 -b 2048 -ub 2048`, MTP 3 / 0.0), order XL, Q8_0, Q8_0, XL**

| | XL (2 runs) | Q8_0 (2 runs) |
|---|---:|---:|
| 64,000-token prompt, t/s (4 values) | 516.3, 512.5, 515.8, 512.3 (mean 514.2) | 519.8, 516.1, 518.8, 516.2 (mean 517.7) |
| 512-token generation with MTP, t/s (6 values) | 38.9, 38.9, 44.6, 42.5, 41.1, 40.0 (mean 41.0) | 43.1, 43.0, 40.2, 41.6, 37.3, 41.1 (mean 41.0) |
| Draft tokens accepted / drafted | 1754 / 3916 (44.8%) | 1752 / 3921 (44.7%) |
| Short chats, t/s | 48.8, 54.9, 52.1, 51.1 | 47.7, 50.9, 48.6, 51.1 |
| Peak MiB per card | 8531 | 9359 |
| Peak C | 64/61/64/66 | 65/61/64/66 |

The MTP generation figures spread by 15% within one model (sampling at temperature 1.0 changes how many drafted tokens are accepted), so the equal means do not show that Q8_0 is free with MTP; they show the difference is smaller than that spread.

**3. Output comparison (`llama-perplexity`, wikitext-2, `-c 4096`, 8 chunks, 16,376 tokens scored; saved logits compared with `kldpos`)**

| Comparison | Mean KLD | Median | p99.9 | Max | Same top token |
|---|---:|---:|---:|---:|---:|
| XL today against XL 2026-10-06 (repeat check) | 0 (0 of 16,376 records differ) | 0 | 0 | 0 | 100% |
| XL against Q8_0 (Q8_0 as reference) | 0.010150 | 0.000797 | 1.4185 | 13.33 | 97.472% |

PPL: XL 6.4767 ± 0.1284, Q8_0 6.5279 ± 0.1303. The difference is inside the error. KLD against Q8_0 says how far apart the two files are, not which is closer to the full-precision model; no full-precision reference was run. For scale, NCCL against non-NCCL on the same file measured 0.0052 on this text.

**4. Agent battery (9 tasks x 3 reps, 600 s limit, same harness) against a Q8_0 copy of the production service**

The service was the production Compose definition with only the model file, the model mount and the container name changed, and no restart policy.

| Task | XL 2026-10-06 ok / wall | Q8_0 2026-10-11 ok / wall |
|---|---:|---:|
| err_python_env | 3/3, 67 s | 3/3, 71 s |
| err_replay_patch | 3/3, 65 s | 3/3, 62 s |
| err_ambiguous_edit | 3/3, 70 s | 3/3, 92 s |
| err_case_search | 3/3, 71 s | 3/3, 83 s |
| err_hidden_search | 3/3, 63 s | 2/3, 61 s |
| err_big_output | 3/3, 64 s | 3/3, 64 s |
| err_multi_dir | 3/3, 61 s | 3/3, 71 s |
| err_inline_script | 3/3, 88 s | 3/3, 152 s |
| err_big_file_read | 2/3, 529 s | 3/3, 484 s |
| **Total** | **26/27 (96%), 120 s** | **26/27 (96%), 127 s** |

Mean wall excluding `err_big_file_read`: XL 69 s, Q8_0 82 s. Q8_0 battery: 3,418 s, 1,004,120 prompt tokens and 58,483 generated tokens served, peak 71/66/71/72 C, peak 9,357 MiB per card, no out-of-memory, abort or context-exceeded line in the server log. The cause of the one failed `err_hidden_search` repetition was not investigated. The two batteries were five days apart and the wall times were not interleaved, so the 13 s difference is an observation, not a controlled result; one failure in 27 on each side does not separate the files.

### f16 KV cache at the serving context (2026-10-12)

Run at the real 262,144-token context, since the 2026-10-11 test below only benchmarked f16 and did not
serve it. Model downloads (network and data-disk writes) were running on the host during the test. A
first launch stopped after three minutes because a helper script had not been copied; nothing was
measured in it.

| Run | Short t/s | Draft accepted | Deep t/s | Draft accepted | 64k prompt t/s | Peak MiB per card |
|---|---:|---:|---:|---:|---:|---:|
| Production (KV q4_0), run 1 | 44.69 | 53.1% | 38.78 | 59.7% | 511.9 | 8555 |
| KV f16, run 1 | 45.87 | 52.7% | 39.82 | 61.6% | 499.9 | 11471 |
| KV f16 and draft KV f16 | 47.22 | 53.2% | 42.72 | 65.9% | 500.2 | 11677 |
| Production, run 2 | 44.70 | 51.5% | 41.62 | 66.1% | 511.9 | 8577 |
| KV f16, run 2 | 46.13 | 52.0% | 41.90 | 65.8% | 497.9 | 11495 |

- KV f16 against q4_0, means of two runs: short +2.9% (46.00 against 44.70), deep +1.6% (40.86 against 40.20, inside the repeat spread), 64k prompt −2.5% (498.9 against 511.9), +2.9 GB per card.
- f16 for the draft cache as well: one run, +5.6% short; not repeated, so not established.
- The deep figures rest on 768 tokens per run and differ by up to 7% between repeats, so they show no
  clear difference. This closes the gap noted in the next section ("f16 KV was not run on the server").

### KV cache type and MTP settings (2026-10-11)

Production was idle before it was stopped. No fork source changes; the out-of-tree logit dump tool gained
two arguments (KV type, number of cards).

**1. llama-bench by KV cache type (`-r 3`, `-b 32768 -ub 2048`)**

| KV type | tg512 t/s | vs q4_0 | pp2048 @16384 t/s | vs q4_0 | Peak MiB per card at 16k |
|---|---:|---:|---:|---:|---:|
| q4_0 (production) | 39.21 ± 0.38 | — | 582.53 ± 1.52 | — | 8517 |
| q8_0 | 35.80 ± 0.31 | −8.7% | 569.56 ± 2.12 | −2.2% | 8567 |
| f16 | 40.35 ± 0.39 | +2.9% | 569.90 ± 1.89 | −2.2% | 8701 |

**2. Output by KV cache type at 65,536 tokens** (wikitext-2 raw test, first 65,536 tokens as one context, every position's distribution saved by the out-of-tree dump tool, n_batch 2048, `-ub 2048`, 4 cards; compared with `kldpos`)

Tool check at `-c 4096`, q4_0: the dump tool's records match today's `llama-perplexity` file bit for bit (0 of 2,047 differ). A second f16 run gave the same hash as the saved one (`a0ace4e829f16638`).

| Comparison | Mean KLD | Median | p99.9 | Max | Same top token |
|---|---:|---:|---:|---:|---:|
| q8_0 against f16 | 0.000124 | 0.000037 | 0.0043 | 0.184 | 99.487% |
| q4_0 against f16 | 0.002084 | 0.000924 | 0.0553 | 2.637 | 97.963% |
| q4_0 against q8_0 | 0.002101 | 0.000932 | 0.0563 | 2.709 | 97.943% |

Mean KLD of q4_0 against f16 by position, 8,192 positions per bucket: 0.00181, 0.00220, 0.00198, 0.00226, 0.00256, 0.00202, 0.00186, 0.00198. No upward trend with depth. PPL over all positions: f16 5.6174, q8_0 5.6165, q4_0 5.6315 (+0.25%). Peak MiB per card during the dump: f16 9931, q8_0 9453, q4_0 9197.

For scale, on this project's earlier measurements: NCCL against non-NCCL at 65k was 0.00089 and the batch-size yardstick 0.00152 (3 cards); UD-Q6_K_XL against Q8_0 model files at 4k was 0.0102. The q4_0 cache's effect is a fifth of the gap between the two model files.

**3. Server with the serving flags (`-c 262144 -b 2048 -ub 2048`), one setting changed per run**

Client per run: 12 generations of 384 tokens (3 prompts x 4 seeds; "short"), one 64,000-token prompt, then 3 generations of 256 tokens on top of that prompt from the slot's cache ("deep"). t/s is total tokens over total generation time. Production was run first and last; the two runs differ by 2.8% (short) and 2.6% (deep), so smaller differences are noise. The deep figure rests on 768 tokens per run.

| Run | Short t/s | Draft accepted | Deep t/s | Draft accepted | 64k prompt t/s | Peak MiB per card |
|---|---:|---:|---:|---:|---:|---:|
| Production, run 1 (n-max 3, top-k 20, q4_0) | 45.78 | 52.7% | 41.95 | 66.2% | 512.1 | 8577 |
| Production, run 2 | 44.52 | 50.8% | 40.90 | 63.1% | 511.2 | 8577 |
| n-max 2 | 42.02 | 65.2% | 39.83 | 70.8% | 511.8 | 8495 |
| n-max 4 | 43.84 | 42.5% | 43.92 | 64.3% | 508.1 | 8591 |
| n-max 5 | 40.73 | 35.7% | 38.46 | 62.9% | 510.7 | 8653 |
| n-max 4 + `LLAMA_SPEC_DRAFT_TOPP=0.95` | 44.77 | 44.0% | 43.22 | 64.2% | 508.4 | 8615 |
| `LLAMA_SPEC_DRAFT_TOPP=0.95` | 43.97 | 49.2% | 42.58 | 69.3% | 511.5 | 8577 |
| `LLAMA_SPEC_DRAFT_TOPK=10` | 44.73 | 50.7% | 42.81 | 65.5% | 511.1 | 8553 |
| `LLAMA_SPEC_DRAFT_TOPK=40` | 44.50 | 50.5% | 41.58 | 68.4% | 511.1 | 8577 |
| KV q8_0, run 1 | 44.22 | 51.3% | 38.91 | 60.0% | 496.9 | 9579 |
| KV q8_0, run 2 | 42.89 | 48.5% | 40.05 | 63.3% | 497.5 | 9557 |
| KV q8_0 and draft KV q8_0 | 43.72 | 51.5% | 38.25 | 63.9% | 497.4 | 10663 |
| MTP off | 39.24 | — | 37.31 | — | 536.2 | 8103 |

- MTP on against off: +15% short (mean 45.15 against 39.24), +11% deep (41.4 against 37.3), and −4.5% on the 64,000-token prompt (511.7 against 536.2), for 474 MiB more per card.
- n-max 2 and 5 are slower than 3 (−7% and −10% short). n-max 4 is −3% short and +6% deep; with only 768 deep tokens per run and a 2.6% repeat spread, that deep gain is not established.
- The three draft cut-off settings are all within noise of production.
- KV q8_0 on the server: −3.5% short, −4.7% deep, −2.9% on the 64,000-token prompt, +1.0 GB per card (mean of two runs). A q8_0 draft cache adds another 1.1 GB per card for no gain.
- f16 KV was not run on the server at `-c 262144` in this test; the 2026-10-12 test above did that.

Not measured in these three tests: output beyond 65,536 tokens, and any agent battery on the KV or MTP
settings. KLD against Q8_0 says how far apart two model files are, not which is closer to full precision;
no full-precision reference was run.
