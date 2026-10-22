# WAM Data Eligibility Reason-Code Taxonomy

**Taxonomy version:** 0.2  
**Purpose:** Provide stable, machine-readable explanations for eligibility and routing decisions  
**Unit:** One reason code applied to one transition packet and, when relevant, one lane

## Usage rules

1. Every `ineligible` or `unknown` lane verdict requires at least one reason code.
2. Positive codes may explain why a packet was released, but they do not replace contract checks.
3. Record the narrowest applicable code. Do not use a generic code when the specific failure is known.
4. Codes describe evidence, not blame. Collector performance should not be inferred from one code.
5. Multiple codes may apply when they identify independent conditions.
6. Free text may add context but must not redefine the controlled code.
7. Changes to a code's meaning require a new taxonomy version.

## Code format

```text
CATEGORY_CONDITION
```

Categories describe the evidence domain. Conditions describe the observed state. Codes use uppercase snake case and remain stable after publication.

## 1. Source and lineage

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `SOURCE_IDENTITY_MISSING` | Parent dataset or episode cannot be established | Hold all lanes | Restore source registry link |
| `SOURCE_TIME_UNRECOVERABLE` | Derived packet cannot be mapped to source timestamps | Ineligible for release | Recover mapping; otherwise reject |
| `CAMERA_IDENTITY_UNKNOWN` | Camera or viewpoint stream is ambiguous | Hold affected lanes | Recover stream metadata |
| `PROCESSING_VERSION_UNKNOWN` | Transform version cannot be determined | Hold derived channels | Identify or regenerate transform |
| `RIGHTS_STATUS_UNRESOLVED` | Intended release or training use is not covered by known rights | Hold publication and training use | Rights review |
| `LINEAGE_COMPLETE` | Source identity, time, stream, and transforms are reconstructable | Positive evidence only | None |

## 2. Transition completeness

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `MISSING_PRECONDITION` | Required starting state is absent from source or packet | VD-1 and TT-1 ineligible | Extend from parent or recollect |
| `PRECONDITION_AMBIGUOUS` | Starting state is visible but cannot be resolved | VD-1/TT-1 unknown | Specialist review or better view |
| `CAUSAL_EVENT_NOT_OBSERVED` | Intervention or physical causal moment is absent | VD-1 and TT-1 ineligible | Extend, switch view, or recollect |
| `CAUSAL_EVENT_AMBIGUOUS` | Causal event may be present but cannot be distinguished | Relevant lanes unknown | Review alternate view or metadata |
| `MISSING_CONSEQUENCE` | Resulting state is absent | VD-1 and TT-1 ineligible | Extend from parent or recollect |
| `STABLE_POSTCONDITION_REQUIRED` | Target claim requires follow-through showing persistence | Hold/reprocess relevant lanes | Preserve longer follow-through |
| `COMPLETE_TRANSITION` | Precondition, causal event, and consequence are observable | Positive evidence for VD-1 | None |

## 3. Temporal integrity and capture

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `TEMPORAL_DISCONTINUITY` | Unrepresented or invalid time gap crosses relevant evidence | Affected lanes ineligible | Restore source ranges or recollect |
| `CAPTURE_DROPOUT_AT_CONTACT` | Source frames are missing around contact or causal event | VD-1, TT-1, MA-1 ineligible | Recollect |
| `DUPLICATED_FRAMES_DURING_EVENT` | Repeated source frames corrupt event timing | Affected temporal uses ineligible | Reprocess if original frames exist |
| `CAMERA_STREAM_MISORDERED` | Camera ordering or identity changed incorrectly | Hold all viewpoint-dependent uses | Correct stream mapping |
| `SOURCE_CLOCK_DRIFT` | Sensor alignment changes over the interval | MA-1/PS-1 ineligible or unknown | Re-estimate synchronization |
| `SENSOR_SYNC_UNVERIFIED` | Alignment has not been validated | MA-1/PS-1 unknown or ineligible | Run synchronization validation |
| `TEMPORAL_INTEGRITY_VERIFIED` | Required source streams remain ordered and synchronized | Positive evidence | None |

## 4. Visibility and observation

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `TARGET_OBJECT_NOT_VISIBLE` | Relevant object is not observable during required interval | VD-1/TT-1 ineligible | Alternate view or recollect |
| `TARGET_INTERACTION_NOT_VISIBLE` | Hand/object relationship or intervention is obscured | VD-1/TT-1 ineligible or unknown | Alternate view or recollect |
| `CONSEQUENCE_OCCLUDED` | Final physical result cannot be observed | VD-1/TT-1 ineligible | Extend or use alternate view |
| `CAMERA_MOTION_DOMINATES` | Viewpoint motion prevents resolving the agent/world change | Lane-dependent unknown/ineligible | Stabilize only if evidence survives; otherwise recollect |
| `VIEWPOINT_SUITABILITY_UNKNOWN` | Visibility exists but usefulness for the declared target is unresolved | TT-1 unknown | Target-specific review |
| `RELEVANT_EVIDENCE_VISIBLE` | Required objects and state change remain observable | Positive evidence | None |

## 5. Outcome, failure, and recovery

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `OUTCOME_UNRESOLVED` | Available evidence cannot distinguish success, failure, or partial completion | CR-1 and target-specific lanes unknown | Extend/review/recollect |
| `COMPLETE_FAILED_TRANSITION` | Authentic failed attempt and consequence are observable | Positive for VD-1/CR-1; not clean expert action | Route and qualify labels |
| `RECOVERY_VISIBLE` | Corrective response is contiguous and observable | Positive for CR-1 | Preserve adjacency |
| `RECOVERY_NOT_CAPTURED` | Source stops before a relevant recovery can be observed | CR-1 ineligible for recovery use | Recollect with follow-through |
| `RECOVERY_CROPPED` | Parent contains recovery but derived packet excludes it | CR-1 currently ineligible | Reprocess boundaries |
| `RECOVERY_CONTINUITY_UNKNOWN` | Gaps prevent proving that recovery follows the attempt | CR-1 unknown | Recover lineage or reject continuity claim |
| `CAPTURE_DEFECT_NOT_BEHAVIOR_FAILURE` | Apparent failure results from recording corruption | CR-1 ineligible as authentic behavior | Route to synthetic/capture robustness set if useful |
| `NO_COUNTERFACTUAL_EVENT` | Packet contains no alternate outcome, failure, or correction | CR-1 ineligible | No repair required |

## 6. Task and mechanics match

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `TARGET_STATE_CHANGE_MISMATCH` | Source and target require different state transitions | TT-1 ineligible | Route to broader VD-1 set |
| `TASK_MECHANICS_MISMATCH` | Contact, constraint, motion topology, or force requirements differ materially | TT-1 ineligible | Find or collect mechanically matched transition |
| `TARGET_START_STATE_MISMATCH` | Source begins in a state not relevant to the declared target | TT-1 ineligible | Match or collect target start state |
| `TARGET_MATCH_UNRESOLVED` | Target definition is too vague to determine compatibility | TT-1 unknown | Refine target contract |
| `MECHANICS_UNRESOLVED` | Evidence does not reveal whether mechanics match | TT-1 unknown | Expert review or alternate view |
| `LANGUAGE_MATCH_INSUFFICIENT` | Eligibility was proposed from shared labels without physical evidence | TT-1 ineligible/unknown | Review state and mechanics |
| `TARGET_MECHANICS_MATCH` | State change and interaction mechanics match the declared target | Positive TT-1 evidence | None |
| `ALTERNATE_VALID_STRATEGY` | Non-modal strategy safely achieves the same target transition | Positive TT-1/contrast evidence | Preserve strategy identity |

## 7. Metric action validity

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `METRIC_CHANNEL_ABSENT` | No metric action or pose channel exists | MA-1 ineligible | Route to non-metric lanes |
| `ACTION_SEMANTICS_UNDEFINED` | Measured point or action variable is unspecified | MA-1 ineligible | Define channel semantics |
| `COORDINATE_FRAME_UNKNOWN` | Frame, axes, or origin are undocumented | MA-1 ineligible | Recover calibration or regenerate |
| `UNITS_UNKNOWN` | Scale or units are undocumented | MA-1 ineligible | Recover measurement contract |
| `TRACKING_INVALID_DURING_EVENT` | Required motion channel fails during the causal interval | MA-1 ineligible | Reprocess if source permits; otherwise recollect |
| `TRACKING_VALIDITY_UNKNOWN` | Channel lacks confidence or missing-data semantics | MA-1 unknown | Validate or add channel validity |
| `RETARGETING_MAPPING_UNDEFINED` | Human-to-target action mapping is unspecified | MA-1 ineligible for retargeting | Define and validate mapping |
| `METRIC_CALIBRATION_UNKNOWN` | Calibration state cannot be established | MA-1 unknown/ineligible | Recover calibration record |
| `METRIC_CONTRACT_VERIFIED` | Semantics, frame, units, timing, and validity are documented | Positive MA-1 evidence | None |

## 8. Proxy-sensor validity

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `PROXY_SENSOR_CHANNEL_ABSENT` | No indirect action-relevant sensor stream exists | PS-1 ineligible | Route to non-proxy lanes |
| `PROXY_SENSOR_CONFIGURATION_UNKNOWN` | Sensor type, placement, or channel configuration is undocumented | PS-1 unknown/ineligible | Recover sensor manifest |
| `PROXY_SENSOR_VALIDITY_UNKNOWN` | Signal-contact quality or missing-data semantics are absent | PS-1 unknown | Recover QC or validate the stream |
| `SENSOR_CONTACT_DEGRADED` | Electrode or wearable contact degrades the required interval | PS-1 ineligible for the affected span | Mask span, reprocess QC, or recollect |
| `PARTICIPANT_SESSION_IDENTITY_MISSING` | Wearer or recording-session identity is unavailable | PS-1 hold/unknown | Restore provenance |
| `PROXY_TARGET_UNVALIDATED` | Proposed prediction target lacks independent validation | PS-1 ineligible for that claim | Validate against an independent reference |
| `REFERENCE_LABEL_UNAVAILABLE` | No independent reference exists for the claimed target | PS-1 unknown or research-only | Acquire controlled reference data |
| `PROXY_SIGNAL_NOT_DIRECT_MEASUREMENT` | Signal is related to, but does not directly measure, pose/contact/force | Requires explicit claim boundary | Relabel as proxy; block direct-measurement claims |
| `PROXY_SENSOR_CONTRACT_VERIFIED` | Configuration, validity, timing, identity, and target claim are documented | Positive PS-1 evidence | None |

EMG-1 motivates these distinctions. It separately grades EMG and multimodal quality, keeps visual-pose gaps, and uses laboratory optical motion capture as an independent articulation reference.

Its article also states that EMG is not direct pose, contact, or force measurement. [EMG-1 article](https://encord.com/data-labs/introducing-emg-1/)

## 9. Derived transformations

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `DERIVED_CROP_INCOMPLETE` | Derived boundaries exclude required evidence found in parent | Current derivative ineligible | Regenerate boundaries |
| `SOURCE_EVIDENCE_RECOVERABLE` | Missing derivative evidence remains available in parent source | Favors reprocess over recollect | Reprocess |
| `REMOVED_GAP_NOT_REPRESENTED` | Concatenated ranges appear falsely continuous | Hold affected derivative | Restore lineage and gap metadata |
| `LOW_MOTION_SEMANTICALLY_RELEVANT` | Proposed idle removal would delete a hold, consequence, or recovery | Do not remove blindly | Retain or mask by consumer |
| `ACTION_LABEL_REQUIRES_SEGMENTATION` | One label collapses attempt, failure, and correction | Ineligible as one clean action | Segment or qualify phases |
| `CHANNEL_MASK_REQUIRED` | Packet is releasable only if unsupported channel is excluded | Route/release with mask | Publish explicit mask |

## 10. Authenticity and set membership

| Code | Meaning | Default consequence | Typical remedy |
|---|---|---|---|
| `EVENT_AUTHENTICITY_UNKNOWN` | Cannot distinguish natural behavior from induced corruption or staging | CR-1 unknown | Recover protocol metadata |
| `SYNTHETIC_DEFECT` | Corruption was deliberately introduced | Exclude from authentic-behavior claims | Route to declared robustness set |
| `STRATEGY_CONTRAST_ELIGIBLE` | Packet supplies a valid alternative strategy for comparison | Positive CR-1/contrast evidence | Preserve matching variables |

## Decision record example

```yaml
packet_id: hypothetical/mug_miss_recovery
lane_decisions:
  video_dynamics:
    verdict: eligible
    reason_codes:
      - COMPLETE_TRANSITION
      - COMPLETE_FAILED_TRANSITION
      - RECOVERY_VISIBLE
  task_matched_transfer:
    verdict: eligible
    reason_codes:
      - TARGET_MECHANICS_MATCH
      - ACTION_LABEL_REQUIRES_SEGMENTATION
  metric_action:
    verdict: unknown
    reason_codes:
      - METRIC_CALIBRATION_UNKNOWN
  proxy_sensor:
    verdict: ineligible
    reason_codes:
      - PROXY_SENSOR_CHANNEL_ABSENT
  counterfactual_recovery:
    verdict: eligible
    reason_codes:
      - COMPLETE_FAILED_TRANSITION
      - RECOVERY_VISIBLE
decision:
  disposition: route
  taxonomy_version: "0.2"
```

## Taxonomy maintenance

Add a code only when the existing taxonomy cannot distinguish a condition that changes eligibility, disposition, or remedy. Do not add synonyms for reporting convenience. Deprecate rather than redefine published codes, and retain mappings from deprecated codes to their replacements.
