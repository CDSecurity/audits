# Uniswap v4 Hooks Comprehensive Security & Design Checklist

A unified checklist for **auditors and developers** working with Uniswap v4 Hooks.  
Each check includes *why* it matters - helping identify the precise failure mode or exploit it prevents.

---

## 1. Hook Configuration & Permission Encoding

- [ ] **Are all permission bits correctly encoded in the Hook address?**  
  Why: Incorrect flag encoding can cause missing callbacks or DoS during swap/modify operations.

- [ ] **Does the Hook inherit from BaseHook?**  
  Why: BaseHook ensures permission correctness and consistent callback behavior.

- [ ] **Are all intended flags (before/after swap, add/remove liquidity, donate) included at deployment?**  
  Why: Missing bits cannot be added post-deployment, breaking intended Hook logic.

- [ ] **Is the Hook deployed using CREATE2 mining to ensure proper permission bits?**  
  Why: PoolManager checks permission bits directly from the address, not the contract code.

- [ ] **Are placeholder functions defined for future upgrades?**  
  Why: Prevents PoolManager from rejecting newly added callbacks post-upgrade.

- [ ] **Does the Hook implement correct return types and selector signatures?**  
  Why: Wrong tuple order or types will cause reverts or corrupted delta accounting.

---

## 2. PoolManager Interaction & Delta Accounting

- [ ] **Does the Hook correctly handle BeforeSwapDelta and BalanceDelta direction (sign)?**  
  Why: Misinterpreted deltas lead to wrong token flow or liquidity misallocation.

- [ ] **Does delta handling align with swap direction (zeroForOne vs oneForZero)?**  
  Why: Prevents asymmetric accounting between token0/token1.

- [ ] **Are all deltas settled before `unlock()` finalizes?**  
  Why: NonzeroDeltaCount > 0 will revert the entire transaction.

- [ ] **Does the Hook always return NonzeroDeltaCount = 0 post-unlock?**  
  Why: Unsettled Hook deltas trigger permanent DoS on swap completion.

- [ ] **Are HookDelta values bounded to prevent swap inversion (exact in/out)?**  
  Why: Violating Uniswap’s swap invariants leads to unexpected execution reverts.

- [ ] **Does the Hook call `settle()`, `take()`, or `settleFor()` properly?**  
  Why: Unsettled balances create stuck funds or reverts in PoolManager.

---

## 3. Async Hook Risks

- [ ] **Does the Hook take custody of swapped tokens (async behavior)?**  
  Why: Async Hooks override Uniswap’s swap logic and can lead to fund loss.

- [ ] **Are all async Hooks returning swapped tokens to PoolManager before transaction end?**  
  Why: Prevents token theft or permanent fund lockup.

- [ ] **Are transfers made only to PoolManager, not arbitrary addresses?**  
  Why: Prevents direct drain by malicious Hooks.

- [ ] **Are `settle()` or `settleFor()` used instead of manual ERC20 transfers?**  
  Why: Ensures PoolManager state consistency and avoids double accounting.

---

## 4. Reentrancy, Native Assets & CEI Pattern

- [ ] **Is `nonReentrant` or equivalent applied to all external callbacks?**  
  Why: Hooks run in complex callback chains where reentrancy can bypass checks.

- [ ] **Are ERC777 or ERC677 tokens sandboxed or blocked?**  
  Why: Their callbacks can reenter PoolManager during transfers.

- [ ] **Are native token (`msg.value`) flows handled safely with refunds?**  
  Why: Prevents trapped ETH or unauthorized withdrawals.

- [ ] **Is the Checks-Effects-Interactions (CEI) pattern enforced before any token transfer?**  
  Why: Ensures state is updated before any external interaction.

---

## 5. Access Control & Authorization

- [ ] **Are all entry points (`before*`, `after*`) restricted to `onlyPoolManager`?**  
  Why: Prevents direct calls by attackers simulating PoolManager.

- [ ] **Does the Hook verify that `msg.sender` equals the PoolManager for that pool?**  
  Why: Guards against spoofed calls from external contracts.

- [ ] **Can the Hook only be initialized once per pool?**  
  Why: Prevents reinitialization by another pool or exploit through duplicate linkage.

- [ ] **If multi-pool, is state isolated per pool key or ID?**  
  Why: Shared state between pools leads to balance corruption.

- [ ] **Are all public/admin functions properly restricted (no arbitrary `setFee`/`updatePool`)?**  
  Why: Prevents unauthorized configuration or fund drain.

---

## 6. Governance, Centralization & Upgradeability

- [ ] **If using UUPS or proxy pattern, is upgrade authority gated by timelock or multisig?**  
  Why: Prevents instant malicious upgrades or rug pulls.

- [ ] **Are storage layouts preserved across upgrades?**  
  Why: Storage misalignment can overwrite critical data.

- [ ] **Is there an immutable governance mechanism for upgrades and parameter changes?**  
  Why: Enforces predictable and transparent change management.

- [ ] **Can owner modify swap fees or execution logic arbitrarily?**  
  Why: Centralized fee control allows censorship or fund extraction.

- [ ] **Are admin functions limited to parameter tuning and not swap execution?**  
  Why: Protects against governance abuse during live trading.

---

## 7. Gas Efficiency & Denial-of-Service Risks

- [ ] **Are all loops bounded or capped?**  
  Why: Prevents DoS from excessive gas usage.

- [ ] **Is Hook logic optimized to remain below block gas limits under load?**  
  Why: Ensures pool remains usable under stress.

- [ ] **Are external contract calls wrapped in try/catch with revert handling?**  
  Why: Prevents ungraceful halts from third-party failures.

- [ ] **Are revert conditions reviewed for non-critical dependencies (oracle, price, access)?**  
  Why: Avoids Hooks blocking swaps unnecessarily.

---

## 8. Dynamic Fee Logic

- [ ] **Is LP fee override value within valid range (≤ 100%)?**  
  Why: Prevents total value loss from invalid fee percentages.

- [ ] **Does dynamic fee logic use validated input and safe math?**  
  Why: Overflow or rounding bugs can block swaps or drain liquidity.

- [ ] **Is fee logic deterministic across same block and conditions?**  
  Why: Prevents MEV or manipulation from varying fee behavior.

- [ ] **Are dynamic fee changes restricted to authorized hooks or admins?**  
  Why: Prevents hostile modification of pool economics.

---

## 9. Oracle & Market Data Dependencies

- [ ] **Are external oracles validated for freshness and trust?**  
  Why: Prevents stale or manipulated data driving incorrect swap logic.

- [ ] **Are fallback or sanity checks implemented for oracle responses?**  
  Why: Ensures Hook doesn’t revert on temporary oracle failure.

- [ ] **If using block.timestamp, is it resistant to miner manipulation?**  
  Why: Block timestamps can be exploited for predictable fee timing.

- [ ] **Are price-dependent fee adjustments MEV-hardened (TWAP/median)?**  
  Why: Reduces risk of sandwich attacks manipulating fee outcome.

---

## 10. State Management & Invariant Safety

- [ ] **Are pool variables isolated in mappings keyed by PoolId or PoolKey?**  
  Why: Prevents cross-pool data corruption.

- [ ] **Do before* and after* functions maintain state symmetry?**  
  Why: Prevents imbalance between swap directions and callbacks.

- [ ] **Are liquidity modifications atomic and revert-safe?**  
  Why: Partial updates can desync internal accounting.

- [ ] **Are accrued fees separated from principal delta?**  
  Why: Prevents slippage logic from penalizing fee balances.

- [ ] **Are balance invariants verified post-swap?**  
  Why: Detects unintended token imbalance or incorrect delta application.

---

## 11. Front-Running, MEV & Market Manipulation

- [ ] **Does the Hook logic depend on pool price or external state?**  
  Why: Allows attackers to front-run based on mempool visibility.

- [ ] **Are price or time-dependent fees hardened against MEV?**  
  Why: Dynamic fees can be gamed by predictable swap sequencing.

- [ ] **Are Hooks tested under sandwich and latency conditions?**  
  Why: Detects profitability windows exploited by bots.

- [ ] **Is sensitive logic moved off the mempool path (e.g., via off-chain precommit)?**  
  Why: Reduces manipulability of fee updates or liquidity reactions.

---

## 12. Testing & Simulation Coverage

- [ ] **Are invariant tests run for delta balancing, NonzeroDeltaCount, and PoolManager unlocks?**  
  Why: Detects balance mismatches and settlement leaks.

- [ ] **Are fuzz tests performed for all Hook entry points?**  
  Why: Uncovers unexpected revert conditions and unsafe edge cases.

- [ ] **Are async and reentrancy edge cases simulated with ERC777, flash loans, and nested unlocks?**  
  Why: Validates complex reentrancy paths in production-like environments.

- [ ] **Is upgrade simulation done on live state to confirm backward compatibility?**  
  Why: Ensures storage persistence and upgrade safety.

- [ ] **Are Hooks benchmarked under gas, mempool, and MEV simulation?**  
  Why: Confirms performance and exploit resistance under real conditions.

---
