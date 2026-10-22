# From Egocentric Video to WAM Training Data

## Designing an eligibility and routing contract

**Document type:** Operational design case study  
**Status:** Proposed operating model; not an empirical evaluation  
**Scope:** Egocentric video used for World Action Model (WAM) development

## Executive summary

Egocentric recordings are often treated as though they become useful training data after passing one quality gate. That framing is too coarse for World Action Models. The same interval may be suitable for learning visual dynamics, unsuitable for metric action supervision, useful for evaluating recovery behavior, and irrelevant to a particular target task—all at the same time.

This case study proposes a data operating model built around **use-specific eligibility**. Instead of assigning an episode a universal quality score, the operation divides it into physically meaningful `transition_packet` records and evaluates each packet against four downstream contracts:

1. video-dynamics learning;
2. task-matched human-to-robot transfer;
3. instrumented action supervision, split into metric-action and proxy-sensor contracts;
4. counterfactual and recovery evaluation.

Each contract asks for different evidence. A failed attempt may be dangerous if presented as expert action, but valuable for learning possible futures. An RGB clip without calibrated motion cannot support metric wrist supervision, but it may still show a complete and useful state transition. A visually clean action may still be irrelevant to a target task because the starting state or physical mechanics differ.

The proposed workflow therefore separates three questions:

- **What happened in the world?**
- **What can the available evidence honestly supervise?**
- **What operational action should follow: release, route, hold, reprocess, recollect, or reject?**

This is a design case study, not a claim that the policy improves model performance. Its contribution is a concrete decision system a WAM data team could implement, audit, and later evaluate.

## 1. Why the conventional quality gate is insufficient

Traditional imitation-learning pipelines have a relatively clear target: retain successful demonstrations paired with trustworthy robot actions. That makes success, synchronization, and action validity natural admission criteria.

WAM pipelines complicate this assumption. Human video can enter the system without becoming a robot-action label. DreamZero reports a human-to-robot experiment in which 72 task-matched human trajectories—12 minutes across nine tasks—contributed only to the video-prediction objective. The paper reports improved task progress on those tasks after co-training, but does not claim that arbitrary wearable footage transfers or that human action labels were recovered from it. [DreamZero paper, §5 Q4](https://arxiv.org/abs/2602.15922)

DreamDojo demonstrates a different mixture. Its human-video sources include a small instrumented collection, EgoDex, and a much larger crowdsourced video corpus. It uses different source mixtures for world-model pretraining and latent-action learning. That is evidence that “human video” is not one homogeneous asset class and that eligibility depends on the intended training stage. [DreamDojo paper, §§3.2–3.3](https://arxiv.org/abs/2602.06949)

EMG-1 makes the operational distinction more concrete. Its 307-hour field collection targets natural manipulation, while a separate 10.4-hour laboratory collection uses optical motion capture for independent pose evaluation.

It grades EMG quality separately from complete multimodal quality. When its vision pipeline cannot produce a reliable pose, it retains the gap and the aligned EMG instead of treating an uncertain prediction as a measured label.

EMG-1 also states that wrist EMG is not a direct measurement of pose, contact, or force. It therefore supports a proxy-sensor contract, not automatic metric-action eligibility. [EMG-1 article](https://encord.com/data-labs/introducing-emg-1/)

These systems motivate a broader operational principle:

> Data quality is not an intrinsic scalar property of a recording. It is a relationship between evidence, transformation history, and an explicitly declared use.

A universal keep/drop decision obscures that relationship. It can discard useful evidence, admit unsupported supervision, and make later failures difficult to trace.

## 2. The atomic unit: a complete physical transition

The proposed unit of data is not the source episode, an arbitrary fixed window, or an annotated verb. It is a **transition packet**:

```text
observable precondition
→ approach or intentional intervention
→ contact or causal event
→ observable consequence
→ recovery or stable postcondition
```

This structure matters because a WAM is expected to represent how the world changes under an intervention. A crop containing only a hand moving toward a drawer does not establish whether the drawer opened. A crop beginning after contact does not show the state from which the transition began. A crop ending at the first successful frame cannot reveal whether the result remained stable or was immediately undone.

Ego4D's Hands and Objects benchmark provides a useful vocabulary—precondition, point of no return, contact, and postcondition—even though its annotations and selection process were designed for its own benchmark rather than this proposed operation. [Ego4D Hands and Objects](https://ego4d-data.org/docs/benchmarks/hands-and-objects/)

The packet is a logical record layered over immutable source video. It does not require destructive clipping. Every derived interval retains its source episode, original timestamps, omitted ranges, camera identity, and processing version.

An illustrative record is:

```yaml
packet_id: source_17/episode_042/transition_006
source:
  episode_id: episode_042
  parent_start_s: 81.40
  parent_end_s: 89.75
transition:
  requested_change: open_drawer
  precondition_visible: yes
  intervention_visible: yes
  consequence_visible: yes
  outcome: failed_attempt
  recovery_contiguous: yes
eligibility:
  video_dynamics: eligible
  task_matched_transfer: eligible
  instrumented_action:
    metric_action: unknown
    proxy_sensor: ineligible
  counterfactual_recovery: eligible
decision:
  disposition: route
  reason_codes:
    - COMPLETE_FAILED_TRANSITION
    - RECOVERY_VISIBLE
    - METRIC_CALIBRATION_UNKNOWN
    - PROXY_SENSOR_CHANNEL_ABSENT
```

Existing annotation systems can populate observed actions, objects, outcomes, mistakes, and recoveries. The new operational work begins after those observations exist: constructing complete packets, checking the evidence required by each consumer, and preserving the reasoning behind every routing decision.

## 3. Four downstream contracts

### 3.1 Video dynamics

This lane learns or evaluates observable changes in the visual world. Its minimum evidence is temporal continuity across a physical intervention and its consequence. Robot actions, calibrated hand poses, and successful completion are not universal requirements.

Useful examples may include:

- successful state changes;
- misses and slips;
- retries and recoveries;
- stable holds that demonstrate persistence of a state;
- alternative outcomes from similar starting conditions.

Typical disqualifiers include a cut over the causal moment, an invisible task object, severe corruption during contact, or an outcome that occurs outside the recorded interval.

### 3.2 Task-matched video-only transfer

This lane is stricter about semantic and mechanical relevance. Matching the verb is insufficient. “Open” can describe a hinged door, sliding drawer, zipper, bottle cap, or software menu. These interactions have different preconditions, contact patterns, constraints, and consequences.

Eligibility therefore depends on:

- the target object's relevant state;
- the intended physical change;
- compatible interaction mechanics;
- visibility of intermediate and final consequences;
- a viewpoint from which the transition is learnable.

A high-quality recording of the wrong mechanics should not enter merely because its caption shares a verb with the target task.

### 3.3 Instrumented action supervision

This lane has two subcontracts. They must not be collapsed because a sensor can be action-relevant without measuring an action variable directly.

#### 3.3.1 Metric action or retargeting

This lane carries the strongest measurement contract. If the data product claims metric motion, the operation must know what point was measured, in which coordinate frame, at what scale, with what synchronization, and under which validity conditions.

Eligibility can require:

- calibrated sensors or a validated reconstruction process;
- known coordinate frames and units;
- timestamp alignment between video and pose;
- confidence and missing-data semantics;
- visibility or instrumentation sufficient for the claimed body point;
- a documented mapping from human motion to the target action representation.

Plain RGB footage should not be upgraded to metric supervision merely because a pose estimator returns coordinates. A packet rejected here can remain eligible for the other lanes.

#### 3.3.2 Proxy-sensor supervision

Proxy sensors capture signals related to action without directly measuring pose, contact, or force. EMG is one example. Its contract must preserve sensor placement, configuration, sampling, contact quality, timing, participant, and session.

EMG-1 illustrates why this distinction matters. Field EMG is sampled at 2,133 Hz while its cameras run near 30 FPS. The project uses shared gyroscope motion to estimate alignment and applies separate signal and multimodal quality grades.

The article reports that fused EMG and camera estimates outperform its camera reference for fingertip tracking. But the fusion uses a camera branch, laboratory references, and future EMG, so it is an offline annotation result—not direct force sensing or real-time control.

A proxy packet may be eligible for learning articulation or bridging visual gaps while remaining ineligible for contact, force, metric pose, or retargeting claims. The validated target and unsupported claims must both be explicit.

### 3.4 Counterfactual and recovery evaluation

This lane preserves authentic alternatives to clean expert completion. DreamDojo includes counterfactual evaluation cases such as reaching toward an object and missing it, while HoloAssist explicitly records mistakes, interventions, and corrections during physical tasks. These sources motivate evaluation of alternate futures, although neither establishes a universal training mixture for failures. [DreamDojo paper, §4.1](https://arxiv.org/abs/2602.06949) [HoloAssist paper](https://openaccess.thecvf.com/content/ICCV2023/papers/Wang_HoloAssist_an_Egocentric_Human_Interaction_Dataset_for_Interactive_AI_Assistants_ICCV_2023_paper.pdf)

Eligibility requires an authentic attempt, an observable outcome, and sufficient continuation to distinguish failure, correction, abandonment, or delayed success. A camera dropout masquerading as a failed attempt is a capture defect, not a counterfactual example.

## 4. The routing decision

Each transition packet receives four independent top-level lane decisions. Instrumented action then receives separate `metric_action` and `proxy_sensor` verdicts.

- `eligible`: the required evidence is present;
- `ineligible`: a known requirement is violated;
- `unknown`: the available evidence cannot resolve the requirement.

The packet then receives an operational disposition:

| Disposition | Meaning |
|---|---|
| `release` | Meets the declared contract and can enter the target data product |
| `route` | Useful, but for a different lane or qualified subset |
| `hold` | Decision requires missing metadata or specialist review |
| `reprocess` | Source evidence is sufficient, but a derived artifact must be regenerated |
| `recollect` | Required evidence never existed or cannot be recovered |
| `reject` | No supported use remains within the current program |

This distinction prevents two common errors. First, a packet is not globally rejected just because it cannot support metric action. Second, a failed action is not released as an unqualified expert demonstration merely because it contains useful visual dynamics.

Reason codes make decisions inspectable. `MISSING_CONSEQUENCE`, `CAPTURE_DROPOUT_AT_CONTACT`, and `METRIC_CALIBRATION_UNKNOWN` describe different deficiencies and lead to different remedies. Free-text notes can supplement them, but should not replace a controlled vocabulary.

## 5. Failure is not the same as defective data

Data operations must distinguish:

1. **task failure:** the attempted intervention did not achieve the intended result;
2. **capture failure:** the recording failed to preserve what occurred;
3. **processing failure:** a transformation corrupted otherwise available evidence;
4. **contract failure:** the evidence is intact but insufficient for the proposed supervision claim.

These cases are operationally different.

DROID released unsuccessful episodes alongside successful demonstrations. DreamZero's published DROID conversion uses `--filter-failed` and applies precomputed non-idle ranges for its training preparation. Neither choice supplies a universal answer: releasing failures does not make them expert demonstrations, and filtering them for one recipe does not prove that they lack value for video dynamics, counterfactual evaluation, or recovery analysis. [DROID dataset documentation](https://github.com/droid-dataset/droid/blob/main/docs/the-droid-dataset.md) [DreamZero DROID conversion guide](https://github.com/dreamzero0/dreamzero/blob/main/docs/DROID_CONVERSION.md)

The proposed operation preserves the source and asks a narrower question for each downstream use. It can mask failed action labels while retaining the visual transition, or place a complete failed attempt into a challenge set while excluding a dropout-corrupted attempt entirely.

## 6. End-to-end operating workflow

The operating sequence is:

```text
Register source and rights
→ preserve immutable episode and timestamps
→ identify candidate physical transitions
→ assemble transition packets
→ verify packet completeness
→ issue four top-level lane decisions and applicable subcontracts
→ assign disposition and reason codes
→ publish lane-specific manifests
→ review mixture coverage and concentration
→ issue targeted collection or reprocessing orders
```

### Source registration

The operation records dataset, participant or collector grouping, site, device, camera arrangement, rights restrictions, and processing version. These fields remain attached through every transformation.

### Candidate discovery

Existing action and event annotations identify likely transitions. Discovery is intentionally recall-oriented: a later eligibility review decides whether the interval is complete and useful.

### Packet assembly

The system extends or adjusts candidate boundaries to preserve precondition, causal event, consequence, and recovery. It records every retained and omitted source-time range rather than pretending that concatenated segments were adjacent.

### Eligibility review

Deterministic checks verify metadata, timing, and known instrumentation requirements. Human review resolves semantic mechanics, visible state changes, ambiguous outcomes, and recovery continuity.

### Manifest publication

Each lane receives a versioned manifest, not a copied and disconnected video archive. The manifest expresses eligibility, masks unsupported channels, records sampling eligibility, and links back to immutable source time.

### Portfolio review

The team reviews coverage over physical transition tuples:

```text
starting state × intended change × interaction mode
× outcome × recovery × conditions
```

Counts remain segmented by source, collector, device, and processing version. Otherwise, a pooled total can hide that one person or dataset supplies nearly every rare recovery.

### Collection feedback

A missing tuple becomes a precise request. “Collect more drawer opening” is replaced by “collect failed first pulls on partially obstructed drawers, including the corrective regrasp and stable final state, from a continuous egocentric view.”

## 7. Worked design questions

The companion worked examples apply the policy to representative situations rather than claiming measured frequencies. They are intended to test whether the proposed contract produces coherent decisions.

Examples include:

- a clean success eligible for dynamics and task transfer but not metric action;
- a miss followed by recovery that belongs in counterfactual evaluation;
- a failure whose causal moment was lost to dropped frames;
- a clip with undocumented metric pose but separately valid proxy-sensor evidence;
- a stable hold likely to be removed by a low-motion filter;
- a mechanically mismatched task sharing the same language label;
- a derived crop that ends before the postcondition;
- an alternative valid strategy that should not be mislabeled as deviation.

The purpose of these examples is not to simulate statistics. It is to expose contradictions before implementing the workflow at scale.

## 8. Governance and auditability

The routing contract must be versioned. A decision records:

- policy version;
- reviewer or automated check version;
- evidence used;
- eligibility verdicts;
- reason codes;
- final disposition;
- override history;
- source and derivative identifiers.

Policy changes should create a new manifest rather than silently rewriting the archive. A bad pose frame should remove metric eligibility without deleting valid RGB or EMG evidence.

Likewise, degraded electrode contact should revoke proxy-sensor eligibility without changing a packet's video-dynamics verdict.

Unknown is a legitimate verdict. Forcing ambiguity into eligible or ineligible hides missing information and makes improvement impossible. The operation should periodically examine unknowns by reason code to decide whether documentation, tooling, specialist review, or recollection would resolve them.

## 9. What this design establishes—and what it does not

This case study establishes a proposed operating logic:

- eligibility is specific to a declared WAM use;
- the physical transition is a more informative unit than a generic clip;
- task failure and data failure require different treatment;
- immutable source-time lineage should survive preprocessing;
- missing evidence should produce a specific remedy;
- coverage should be expressed in state–interaction–outcome terms.

It does **not** establish that:

- route-aware datasets improve policy success;
- failure data should occupy a particular training ratio;
- the four proposed lanes cover every WAM architecture;
- any one source is sufficiently representative;
- the operational cost is lower than a conventional pipeline;
- the proposed thresholds are calibrated.

Those questions require implementation and downstream evaluation. Keeping them explicit is part of the design's credibility.

## 10. Final proposal

The proposed data operation treats an egocentric recording as reusable source evidence rather than a single-purpose labeled example. It preserves complete physical transitions, evaluates them against explicit consumer contracts, withholds unsupported supervision, and routes failures and recoveries without presenting them as expert actions.

The most important change is conceptual:

> The admission question is not “Is this a good clip?” It is “What claim can this evidence support, for which WAM use, under which transformation history?”

That question produces better operational artifacts: lane-specific manifests, inspectable reason codes, repair queues, and collection orders stated in terms of missing physical outcomes. It is a design ready to be challenged with real data without pretending that the empirical work has already been done.
