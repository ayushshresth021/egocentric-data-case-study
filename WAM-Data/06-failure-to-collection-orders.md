# From Model Failure to Egocentric Collection Order

**Document type:** Two hypothetical operational examples  
**Evidence status:** These are designed scenarios, not observed model results  
**Purpose:** Show how a failure report becomes a narrowly specified data request

## Guardrail: a model failure is not automatically a data gap

Before ordering new footage, the data team should distinguish among:

- missing or concentrated training coverage;
- a preprocessing or adapter defect;
- an evaluation artifact;
- a model-capacity or optimization limitation;
- a sensor requirement that RGB ego video cannot satisfy.

Collection is justified only when evidence supports a gap that the requested recording can plausibly address. Otherwise, “collect more data” becomes an expensive way to avoid diagnosis.

EMG-1 adds a useful boundary. Wrist EMG may support articulation through visual gaps, but its authors do not present it as direct pose, object-contact, or force measurement. Sensor upgrades need their own evidence contract. [EMG-1 article](https://encord.com/data-labs/introducing-emg-1/)

## Example 1 — Correct reach, missed contact, no recovery

### Failure report

In a hypothetical rollout, the generated visual plan reaches toward the correct soft-bag handle. The hand approaches the right region but fails to establish a secure grasp. The subsequent plan continues as though contact succeeded, and the bag does not move.

This report contains at least three possible failures:

1. the imagined contact geometry is wrong;
2. the model lacks alternate futures for miss, slip, and recovery;
3. the action system fails to execute an otherwise correct visual plan.

Only the first two are candidates for an egocentric-video collection response.

### Pre-collection triage

| Check | Evidence requested | Decision consequence |
|---|---|---|
| Visual plan inspection | Does the predicted hand visibly miss or pass through the handle? | If yes, continue the data-gap investigation |
| Action-plan agreement | Did execution faithfully follow the predicted contact point? | If no, escalate to the action interface rather than ordering human video |
| Existing mixture search | Are secure grasps, near misses, slips, and regrasp recoveries represented for flexible handles? | If coverage exists across relevant conditions, do not assume a collection gap |
| Adapter audit | Were contact frames cropped, downsampled, or obscured by preprocessing? | Repair the adapter before collecting replacements |
| Sensor sufficiency | Is the failure dominated by articulation, contact, force, or tactile state invisible in RGB? | Match the sensor to the claim; EMG may proxy articulation but does not directly measure contact or force |

### Candidate data-gap hypothesis

The current mixture contains clean successful grasps but few complete transitions showing:

```text
visible handle and hand approach
→ near-contact or partial contact
→ miss or slip consequence
→ corrective reorientation
→ secure regrasp or abandonment
```

The hypothesis is not that failure data universally improves the model. It is that the current data may not distinguish futures immediately around contact under the target conditions.

### Collection order A — Flexible-handle contact packets

**Order ID:** `EGO-CONTACT-FLEX-001`  
**Priority:** Conditional on the triage above confirming a coverage gap  
**Target transition:** Securely grasp and lift a flexible bag handle from a cluttered surface

#### Required matched conditions

- same bag type across outcome branches;
- documented bag fill state;
- same surface class and approximately matched clutter arrangement;
- visible handle before approach;
- hand and handle simultaneously visible through contact;
- uninterrupted recording through the outcome and any correction;
- stable postcondition after lift, slip, or abandonment.

#### Required outcome branches

| Branch | Required content | Intended lane |
|---|---|---|
| Secure first grasp | Approach, contact, load transfer, stable lift | VD-1 and TT-1 |
| Near-contact miss | Approach passes the handle; non-contact consequence remains visible | VD-1 and CR-1 |
| Partial grasp then slip | Initial displacement followed by loss of contact | VD-1 and CR-1 |
| Miss then recovery | Failed attempt, visible corrective movement, successful regrasp | VD-1, TT-1, and CR-1 |
| Miss then abandonment | Failed attempt and stable decision not to continue | VD-1 and CR-1 |
| Alternate valid grasp | Different safe strategy reaching the same lifted state | TT-1 and strategy contrast |

These are requested branches, not fixed mixture ratios. The first pilot should reveal which branches can be collected authentically and repeatably.

#### Capture requirements

```yaml
capture:
  viewpoint: egocentric
  continuity: no cuts from precondition through postcondition
  visibility:
    hand_and_handle_at_contact: required
    bag_state_after_attempt: required
  source_timestamps: retained
  outcome_tag: required
  recovery_tag: required
  metric_hand_pose: optional
  wrist_emg: optional_with_PS-1_contract
  tactile_or_force: optional_but_declared
```

#### Optional EMG upgrade

Add wrist EMG only if the question concerns action-relevant muscle activity or articulation during visual gaps. Do not add it merely because contact is difficult to see.

An EMG-enabled packet must record:

```yaml
proxy_sensor_contract:
  sensor_type: wrist_emg
  electrode_configuration: required
  placement: required
  sampling_rate_hz: required
  skin_contact_or_signal_quality: required
  participant_and_session_identity: required
  video_sync: verified
  validated_target: declared
  unsupported_claims:
    - direct_metric_pose
    - object_contact
    - grip_force
```

EMG-1 aligned high-rate EMG and video using shared gyroscope motion and graded EMG quality separately from complete multimodal quality. This is a practical precedent for PS-1 capture and QC.

#### Acceptance rules

Accept a packet only if:

- the starting handle state is observable;
- contact or non-contact can be distinguished;
- the bag's physical response is visible;
- recovery remains contiguous with the failed attempt;
- a capture dropout is not mislabeled as a miss;
- source time and outcome-branch identity are retained.

#### Rework and rejection

| Condition | Disposition |
|---|---|
| Derived clip begins after approach | Reprocess if parent contains lead-in |
| Clip ends immediately at apparent contact | Reprocess or recollect, depending on parent source |
| Hand blocks all evidence of handle contact | Recollect with adjusted mounting/view |
| Pose exists without documented frame or units | Keep visual lanes; deny MA-1 |
| EMG exists without placement, signal quality, or verified video timing | Keep visual lanes; deny or hold PS-1 |
| EMG is labeled as ground-truth contact or force without a reference | Remove the direct label; retain only a defensible proxy claim |
| Outcome branch was deliberately faked through video corruption | Route to a declared synthetic robustness set, not authentic CR-1 |

### What this order does not claim

It does not claim that new footage will repair execution or that visual contact is sufficient for force control. It also does not treat EMG as tactile ground truth or assume a branch ratio is optimal.

---

## Example 2 — Correct goal, wrong causal order

### Failure report

In a hypothetical multi-step task, the model must open a lidded storage bin, place an object inside, and close the lid. Its visual plan moves the object toward the still-closed bin, becomes blocked, and fails to recover by opening the lid first.

The final goal is represented, but the intermediate causal order is not.

Possible explanations include:

1. training windows overrepresent final states and omit prerequisite actions;
2. long episodes were divided so that the closed-bin precondition and later insertion are no longer linked;
3. captions describe the goal but not the required order;
4. the model cannot maintain the necessary state over the planning horizon;
5. the action system—not the visual plan—reordered otherwise correct steps.

### Pre-collection triage

| Check | Evidence requested | Decision consequence |
|---|---|---|
| Predicted sequence review | Does the visual plan itself attempt insertion before opening? | If no, investigate action sequencing rather than video coverage |
| Packet-lineage audit | Were prerequisite and consequence split across windows or gaps? | Repair packetization before collecting new episodes |
| Existing transition search | Do examples show the same starting state and full prerequisite chain? | If yes, inspect sampling and weighting before acquisition |
| Label audit | Does one global label hide several ordered state changes? | Add transition structure; more footage may not be necessary |
| Horizon check | Does failure persist even when the complete sequence is supplied in context? | If yes, the limitation may not be resolved through collection alone |

### Candidate data-gap hypothesis

The mixture contains many isolated “place object in bin” clips, but insufficient continuous evidence connecting:

```text
bin closed
→ lid opened
→ opening verified and maintained
→ object inserted
→ object visibly inside
→ lid closed
→ final state remains stable
```

It also lacks authentic examples of premature insertion attempts followed by recovery.

### Collection order B — Prerequisite-order packets

**Order ID:** `EGO-ORDER-BIN-001`  
**Priority:** Conditional on lineage, sampling, and action-sequencing audits  
**Target transition:** Place a specified object inside a lidded bin while preserving prerequisite order and observable intermediate states

#### Required sequence families

| Family | Sequence | Purpose |
|---|---|---|
| Canonical completion | Open → verify open → insert → verify inside → close | Complete positive transition chain |
| Premature insertion and recovery | Attempt insert while closed → observe blockage → set object safely → open → insert | Counterfactual causal order and correction |
| Interrupted opening | Begin opening → lid falls closed → reopen → insert | Persistence failure and recovery |
| Valid alternate strategy | Secure object first, then open with free hand, or open first and retrieve object | Multiple valid plans with the same state constraints |
| Abandoned attempt | Encounter blocked or unsafe state → stop without pretending success | Authentic unresolved/abandoned future |

The operation must not instruct participants to create unsafe failures. Requested alternate branches must remain physically safe and authentic.

#### State checkpoints

Every accepted packet must make these checkpoints observable or explicitly mark them unknown:

```yaml
state_checkpoints:
  initial_bin_state: closed|open|unknown
  object_initial_location: visible_description
  lid_open_before_insertion: yes|no|unknown
  opening_maintained_during_insertion: yes|no|unknown
  object_inside_after_insertion: yes|no|unknown
  final_lid_state: open|closed|unknown
  final_state_stable: yes|no|unknown
```

#### Boundary requirements

- Begin before the participant commits to the first prerequisite action.
- Preserve transitions between subtasks; do not export only isolated verbs.
- Retain pauses that show whether the lid remains open.
- Continue through a stable final state or an explicit abandonment.
- If fixed-length model windows are required, record overlap and adjacency so the full parent chain remains reconstructable.
- Never concatenate nonadjacent retained ranges without representing the gap.

#### Acceptance rules

Accept the packet for VD-1 when the relevant state transitions remain observable and temporally coherent. Accept it for TT-1 only when bin mechanics, starting state, and target sequence match the declared task. Accept a premature-order example for CR-1 only when the failed attempt and recovery are authentic and continuous.

Metric pose is optional. Its presence does not compensate for an invisible lid state or missing intermediate consequence.

EMG is also optional and usually unnecessary for this order. The suspected gap concerns observable prerequisite state and temporal order, not hidden articulation. Instrumentation should follow the hypothesis rather than be added by default.

#### Rework and rejection

| Condition | Disposition |
|---|---|
| Parent episode contains full order, but action clips isolate verbs | Reprocess into linked transition packets |
| Global caption states only “put object away” | Retain caption; add state checkpoints rather than inventing detailed intent |
| Camera turns away while lid state changes | Recollect or use a valid alternate view |
| Clip ends when object crosses the rim | Extend from parent or recollect stable postcondition |
| Action logs and video disagree on event order | Hold and investigate synchronization |

### What this order does not claim

The order does not assume that longer clips always improve planning. It preserves the causal chain so a downstream consumer can choose windows without losing state adjacency. It also does not assume that additional ordered examples will solve a model's horizon or memory limitation.

## Collection-order template

The two examples follow a reusable structure:

```yaml
order_id: string
trigger:
  observed_failure: string
  non_data_causes_checked: []
  coverage_evidence: []
gap_hypothesis:
  missing_start_state: string
  missing_transition_or_branch: string
  target_conditions: []
request:
  transition: string
  matched_conditions: []
  required_outcome_branches: []
  required_state_checkpoints: []
capture_contract:
  required_views: []
  continuity: string
  required_channels: []
  optional_channels: []
  proxy_sensor_contract:
    validated_targets: []
    unsupported_claims: []
  lineage_fields: []
acceptance:
  rules: []
  rework_triggers: []
  rejection_triggers: []
supported_lanes: []
unsupported_claims: []
```

The template forces the requesting team to connect a model failure to missing evidence, declare what must remain observable, and state what the resulting footage still cannot prove.
