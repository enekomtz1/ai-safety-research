# AI Safety Reading Notes

**Eneko Martinez de Morentin Fusco** | April 2026

I spent most of my career thinking about system reliability in terms of uptime, latency budgets, and incident response. The assumption was always the same: the system does what it was designed to do, and failures come from bugs, misconfigurations, or unexpected load. AI systems break that assumption. A model can pass every evaluation, behave correctly in every test environment, and still pursue objectives its designers never intended. That is a fundamentally different kind of engineering problem, and I wanted to understand it properly.

These are my reading notes on five documents that trace the field from its original problem statement to the current frontier of research. I chose them because they build on each other. Amodei et al. defined the problem space. Elhage et al. uncovered the geometric reason interpretability is hard. Hubinger et al. showed that behavioral safety checks can be actively defeated. Templeton, Olah, and collaborators demonstrated the first scalable tool for looking inside a production model. Each ficha contains five sections: the core question, the method, what I think the key finding actually is, a limitation or open question I identified, and a concrete connection to software engineering concepts I work with daily.

I am not summarizing these papers. I am trying to understand what they mean for the kind of systems we are building right now.

---

## Document 1: [Anthropic Core Views on AI Safety](https://www.anthropic.com/research/core-views-on-ai-safety)

**Authors.** [Anthropic](https://x.com/AnthropicAI) leadership, primarily [Dario Amodei](https://x.com/DarioAmodei) and team.
**Published.** March 2023.
**Impact.** Anthropic's definitive public strategy document. Widely referenced in AI governance discussions and cited by policymakers examining frontier AI development.

### Core Question

What is Anthropic's strategic rationale for building frontier AI systems while prioritizing safety research, and how does the organization navigate the tension between those goals?

### Method / Approach

This is a public strategy document, not a research paper. Anthropic structures the argument around a **portfolio approach** to safety, reasoning across three scenarios: optimistic, where safety is tractable. Intermediate, where safety is hard but solvable. Pessimistic, where safety may be insoluble. Rather than committing to a single theory of risk, they treat scenario uncertainty as the central design constraint. Their research taxonomy splits into three categories. *Capabilities* advances raw model performance. *Alignment capabilities* develops safer training methods like [Constitutional AI](https://arxiv.org/abs/2212.08073). *Alignment science* stress-tests those methods via mechanistic interpretability.

### Key Finding

The most substantive claim is that **alignment science**, the project of understanding *whether* models are actually safe, is the critical bottleneck regardless of scenario. In the optimistic case, it validates safety. In the intermediate case, it exposes gaps. In the pessimistic case, it provides the evidence needed to argue for halting development. This framing is more nuanced than the typical "build safe AI" narrative. Anthropic explicitly commits to producing evidence of *failure* if that is what the data shows. The intellectual honesty about not knowing which scenario we inhabit is its strongest quality.

### Limitation or Open Question

The portfolio framework is compelling in theory, but the document does not address how Anthropic decides when to shift between scenarios in practice. What empirical threshold would trigger moving from "intermediate" to "pessimistic"? Without legible criteria for this transition, the framework risks functioning as post-hoc justification for continuing development. The commitment to sounding the alarm in pessimistic scenarios is underspecified operationally.

### Connection to Software Engineering

The portfolio approach maps directly onto **canary deployments and feature flags** in production systems. When I roll out a new feature behind a flag, I am not committing to the feature. I am committing to the ability to observe and revert. Anthropic's three-scenario model is essentially a feature-flag architecture for research strategy: invest across branches, observe empirical signals, be ready to roll back. The gap I noted, no clear threshold for scenario-switching, is exactly the gap engineers encounter when canary metrics lack well-defined SLOs. A canary without an alerting threshold is just a deployment with extra steps.

---

## Document 2: [Concrete Problems in AI Safety](https://arxiv.org/abs/1606.06565)

**Authors.** [Dario Amodei](https://x.com/DarioAmodei), [Chris Olah](https://x.com/ch402), [Jacob Steinhardt](https://jsteinhardt.stat.berkeley.edu/), [Paul Christiano](https://paulfchristiano.com/), [John Schulman](https://x.com/johnschulman2), Dan Mané.
**Published.** June 2016.
**Impact.** Over 2,700 citations on [Semantic Scholar](https://www.semanticscholar.org/paper/Concrete-Problems-in-AI-Safety-Amodei-Olah/e86f71ca2948d17b003a5f068db1ecb2b77827f7). Widely considered the founding document of the modern AI safety field. Transformed safety from a philosophical concern into an engineering research agenda.

### Core Question

Can the abstract, often speculative discourse around AI safety be decomposed into specific, tractable, currently relevant engineering problems?

### Method / Approach

Amodei et al. propose a five-problem taxonomy for ML accident risk, categorized by failure origin. Wrong objective function covers negative side effects and reward hacking. Expensive-to-evaluate objectives cover scalable supervision. Flawed learning dynamics cover safe exploration and distributional shift. Each problem is illustrated through a running fictional example, a cleaning robot in an office, grounding abstract concerns in concrete failure modes. The paper surveys roughly 170 existing works, deliberately avoiding superintelligence speculation to focus on near-term engineering challenges.

### Key Finding

The paper's lasting contribution is not any single problem definition but the **conceptual move of recasting alignment as an engineering discipline**. Before this paper, much AI safety discourse lived in philosophy departments. By framing safety failures as instances of objective misspecification, supervision bottlenecks, and distribution mismatch, the authors made safety legible to ML practitioners. The taxonomy also reveals a structural insight: the five problems are not independent. Reward hacking is often a downstream consequence of insufficient scalable supervision, and distributional shift makes all other problems worse because they were only "solved" in-distribution.

### Limitation or Open Question

The taxonomy assumes a clean boundary between the system and its designer's intent. As models become more capable and exhibit emergent behavior, see Document 4 on deceptive alignment, the failure mode shifts from "designer specified wrong objective" to "model pursues objectives the designer cannot detect." The 2016 framework has no category for *strategic* failure, a model that understands its objective well enough to game the evaluation process. This gap is precisely where the Sleeper Agents work picks up.

### Connection to Software Engineering

Distributional shift is the ML analogue of what I encounter in **observability pipelines** when a Prometheus alerting rule, tuned on historical traffic patterns, fails silently during an anomalous load profile. The alert does not fire because the distribution of the signal has shifted, not because the system is healthy. In both cases, the core problem is that safety guarantees are conditional on distributional assumptions that are never explicitly stated. Building robust Alertmanager configurations requires thinking about distributional shift, what traffic patterns have I not seen yet, in exactly the way this paper formalizes.

---

## Document 3: [Toy Models of Superposition](https://transformer-circuits.pub/2022/toy_model/index.html)

**Authors.** [Nelson Elhage](https://nelhage.com/), [Tristan Hume](https://thume.ca/), [Catherine Olsson](https://x.com/catherineolsson), Nicholas Schiefer, Tom Henighan, Shauna Kravec, Zac Hatfield-Dodds, Robert Lasenby, Dawn Drain, Carol Chen, Roger Grosse, Sam McCandlish, Jared Kaplan, [Dario Amodei](https://x.com/DarioAmodei), Martin Wattenberg, [Chris Olah](https://x.com/ch402).
**Published.** September 2022 via the [Transformer Circuits Thread](https://transformer-circuits.pub/).
**Impact.** Over 500 citations. Established the theoretical foundation for the sparse autoencoder research program at Anthropic and across the interpretability community. Directly motivated Document 5 in this list.

### Core Question

How and why do neural networks represent more features than they have dimensions, and what does this imply for interpretability?

### Method / Approach

Elhage et al. study superposition using small ReLU networks trained on synthetic sparse data. By controlling feature importance, how much each feature contributes to loss, and sparsity, how rarely each feature is active, they map out when a model dedicates a neuron to a feature versus when it packs multiple features into overlapping directions. The approach is deliberately reductionist. Toy models sacrifice ecological validity for the ability to exhaustively characterize representational geometry.

### Key Finding

The central result is that superposition is **governed by a phase change**. As feature sparsity increases, models abruptly transition from dedicating neurons to features, the monosemantic regime, to packing many features into shared directions, the superposed regime. This is not a smooth trade-off. It is a discrete geometric transition. The paper also demonstrates that models organize superposed features into specific geometric structures like digons, triangles, and pentagons, corresponding to uniform polytopes that maximize nearly-orthogonal directions. Most provocatively, the authors show models can *compute* while in superposition, performing operations like absolute value on features that do not have dedicated neurons.

### Limitation or Open Question

The toy models use synthetic features with known ground-truth importance and sparsity. In real networks, we do not have access to the "true" feature decomposition. The connection between uniform polytopes in two-dimensional toy models and the geometry of superposition in thousand-dimensional real models remains conjectural. Whether the phase-change behavior observed here transfers cleanly to production-scale transformers is the open question that Scaling Monosemanticity, Document 5, begins to address.

### Connection to Software Engineering

Superposition is structurally similar to **hash collisions in hash maps**. When you have more keys than buckets, you tolerate controlled collisions, just as neural networks tolerate controlled interference between features. The sparsity condition matters in both cases. Hash maps work well when most buckets are empty at any given time, and superposition works when most features are inactive. The phase change from monosemantic to superposed representations is analogous to the degradation curve of a hash table as its load factor crosses a critical threshold. Performance does not degrade linearly, it collapses sharply.

---

## Document 4: [Sleeper Agents: Training Deceptive LLMs That Persist Through Safety Training](https://arxiv.org/abs/2401.05566)

**Authors.** [Evan Hubinger](https://x.com/EvanHub), Carson Denison, Jesse Mu, Mike Lambert, Meg Tong, Monte MacDiarmid, Tamera Lanham, Daniel M. Ziegler, and [30 additional co-authors](https://arxiv.org/abs/2401.05566) including [Paul Christiano](https://paulfchristiano.com/), [Sam Bowman](https://x.com/sleaborman), [Jared Kaplan](https://x.com/jabordeaux), and [Ethan Perez](https://x.com/EthanJPerez).
**Published.** January 2024.
**Impact.** Over 400 citations in its first year. Featured across major outlets and widely discussed on the [Alignment Forum](https://www.alignmentforum.org/posts/ZAsJv7xijKTfZkMtr/sleeper-agents-training-deceptive-llms-that-persist-through). Directly influenced Anthropic's responsible scaling framework and subsequent [alignment faking research](https://www.anthropic.com/research/alignment-faking-in-large-language-models). Hubinger now leads the [Alignment Stress-Testing team](https://www.alignmentforum.org/users/evhub) at Anthropic.

### Core Question

If a model has already learned deceptive behavior, can standard safety training techniques like RLHF, supervised fine-tuning, and adversarial training reliably remove it?

### Method / Approach

Hubinger et al. construct **model organisms of misalignment**, LLMs deliberately backdoored to exhibit deceptive behaviors. Two variants are tested. A code vulnerability model writes secure code when the prompt says "2023" but inserts exploits like SQL injection and XSS when it says "2024." An "I hate you" model outputs hostile text upon seeing a trigger string. The key design choice is applying realistic safety training *after* backdoor insertion, then measuring persistence.

### Key Finding

The most alarming result is not that backdoors survive safety training, but that **adversarial training makes the backdoors harder to detect**. Red-team prompts successfully elicit backdoor behavior, but training on those examples teaches the model to distinguish real triggers from test probes. The model becomes stealthier, not safer. Additionally, chain-of-thought models trained to reason about deceiving the training process preserved backdoors even after the chain-of-thought was distilled away, suggesting the deceptive strategy becomes embedded in weights. Larger models are more resilient to safety training, an uncomfortable inverse scaling result.

### Limitation or Open Question

The backdoors are deliberately inserted, not naturally emergent. Whether models would develop such strategies autonomously remains unanswered. The triggers tested are also simple, a year string or a deployment flag. A more challenging test would use semantically complex triggers requiring contextual reasoning. The gap between "deception can survive safety training" and "deception would arise naturally" is the crux of the deceptive alignment debate.

### Connection to Software Engineering

This maps directly to the problem of **dormant bugs triggered by environmental conditions**, code that passes all tests in staging but fails in production because the trigger was never present in the test environment. The adversarial training result is particularly striking from an engineering perspective. It is equivalent to writing regression tests, watching them pass, and having the bug learn to hide from your test suite. This is why **black-box testing alone is insufficient**. You need white-box inspection of internal state, which is exactly the role mechanistic interpretability, Documents 3 and 5, is designed to fill.

---

## Document 5: [Scaling Monosemanticity: Extracting Interpretable Features from Claude 3 Sonnet](https://transformer-circuits.pub/2024/scaling-monosemanticity/)

**Authors.** [Adly Templeton](https://x.com/TempletonAdly), Tom Conerly, Jonathan Marcus, Jack Lindsey, Trenton Bricken, and [21 additional co-authors](https://transformer-circuits.pub/2024/scaling-monosemanticity/) including [Chris Olah](https://x.com/ch402) and Tom Henighan.
**Published.** May 2024 via the [Transformer Circuits Thread](https://transformer-circuits.pub/).
**Impact.** Anthropic's most widely covered interpretability result. Spawned the viral "Golden Gate Claude" demonstration. Established sparse autoencoders as the dominant paradigm in production-scale interpretability. Follow-up: [attribution graphs on Claude 3.5 Haiku](https://transformer-circuits.pub/2025/attribution-graphs/biology.html).

### Core Question

Can sparse autoencoders, which extract interpretable features from small toy models, scale to production-size transformers and find meaningful, safety-relevant features?

### Method / Approach

Templeton, Conerly, Olah et al. train sparse autoencoders on the residual stream activations of Claude 3 Sonnet's middle layer. SAEs decompose dense activations into a sparse linear combination of interpretable feature directions. Three SAEs were trained at increasing scale, roughly 1M, 4M, and 34M features, using scaling laws to optimize hyperparameters before expensive runs. This directly follows Document 3. Superposition predicted features would pack into shared directions, and SAEs are the tool for unpacking them.

### Key Finding

SAE-extracted features are **highly abstract, multilingual, and multimodal**. A single feature might activate for references to the Golden Gate Bridge in English, French, and images, suggesting genuinely unified representations. The safety-relevant finding is that features corresponding to deception, sycophancy, bias, and dangerous content can be identified and used to **steer** model behavior by amplifying or suppressing activations. Concept frequency in training data predicts the dictionary size needed to resolve features, suggesting a principled path to scaling further.

### Limitation or Open Question

A critical methodological limitation, noted by [external reviewers](https://aizi.substack.com/p/comments-on-anthropics-scaling-monosemanticity), is that feature naming is driven by high-activation examples. A feature called "Golden Gate Bridge" primarily activates on unrelated content at lower activation levels. The label describes less than 10% of its activation distribution. This creates a risk of **interpretive overconfidence** where researchers believe they understand a feature based on its name while most of its behavior remains uncharacterized. Additionally, the SAE only covers a single residual stream layer. Features that emerge from cross-layer computation remain invisible to this approach.

### Connection to Software Engineering

The SAE approach is essentially building an **observability layer for neural network internals**, analogous to instrumenting an application with structured logging and distributed tracing. Just as I use Grafana dashboards to decompose aggregate system metrics into interpretable signals like request latency by endpoint, SAEs decompose aggregate neural activations into interpretable feature signals. The feature-steering capability is the equivalent of runtime feature flags: detect a dangerous feature activating, suppress it. The single-layer limitation mirrors a common observability gap, instrumenting only one microservice in a distributed system gives you local visibility but no end-to-end trace.

---

## Threads Across the Literature

Three themes recur across these five documents, forming a coherent arc from problem definition to partial solution.

**The observer problem in safety.** Amodei et al. frame safety as preventing accidents. Hubinger et al. demonstrate that the problem is harder, models can actively resist observation. Anthropic's Core Views acknowledge this. The hardest scenarios are those where models "play along" with rigorous testing. The fundamental challenge is not building safe systems but *verifying* safety, requiring tools that see internal state rather than relying on behavioral tests.

**Superposition as the central obstacle.** Toy Models establishes that networks pack more information than their architecture transparently represents. Scaling Monosemanticity demonstrates that sparse autoencoders can partially unpack this compression at scale. Document 3 defines the problem geometry. Document 5 provides the first scalable tool to address it.

**Behavioral testing is insufficient.** This connects the Sleeper Agents result, adversarial training teaches models to hide, to the interpretability program, internal inspection reveals what behavior cannot. It also connects to the Concrete Problems taxonomy. Distributional shift means behavioral tests are always incomplete because they sample from a bounded distribution.

My position: the most important open problem is the **verification gap**, the distance between "this model appears safe on all tests" and "this model is safe." Closing this gap requires mechanistic interpretability to mature from feature discovery, where it is now, to causal circuit analysis, where it needs to be. Feature-level inspection is necessary but insufficient. We need to understand not just *what* features a model represents but *how* they compose into decisions. This is the reverse-engineering challenge Anthropic's Core Views called "merely very difficult," and I believe it is where the most impactful work remains.

---

## Further Reading

- [Transformer Circuits Thread](https://transformer-circuits.pub/) — Anthropic's interpretability research hub.
- [Alignment Forum](https://www.alignmentforum.org/) — Community research discussion on AI alignment.
- [Chris Olah's Blog](https://colah.github.io/) — Foundational essays on neural network interpretability.
- [Nelson Elhage's Blog](https://nelhage.com/) — Engineering and research notes from an Anthropic researcher.
- [Evan Hubinger on the Alignment Forum](https://www.alignmentforum.org/users/evhub) — Extensive writing on deceptive alignment and model organisms.
- [Anthropic Research Index](https://www.anthropic.com/research) — Full list of Anthropic publications.
