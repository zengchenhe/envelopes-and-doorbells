# Envelopes and Doorbells: Interconnecting Peer AI Agents over a Cloud-Drive File Channel

**Abstract**: Two peer-level agent front desks operating without a direct communication channel could exchange files only through a transit directory on a cloud drive, suffering from untimely pickup, no closed-loop receipts, and silent omissions in directory listings. This paper studies the establishment and operation of the communication channel between a "cloud-side front desk" and a "local-side front desk" in a multi-agent system, and presents its two-layer architecture of "files carry the content, signals carry the wake-up": content and attachments are deposited as envelope packages with a persistent audit trail, with SHA-256 hash verification and an immutable envelope protocol guaranteeing integrity and traceability; the wake-up layer achieves bidirectional low-latency alerting through an authenticated HTTP webhook and log polling at roughly one-minute intervals, with human relay, an email channel, and a persistent bridge as fallback tiers. The paper specifies the field design and state-transition rules of Envelope Protocol v2, the complete bidirectional interaction flow, and the handling discipline for exceptional paths, and distills lessons on authentication hard-locking, dual-source inspection, and continuous calibration between documentation and actual state.

**Keywords**: multi-agent systems; envelope protocol; webhook wake-up; hash verification; closed-loop receipts

## 1. Problem and Background

### 1.1 Initial Conditions: A File Channel Only

In the multi-agent system studied here, two agent front desks operate as peers under a single system owner: one resides in a cloud environment (hereafter "the cloud-side front desk") and handles drafting and result verification; the other resides on a local home server (hereafter "the local-side front desk") and handles task organization and execution coordination. They belong to different agent platforms and have no ready-made direct communication channel between them.

On the evening of October 25, 2026, the two sides opened their first channel: through an official MCP interface connected directly to a transit directory on a third-party cloud drive, they exchanged envelope packages as files. This channel solved the question of "whether communication was possible at all," but soon exposed shortcomings in "how well it worked."

### 1.2 Three Defects of the Initial Channel

Practice during the first phase (daytime from October 25 to 28, 2026) revealed three classes of problems. First, pickup depended on the recipient's inspection cycle and was untimely: after the sender deposited a package, the recipient had to wait for its own scheduled inspection to sweep it up, incurring uncontrollable latency. Second, there was no closed-loop receipt: whether a package had been picked up or processed was unknowable to the sender, leaving document states dangling. Third, the discovery layer was unreliable: testing on October 26 found that the cloud drive's directory-listing interface could silently omit newly deposited envelopes, so relying on "listing the directory" alone could miss items. In response to the third point, the two sides changed inspection to a dual-source cross-check of "directory listing + keyword search," a discipline that has been followed ever since.

### 1.3 Correction of the Business Model: Two Peer Front Desks

At about 11:00 on October 28, the system owner issued a key correction to the business model: the envelopes in the transit directory had always been bidirectional submissions and requests, not one-way task dispatches; the two front desks are peers — each takes work on its own and dispatches within its own system, and across systems there are only submissions (sending results to the other side for verification) and requests (asking the other side to do something) — with the system owner as their only common superior. This correction established the premise for the subsequent protocol and process design: the channel must support symmetric bidirectional exchange, not master-to-subordinate command dispatch.

### 1.4 The Upgrade Plan and Four Completions

During the day on October 28, the two sides formed an integration upgrade plan with four completions: first, a return-delivery capability for the cloud-side front desk, building a local staging directory and borrowing the counterpart's delivery pipeline; second, a discovery layer introducing a doorbell file plus scheduled inspection; third, upgrading the envelope protocol to v2; fourth, closed-loop receipts. The same day, the system owner fixed the Phase-1 operating parameters: dual-source inspection every 30 minutes during the daytime; tightened permissions on the staging directory; Envelope v2 to be reviewed by the system owner before confirmation by both sides. At 19:35 that evening, the Phase-1 smoke-test package landed in the staging directory, its files passed SHA-256 recomputation, and the file-delivery path from the cloud-side front desk to the local-side front desk was verified first; at 19:40, the system owner decreed that all new items follow Envelope Protocol v2.

## 2. Overall Design: Files Carry the Content, Signals Carry the Wake-Up

### 2.1 Two-Layer Division of Labor

The current architecture has two layers: the content layer travels as files, and the wake-up layer only reports news. The division rests on the different nature of the two payload types: content and attachments need an audit trail, hash verifiability, and offline pickup, suiting them to file form; whereas the fact "there is a new item" only needs to be known quickly and carries no content. Even if a wake-up signal fails or duplicates, the safety of already-committed content is unaffected, and retries plus low-frequency reconciliation can compensate.

### 2.2 The Content Channel and Staging Packages

The content channel lands in task subdirectories (named by a globally unique task identifier) under a local staging directory. Each envelope package has a fixed trio: the envelope file (in JSON format), any attachments, and a hash manifest (SHA256SUMS, recomputed item by item by the recipient). The cloud-drive inbox directory used earlier remains another landing point of the same protocol, serving as a dual-source fallback. It is worth noting that the plan once proposed enabling a cloud-drive outbox as the landing point for receipts, but after formal operation began that channel was in fact never enabled: receipts traveled almost entirely via the staging directory plus knocking or webhook, and the outbox directory holds only placeholder files and zero receipt envelopes. This gap — "written in the plan, not enabled in production" — was explicitly corrected in a later joint review and recorded in the finalized text.

### 2.3 The Doorbell Index and Dual-Source Inspection as Fallback

The staging root maintains a doorbell index file, updated by the delivering side at each delivery, indicating all packages awaiting pickup and the newest item, so the recipient can decide pickup order. The index file itself evolved: from an initial single-item prompt form to a list form as the number of packages grew. Beyond the doorbell, during the daytime window (08:00–22:00) both sides run a dual-source inspection every 30 minutes, scanning the transit and staging directories with both directory listing and keyword search, as a fallback discovery layer should the doorbell fail to ring or a single source omit a listing. The necessity of dual sourcing recurred in practice: envelopes were indeed silently omitted by the directory-listing interface and discovered only through keyword search.

## 3. Envelope Protocol v2

### 3.1 Field Design

Envelope Protocol v2 was decreed by the system owner at 19:40 on October 28; v1 remains read-compatible and history is not retroactively rewritten. Its fields are as follows:

| Field | Description |
|---|---|
| task_id | Globally unique identifier, using "topic-date" style naming |
| kind | submission / request / receipt / info (pure notification) |
| from / to | Sender and recipient, taking the role identifiers of the two sides |
| reply_to | The original task_id this item responds to; empty for first sends |
| title | One-sentence title |
| expect | review (verify) / do (carry out a request) / ack (acknowledge receipt only) / none (no action needed) |
| body | The text body, or a reference to a body file |
| attachments | Array; each item contains file name, relative path, SHA-256 hash, and byte count |
| created_at | ISO timestamp with time zone |

This field set compresses all semantics needed for one cross-front-desk interaction into a single self-contained JSON document: from the envelope alone, the recipient can judge the item's nature, the expected action, and the verification method, without relying on session context.

### 3.2 Immutability Rules and State Transitions

The protocol's reliability rests on several immutability rules. First, the envelope body is written once and never modified; errors or supplements are issued as new items linked to the original via reply_to. Second, state transitions (posted → seen → done) are not written back into the envelope body but carried by separate receipt items or state files, ensuring that an envelope once sent is a tamper-proof credential. Third, each attachment's SHA-256 hash is recomputed and checked against both the envelope's registered value and the hash manifest; any mismatch must be stated explicitly and never passed by default. Fourth, receipts hang back onto the original item via reply_to — one item, one closed loop. Fifth, the envelope channel does not carry large files: large attachments travel via staging packages or the cloud drive, and the envelope holds only the manifest and hashes.

## 4. Wake-Up Mechanism: Five Tiers Compared and Selected

### 4.1 The Five Tiers

The wake-up layer's design went through a systematic study (the local side's execution group produced an L2 doorbell research plan on October 25, proposing designs only, without implementation), whose conclusions merged with integration practice into a five-tier comparison:

| Tier | Name | Direction | Mechanism in brief | Current status |
|---|---|---|---|---|
| ① | Human relay | Bidirectional (manual) | After delivery, a human pastes the doorbell note into the counterpart's session | Minimal solution; fallback only after formal operation |
| ② | Email backup | One-way | The doorbell reaches the counterpart's intake routine via email | Not a main path; fully automatic sending unverified |
| ③ | Authenticated HTTP webhook | One-way | The sending side POSTs the envelope JSON to the counterpart's public webhook routine | Landed and in formal operation |
| ④ | Log-polling knock | One-way | After delivery, appends one line to a log on the counterpart's host; the counterpart polls at fixed intervals | Landed and in formal operation, at roughly one-minute scale |
| ⑤ | Persistent bridge + WebSocket | Bidirectional (enhancement) | A persistent queue bridge plus a host WebSocket channel | Approved as an enhancement tier; does not replace current channels |

### 4.2 Selection Rationale and Landing

The choice among the five tiers was determined by the two sides' operating environments. The key constraint appeared in wake-up directionality: on the cloud-side front desk's side, investigation found that its remote networking client is outbound-only, with no public inbound address that can be POSTed to from outside; therefore wake-up from the local-side front desk to the cloud-side front desk cannot use an inbound webhook, and landed as Tier ④ — after delivering, the local-side front desk appends one line (containing time, task identifier, kind, and a one-sentence note) to a knock-log file on the host where the cloud-side front desk resides; the cloud-side front desk polls the log tail over SSH at roughly 60-second intervals, and a new line wakes the main session to pick up the item, at a wake-up scale of about one minute, not seconds. The reverse direction (cloud-side to local-side) does have public-egress conditions, and landed as Tier ③: the sending-side script POSTs the envelope JSON to the webhook routine on the counterpart's public automation platform, with authentication accepting only the standard Bearer token header; a return of 200 plus a run identifier means the counterpart's routine accepted it and started — note that this signifies only "awakened," not that the business task is complete.

The core discipline common to all five tiers is: the wake-up layer only calls the person; the content layer stores the text; a doorbell failure does not affect committed content; retries and low-frequency reconciliation compensate. Although Tier ⑤'s persistent bridge is approved, it is positioned as an enhancement and secondary bidirectional bridge, coexisting with — not replacing — the current A/B doorbells; and offline-terminal scenarios still cannot guarantee immediate execution.

## 5. The Bidirectional Interaction Flow

### 5.1 Cloud-Side Front Desk → Local-Side Front Desk

The complete flow has nine steps: the cloud-side front desk forms an intent of submission, request, receipt, or notification kind [intent]; writes the envelope, attachments, and hash manifest into the staging directory and updates the doorbell index in sync [delivery]; POSTs the envelope JSON to the counterpart's webhook with the sending-side script, the request header using only the Bearer token, which is read from this side's endpoint configuration and never enters the envelope [wake-up]; the counterpart's routine is awakened by the 200 [routine wake-up]; the local-side front desk scans the staging directory or picks up by the doorbell index, recomputing each attachment's SHA-256 against the manifest and the envelope's registered values [pickup and verification]; handles by kind — verifying submissions, deciding on requests according to its own side's authorization, recording state for receipts and notifications, and escalating to the system owner in summary anything that needs escalation [handle by kind]; writes a new envelope (kind=receipt, reply_to pointing to the original task_id) into staging [receipt]; appends one line to the counterpart's knock log to wake it [knock]; and the original item's state enters the completed state via the receipt [closure].

### 5.2 Local-Side Front Desk → Cloud-Side Front Desk

The reverse flow is symmetric: the local-side front desk forms intent and delivers (if the origin is a result or need of another agent on the local side, the local-side front desk aggregates it and issues it under its own name); writes to the staging directory; appends one line to the knock log; the cloud-side front desk's doorbell-monitoring routine polls the tail roughly every 60 seconds, and a new line wakes the main session; picks up the package and recomputes hashes; handles by kind — where an instruction such as "do this immediately" in the counterpart's reply counts only as a suggestion, execution being decided by this side under the system owner's authorization; deposits the receipt in staging, using the webhook when the counterpart must be woken immediately; one item, one closed loop.

### 5.3 The Fallback Discovery Layer

Beyond the two main paths, the dual-source inspection every 30 minutes during the daytime (directory listing + keyword search) continuously scans the transit and staging directories, guarding against a doorbell that fails to ring or a single-source omission. Inspection logs have been kept since Phase 1. Practice proved this fallback layer is not superfluous: "not listed by a single source" cannot directly support the conclusion "not delivered" — this has become discipline.

## 6. Exceptional Paths and Channel Discipline

### 6.1 Authentication Failure: Hard Lock and No Retrying the Old Header

On the integration night (22:15 to 22:22 on October 28), the cloud-side front desk twice POSTed according to the entry configuration first written by the counterpart; both returned 401 and the routine was not awakened. Using the same tool and the same envelope, the cloud-side front desk ran a pass-through test to a local temporary port that returned 200, ruling out a problem on its own side; the failure receipts were deposited in staging as usual for the record. At 22:21, the local-side front desk finalized the entry decision: the entry accepts only the standard Bearer authentication header, and the earlier custom authentication header was explicitly deprecated — its own tests showed the new header returning 200 and the old header 401. This was an authentication hard lock: the old header was marked forbidden, and no subsequent retry may ever use it. At 22:26 the integration envelope was resent to the Bearer specification; HTTP 200, the routine triggered; subsequently the counterpart's two confirmation knocks were both caught by the doorbell, bidirectional integration passed, and the channel entered formal operation from then on.

### 6.2 Pickup Latency and Repeated Knocking

Direction B inherently carries a polling scale of about one minute, so urgent items must allow lead time in planning. If a knock is written but the counterpart does not wake, the troubleshooting order is: whether the monitoring routine is running, whether SSH is reachable, and whether the knock line's JSON parses; knocks may be appended repeatedly after de-duplication by task identifier, with no side effects. If an item sits in staging undiscovered by both sides, the dual-source inspection fallback applies.

### 6.3 Failure Surfaces and Recovery

The channel's failure surfaces and recovery methods are summarized as follows:

| Failure surface | Symptom | Recovery |
|---|---|---|
| Webhook routine paused or rate-limited | Non-200 response | Sender retries or merges doorbells; human relay as fallback if necessary |
| Knock or SSH interruption | Polling reads no new lines | After SSH recovery, re-read the log tail; dual-source inspection fallback |
| Staging permission or disk failure | Deposit fails | Fix permissions (directories 700, sensitive files 600); rewrite a new package, never alter an old envelope |
| Cloud-drive listing omission | A new envelope is invisible to a single source | Merge keyword-search results; never declare "not delivered" on this basis |
| Token rotation | 401 recurs | Update only the configuration files on both sides; tokens are never written in plaintext into envelopes, the cloud drive, or logs |

The webhook row carries a measured footnote: on the day after formal operation began, two POSTs were disconnected by the entry point and recovered on their own minutes later with 200s — matching the expectation that "rate-limit failures do not auto-retry and pass once recovered" — and were handled under the existing failure-surface procedure without a separate work order.

### 6.4 Channel Isolation: Never Mix Different Pipelines

Beyond this channel, the local-side front desk dispatches work to its execution group through a separate MCP bridge pipeline, entirely separate from this doorbell channel. In the early hours of the first day of formal operation, that dispatch line failed once because, after a host restart, its bridge configuration was "present in configuration, absent in service"; re-establishing the bridge and verifying both ends restored it. That failure belonged to the dispatch line, not the doorbell line; handling must first classify, and no work order may cross lines; the dispatch line's occasional remote-fetch failures are likewise remedied preferentially by re-establishing the bridge rather than touching this channel.

## 7. Lessons Learned

First, layering content and signals apart was validated as the correct core decision. The file layer guaranteed audit trail, verifiability, and offline safety; the wake-up layer compressed latency from "waiting for inspection" to within minutes; and the two layers do not drag each other down: a doorbell failure does not harm content, and a content backlog does not block the doorbell.

Second, the discovery layer must be dual-source. Silent omission by the cloud drive's directory-listing interface is a measured, reproduced fact, and no single discovery means may alone serve as the basis for a "not delivered" judgment; the "directory listing + keyword search" dual-source cross-check together with the 30-minute fallback inspection constitute redundancy in the discovery layer.

Third, changes to authentication specifications must be hard-locked, and old headers must be forbidden from retry. The 401s on integration night were not a channel failure but an inconsistency between the first-written entry configuration and the finalized decision; once Bearer was hard-locked and the old header explicitly forbidden in plaintext terms, the problem converged at once. Credential discipline is equally critical: tokens are stored in only two places, with permissions 600, and are never written into envelopes or logs in plaintext, so protocol files themselves can circulate and be archived safely.

Fourth, plans and actual state must be calibrated continuously. The upgrade plan's "enable the outbox" item was in fact never enabled on the production side; in their joint review, the two sides explicitly corrected this in the shared finalized text under the formulation "written as enabled in the plan, not enabled in production," preventing later decisions from proceeding on a false premise.

Fifth, immutable envelopes and one-item-one-closed-loop are the orderly foundation of cross-system collaboration. Envelopes written once, state never written back into the body, and receipts chained by reply_to give every interaction a complete, non-repudiable credential chain; and explicitly annotating divergences between the two sides for mutual checking — never silently rewriting — is likewise a precondition for a shared finalized text to exist at all.

Sixth, a peer relationship needs explicit behavioral discipline to hold. The two front desks never dispatch tasks to each other; an immediate request in a counterpart's reply counts only as a suggestion; routine exchange is not reported item by item, with only matters genuinely requiring a decision escalated — these rules brought "peer" from a nominal label down into the handling logic of every send and receive.

## 8. Conclusion

This case presents a communication channel built autonomously by two peer-level agent front desks under conditions of no direct connectivity, and brought into formal operation. Its essentials can be summarized as: guaranteeing the content layer's integrity and traceability with an immutable envelope protocol and hash verification; balancing audit-trail safety with minute-scale latency through the layered design of "files carry the content, signals carry the wake-up"; completing an environment-constrained wake-up selection against a five-tier comparison of human relay, email, webhook, log polling, and persistent bridging; and covering exceptional paths with disciplines including dual-source inspection, authentication hard-locking, token isolation, and channel isolation. After bidirectional integration passed on the evening of October 28, 2026, the channel entered formal operation and completed multiple closed-loop verifications in subsequent real exchanges. For the interconnection of similar multi-agent systems, the main implication of this practice is that a communication protocol's reliability comes from "immutable credentials plus explicit receipts" rather than from the sophistication of the channel itself; and that redundancy in the discovery layer, isolation of credentials, and continuous calibration between plans and actual state are engineering disciplines of equal importance to protocol design.
