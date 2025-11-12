# Cross-Chain Bridge Security Mega Checklist

A unified security checklist covering **LayerZero**, **Wormhole**, and **General Bridge Security**.  
It’s designed for smart contract developers, security researchers, and auditors analyzing cross-chain communication protocols and bridge architectures.

## Structure
1. **LayerZero Security Checklist** – for omnichain messaging, EndpointV2, OFT, lzRead/lzReceive logic.  
2. **Wormhole Security Checklist** – for guardian-based bridges relying on multisig verification and VAAs.  
3. **Bridge Security Review Checklist (General)** – for evaluating bridge architecture, governance, and economic risk.

Use this document as a practical, audit-ready framework to systematically review bridge protocols and cross-chain integrations.

---

# LayerZero Security Checklist

A practical, audit-focused checklist for reviewing LayerZero integrations and applications (EndpointV2, lzRead, lzReceive, OFT, lzCompose, Executor/options).  
Intended for developers and auditors assessing cross-chain messaging reliability, nonce handling, and execution safety in omnichain environments.

---

## 1. EndpointV2 / Send / Receive Validation

- [ ] Validate `refundReceiver` in `send()` for reentrancy, DoS or gas griefing.  
  **Why:** RefundReceiver receives ETH refunds and can re-enter or consume gas unexpectedly.

- [ ] Enforce `lzReceive` callable only by Endpoint (check `msg.sender == endpoint`).  
  **Why:** Prevents arbitrary contracts from triggering receive logic and corrupting inbound state.

- [ ] Confirm `isValidReceiveLibrary()` and send/receive library configuration are set and locked.  
  **Why:** Prevents fallback to unsafe defaults controlled by LayerZero.

- [ ] Ensure lower nonces are processed before higher ones to avoid `_clearPayload` OOG loops.  
  **Why:** High nonces trigger excessive iterations when earlier nonces remain unprocessed.

- [ ] Restrict `skip()` usage to authorized roles only.  
  **Why:** Skipping nonces without verification can desynchronize inbound state.

- [ ] Validate inbound payload hashes before lazy nonce increments.  
  **Why:** Missing hashes block message processing until verified.

---

## 2. lzCompose / Composed Message Execution

- [ ] Validate `_from` equals the original OFT contract address in `lzCompose`.  
  **Why:** Prevents spoofed composed message execution by unknown senders.

- [ ] Check `msg.sender == EndpointV2` for all composed message executions.  
  **Why:** Ensures only verified endpoints execute composed payloads.

- [ ] Invalidate `composeQueue` entries after execution.  
  **Why:** Prevents replay of composed messages.

- [ ] Handle exceptions in `ILayerZeroComposer(_to)` safely.  
  **Why:** Avoids cascading reverts or locked funds during destination execution.

---

## 3. Execution Options, Gas, and Refunds

- [ ] Validate execution options for gas and `msg.value` correctness.  
  **Why:** Off-chain executors can invoke with arbitrary values, breaking assumptions.

- [ ] Embed `msg.value` in payload and enforce on receive.  
  **Why:** Prevents mismatch between intended and actual value transferred.

- [ ] Whitelist executors or enforce safe gas thresholds.  
  **Why:** Stops under-gassed executions that block future messages.

- [ ] Combine user-supplied options with `enforcedOptions`.  
  **Why:** Guarantees minimum gas and fee requirements for all executions.

- [ ] Audit refund logic for DoS or gas grief potential.  
  **Why:** RefundReceiver can execute arbitrary logic, blocking message loops.

---

## 4. Ordered Execution and Nonce Handling

- [ ] Implement `nextNonce()` correctly in ordered OApps.  
  **Why:** Incorrect logic blocks message queues indefinitely.

- [ ] Test ordered vs unordered modes for revert behavior.  
  **Why:** Ensures a failed message doesn’t stall the system.

- [ ] Provide recovery or manual override for stuck nonces.  
  **Why:** Enables restoring message flow after failed transactions.

---

## 5. OFT (Omnichain Fungible Token) Logic

- [ ] Call `_removeDust()` after applying fees or conversions.  
  **Why:** Maintains precision alignment between local and shared decimals.

- [ ] Check consistent `sharedDecimals` and `localDecimals` across chains.  
  **Why:** Prevents rounding mismatches between networks.

- [ ] Prevent transfers larger than `uint64.max` when decimals match.  
  **Why:** `_toSD()` truncates silently, causing value loss.

- [ ] Validate decimal conversion rates before transfers.  
  **Why:** Wrong conversion rate causes incorrect supply and token loss.

---

## 6. LayerZero Read (lzRead) Integrity

- [ ] Fuzz `lzRead` targets to ensure they never revert.  
  **Why:** A revert blocks all subsequent messages for that nonce.

- [ ] Set sufficient confirmations for target chain finality.  
  **Why:** Low confirmations risk accepting data from reorged blocks.

- [ ] Match `returnDataSize` exactly to the returned byte size.  
  **Why:** Mismatch reverts the lzRead response.

- [ ] Validate `targetEid` for intentional cross/same-chain reads.  
  **Why:** Prevents data leakage or infinite loop reads.

- [ ] Protect `skip()` logic for failed reads with access control.  
  **Why:** Unauthorized skips allow censorship or corruption.

---

## 7. Library & Immutability Management

- [ ] Explicitly configure and lock send/receive libraries.  
  **Why:** Defaults can change, bricking unconfigured protocols.

- [ ] Verify `MessageLibManager` mappings for send/receive correctness.  
  **Why:** Ensures consistent routing of cross-chain messages.

- [ ] Only use trusted libraries (SendUln302, ReceiveUln302, etc.).  
  **Why:** Rogue libraries can alter validation flow.

- [ ] Lock library configuration post-deployment.  
  **Why:** Ensures immutability of protocol communication.

---

## 8. Executor and Native Airdrop Controls

- [ ] Validate `nativeCap` and per-chain Executor configs.  
  **Why:** Prevents excessive native token distribution.

- [ ] Check Executor gas settings (`lzReceiveBaseGas`, `lzComposeBaseGas`).  
  **Why:** Misconfigurations cause underpayment or failed executions.

- [ ] Monitor Executor performance and refunds.  
  **Why:** Detects gas grief or payment mismatches in production.

---

## 9. Common Gotchas

- RefundReceiver can grief execution via reentrancy.  
- Ordered execution blocks all nonces if one fails.  
- `_toSD` truncation silently loses token precision.  
- `lzRead` results represent past state; handle time gaps.  
- Do not assume tokens return bools for mint/burn.  
- Library defaults can change; always configure explicitly.

---


# Wormhole Security Checklist

A technical checklist for auditing Wormhole-based protocols, guardian-verified bridges, and cross-chain message handlers.  
Focuses on guardian set validation, VAA processing, token normalization, replay protection, and relayer integrity.

---

## 1. Core Trust Model

- [ ] Understand the guardian-based security assumption (13/19 signatures).  
  **Why:** Bridge integrity depends entirely on the guardian committee.

- [ ] Treat relayers as untrusted.  
  **Why:** They can drop or reorder VAAs but cannot forge them.

- [ ] Implement replay protection for VAAs.  
  **Why:** Prevents multiple executions of the same message.

- [ ] Verify VAAs on-chain before use.  
  **Why:** Prevents execution of fake or tampered VAAs.

---

## 2. Message Publication

- [ ] Use unique nonces per sender.  
  **Why:** Duplicate nonces enable replay attacks.

- [ ] Ensure payload encoding/decoding consistency.  
  **Why:** Mismatched encoding leads to corrupted data.

- [ ] Adjust `consistencyLevel` based on chain finality.  
  **Why:** Low levels risk reorg-based invalidations.

- [ ] Always fetch live fee values via `messageFee()`.  
  **Why:** Prevents outdated hardcoded fees from causing failures.

---

## 3. Fee and Relayer Handling

- [ ] Pass correct value to `publishMessage()` at runtime.  
  **Why:** Underpaying leads to undelivered messages.

- [ ] Separate relayer fees from protocol message fees.  
  **Why:** Clarifies accounting and prevents overcharging.

- [ ] Validate relayer authenticity using `isRegisteredSender`.  
  **Why:** Prevents unauthorized message triggers.

- [ ] Cross-check `msg.sender` equals the Wormhole relayer.  
  **Why:** Stops arbitrary contracts from invoking callbacks.

---

## 4. Token Normalization and Bridging

- [ ] Handle normalization and denormalization correctly.  
  **Why:** Prevents trapped dust or incorrect conversions.

- [ ] Avoid double normalization.  
  **Why:** Redundant scaling introduces balance mismatches.

- [ ] Enforce token-specific max transfer limits.  
  **Why:** Avoids overflow on smaller decimal tokens.

- [ ] Validate per-token decimals dynamically.  
  **Why:** Prevents wrong scaling for non-standard tokens.

---

## 5. Guardian Set and Signature Verification

- [ ] Ensure overlapping guardian sets coexist during transitions.  
  **Why:** Avoids downtime between rotations.

- [ ] Validate guardian set size > 0 during upgrades.  
  **Why:** Empty sets disable message verification.

- [ ] Enforce unique guardian signatures per message.  
  **Why:** Duplicate indices can spoof quorum.

- [ ] Verify signatures match known guardian keys.  
  **Why:** Prevents forged guardian approvals.

- [ ] Enforce quorum (>⅔) before message acceptance.  
  **Why:** Maintains bridge-level security assumptions.

---

## 6. Governance & Upgrade Security

- [ ] Quorum verification required for all upgrades.  
  **Why:** Ensures legitimate guardian-signed governance VAAs.

- [ ] Replay protection for governance VAAs.  
  **Why:** Prevents reusing old upgrade messages.

- [ ] Check module IDs in upgrade payloads.  
  **Why:** Avoid accepting arbitrary governance actions.

---

## 7. Monitoring and Recovery

- [ ] Track guardian rotation and quorum on-chain.  
  **Why:** Detects compromised or inactive guardians.

- [ ] Log duplicate message attempts.  
  **Why:** Aids in detecting replay or relayer manipulation.

- [ ] Maintain replay and relayer metrics.  
  **Why:** Supports forensic tracking post-incident.

---


# Bridge Security Review Checklist

Comprehensive checklist for evaluating bridge architecture, validator design, economic assumptions, and operational resilience.  
For security reviewers and DeFi teams integrating or auditing L1/L2 cross-chain bridges.

---

## 1. Bridge Design and Architecture

- [ ] Define bridge type: Lock & Mint / Burn & Mint / Liquidity Pool / Hybrid.  
  **Why:** Architecture dictates custody model and attack surface.

- [ ] Verify asset peg logic for wrapped tokens.  
  **Why:** Broken pegs lead to insolvency or over-minting.

- [ ] Ensure redemption path matches minting logic.  
  **Why:** Prevents supply imbalance across chains.

---

## 2. Validator / Attester Security

- [ ] Governance must manage validator sets, not a single entity.  
  **Why:** Decentralizes trust and limits insider risk.

- [ ] Regular key rotation with proof.  
  **Why:** Mitigates long-term key compromise.

- [ ] Unique validator signatures per message.  
  **Why:** Prevents replay of approvals.

- [ ] On-chain visibility for validator changes.  
  **Why:** Enables transparency and tracking.

- [ ] Define quorum and penalty mechanism.  
  **Why:** Discourages collusion and downtime.

---

## 3. Economic Security

- [ ] Quantify total value locked (TVL) and secured historically.  
  **Why:** Contextualizes risk scale.

- [ ] Define cost-to-corrupt (validator bribing, key compromise).  
  **Why:** Estimates system resilience.

- [ ] Limit exposure per chain or token.  
  **Why:** Caps losses from weak chains.

- [ ] Create fallback withdrawal mechanisms.  
  **Why:** Allows partial recovery after compromise.

---

## 4. Message Validation and Resilience

- [ ] Explicitly state verification type: External / Native / Optimistic / Canonical.  
  **Why:** Defines trust boundaries and latency model.

- [ ] Validate proof replay and expiration windows.  
  **Why:** Prevents replays post-fork or upgrade.

- [ ] Simulate reorg and latency attacks in testing.  
  **Why:** Measures behavior under unstable consensus.

---

## 5. Governance and Control

- [ ] Use DAO or multi-sig for upgrade rights.  
  **Why:** Prevents unilateral configuration changes.

- [ ] Apply time-locks on upgrades.  
  **Why:** Gives users time to react.

- [ ] Restrict pause/emergency controls via verified roles.  
  **Why:** Prevents abuse or censorship.

---

## 6. Monitoring and Incident Response

- [ ] Monitor guardian/validator uptime and message failure metrics.  
  **Why:** Detects degradation in live conditions.

- [ ] Emit logs for every mint, burn, and validation.  
  **Why:** Enables traceability and forensics.

- [ ] Maintain public communication plan for exploits.  
  **Why:** Reduces panic and improves recovery transparency.

- [ ] Document rollback and freeze procedures.  
  **Why:** Ensures structured mitigation in emergencies.

---
