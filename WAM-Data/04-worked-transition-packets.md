# Worked Transition Packets

**Purpose:** Exercise the proposed routing policy before applying it to a real cohort  
**Evidence status:** All scenarios below are hypothetical and illustrative; no frequencies or empirical results are implied  
**Policy dependency:** WAM Egocentric Data Contract v0.2

## How to read the examples

Each example starts with a requested use. It evaluates four top-level lanes, with two subcontracts under instrumented action:

- `VD-1`: video dynamics;
- `TT-1`: task-matched video-only transfer;
- `MA-1`: metric pseudo-action or retargeting;
- `PS-1`: indirect proxy-sensor supervision, including EMG;
- `CR-1`: counterfactual and recovery evaluation.

The final disposition answers an operational question, not whether the recording is globally “good” or “bad.”

## Packet 1 — Clean success without metric instrumentation

### Scenario

A head-mounted RGB camera records a person opening a kitchen drawer. The closed drawer is visible before approach. The hand grasps the handle, pulls, and releases. The open drawer remains visible for two seconds. No hand-pose or body-pose channel exists.

**Requested use:** Task-matched transfer for opening a pull-out drawer.

### Evidence

```yaml
packet_id: hypothetical/clean_drawer_success
transition:
  starting_state: drawer_closed
  intended_change: drawer_open
  interaction_mode: grasp_handle_and_pull
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: yes
  outcome: success
  recovery_contiguous: not_applicable
instrumentation:
  rgb: present
  metric_hand_pose: absent
  proxy_sensor: absent
```

### Verdicts

| Lane | Verdict | Reason |
|---|---|---|
| VD-1 | `eligible` | A continuous and observable state transition is present |
| TT-1 | `eligible` | Starting state, mechanics, and intended change match the declared target |
| MA-1 | `ineligible` | No metric action channel or calibration exists |
| PS-1 | `ineligible` | No proxy-sensor channel exists |
| CR-1 | `ineligible` | No alternate outcome, failure, or recovery is present |

**Disposition:** `release` to VD-1 and TT-1; mask MA-1, PS-1, and CR-1.

**Reason codes:** `COMPLETE_TRANSITION`, `TARGET_MECHANICS_MATCH`, `METRIC_CHANNEL_ABSENT`, `NO_COUNTERFACTUAL_EVENT`

**Why this example matters:** Lack of metric pose does not make the visual transition unusable. The contract prevents an unsupported action claim without discarding the episode.

---

## Packet 2 — Miss followed by a successful recovery

### Scenario

A person reaches for a mug handle, initially misses, pauses, changes wrist orientation, grasps the handle, and lifts the mug. The mug, hand, first miss, corrective movement, and final lift remain visible continuously.

**Requested use:** Clean expert-action supervision.

### Evidence

```yaml
packet_id: hypothetical/mug_miss_recovery
transition:
  starting_state: mug_on_table
  intended_change: mug_lifted
  interaction_mode: reach_grasp_lift
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: yes
  outcome: recovered
  recovery_contiguous: yes
behavior:
  first_attempt: miss
  corrective_action: reorient_and_regrasp
```

### Verdicts

| Lane | Verdict | Reason |
|---|---|---|
| VD-1 | `eligible` | The packet contains coherent miss, correction, and successful lift dynamics |
| TT-1 | `eligible` | The full transition is relevant to the target, provided it is not labeled as a single clean expert action |
| MA-1 | `unknown` | No calibration evidence was supplied with the scenario |
| PS-1 | `ineligible` | No proxy-sensor stream was captured |
| CR-1 | `eligible` | The failed attempt and corrective response are authentic and contiguous |

**Disposition:** `route`. Exclude from a clean-success-only action subset; route to VD-1, qualified TT-1, and CR-1.

**Reason codes:** `COMPLETE_FAILED_TRANSITION`, `RECOVERY_VISIBLE`, `ACTION_LABEL_REQUIRES_SEGMENTATION`, `METRIC_CALIBRATION_UNKNOWN`

**Why this example matters:** The behavioral failure is valuable evidence. Treating “not clean expert execution” as “bad data” would delete the corrective future.

---

## Packet 3 — Dropped frames at contact

### Scenario

A recording shows a hand approaching a light switch. The video jumps from three centimeters before contact to the hand withdrawing. The light is on afterward, but frames around contact are absent from the source recording. Timestamps confirm a 700 ms capture gap.

**Requested use:** Video-dynamics learning.

### Evidence

```yaml
packet_id: hypothetical/light_switch_capture_gap
transition:
  starting_state: light_off
  intended_change: light_on
  interaction_mode: press_switch
  precondition_visible: yes
  causal_event_visible: no
  consequence_visible: yes
  outcome: success
capture:
  source_gap_ms: 700
  gap_overlaps_contact: true
```

### Verdicts

| Lane | Verdict | Reason |
|---|---|---|
| VD-1 | `ineligible` | The causal event is missing from the source, so continuity cannot be reconstructed |
| TT-1 | `ineligible` | TT-1 inherits the missing causal evidence from VD-1 |
| MA-1 | `ineligible` | The required action interval is missing and synchronization is discontinuous |
| PS-1 | `ineligible` | No proxy-sensor channel exists to preserve evidence through the visual gap |
| CR-1 | `ineligible` | The gap is a capture defect, not an authentic alternate outcome |

**Disposition:** `recollect` if this transition is strategically important; otherwise `reject` from the current release.

**Reason codes:** `CAPTURE_DROPOUT_AT_CONTACT`, `CAUSAL_EVENT_NOT_OBSERVED`, `TEMPORAL_DISCONTINUITY`

**Why this example matters:** A visible before-and-after state is not always sufficient. The policy distinguishes a real-world transition from evidence that can support a temporally controllable model.

---

## Packet 4 — Invalid metric pose with valid EMG evidence

### Scenario

RGB, a smooth 3D wrist trajectory, and wrist EMG accompany a bottle-opening episode. The pose lacks a frame, units, and calibration. The EMG configuration, placement, sampling, signal validity, and video synchronization are documented.

The EMG target is limited to finger-articulation representation during visual gaps. It is not labeled as direct pose, object contact, or force.

**Requested use:** Metric wrist-action supervision.

### Evidence

```yaml
packet_id: hypothetical/bottle_unknown_pose_frame
transition:
  starting_state: bottle_capped
  intended_change: bottle_uncapped
  interaction_mode: stabilize_and_twist
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: yes
  outcome: success
instrumentation:
  pose_samples: present
  tracking_continuity: visually_plausible
  coordinate_frame: unknown
  units: unknown
  video_pose_sync: undocumented
  wrist_emg:
    channel_present: true
    placement_and_configuration: documented
    sampling_rate_hz: documented
    contact_or_signal_quality: valid
    video_sync: verified
    validated_target: finger_articulation_proxy
    unsupported_claims:
      - direct_metric_pose
      - object_contact
      - grip_force
```

### Verdicts

| Lane | Verdict | Reason |
|---|---|---|
| VD-1 | `eligible` | RGB preserves the complete physical transition |
| TT-1 | `eligible` | The task mechanics match a twist-cap target, assuming the target is declared accordingly |
| MA-1 | `ineligible` | Smooth coordinates do not establish metric meaning, units, frame, or synchronization |
| PS-1 | `eligible` | EMG satisfies its proxy contract for the declared target without being presented as metric truth |
| CR-1 | `ineligible` | No alternate outcome or recovery occurs |

**Disposition:** `route` to VD-1, TT-1, and PS-1. Hold MA-1 only if calibration documentation can realistically be recovered.

**Reason codes:** `COMPLETE_TRANSITION`, `COORDINATE_FRAME_UNKNOWN`, `UNITS_UNKNOWN`, `PROXY_SENSOR_CONTRACT_VERIFIED`, `PROXY_SIGNAL_NOT_DIRECT_MEASUREMENT`

**Why this example matters:** A bad metric channel does not invalidate a separate proxy channel. EMG-1 applies this logic by retaining aligned EMG when visual pose is unreliable. [EMG-1 article](https://encord.com/data-labs/introducing-emg-1/)

---

## Packet 5 — Low-motion interval preserving a stable postcondition

### Scenario

A person places a flexible pouch upright, supports it for one second, releases it, and watches it remain stable for four seconds. An activity filter marks the final four seconds as idle and proposes removing them.

**Requested use:** Video-dynamics learning after low-motion filtering.

### Evidence

```yaml
packet_id: hypothetical/pouch_stability_hold
transition:
  starting_state: pouch_flat
  intended_change: pouch_upright_and_stable
  interaction_mode: lift_place_support_release
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: yes
  outcome: success
derived_transform:
  proposed_idle_removal_s: 4.0
  removed_content: stable_postcondition
```

### Verdicts

| Lane | Verdict before trimming | Effect of proposed trimming |
|---|---|---|
| VD-1 | `eligible` | May become `unknown` because stability is no longer demonstrated |
| TT-1 | `eligible` | May lose the target's required stable outcome |
| MA-1 | `unknown` | Unchanged; instrumentation remains unspecified |
| PS-1 | `unknown` | No proxy-sensor evidence was supplied with the scenario |
| CR-1 | `ineligible` | No counterfactual event occurs |

**Disposition:** `reprocess`. Retain enough post-release time to establish stability; do not delete the source interval.

**Reason codes:** `STABLE_POSTCONDITION_REQUIRED`, `LOW_MOTION_SEMANTICALLY_RELEVANT`, `DERIVED_CROP_INCOMPLETE`

**Why this example matters:** Low visual activity can carry high causal value. Activity is not a proxy for supervision value.

---

## Packet 6 — Derived clip ends before the consequence

### Scenario

The parent episode shows a person inserting a key, turning it, opening a door, and walking through. An automatically derived action clip ends immediately after the key turns. The remainder is still present in the immutable parent episode.

**Requested use:** Task-matched transfer for unlocking and opening a door.

### Evidence

```yaml
packet_id: hypothetical/key_turn_crop
transition:
  starting_state: door_closed_and_locked
  intended_change: door_unlocked_and_open
  interaction_mode: insert_turn_pull
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: no
  outcome: unresolved
lineage:
  consequence_exists_in_parent: true
  derived_clip_ends_early: true
```

### Verdicts

| Lane | Current derived clip | After boundary repair |
|---|---|---|
| VD-1 | `ineligible` | `eligible` if the door opening and final state are restored |
| TT-1 | `ineligible` | `eligible` if the complete target mechanics are present |
| MA-1 | `unknown` | Remains dependent on instrumentation evidence |
| PS-1 | `unknown` | Remains dependent on proxy-sensor metadata and target definition |
| CR-1 | `ineligible` | Remains ineligible unless an alternate outcome exists |

**Disposition:** `reprocess`, not recollect.

**Reason codes:** `MISSING_CONSEQUENCE`, `DERIVED_CROP_INCOMPLETE`, `SOURCE_EVIDENCE_RECOVERABLE`

**Why this example matters:** The correct remedy depends on provenance. Recollection would waste effort because the source already contains the required evidence.

---

## Packet 7 — Same verb, incompatible mechanics

### Scenario

The target capability is opening a heavy sliding workshop drawer by pulling a recessed handle. The candidate packet shows a person opening a lightweight hinged kitchen cabinet by rotating its door outward. Both carry the language label “open cabinet.”

**Requested use:** Task-matched transfer for the sliding drawer.

### Evidence

```yaml
packet_id: hypothetical/open_verb_mechanics_mismatch
transition:
  starting_state: hinged_cabinet_closed
  intended_change: hinged_cabinet_open
  interaction_mode: grasp_and_rotate_about_hinge
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: yes
  outcome: success
target:
  object: heavy_sliding_drawer
  interaction_mode: recessed_grasp_and_linear_pull
```

### Verdicts

| Lane | Verdict | Reason |
|---|---|---|
| VD-1 | `eligible` | The source contains a coherent physical transition |
| TT-1 | `ineligible` | Constraints, force direction, handle geometry, and motion topology differ from the target |
| MA-1 | `unknown` | Instrumentation evidence is unspecified |
| PS-1 | `unknown` | Proxy instrumentation and target claim are unspecified |
| CR-1 | `ineligible` | No alternate outcome or recovery occurs |

**Disposition:** `route` to generic VD-1; do not include in the target-specific TT-1 manifest.

**Reason codes:** `TASK_MECHANICS_MISMATCH`, `TARGET_STATE_CHANGE_MISMATCH`, `LANGUAGE_MATCH_INSUFFICIENT`

**Why this example matters:** Task names can inflate apparent coverage. State and mechanics—not shared verbs—determine task-transfer eligibility.

---

## Packet 8 — Alternative valid strategy

### Scenario

Most examples show a storage bin being opened with the dominant hand while the other hand stabilizes the container. This packet shows a person pinning the bin against the counter with their forearm and opening the latch with the nondominant hand. The bin opens safely and remains open.

**Requested use:** Task-matched transfer and strategy coverage.

### Evidence

```yaml
packet_id: hypothetical/bin_alternative_strategy
transition:
  starting_state: latched_bin_closed
  intended_change: bin_open
  interaction_mode: forearm_stabilize_nondominant_unlatch
  precondition_visible: yes
  causal_event_visible: yes
  consequence_visible: yes
  outcome: success
strategy:
  differs_from_modal_pattern: true
  safe_and_effective: true
```

### Verdicts

| Lane | Verdict | Reason |
|---|---|---|
| VD-1 | `eligible` | Complete physical transition is visible |
| TT-1 | `eligible` | The starting state, mechanics, and target outcome match despite strategy variation |
| MA-1 | `unknown` | Metric channel requirements remain unspecified |
| PS-1 | `unknown` | No proxy-sensor evidence or validated target was supplied |
| CR-1 | `eligible` for a strategy-contrast set | It supplies a valid alternate future from a similar task condition |

**Disposition:** `release` to VD-1 and TT-1; `route` a reference to the strategy-contrast subset of CR-1.

**Reason codes:** `COMPLETE_TRANSITION`, `ALTERNATE_VALID_STRATEGY`, `TARGET_MECHANICS_MATCH`

**Why this example matters:** Deviation from the modal demonstration is not necessarily error. A data operation should preserve valid behavioral diversity without conflating it with failure.

## Cross-example conclusions

These examples expose five policy properties:

1. **Eligibility is genuinely multi-lane.** Packet 4 qualifies for PS-1 while failing MA-1; packets 1 and 7 remain useful without either instrumented-action subcontract.
2. **The remedy follows the evidence location.** Packet 6 is reprocessed because the source contains the consequence; packet 3 requires recollection because it does not.
3. **Failure and corruption remain distinct.** Packet 2 is an authentic miss and recovery; packet 3 is a capture gap.
4. **Inactive frames can be causally important.** Packet 5 needs low-motion follow-through to establish stability.
5. **Labels do not determine physical relevance.** Packet 7 shares a verb but not mechanics, while packet 8 differs in strategy but reaches the same valid state.

The examples test the coherence of the design. They do not show how often these conditions occur in a real dataset or whether the routing policy improves a model.
