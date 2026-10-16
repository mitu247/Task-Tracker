# From Conversation History to Task State: Diagnosing Small Language Models under Incremental Observations

Sushmita Paul

**Collaborators:** Apurbo Banik Turjo, Dr. Ch. Md. Rakin Haider

Research progress updated **15 October 2026**. Paper in preparation.

This repository presents the study's design and current aggregate results. Raw
trajectories, full prompts, and run manifests are not included. This overview
is synchronized with the shared [Task-Tracker study](https://github.com/excellencior/Task-Tracker).

## Research question

**Can small language models manage their own task memory as observations arrive,
and when do they need externally maintained state?**

An agent acting over many turns must incorporate new information without losing
what it has already learned. We study conversation history, verified external
state, state-report correction, and model-managed memory using interactive
breadth-first search (BFS) and depth-first search (DFS) on partially observed
trees. Each turn reveals only the neighbors of the node reached.

These controlled tasks let us distinguish visiting all nodes from following the
required traversal order. They are diagnostics, not the full range of agents
we hope to study.

![External-state experimental setup.](assets/research_setup.png)

## Experimental setup

Completed behavioral experiments contain **57,600 trajectories** across two
external-state campaigns and two model-managed-memory variants. All use
Qwen3-1.7B, Qwen3.5-2B, Qwen3.5-4B, and Gemma-4-E4B-IT; normal and hard BFS/DFS
instances; 200 matched graphs per exact cell; greedy decoding; a 4,096-token
output cap; a 2,048-token thinking budget; and a 32,768-token context limit.

| Campaign | Trajectories | State or memory | History and execution |
|---|---:|---|---|
| Initial external state | 25,600 | BFS: current node and visited nodes, no queue. DFS: current node, visited nodes, and stack. RAW, PSR, SSR, VI. | Retain/clear injections; `p1_concise_legacy`; H100 NVL; up to 16 parallel trajectories. |
| Matched control-state follow-up | 25,600 | BFS queue supplied; DFS stack omitted. Same four conditions. | Same task prompt, graphs, decoding, limits, and scorer; retain/clear injections; H100; 64 parallel trajectories. |
| Model memory + history | 3,200 | Model-chosen free-schema JSON memory, capped at 1,024 tokens; no verified queue or stack. | Full conversation plus latest memory; `p1_model_memory_injection`; H100; 64 parallel trajectories. |
| Model memory only | 3,200 | Model-owned JSON store is the only cross-turn memory beyond current observation and feedback. | Earlier conversation omitted; `p1_model_memory_memory_only`; H100; 64 parallel trajectories. |

Together, the external-state campaigns cover BFS with and without its queue and
DFS with and without its stack. The model-managed variants ask whether the model
can choose what to store and preserve it. They use different prompts and response
contracts, so their absolute results do not isolate memory ownership alone.

## External state and history

| Condition | State support |
|---|---|
| RAW | Current observation, without an injected state reminder. |
| PSR: Prose State Reminder | Verified state restated in ordinary language. |
| SSR: Structured State Reminder | The same task-specific facts in a tagged block. |
| VI: Verifiable Injection | SSR plus a required state report, checked against the environment; missing, malformed, or incorrect reports receive feedback on the next turn. |

Earlier versions of this overview called the verification condition **SRVC**.
The current study uses **VI**.

Under **retain**, earlier injected reminders remain. Under **clear injection**,
only older reminder text is removed: earlier observations and visible replies
remain, and the current reminder is always present. This differs from the
model-memory-only variant, which omits earlier conversation history.

![Retaining or clearing older injected reminders.](assets/history_policies.png)

## Model-managed memory

The model decides what information to keep in a JSON store and updates that
store as it acts. We compare access to this store alongside conversation history
with access to the store without that history. This tests more than recall:
the model must select useful information, update it correctly, and use it to
choose an action.

The initial pilot also tested compliance with an outer JSON response format.
Format failures therefore confound conclusions about memory quality. An ongoing
**25,600-trajectory memory-responsibility study** compares RAW, oracle state,
free-form model memory, and fixed-schema model memory, each with and without
conversation history. Its outer JSON response is constrained to reduce this
formatting confound.

## Evaluation

- **Graph completion:** whether every node was visited, regardless of order.
- **Correct traversal prefix:** the uninterrupted initial sequence of
  algorithm-correct accepted moves divided by recorded node-history length.
  Later recovery does not restore the prefix.
- **Algorithm-correct completion:** completion with a perfect traversal prefix.
- **Zero-error FIFO success:** strict FIFO traversal success without invalid
  attempts or format failures.
- **Valid-move rate:** admissible attempts divided by all recorded attempts.
  A legal move can still violate traversal order.
- **Final decoverage:** the fraction of graph nodes left unvisited.

Accumulated decoverage is interpreted alongside completion and episode length:
a short failed episode can have a misleadingly small accumulated value.

The initial BFS runs used a strict FIFO-head checker. Recorded actions were
also replayed under the historical AQA-Bench BFS frontier checker. The first
table below reports replayed AQA success; the follow-up and memory tables report
strict recorded traversal success. These success columns are not interchangeable.
DFS scores coincide under the two rules.

## Current behavioral results

Each external-state row pools 3,200 trajectories across models, difficulties,
and history policies.

### Initial external-state campaign

| Task | Control state | Condition | Graph completion | AQA algorithm-correct completion |
|---|---|---|---:|---:|
| BFS | Queue not supplied | RAW | 10.94% | 0.44% |
| BFS | Queue not supplied | PSR | 24.50% | 1.63% |
| BFS | Queue not supplied | SSR | 24.53% | 2.56% |
| BFS | Queue not supplied | VI | 16.41% | 3.03% |
| DFS | Stack supplied | RAW | 22.69% | 14.88% |
| DFS | Stack supplied | PSR | 72.84% | 58.13% |
| DFS | Stack supplied | SSR | 79.19% | 70.00% |
| DFS | Stack supplied | VI | 74.59% | 62.81% |

### Matched control-state follow-up

| Task | Control state | Condition | Graph completion | Strict success |
|---|---|---|---:|---:|
| BFS | Queue supplied | RAW | 10.00% | 0.28% |
| BFS | Queue supplied | PSR | 82.09% | 33.19% |
| BFS | Queue supplied | SSR | 87.75% | 45.91% |
| BFS | Queue supplied | VI | 80.25% | 35.31% |
| DFS | Stack omitted | RAW | 25.41% | 16.38% |
| DFS | Stack omitted | PSR | 51.16% | 40.53% |
| DFS | Stack omitted | SSR | 46.88% | 38.38% |
| DFS | Stack omitted | VI | 41.75% | 32.72% |

Supplying the BFS queue raises externally assisted BFS performance, while
removing the DFS stack reduces corresponding DFS performance. RAW changes
little. The original task gap therefore partly reflected supplied control
state; this does not establish that BFS and DFS are intrinsically identical.

### Model-managed-memory pilot

Each row pools 1,600 trajectories across four models and two difficulties.

| Task | Memory variant | Graph completion | Strict success |
|---|---|---:|---:|
| BFS | Memory with full history | 1.38% | 0.12% |
| BFS | Memory as the only cross-turn store | 3.25% | 0.19% |
| DFS | Memory with full history | 22.69% | 12.69% |
| DFS | Memory as the only cross-turn store | 2.69% | 1.00% |

Allowing a model to create a store does not ensure it preserves sufficient task
state when history is removed. BFS remains weak in both variants. These runs
jointly test memory design, retention, action selection, and format compliance;
they are not an isolated causal comparison with externally verified state.

## Reasoning-trace analysis

A separate primary LLM-judge pass evaluated 2,560 paired trajectories using
`claude-opus-4-6-thinking`. It rated state tracking, algorithmic reasoning and
action coherence, history and feedback use, and visible response quality on
1–5 scales. Model, condition, history-policy labels, and aggregate outcome
scores were withheld, though prompt contents revealed some state-support cues.

Equal-weight mean ratings were RAW **1.82**, PSR **2.85**, SSR **3.03**, and VI
**2.59**. These are descriptive ratings, not validated measurements of hidden
reasoning. All judgments passed schema extraction; seven semantic warning flags
were inspected. The selected 246 repeat judgments and independent blinded human
calibration remain incomplete. Behavioral results remain the primary evidence.

## Limitations

- Four checkpoints, selected trees, finite token limits, and one execution per
  exact cell limit generalization.
- The initial external-state campaign was asymmetric. The follow-up covers the
  complementary queue/stack settings; matched analysis is ongoing.
- Model-managed variants differ in prompts and JSON contracts. The new study
  aims to separate memory behavior from response-format compliance.
- VI combines reporting, delayed feedback, and prompt-format tension. In 4,625
  of 6,400 VI trajectories, at least one correction about a previous turn appears
  alongside different current-state values. This does not identify the causal
  effect of verification alone.
- Selected traces illustrate failures but do not establish their prevalence.
- Transfer beyond graph traversal has not yet been established.

## Research progress

### Completed

- [x] Build and validate the BFS/DFS evaluation pipeline.
- [x] Run the initial 25,600-trajectory external-state campaign.
- [x] Audit behavioral metrics and complete the 2,560-trajectory primary judge pass.
- [x] Run the matched 25,600-trajectory queue/stack follow-up.
- [x] Run the 6,400-trajectory model-managed-memory pilot.
- [x] Back up follow-up trajectories and preserve run manifests.

### In progress

- [ ] Complete the 25,600-trajectory memory-responsibility study.
- [ ] Complete matched analysis, including model- and difficulty-level effects.
- [ ] Trace the first memory inconsistency and distinguish memory-selection,
  memory-update, and action-selection failures.

### Next

- [ ] Complete judge repeatability and independent human calibration.
- [ ] Evaluate a time-aligned VI protocol.
- [ ] Test transfer beyond graph traversal.

---

**Sushmita Paul** · Dhaka, Bangladesh  
[GitHub](https://github.com/mitu247) · [Portfolio](https://sushmitapaul.onrender.com) · [Email](mailto:sushmitap247@gmail.com)
