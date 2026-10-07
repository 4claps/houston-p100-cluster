# Four cards (2026-10-06)

A fourth Tesla P100 was added to `houston` on 2026-10-06. This page records the test of three
cards against four on the production configuration, the switch of production to four cards,
and the agent battery run on the four-card service. The rest of this folder describes the
three-card machine.

**Summary**

- Four cards are faster on everything measured: a 64,000-token prompt processes at 514.5
  against 388.5 tokens/s (+32%), and generation with MTP runs at 42.1 against 37.4 (+13%).
- Memory per card drops from about 11 GB to 8.5 GB.
- Output on four cards differs from three by about as much as turning NCCL on does, and two
  four-card runs were byte-identical.
- The agent battery on the four-card service passed 26 of 27 tasks, its best result so far.
- Not measured on four cards: output quality at long context (the check was at 4,096 tokens),
  a long soak, and temperatures under the temperature-driven fan service.

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
