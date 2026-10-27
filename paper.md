# Quote the instruction: a training-format change that fixes misreading in small models

Bentley Moon · futurisminstitute@gmail.com · draft, 2026-10-22

## Abstract

Small language models fine-tuned on step-by-step worked solutions often fail on problems that contain an
instruction they never saw in training. We find that in our setting these failures are failures of *reading*, not of
computation. The model drops the unfamiliar step, or replaces it with a familiar one, while every arithmetic step
it does write is correct. A one-line change to the training format repairs most of this: each step names the
instruction it carries out ("Step 2 ('Square it'): 23 x 23 = 529"). On held-out problems with two operations absent
from training, accuracy rose from 0.63 to 0.91 for Qwen2.5-1.5B and from 0.07 to 0.48 for SmolLM-1.7B, with
no loss on familiar problems.
- **Replication:** the gain held on three new seeds (+0.26 and +0.41).
- **Unseen operations:** it extended to two operations never trained or tested (+0.16 and +0.20).
- **More data:** four times the training data without the change did not substitute for it (Qwen: 0.59
  unquoted at 1.6M tokens against 0.91 quoted at 400k; SmolLM: 0.16 against 0.48).
- **A second task:** on word-editing chains with two operations absent from training, quoting raised Qwen's
  accuracy from 0.09 to 0.33. Almost every remaining quoted failure names the right instruction and carries it out
  wrongly.

Every experiment was registered, with its predictions and scoring, before it ran. The scale is small and the task
synthetic; we state what does not follow.

## 1. Setup

**Task.** Arithmetic chains with exact answers ("Start with 7. Multiply by 3. Subtract 4. ...").
- **Training chains:** 3-5 steps using Add, Subtract and Multiply by.
- **Evaluation families, 60 problems each:**
  - *in*: new problems of the training kind.
  - *long*: 6 steps, familiar operations.
  - *newop*: 4-5 steps containing two of "Double it" / "Square it", which never appear in training.
  - *held*: 6 steps containing two of them.
  - *novel*: "Triple it" / "Negate it", never trained or tested before.

**Training.** Each record is a worked solution followed by a check of every step. Half the records start with a
correct attempt that the check confirms, and half start with one arithmetic slip that the check finds and
corrects (the format of our earlier experiment D3).
- **Models and adapter:** LoRA rank 16 on the attention projections of Qwen2.5-1.5B or SmolLM-1.7B (base models).
- **Budget:** 400,000 training tokens (also 800k and 1.6M in the data experiment), 3 seeds per condition.
- **Evaluation:** greedy decoding, up to 384 new tokens; a reply is correct when its final "Answer: N" is right.

**The change.** The only difference between conditions is how a step is written: "Step 2: 7 x 5 = 35."
(unquoted) or "Step 2 ('Multiply by 5'): 7 x 5 = 35." (quoted). Quoted steps are longer, so under the fixed
token budget the quoted models saw fewer problems (about 1,960 against 2,256 for Qwen).

## 2. The failure is reading

On the held family, most wrong answers from unquoted models contain no arithmetic error at all. Among wrong
replies to problems containing "Square it", a correct squaring step was missing in 270 of 350 (Qwen) and 589 of 656
(SmolLM). The model either dropped the step or wrote it as multiplication by a constant ("21 x 2 = 42").

Two tests separate this from a lack of skill:
- **Length is not the cause.** Six-step problems with familiar operations are solved 0.98-0.99 of the time.
- **The model can square.** Given a step already written as "square 45 =", with the true value nowhere in
  context, Qwen computes it correctly 0.90 of the time.

A check that verifies each step's arithmetic cannot see these failures, because the arithmetic it checks is right.

## 3. Results

| solve accuracy (seed means) | in | long | newop | held | all steps present (newop+held) |
|---|---:|---:|---:|---:|---:|
| Qwen2.5-1.5B, unquoted | 0.98 | 0.98 | 0.66 | 0.63 | 0.78 |
| Qwen2.5-1.5B, quoted | 1.00 | 0.98 | 0.92 | 0.91 | 1.00 |
| SmolLM-1.7B, unquoted | 0.90 | 0.61 | 0.21 | 0.07 | 0.51 |
| SmolLM-1.7B, quoted | 0.91 | 0.69 | 0.57 | 0.48 | 0.97 |

**Replication.** On seeds 3-5 the held quoted-minus-unquoted gap was +0.26 (Qwen) and +0.41 (SmolLM), against
+0.27 and +0.41 on seeds 0-2.

**Unseen operations.** On the novel family, quoting added +0.16 (Qwen, 0.77 against 0.61) and +0.20 (SmolLM,
0.41 against 0.21).

**Data does not substitute.** Held accuracy by training budget:

| | 400k | 800k | 1.6M |
|---|---:|---:|---:|
| Qwen, unquoted | 0.63-0.66 | 0.58 | 0.59 |
| Qwen, quoted | 0.91-0.92 | 0.93 | 0.93 |
| SmolLM, unquoted | 0.07-0.11 | 0.12 | 0.16 |
| SmolLM, quoted | 0.48-0.51 | 0.49 | 0.51 |

**What remains.** For quoted Qwen, 31 of 1,080 held, newop and novel solve replies were wrong; none misread an
instruction. 25 were arithmetic slips, mostly in products above 1,000. The model's own step check passed 24 of 25
of them: the checker shares the solver's arithmetic limit. A check that recomputes each product by a second route
(splitting into tens or hundreds) flagged 0.61 of such slips in a follow-up experiment.

**A second task (T).** Word-editing chains ("Start with: lamp river stone. Reverse the words. Uppercase word
2."), trained on reverse, uppercase, drop, append and swap, and tested on chains containing "Sort the words." and
"Duplicate word N.", which never appear in training. Same model, adapter, budget and seeds.

| solve accuracy (seed means) | in | held |
|---|---:|---:|
| Qwen2.5-1.5B, unquoted | 0.92 | 0.09 |
| Qwen2.5-1.5B, quoted | 0.96 | 0.33 |

In 119 of the 120 quoted held failures, every step names the instruction it was given, in order. The error is in
carrying out an operation the model never practised. Quoting mostly fixed "Duplicate word N" (95 to 42 first
errors) and not "Sort the words" (58 to 67). The lower ceiling than in arithmetic (0.33 against 0.91) fits that
reading: the arithmetic operations were familiar procedures under unfamiliar names, and these are new procedures.
Quoting fixes reading; it does not teach a procedure.

**Content, not a pointer (FMT).** Two seeds per format, same budget. A step label that only points ("Step 2
(instruction 2):") gave 0.57 on held problems, no better than no label (0.66). A one-word verb ("Step 2
(multiply):") gave 0.88, close to the full quote (0.90), but not on the novel operations (0.62 against 0.75). The
gain comes from writing the operation's name beside the work.

## 4. Registered predictions

- **Quoting (D11, D12):** all 8 registered predictions held.
- **Data and generalisation (DX):** 6 of 8 registered checks held. SmolLM's quoted held accuracy did not rise with
  data (predicted +0.10, observed +0.00 to +0.03), and a recipe combining quoting with a problem-reading check and
  masked mistakes scored 0.78 held (predicted at least 0.88).
- **Second task (T):** 2 of 3 held. The miss: unquoted failures were predicted to change the step count, the
  arithmetic signature of misreading, in at least 0.30 of cases. They did in 0.01; in text the misreading shows as a
  wrong edit inside a step that is present.
- **Format (FMT):** 3 of 5 held. Misses: the pointer gave no gain (predicted +0.10), and quoting raised obedience
  to a planted note rather than lowering it (see Limitations).
- **Corrections:** two explanations written during the work were wrong and are corrected in the record. The
  failures were first attributed to chain length, then to an inability to square.

## 5. Related work

- **Step-by-step formats:** intermediate steps written out in training or prompting are well studied, as
  scratchpads (Nye et al., 2021, arXiv:2112.00114) and chain-of-thought (Wei et al., 2022, arXiv:2201.11903).
- **Checking steps:** checking each step rather than the final answer is studied as process supervision (Lightman
  et al., 2023, arXiv:2305.20050).
- **Re-reading:** having a model read the question again at inference improves reasoning (RE2; Xu et al., EMNLP
  2024, arXiv:2309.06275). Our change is the step-local, trained version: each step re-reads the one instruction
  it executes, inside the training records. A prompt-level test in larger models (QP) is the direct comparison
  and is registered.
- **Instruction chains:** instruction tuning on chains of subtasks is studied as Chain-of-Instructions (Hayati et
  al., AAAI 2025, arXiv:2402.11532).
- **Our contribution** is narrower. A step that cites the instruction it executes repairs a reading failure that
  step-level checking cannot detect. The effect is measured against matched controls, data scaling and unseen
  operations.
- **Adapter:** LoRA is Hu et al. (2021, arXiv:2106.09685).

*(Still to check before deposit: copying and induction mechanisms in small models, and instruction following under
distribution shift.)*

## 6. Limitations

- **Scale and task:** two base models of 1.5-1.7B parameters, synthetic English arithmetic and (for one model)
  synthetic word edits, three seeds per condition, and one family of unfamiliar operations plus one novel pair. Instructions here are short and can be
  quoted verbatim; open-ended tasks, long instructions and larger models are untested.
- **Planted instructions:** quoting makes the model follow instruction-like text more faithfully, including text
  that is not an instruction. With a marked decoy note in the problem ("A margin note says step 2 should be: Add
  16"), quoted models obeyed it in 114 of 120 replies, against at least 33 unquoted. In those replies every quote
  matched the arithmetic actually done (2,875 of 2,875 steps), so comparing quotes with the real instruction list
  flagged all 116 wrong answers. That check needs to know which text is the instruction. Quoting is a record of what
  was executed, not a defence. Training on problems whose notes the solution ignores removed the obedience only for
  notes in the trained position: a note after the problem was then ignored (correct 0.99-1.00, also in a new
  wording), but one placed before the problem was obeyed 120 of 120 times. Across these tests the quote comparison
  flagged 619 of the 621 wrong replies from quoted models.
- **Unequal data:** the quoted condition saw about 13% fewer unique problems under the same token budget, which
  works against it.

## 7. Reproducibility

The pre-registrations, scoring code, result files and raw replies are available on request at the address above.
The code runs with Python, PyTorch, Transformers and PEFT on a 16 GB Apple M4 (MPS); one run takes about 1-2 hours.
