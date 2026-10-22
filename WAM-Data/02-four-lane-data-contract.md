# WAM Egocentric Data Contract

**Contract version:** 0.2  
**Applies to:** Derived transition packets from egocentric physical-interaction recordings  
**Decision model:** Eligibility is evaluated independently for each release lane

## Contract principle

> A transition packet may enter only the lanes whose evidence requirements it satisfies. Rejection from one lane does not imply global rejection.

The atomic unit is a complete physical transition:

```text
visible precondition
→ intervention
→ causal event
→ visible consequence
→ recovery or stable postcondition
```

All derived packets must retain their parent episode, immutable source timestamps, camera identity, processing version, and any omitted source-time ranges.

## Lane contracts

| Contract | Intended use | Minimum required evidence | Hard disqualifiers | Evidence not universally required |
|---|---|---|---|---|
| **VD-1: Video dynamics** | Learning or evaluating observable physical futures | Temporally coherent precondition, intervention, and consequence; relevant objects visible through the causal event | Cut or unrecoverable corruption across the causal event; consequence absent; task object not observable; temporal order uncertain | Success; robot actions; metric pose; task-language match |
| **TT-1: Task-matched transfer** | Video-only adaptation to a declared robot task or evaluation | VD-1 eligibility plus matching starting state, intended state change, compatible physical mechanics, and usable viewpoint | Same verb but incompatible mechanics; target state absent; materially different constraints; outcome not visible | Metric human motion; robot embodiment; successful human action in every packet |
| **MA-1: Metric action** | Pseudo-action extraction, retargeting, or metric motion supervision | Known body point or action definition; calibrated coordinate frame and units; verified timestamps; valid synchronization; channel-level confidence or missing-data semantics | Unknown scale/frame; unverified reconstruction; timestamp ambiguity; tracking loss across required motion; undocumented human-to-robot mapping | Task success; language annotation; counterfactual outcome |
| **PS-1: Proxy sensor** | Learning action-relevant representations from EMG, IMU, or another indirect signal | Sensor type, placement, configuration, sampling, contact/validity state, synchronization, participant/session identity, and a declared prediction target | Signal presented as direct pose/contact/force; unknown placement; degraded contact without validity flags; unverified timing; unsupported target claim | Metric coordinates; task success; direct object-contact label |
| **CR-1: Counterfactual and recovery** | Testing or learning distinguishable alternate outcomes and corrective behavior | Authentic attempt; observable outcome; enough continuation to distinguish miss, failure, correction, abandonment, or delayed success | Capture defect mistaken for world failure; outcome cannot be resolved; recovery separated by an unrecorded gap; synthetic corruption presented as authentic behavior | Metric pose; clean execution; successful final completion |

MA-1 and PS-1 are subcontracts of the instrumented-action lane. They receive separate verdicts. A packet may satisfy either, both, or neither.

EMG-1 demonstrates the distinction. It releases synchronized wrist EMG and video, but states that EMG is not direct pose, contact, or force measurement. Its field and laboratory collections also serve different evidence roles.

The project retains unreliable visual-pose gaps and the corresponding EMG rather than treating uncertain poses as measurements. [EMG-1 article](https://encord.com/data-labs/introducing-emg-1/)

## Allowed verdicts

Each lane receives exactly one verdict:

| Verdict | Meaning |
|---|---|
| `eligible` | All required evidence is present and no hard disqualifier applies |
| `ineligible` | A known contract requirement is violated |
| `unknown` | Available evidence cannot resolve a required condition |

`Unknown` must not be silently converted to `eligible`. It requires a reason code and an owner for resolution or acceptance of the uncertainty.

## Operational disposition

After lane verdicts, assign one packet-level disposition:

| Disposition | Use when |
|---|---|
| `release` | The packet can enter its declared target lane as-is |
| `route` | The packet is ineligible for its requested lane but eligible for another |
| `hold` | Metadata, adjudication, or policy clarification may resolve the decision |
| `reprocess` | Required source evidence exists, but a derived artifact is incorrect or incomplete |
| `recollect` | The missing evidence was never captured or cannot be recovered |
| `reject` | No current lane can support the packet and no proportionate remedy exists |

## Cross-lane rules

1. A failed physical attempt is not automatically defective data.
2. A capture defect is not a counterfactual outcome.
3. `MA-1: eligible` requires positive evidence of calibration and synchronization; absence of a warning is insufficient.
4. `PS-1: eligible` requires a validated target claim. Action relevance does not make a proxy signal a metric label.
5. Missing visual pose may coexist with eligible proxy-sensor evidence; preserve the gap rather than imputing a measured pose.
6. `TT-1: eligible` requires matching mechanics, not merely a matching task verb or object category.
7. Low motion is not a universal rejection reason. Stable holds and postconditions may be required by VD-1 or CR-1.
8. Cropping must not manufacture adjacency. Every removed gap remains represented in lineage.
9. A packet may be released to multiple lanes with different channel masks.
10. No transform may overwrite immutable source identity or source time.

## Required manifest fields

```yaml
packet_id: string
source:
  dataset: string
  episode_id: string
  camera_id: string
  parent_start_s: number
  parent_end_s: number
  retained_time_ranges: []
  omitted_time_ranges: []
  processing_version: string

transition:
  intended_change: string
  starting_state: string
  interaction_mode: string
  precondition_visible: yes|no|unknown
  causal_event_visible: yes|no|unknown
  consequence_visible: yes|no|unknown
  outcome: success|failed_attempt|recovered|abandoned|unresolved
  recovery_contiguous: yes|no|not_applicable|unknown

instrumentation:
  metric_action:
    channel_present: boolean
    coordinate_frame: string|null
    units: string|null
    synchronization_status: verified|unverified|not_applicable
  proxy_sensor:
    channel_present: boolean
    sensor_type: string|null
    placement_and_configuration: string|null
    sampling_rate_hz: number|null
    contact_or_signal_quality: valid|degraded|unknown|not_applicable
    participant_and_session_retained: boolean
    synchronization_status: verified|unverified|not_applicable
    validated_targets: []
    unsupported_claims: []

eligibility:
  video_dynamics: eligible|ineligible|unknown
  task_matched_transfer: eligible|ineligible|unknown
  metric_action: eligible|ineligible|unknown
  proxy_sensor: eligible|ineligible|unknown
  counterfactual_recovery: eligible|ineligible|unknown

decision:
  disposition: release|route|hold|reprocess|recollect|reject
  reason_codes: []
  policy_version: string
  decided_by: string
  decided_at: string
```

## Release condition

A release manifest is valid only when:

- every packet names its intended lane or lanes;
- every lane has an explicit verdict;
- every `ineligible` or `unknown` verdict has at least one reason code;
- released channels satisfy the corresponding contract;
- masked or unavailable channels are explicit;
- source-time lineage is reconstructable;
- policy and processing versions are recorded.

This contract governs what the evidence may claim. It does not prescribe a model architecture, training weight, or optimal mixture ratio.
