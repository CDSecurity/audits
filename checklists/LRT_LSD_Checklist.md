# Liquid Restaking and Liquid Staking Derivatives Checklist

A comprehensive audit checklist for teams developing or reviewing **Liquid Staking (LSD)** and **Liquid Restaking (LRT)** protocols.  
Use this to validate protocol safety, accounting integrity, and resilience against current staking and restaking vulnerabilities.

---

## 1. General Design

- [ ] Is the protocol design and threat model documented and up to date?  
  Why: auditors must know intended invariants and actor powers.  

- [ ] Are assumptions about external systems (Beacon Chain, EigenLayer, oracles) explicitly defined and verified?  
  Why: prevents blind trust in dynamic external inputs.  

- [ ] Are upgrade and governance paths time-locked and documented?  
  Why: prevents accidental or malicious parameter changes.  

- [ ] Are emergency controls (pause, recovery, upgrade) restricted to verified roles?  
  Why: limits damage during exploits.  

---

## 2. Deposit and Validator Registration

- [ ] Does the deposit flow validate that the correct `deposit_root` or proofs are used when calling the deposit contract?  
  Why: prevents stale or forged deposits.  

- [ ] Can a malicious validator front-run setting withdrawal credentials?  
  Why: A validator can front-run a deposit with their own withdrawal credentials using 1 ETH to hijack the withdrawal address.  

- [ ] Are partial deposits and top-ups correctly handled?  
  Why: ensures attackers cannot set credentials with a small initial deposit.  

- [ ] Are `create2` deterministic address calculations stable under compiler or metadata changes?  
  Why: avoids broken withdrawal vaults due to changing bytecode metadata.  

- [ ] Are batching mechanisms gas-bounded and safe under heavy load?  
  Why: prevents gas exhaustion and DoS during deposit bursts.  

- [ ] Are validator activations and reward accruals aligned with current Beacon Chain consensus rules?  
  Why: avoids accounting drift between deposits and live validators.  

---

## 3. Delegation and Operator Logic

- [ ] Can inter-related storage be corrupted, especially storage related to operators or validators?  
  Why: Multi-structure storage can become inconsistent if updates are incomplete or overwritten.  

- [ ] Does the protocol iterate over the entire set of operators or validators?  
  Why: Large iteration loops can exhaust gas or open DoS vectors.  

- [ ] Are forced undelegations properly bounded by cooldowns?  
  Why: prevents immediate redelegation abuse.  

- [ ] Are operator rotations authenticated and event-logged?  
  Why: ensures transparent operator lifecycle changes.  

- [ ] Can delegation approval or revocation be manipulated by an untrusted party?  
  Why: prevents unauthorized operator swaps.  

---

## 4. Withdrawals and Redemption

- [ ] Are Beacon Chain withdrawals compatible with post-Deneb (EIP-4844) proof formats?  
  Why: ensures the protocol correctly processes withdrawals after the Dencun upgrade (March 2024).  

- [ ] Are withdrawal queue entries unique and replay-protected?  
  Why: avoids double-execution of queued withdrawals.  

- [ ] Can the exchange rate repricing update be sandwich-attacked to drain ETH?  
  Why: If repricing occurs without queue batching, attackers can front-run deposits and back-run withdrawals for profit.  

- [ ] Is reentrancy possible during withdrawals, reward claims, or NFT mints?  
  Why: Reentrancy in `_safeMint` or ETH sends can drain contract balance.  

- [ ] Are paused states enforced for all restricted actions?  
  Why: Missing `whenNotPaused` modifiers allow unsafe actions while paused.  

- [ ] Is partial withdrawal accounting precise and consistent with exchange rates?  
  Why: incorrect share redemption can overpay or underpay users.  

---

## 5. Restaking and EigenLayer Integration

- [ ] Are restaked assets fully collateralized by verified ETH or LSTs?  
  Why: ensures no synthetic over-leverage.  

- [ ] Do restaking credentials match validator assignments across Beacon and EigenLayer?  
  Why: prevents lost delegation records and reward mismatches.  

- [ ] Are AVS operator approvals validated before restaking?  
  Why: blocks unauthorized use of delegated stake.  

- [ ] Are EigenLayer withdrawals reconciled to prevent double-counting?  
  Why: avoids inflating TVL or user balances.  

- [ ] Does restaking logic handle native ETH and LST deposits safely?  
  Why: both asset types must have identical accounting logic.  

- [ ] Is EigenLayer slashing integration tested and enabled where applicable?  
  Why: as of 2025, EigenLayer slashing support is partially active and must be handled correctly.  

---

## 6. Accounting and Token Economics

- [ ] Can an arbitrary exchange rate be set when processing queued withdrawals?  
  Why: Arbitrary rate setting during redemption can rug user balances.  

- [ ] Does unnecessary precision loss occur in deposit, withdrawal, or reward calculations?  
  Why: Division before multiplication or double rounding causes token inflation or deflation.  

- [ ] Are derivative tokens minted or burned strictly based on verified ETH balance?  
  Why: ensures solvency and invariant maintenance.  

- [ ] Are inflation or top-up events prevented from artificially inflating share price?  
  Why: avoids yield manipulation via timing attacks.  

---

## 7. Reentrancy and External Calls

- [ ] Are all ETH transfers or external calls protected against reentrancy?  
  Why: prevents recursive balance drains.  

- [ ] Are shared-state contracts isolated from one another’s reentry paths?  
  Why: multi-contract systems can bypass nonReentrant guards.  

- [ ] Do callback-capable external contracts lack state-changing privileges?  
  Why: prevents unauthorized state mutations mid-call.  

---

## 8. Gas and Loop Safety

- [ ] Are loops over validators or operators bounded or batched?  
  Why: unbounded loops can trigger DoS via gas exhaustion.  

- [ ] Can a missing array increment or validation cause an infinite loop?  
  Why: Missing iterator increments in nested loops can halt execution.  

- [ ] Does the protocol handle large array data safely without exceeding block gas limits?  
  Why: prevents halts under stress conditions.  

---

## 9. Proofs, Oracles, and Beacon Data

- [ ] If using a Proof of Reserves Oracle, does the protocol check for stale data?  
  Why: Outdated off-chain oracle data can be processed as current.  

- [ ] Can Beacon Chain state root manipulation cause invalid proofs to pass?  
  Why: incorrect state roots can approve fake deposits or withdrawals.  

- [ ] Are Merkle proof verifiers flexible to tree depth or schema changes?  
  Why: prevents post-upgrade proof validation failures.  

- [ ] Is proof validation fuzz-tested using real Beacon mainnet data?  
  Why: identifies malformed proof edge cases.  

---

## 10. Slashing and Finalization

- [ ] Are slashing events propagated to delegators and operators deterministically?  
  Why: inconsistent propagation creates insolvency risk.  

- [ ] Can malicious operators avoid slashing by griefing or early withdrawals?  
  Why: prevents accountability bypass.  

- [ ] Are slashing insurance or coverage modules implemented and tested?  
  Why: mitigates total loss for delegators.  

- [ ] Is validator exit logic aligned with consensus finalization rules (post-Dencun)?  
  Why: ensures consistent ETH accounting during exits in current mainnet conditions.  

---

## 11. Economic and Market Manipulation

- [ ] Can attackers manipulate TVL, share value, or token price via flash loans?  
  Why: prevents temporary economic distortions.  

- [ ] Are cooldowns and penalties immune to timing or sandwich exploits?  
  Why: prevents arbitrage abuse of withdrawal timing.  

- [ ] Is reward recalculation atomic and protected from frontrunning?  
  Why: avoids yield misallocation.  

---

## 12. Access Control and Governance

- [ ] Are privileged roles minimal and secured via multisig or DAO governance?  
  Why: prevents single-key protocol compromise.  

- [ ] Are role modifiers correctly enforced across the codebase?  
  Why: avoids unauthorized execution of admin-only functions.  

- [ ] Are upgrade and emergency functions restricted and logged?  
  Why: ensures traceability of sensitive actions.  

- [ ] Are governance proposals subject to timelocks and quorum requirements?  
  Why: protects users from rushed parameter changes.  

---

## 13. Testing, Monitoring, and Upgrades

- [ ] Do tests cover staking, withdrawal, and restaking workflows end-to-end?  
  Why: validates state transitions under live 2025 conditions.  

- [ ] Are fuzz and property tests run for arithmetic, proof, and queue logic?  
  Why: detects non-deterministic or edge-case behavior.  

- [ ] Are monitoring tools comparing on-chain ETH vs. internal accounting?  
  Why: enables early detection of accounting drift.  

- [ ] Are post-upgrade contracts regression-tested against historical vulnerabilities?  
  Why: prevents reintroducing known exploits in current Ethereum environment.  

---

### Usage

This checklist is designed for internal and public security reviews of LSD and LRT systems.  
Use it to guide audits, continuous security monitoring, and public disclosure readiness.  
