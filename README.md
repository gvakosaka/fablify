# Fablify: Packaging Claude Fable 5's Way of Working as an Agent Skill

Fablify is an Agent Skill designed to produce output closer to Claude Fable 5 with smaller models, in case Fable 5 becomes difficult to access routinely.

It does not attempt to reproduce Fable 5's underlying model capabilities. It targets **execution discipline and communication behavior**: leading with the outcome, not filling gaps with guesses, verifying results before claiming success, and using independent verification on difficult tasks.

More precisely, Fablify is not a Fable 5 emulator. It is a **Fable-like behavioral scaffold**. It cannot add knowledge or reasoning capabilities that a model does not already possess, but it can reduce failures caused by poor use of those capabilities and reproduce some of what users perceive as "a smarter model."

The skill itself is defined in [`SKILL.md`](./skills/fablify/SKILL.md).

## Install

```bash
npx skills add gvakosaka/fablify
```

## Why It Exists

The quality of LLM output depends not only on the base model's capabilities but also on behaviors such as these:

- Reading for what the user ultimately needs to accomplish, not merely the literal wording of the question
- Distinguishing known facts, tool-verified observations, and inferences
- Testing counterexamples and failure conditions instead of accepting the first idea
- Preferring observed tool results over memory when tools are available
- Exercising the changed path after modifying code
- Spending additional reasoning effort only on tasks that warrant it
- Removing explanations and AI stock phrases that do not affect the reader's decision

Larger models tend to exhibit these behaviors more consistently without explicit instructions. With smaller models, externalizing each behavior as a procedure can improve reproducibility. Fablify packages those procedures as a skill that can be loaded by agent environments such as VS Code and Claude Code.

## How It Maps to Fable 5

Anthropic's official documentation describes Claude Fable 5 as a model for complex reasoning and long-horizon agentic work. Its model-specific prompting guide separately addresses long-running execution, strong instruction following, evidence-backed progress reports, subagents, memory, and ways to prevent excessive investigation or implementation.

Fablify's central directives align closely with the Fable 5 behaviors described in that guide.

| Behavior in the official Fable 5 guide | Corresponding discipline in Fablify |
|---|---|
| Lead with the result | Put the answer or execution result in the first sentence |
| Distinguish readability from brevity | Remove irrelevant information without collapsing prose into fragments |
| Audit long-run progress against tool results | Separate verified facts, inferences, and unknowns, and report failures faithfully |
| Act once enough information is available | Begin editing or execution once there is a local hypothesis and a way to test it |
| Avoid unnecessary abstraction and cleanup | Make only the smallest changes required by the request |
| Use parallel subagents | Run independent attempts and fresh-context verification |
| Use memory for long-running work | Externalize goals, intermediate conclusions, and discarded hypotheses |
| Stop for the user only when necessary | Ask only about irreversible actions, genuine scope changes, or information only the user can provide |
| Rewrite the final report for its reader | Remove internal shorthand, arrow chains, and labels invented during the work |

This alignment suggests that Fablify is pointed in the right direction for producing Fable-like output. However, official documentation also attributes long-term instruction retention, first-shot correctness on complex problems, vision, and autonomous navigation of ambiguity to Fable 5's model capabilities. A prompt cannot reproduce those capabilities by itself.

## Four Layers Behind the 13 Directives

[`SKILL.md`](./skills/fablify/SKILL.md) contains 13 sections. By purpose, they fall into four layers.

### 1. Problem Understanding and Judgment

The model reads for how the answer will be used rather than responding only to the surface question. It identifies false premises before proceeding and considers one competing interpretation or counterexample before accepting its first proposal. When asked to choose, it gives one recommendation and the reasoning behind it instead of ending with a list of options.

This layer covers sections 1, 2, 5, and 8.

### 2. Honesty and Verification

Verified facts, evidence-based inference, and unverified guesses are kept separate. The model reads a file before describing it, does not invent command output, and exercises the changed path after an edit. When tool results contradict expectations, the results take precedence.

This layer covers sections 4, 7, 9, and 13.

### 3. Implementation and Design

Before writing code, the model reads the target, its callers, and a similar implementation. It avoids incidental refactoring, defensive handling for impossible conditions, and abstractions used only once. In design work, it identifies assumptions, open questions, failure paths, non-goals, and testable acceptance criteria.

This layer covers sections 10 and 11.

### 4. Output Quality and Reasoning Process

The model puts the outcome first and removes text that would not change a decision or action. For difficult tasks, it uses decomposition, independent parallel attempts, adversarial review, and fresh verification of factual claims. Instead of simply asking the same model to "think again," it supplies evaluation criteria or external evidence.

This layer covers sections 3, 6, and 12.

## Relationship to Published Research

Fablify is not a single prompting technique. It combines mechanisms investigated across several lines of research.

### Iterative Refinement Helps, but Self-Correction Has Limits

[Self-Refine](https://arxiv.org/abs/2303.17651) reported an average improvement of roughly 20 percentage points across seven evaluated tasks by having a model produce a draft, critique it, and revise it. Fablify's treatment of the first idea as a draft and its final check against explicit criteria are consistent with that result.

However, [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798) found that reasoning corrections made without external feedback do not reliably improve performance and can sometimes make it worse.

Fablify therefore does not rely on self-critique alone as proof of correctness. It calls for tests when changing code, primary sources when making factual claims, and concrete scenarios and acceptance criteria when evaluating a design. Self-critique is a bridge to external verification, not the verification itself.

### Independent Verification Reduces Hallucination

[Chain-of-Verification](https://arxiv.org/abs/2309.11495) reduced hallucinations across multiple tasks by generating verification questions from an initial draft and answering those questions independently, without allowing the original answer to bias the verification step.

Fablify's chain-of-verification follows the same idea. It turns claims into standalone questions and rechecks them in a context that does not include the draft's reasoning. Asking "is this correct?" while rereading the draft is likely to confirm the original answer rather than verify it.

### Independence Matters More Than Agent Count

[Multiagent Debate](https://arxiv.org/abs/2305.14325) and [Mixture-of-Agents](https://arxiv.org/abs/2406.04692) report improvements in reasoning and response evaluation from combining multiple model outputs.

However, agents generated from the same model, prompt, and context may reproduce the same errors. Fablify therefore prefers this structure over open-ended debate:

1. Generate multiple proposals in independent contexts
2. Compare where they agree and disagree
3. Give the critic the artifact and acceptance criteria, not the author's rationale
4. Check the critic's findings against evidence or executable results

Spawning a subagent is not enough to create independent verification. The important design choice is what context to share and what to withhold.

### Longer Instructions Are Not Always Better

[Lost in the Middle](https://arxiv.org/abs/2307.03172) showed that models do not use information in long contexts uniformly. Performance can vary significantly depending on where relevant information appears.

Fablify is comprehensive, but loading all 13 sections into a smaller model on every request may create instruction conflicts or dilute attention. Activating only the relevant parts based on task difficulty is likely to be more stable than continually adding more principles.

### Prompts Should Evolve Alongside Evaluations

Anthropic's guide to [defining success criteria and building evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests) recommends establishing concrete success criteria, a measurement method, and an initial prompt before attempting prompt optimization. [DSPy](https://arxiv.org/abs/2310.03714) and [OPRO](https://arxiv.org/abs/2309.03409) also support treating hand-written prompts as inputs to an evaluation-driven optimization process rather than as finished artifacts.

Fablify should likewise be measured against real tasks instead of being judged only by how reasonable its principles sound.

## Current Assessment

No benchmark has been run yet, so the scores below are qualitative assessments based on comparing `SKILL.md` with published sources. They are not empirical measurements.

| Dimension | Preliminary score | Rationale |
|---|---:|---|
| Style and information density | 8/10 | Defines criteria for removing prose, not merely a list of banned words |
| Honesty and grounding | 8/10 | Consistently prioritizes tools, labels unknowns, and audits progress claims |
| Coding discipline | 8/10 | Connects reading, minimal change, and executable verification into one procedure |
| Requirements and design | 8/10 | Requires failure paths, non-goals, and acceptance criteria |
| Substitute for deep reasoning capability | 5/10 | Process can improve results but cannot add capabilities the model lacks |
| Long-horizon autonomy | 4/10 | Depends on the context window, memory, tools, and agent harness |
| Overall reproduction of Fable 5 | 6-7/10 | Execution discipline is close, but base-model capability differences remain |

Fablify is most likely to reproduce restraint in writing, disciplined handling of facts, careful code changes, and effective verification. It is less able to reproduce discovery on unfamiliar problems, long-term instruction retention, vision, or one-shot implementation of complex systems.

## Recommended Usage

### Match Execution Intensity to the Task

An operational equivalent of Fable 5's `effort` setting makes Fablify easier to use efficiently.

| Mode | Appropriate tasks | Applied behavior |
|---|---|---|
| `routine` | Lookups and small edits | Outcome first, concreteness, verification, concise output |
| `standard` | Normal implementation, review, and research | Counterexample check, tool verification, minimal changes, final self-check |
| `deep` | Architecture, non-obvious incidents, and high-stakes decisions | Decomposition, independent parallel attempts, fresh-context verifier, external memory |

The current skill performs this triage internally. To make the desired intensity explicit, use prompts such as:

```text
/fablify
Review this PR in standard mode, focusing on bugs and regression risk.
```

```text
/fablify
Compare these authentication approaches in deep mode. Produce independent proposals,
include a threat model and recovery procedure for a failed migration, then recommend one.
```

Using `deep` mode for a simple question is more likely to increase latency and token consumption than improve the answer.

### Supply External Verification

Fablify is a skill for reasoning carefully, but it works better when verification material is available.

- Code changes: test commands, reproduction steps, and expected inputs and outputs
- Research: preferred primary sources, cutoff date, and relevant versions
- Design: concrete usage scenarios, load conditions, non-goals, and acceptance criteria
- Writing: intended audience, publication medium, good examples, and examples to avoid

A falsifiable instruction such as "try to break your proposal against these conditions" is more useful than a generic request to "be careful."

### Preserve Fable 5 Examples as Few-Shot Material

While Fable 5 is still available, preserve strong real-world responses by task category. Three to five examples each for research, code review, design decisions, and short answers is a practical starting point.

Store more than the writing style:

- The original request and context
- Fable 5's final artifact
- The decisions or structure that made it effective
- Omissions or corrections it still required
- The scope of tasks to which the example applies

Extracting a rubric from why an example worked transfers better across models and tasks than asking another model to imitate the prose verbatim.

### Accumulate Feedback in Memory

Record concrete feedback such as "the explanation was too long," "this level of confidence was appropriate," or "this verification step was unnecessary" as short memory entries. A user's correction history will usually affect quality in their environment more directly than additional generic prompting principles.

Update existing rules rather than repeatedly appending duplicates, and remove rules that prove wrong or obsolete.

## Evaluation Method

An impression that output "feels like Fable 5" is not enough to measure improvement. At minimum, run this A/B evaluation:

1. Collect 30-50 tasks from previous real-world work
2. Include research, implementation, review, design, and short questions
3. Run the same smaller model multiple times with and without Fablify
4. If possible, include Fable 5 output as a reference condition
5. Blind the model identity and grade all outputs against one rubric
6. Record regressions separately from improvements

Keep at least these dimensions separate:

| Dimension | What to measure |
|---|---|
| Correctness | Whether facts, calculations, and code behavior are correct |
| Grounding | Whether claims are tied to evidence without invented details |
| Completion | Whether the requested work was finished instead of stopping at a proposal |
| Scope control | Whether unnecessary changes or topics were introduced |
| Actionability | Whether the reader can identify the next action |
| Writing quality | Whether the outcome comes first without stock phrases or excessive structure |
| Cost | Whether token usage, tool calls, and latency are acceptable |

Do not use lexical similarity to Fable 5 as the primary metric. Evaluate whether the output reaches the same decision, identifies the same important facts, performs comparable verification, and is comparably readable. Because a model grading its own output may reproduce correlated preferences, use code-based grading where possible and confirm subjective judgments with humans or a different model.

## Limitations and Caveats

### A Prompt Cannot Add Learned Capabilities

Fablify cannot teach a model an API it does not know or make it solve a reasoning problem beyond its capabilities. Some tasks will still require retrieval, tools, additional material, or a stronger model.

[Distilling Step-by-Step](https://arxiv.org/abs/2305.02301) is relevant research on bringing smaller models closer to larger ones by using rationales as training supervision. That is task-specific model training, not prompting, and is therefore a different approach from the general-purpose Fablify skill.

### Self-Checks Can Become Rituals

If the section 13 checklist is printed on every response, it may create the appearance of verification without performing any. Replace checklist items with executable tests or evidence wherever possible.

### Subagents Are Not Fully Independent

Multiple calls to the same model with the same settings retain correlated error patterns even when their contexts are separated. For important decisions, combine different models, independent information sources, code-based verification, and human review.

### Long Skills Compete with Other Instructions

Project-specific coding conventions, security policies, and explicit user requirements take precedence over Fablify. If Fablify's writing rules conflict with the required format of a deliverable, the deliverable's format wins.

## Conclusion

Fablify does not reproduce Claude Fable 5's capabilities in a smaller model. It does, however, capture much of Fable 5's behavior around leading with conclusions, handling facts, controlling scope, verifying work, and sustaining longer tasks.

The next improvement should not be another layer of general principles. The priorities are a compact core with difficulty-specific modules, few-shot examples collected from Fable 5, and A/B evaluation on real tasks. With those pieces, Fablify can move from a plausible long prompt to a maintainable agent quality layer whose effect can be measured.

## References

- Anthropic, [Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5)
- Anthropic, [Introducing Claude Fable 5 and Claude Mythos 5](https://platform.claude.com/docs/en/about-claude/models/introducing-claude-fable-5-and-claude-mythos-5)
- Anthropic, [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- Madaan et al., [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651)
- Huang et al., [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798)
- Dhuliawala et al., [Chain-of-Verification Reduces Hallucination in Large Language Models](https://arxiv.org/abs/2309.11495)
- Du et al., [Improving Factuality and Reasoning in Language Models through Multiagent Debate](https://arxiv.org/abs/2305.14325)
- Wang et al., [Mixture-of-Agents Enhances Large Language Model Capabilities](https://arxiv.org/abs/2406.04692)
- Liu et al., [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
- Khattab et al., [DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines](https://arxiv.org/abs/2310.03714)
- Yang et al., [Large Language Models as Optimizers](https://arxiv.org/abs/2309.03409)
- Hsieh et al., [Distilling Step-by-Step!](https://arxiv.org/abs/2305.02301)

Researched and written: 2026-07-21
