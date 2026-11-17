# Learning Log

30-day sprint: evals and LLM inference. One entry per day, newest at the bottom.

Each entry has three lines:
- **Did:** what I worked on
- **Measured:** one number I produced
- **Confused:** what I still don't understand

**How Days 1-15 were produced.** I fell behind after Day 0. On 2026-11-17 I asked Claude Code to run the
runnable tasks for Days 1-15 on my machine and write the results here. Every number below comes from a real
run on my M4 (16 GB), and the output files are in `results/`. Two things are not mine yet:
(1) **the reading, writing and posting tasks are marked NOT DONE** and are still mine to do, and
(2) **the dataset is template-generated, not hand-labeled** (see `DATASHEET.md`). Where I have no real confusion yet,
the "Confused" line holds an open question from the run, not a claim about my understanding.

Model for everything below: Qwen2.5-1.5B-Instruct (local folder), not Qwen3, because that is what was downloaded.

---

## Day 0 (2026-10-20)
- **Did:** Made the repo, cloned it, created the venv, wrote `bench.py` for the Gemma 4 12B GGUF (never ran it; needs `llama-cpp-python`).
- **Measured:** nothing yet.
- **Confused:** Whether to use the GGUF model or a Hugging Face Qwen for evals. Decided on Qwen, since lm-eval's log-likelihood tasks need a Hugging Face model.

## Day 1 (run 2026-11-17)
- **Did:** Loaded Qwen2.5-1.5B-Instruct with transformers on MPS (float16) and timed greedy decoding of 128 tokens, 3 runs after a warm-up (`day01/bench_hf.py`). NOT DONE: reading Hamel Husain's eval posts, and the one-paragraph-per-metric write-up (exact match, F1, pass@k, judge score).
- **Measured:** **17.5 tokens/s decode** (runs 17.5, 17.5, 17.3). Time to first token 0.178 s for a 45-token prompt.
- **Confused:** The first token takes 0.18 s but every later token takes ~57 ms. Why is decoding so much slower per token than the prefill rate suggests? (Day 15 arithmetic is meant to answer this.)

## Day 2 (run 2026-11-17)
- **Did:** Ran lm-eval on `arc_easy` (log-likelihood, 200 questions) and `gsm8k` (generative, 5-shot, first 50), plus gsm8k again with `--apply_chat_template`. Read the task YAMLs and 10 logged gsm8k samples. NOT DONE: the flag cheat sheet in my own words (flags used are in `day02/run.sh`).
  - `float16` segfaulted under lm-eval on MPS (it worked in my own script), so the runs use `dtype=bfloat16`.
  - **YAML trace, arc_easy:** prompt `doc_to_text: "Question: {{question}}\nAnswer:"`; target `doc_to_target: "{{choices.label.index(answerKey)}}"` (the index of the right choice); candidates `doc_to_choice: "{{choices.text}}"`; metrics `acc` and `acc_norm` (normalized by answer length). The model never generates text. It scores the log-probability of each choice.
  - **YAML trace, gsm8k:** target is the whole worked solution `{{answer}}`; `exact_match` strips everything up to `#### `. Two filters read the model output: `strict-match` needs the literal `#### <number>`, and `flexible-extract` takes the last number in the text.
  - **Scorer error found (doc 5, doc 9):** the model wrote a correct solution ending "Therefore, the answer is 64." (target 64) and "...is 460." (target 460). Strict-match marked both wrong (`[invalid]`), because it never wrote `####`. Flexible-extract gets them right.
- **Measured:** arc_easy acc **0.710 +/-0.032**, acc_norm 0.740 +/-0.031 (81 s). gsm8k 5-shot, n=50: strict **0.42 +/-0.071**, flexible 0.52 +/-0.071 (484 s). With chat template: strict 0.30, flexible 0.48.
- **Confused:** Why did the chat template lower the strict score (0.42 to 0.30) while the flexible score barely moved? My guess is that the instruct model stops copying the `####` format, but I have not checked the samples.

## Day 3 (run 2026-11-17)
- **Did:** Wrote three prompt formats for arc_easy as custom task YAMLs (`day03/tasks/`) and ran each on the same 200 questions, plus 5-shot. My `arc_easy_plain` YAML reproduces the built-in task exactly (0.71 / 0.74), which confirms the setup. NOT DONE: reading on contamination and the Sclar et al. paper.
- **Measured:** plain 0.710, instruction-style 0.725, **lettered options (A/B/C/D) 0.905**, plain 5-shot 0.775. All stderr about 0.03. **Prompt format moved the score by 19.5 points; stderr was 3.2 points.**
- **Confused:** The lettered format scores 20 points higher while asking the same question. Is the model better at *choosing a letter* than at *assigning probability to answer text*? Which of these is the "true" arc_easy score?

## Day 4 (run 2026-11-17)
- **Did:** Built a judge test with 20 factual question pairs, one right and one wrong answer each, using Qwen2.5-1.5B as the judge (`day04/judge_bias.py`). The judge answers A or B, read from the token logits. Each pair is judged in both orders, and I padded wrong answers with confident filler to test verbosity bias. NOT DONE: MT-Bench and "not fair evaluators" papers.
- **Measured:** **15% flip rate** (3 of 20 pairs change winner when the order is swapped). Accuracy 92.5% overall. Picks position A 47.5% of the time (no big position bias on average). **Padded wrong answers were chosen 47.5% of the time**, against a 7.5% error rate unpadded.
- **Confused:** A 1.5B judge is clearly vulnerable to padding. I don't know whether a larger judge shrinks this or just hides it.

## Day 5 (run 2026-11-17)
- **Did:** Installed `inspect-ai` and `inspect-evals`. Ran the built-in `inspect_evals/arc_easy` on the local model (50 questions) and wrote a 10-question custom task (`day05/mini_eval.py`). I looked only at the class signatures in the Inspect source (Task, Solver, Scorer, Sample), not a full read. NOT DONE: reading the Solver, Scorer and Dataset source properly, and drawing the diagram myself.
  - Flow as I understand it from the run: `Dataset` of `Sample(input, target)` -> `Task` runs each sample through the `solver` list (`system_message`, then `generate`) which builds up the conversation state -> the `scorer` compares the final output with `target`.
- **Measured:** Inspect arc_easy **0.880 +/-0.046** (n=50, 52 s). Custom task **0.900** (9 of 10, 11 s).
- **Confused:** Inspect scores 0.88 and lm-eval 0.71 on the "same" benchmark. Inspect shows lettered options and asks the model to answer, so it is probably the same effect as Day 3's lettered format (0.905). I haven't verified that.

## Day 6 (run 2026-11-17)
- **Did:** Chose the task: **customer message to JSON extraction**. Programmatic scoring, and small models land in the middle. Built 200 examples (more than the 30 the plan asks for today) and wrote rubric v0. Ran 10 examples as a saturation check (`results/day06_first10.jsonl`).
- **Measured:** **6 of 10 exact (0.60)**, so the task is neither at 0% nor 100%. The failures were all on the `issue` field.
- **Confused:** I did not write these examples by hand, so I don't yet have a feel for what real messages look like. That is a limit of this dataset.

## Day 7 (review, run 2026-11-17)
- **Did:** Listed candidate open-source issues (titles only, **not yet read in full**) from the repos' `good first issue` labels:
  1. inspect_ai #6358: `csv_dataset()` reads a blank `id` cell as `""`, so two blank rows break `eval()`
  2. inspect_ai #5950: Google provider never sends `seed`
  3. lm-evaluation-harness #2552: Hendrycks Math extraction rule seems too strict
  4. vllm #32268: refactor `Int8ScaledMMLinearLayerConfig` to use `QuantKey`
  5. lm-evaluation-harness #1613: better test coverage for models

  NOT DONE: the one-page "what I know about evals" note, and the public post.
- **Measured:** 5 issues listed. Issue numbers are from GitHub on 2026-11-17 and may be claimed by now.
- **Confused:** Which of these is small enough to finish in two days? Not known until I read each thread.

## Day 8 (run 2026-11-17)
- **Did:** Used the 100-example subset `data/v1_100.jsonl` (20 per category) and wrote rubric v1 with the borderline cases (`RUBRIC.md`). The labels are exact by construction.
- **Measured:** 100 examples, 5 categories, 4 issue types, 6 fields each.
- **Confused:** The main rubric question is whether "never showed up" is `late` or lost. I settled on `late`, and it turned out to matter (Day 13).

## Day 9 (run 2026-11-17)
- **Did:** Generated all 200 examples (seeded, reproducible) and split each category 30 dev / 10 test (150 / 50). Wrote `DATASHEET.md`. Verified that every name, order id and product word appears verbatim in its message. Found and fixed a bug: typos were landing on names and products, which broke the labels.
- **Measured:** 200 examples; 150 dev, 50 test. **Test has not been run.**
- **Confused:** Is template-built data a fair eval at all? It tests extraction, but not the messy language of real tickets.

## Day 10 (run 2026-11-17)
- **Did:** Wrote the harness `evalkit/`: loader with validation, runner (batched MPS generation, on-disk cache), scorer, CLI. Ran the zero-shot dev baseline.
- **Measured:** **Exact match 0.840 (95% CI 0.780-0.893)**, n=150. Per field: name .987, order_id .973, product 1.0, quantity 1.0, issue .873, amount 1.0. No unparseable outputs.
- **Confused:** The scorer accepts `ORD-31640` for order id 31640. Should it?

## Day 11 (run 2026-11-17)
- **Did:** Added an LLM judge (same 1.5B model; no API key was available for a stronger one) and compared it with the programmatic scorer on 50 dev outputs. Tried two prompts (`day11/judge_agreement.py`).
- **Measured:** v1 prompt: agreement 0.86, **kappa 0.00** (it said "Yes" to all 50; 0.86 is just the pass rate). v2 rubric prompt: agreement 0.78, **kappa 0.03**.
- **Confused:** The judge is useless at this size, and the better prompt made agreement *worse*. A bigger judge model is the next thing to try. Is there any judge small enough to run locally that works?

## Day 12 (run 2026-11-17)
- **Did:** Added bootstrap CIs and a paired bootstrap (`evalkit/stats.py`). Ran 3 seeds at temperature 0.7, and a 3-shot prompt at temperature 0 on the same 150 examples (`day12/compare.py`). NOT DONE: reading the error-bars paper.
- **Measured:** 3 seeds: 0.833 / 0.833 / 0.847 (**mean 0.838, sd 0.008**). 3-shot 0.907 [0.860, 0.947]. **Few-shot minus zero-shot: +6.7 points, paired 95% CI [+0.7, +12.7].** It fixes 15 examples and breaks 5. The gain is real but near the edge.
- **Confused:** Seeds differ by under 1.5 points while the CI is +/-6 points. So the CI measures *which questions* I sampled, not sampling randomness. When does seed variance matter then?

## Day 13 (run 2026-11-17)
- **Did:** Read all 24 dev failures and grouped them (`results/day13_error_analysis.txt`). NOT DONE: reading the Hamel Husain field guide on open coding.
- **Measured:** 24 of 150 fail. **"never showed up": 0 of 13 labeled `late`**, always `wrong_item`. Other late phrasings are 25 of 29 correct. Tracking number taken as the order id: 4. Support agent taken as the customer name: 2.
- **Confused:** One phrase is worth about 9 points of the score. Is that the model's mistake or my rubric's? It is a fair label, but a different taxonomy would give a different headline number.

## Day 14 (run 2026-11-17)
- **Did:** Wrote the `README.md` (motivation, task, data, rubric, results with CIs, failure analysis, limitations, how to reproduce). The two issues I would read first are inspect_ai #6358 and lm-eval #2552. NOT DONE: making the repo public, the public post, and the hiring-manager check.
- **Measured:** README covers 6 result tables. Test split still untouched.
- **Confused:** Should this repo be public before I have hand-labeled data or run the test split? The README says plainly that the data is synthetic.

## Day 15 (run 2026-11-17)
- **Did:** Worked out KV-cache sizes with `day15/kv_cache.py`. Qwen numbers come from the local `config.json`. Llama-3.1-8B numbers use its config as I remember it (32 layers, 8 KV heads, head_dim 128), **not read from disk**. NOT DONE: reading kipply's inference arithmetic post and the Orca paper; writing the prefill vs decode explanation in my own words.
- **Measured:** KV bytes per token = 2 x layers x kv_heads x head_dim x 2 bytes. **Qwen2.5-1.5B: 28 KiB/token. Llama-3.1-8B: 128 KiB/token.** With 4096-token sequences and 90% of GPU memory usable, 8B bf16 (16 GB weights) fits **10 sequences on a 24 GB GPU and 104 on an 80 GB GPU**. Qwen 1.5B fits 157 and 586.
- **Confused:** The calculation ignores activations and fragmentation. How far off is it in practice? That is what the vLLM load test in Week 3 would show.

---

## Still to do (Days 16-30)
Weeks 3-4 need an NVIDIA GPU (vLLM, nsys, quantized variants). I have not started them.
