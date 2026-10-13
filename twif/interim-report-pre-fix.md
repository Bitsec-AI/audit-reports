# TokenLock — Fix-Review Audit of the Corrected Source

| | |
|---|---|
| **Target** | `src/TokenLock.sol` — 128 lines · sha256 `11c72cbcb441711800f9999eb4e79b011be32b00ee4161d480af72e34bda9e35` |
| **Previously audited** | the same file at 84 lines, sha256 `ff3366fa…f242d` → `security-audit/`, 19 findings reported in `CLIENT-REPORT.md` v1.1 |
| **Question put to this run** | *did fixing those 19 findings introduce new exploits?* |
| **Method** | full re-audit from scratch: Phase 1 context rebuild, 5 parallel discovery themes, sequential lead triage, 25 lead-written PoCs, bytecode disassembly |
| **Baseline** | `forge build` clean · `forge test` **67/67** · invariant run 128,000 calls, 0 reverts · `git status` in the audited repo **empty before and after** |
| **Date** | 2026-10-06 |

## Answer in one line

**No new exploit.** Nothing in the fixed source lets an attacker take funds, shorten the lock, or act as the
beneficiary — the four invariants that matter (access control, monotonic `unlockTime`, the locked token cannot
leave early, every payment goes to the immutable beneficiary) survive every attack the five themes could
construct, including at arbitrary re-entry depth. **But the fixes introduced five new permanent-denial paths
and two false security claims**, and one fix does not address the finding it closed.

| | Count | |
|---|---|---|
| Critical / High / Medium | **0** | no attacker profits anywhere in this run |
| Low | **11** | new denial paths, a false in-source rationale, a control that misses its own finding |
| Info | **12** | observability, portability, error-surface, test-coverage |
| Out-of-scope tooling | **4** | the deploy script and verifier no longer agree with the contract |

## What the fixes achieved — stated first, because it is the larger part

- **VULN-011's unbounded ratchet is genuinely closed, and better than the review recommended.** The vendor
  rejected the review's one-line bound and shipped an immutable `maxUnlockTime` anchored at deployment. Their
  reasoning is **correct** and we verified it independently: the recommended line was non-cumulative under both
  readings, and under the fair reading it would **not** have closed the review's own *Act on this now* case.
  The shipped anchor is in bytecode (runtime `SSTORE` = 1), so no call sequence can move it.
- **14 of 19 findings closed**, DOC-2 and DOC-3 closed, and the review's FIX-3/FIX-4 prerequisites shipped in
  the right commit order.
- **Three findings nobody credited**: the ceiling also closes VULN-013's backward-clock leg (a reorg no longer
  leaves the contract inert); the chain binding closes the mechanism of VULN-039; and the constructor probe
  closes the **transposed-argument deploy** (the Safe pasted as the token), which the old guard accepted and
  which strands everything.
- **Verified sound**: the masked-bytecode verifier cannot be defeated by hiding code in an immutable span
  (masked spans are `PUSH32` operands; jumpdest analysis makes them unreachable); no guard transposition
  anywhere changes the *accepted* set, and the shipped `sweep` ordering is the diagnostically correct one; no
  reachable solc 0.8.26 codegen bug; the `paris` pin and single-storage-slot properties hold.

## The five things worth acting on

### 1. One line was deleted that should have been kept — fixes three findings at once
`VULN-101` · `VULN-122` · DELTA-4, plus the gas-sink leg of `VULN-107`. **Verified by applying it** (below). The VULN-001 fix **replaced** `code.length == 0` with a
`balanceOf` probe instead of adding to it. The probe is a real improvement, but the deleted predicate covered a
**disjoint** class: codeless accounts that answer with a word (`0x02`/`0x03`/`0x04`, exhaustively enumerated).
For those, VULN-001's impact returns unchanged — the real asset becomes `otherToken` and `sweep` releases 100%
before `unlockTime`, with the documented smoke test still passing. Then the new codehash pin makes it worse:
`EXTCODEHASH` is `0` for an empty account, so **anyone can send 1 wei to the token address and brick `withdraw`
permanently** (PoC-confirmed; an ordinary contract token is provably immune). And `sweep` now *refuses* the very
class the constructor admits, so the permissiveness buys nothing.

Decisive: **`script/DeployTokenLock.s.sol:44` still requires `code.length > 0`**, so the sanctioned deploy path
refuses this class anyway. Restoring the line costs nothing real and the contract then agrees with its own
script and its own `sweep`.

```solidity
if (address(token_).code.length == 0) revert NotAContract();   // restore, before the probe
```

**Verified empirically (2026-10-06), fix applied and then reverted:** all 67 existing tests still pass, and
seven PoC assertions flip from pass to fail — every precompile-acceptance test now reverts `NotAContract()` at
construction, the 1-wei brick can no longer even be set up, and the C1/C8 disagreement is resolved (both sites
now refuse). The gas-sink precompiles `0x05`–`0x09` are now refused for ~33k gas with a named error instead of
burning the probe's allowance. **It does NOT fix the returndata-bomb leg of VULN-107** — a hostile token with
real code still passes the code check and the constructor still copies 4 MiB for 180.7M gas; that needs the
bounded read in VULN-107's own mitigation.

### 2. Two lines should be transposed — free, and it makes a dead lock visible
`VULN-137`. **Verified by applying it** (below). The codehash pin is checked **after** the time gate, so for the whole locked period a `withdraw`
simulation returns `StillLocked` whether the lock is healthy or already permanently bricked. The Safe guide
makes exactly that simulation the pre-funding health check, immediately before "send the real amount". PoC: the
revert data is **byte-identical** in both cases. Transposing changes nothing about what is accepted.

**Verified empirically (2026-10-06), fix applied and then reverted:** all 67 existing tests still pass —
including `test_withdraw_beforeUnlock_reverts`, which still gets `StillLocked` for a healthy token — and the
PoC's equality assertion now fails exactly as intended: pre-unlock, a healthy lock returns `0x16b82bbe…`
(`StillLocked`) while a bricked one returns `0x3a337af0…` (`TokenCodeChanged`). The brick is now visible to a
free pre-unlock `eth_call`.

### 3. The no-reentrancy-guard rationale is false in two clauses
`VULN-113`. DOC-1 was raised because the source gave a false reason; the fix replaced it with a longer reason
that is still false where it matters. "During a transfer the only contract that can call back is the token,
which can never be the beneficiary" — the token's `transfer` may call **anything, including the beneficiary**,
whose callback satisfies the identity check. PoC: an ordinary contract beneficiary with a token-receive hook
re-entered `extend` **in the middle of an in-flight `withdraw`** (`unlockTime` 86401 → 8812801). No relay, no
multicall, no misconfiguration. The decision to ship without a guard is still right — for the prior audit's
reasons, not the shipped ones. Consequences: `Withdrawn` can **exceed the amount requested**
(PoC: `Withdrawn(999000000000000000001)` for a 999e18 request) and a re-entrant credit can **mask** the new
`NothingDelivered` check, so the integrity control the fix added is itself defeatable.

### 4. The measured-delivery fix measures the recipient, not the custodian
`VULN-111` · `VULN-114` · `VULN-115` · `VULN-112`. `balanceAfter > balanceBefore` is a claim about what the
beneficiary *retains when the call returns*, not about whether value *left the lock*. Making that the basis of a
**revert on the only exit** created a new permanent-denial path, and it put two adversary-controlled reads on
that exit — a token that reverts in `balanceOf` for the beneficiary alone now bricks `withdraw` while `transfer`
still works (PoC). Keep the recipient-side figure for the **event** (that is what VULN-002 asked for) and move
the **assertion** to the custodian side, which no third party can inflate:

```solidity
uint256 selfBefore = t.balanceOf(address(this));
...
if (amount != 0 && t.balanceOf(address(this)) == selfBefore) revert NothingDelivered();
```

### 5. The chain binding does not close the finding it was added for
`VULN-125`. `deployChainId` is recorded **at deployment on whichever chain the deploy happens**, so a second
deployment on chain B records B's id and functions normally there — with the wrong token, or with an uncallable
beneficiary. **Neither of VULN-033's two legs is addressed**, and leg 2's own recommended
`beneficiary_.code.length` check was not added. What the binding does close is the fork/replay mechanism
(VULN-039). Its cost is real: sitting in the modifier, it gates `sweep` and `extend` too, so a state-preserving
chain-id renumber permanently strands the locked token **and every unrelated airdrop** (PoC). This corrects both
the vendor's response and our own first-pass delta review.

---

## Regression table — all 19 reported findings + DOC-1/2/3

| ID | Sev | Verdict | Evidence / residual |
|---|---|---|---|
| VULN-001 | M | **PARTIALLY CLOSED** | `:65-66`. Reported classes closed; codeless-but-answering class newly admitted → VULN-101 |
| VULN-002 | L | **CLOSED** | `:121-127`, tolerant form as asked, event topics unchanged. New side effects → VULN-111/112/114/115 |
| VULN-003 | L | **CLOSED** | `:68`, FIX-3 verbatim |
| VULN-004 | I | **CLOSED** | `:103`, named error before any call; the constructor's gas-burn twin remains → VULN-107 |
| VULN-011 | M | **CLOSED, better than recommended** | `:30/:69/:73/:111`; invariant 128k calls |
| VULN-012 | L | **PARTIALLY CLOSED** | bounded, not closed — one call still freezes everything to the ceiling, surviving rotation → VULN-124. **Vendor's "closed" is wrong** |
| VULN-013 | I | **leg 1 NOT CLOSED (worse), leg 2 CLOSED** | `:70` still accepts `== ceiling` → `extend` dead **permanently** (was 1 s) → VULN-105. Backward-clock leg genuinely fixed |
| VULN-015 | I | **ACCEPTED-BY-DESIGN** | reasoning sound; the new `Locked` event makes a sham lock externally auditable, which is the better mitigation |
| VULN-016 | L | **ACCEPTED-BY-DESIGN** | documented; compounded by the pin's new permanent revert |
| VULN-017 | I | **PARTIALLY CLOSED** | `Locked` at `:39/:77` closes the main gap; omits `deployChainId`/`tokenCodehash`, the two new brick-capable immutables; `Withdrawn`/`Extended` still unindexed |
| VULN-021 | L | **CLOSED** | `:102`, runtime placement as required |
| VULN-022 | L | **CLOSED to the extent possible** | `:15-18`; more now rides on the wording, since `_deliver` reads after a call |
| VULN-025 | I | **CLOSED** | `:102` evaluated per call, with the reason stated |
| VULN-026 | I | **PARTIALLY CLOSED** | the emit-before-call half was not adopted; re-entry is still reachable via a beneficiary hook, and now corrupts the event's **number**, not just its order → VULN-114. **Vendor credits the wrong mechanism** |
| VULN-031 | M | **PARTIALLY CLOSED** | `:75/:90-91` shipped; degenerate pin → VULN-122; invisible pre-unlock → VULN-137; misses proxy upgrades → VULN-123; Q1 answer still "proposed" |
| VULN-033 | M | **NOT CLOSED BY CODE** | → VULN-125. Closed only by the Q2 policy, still "proposed", plus the deploy script's checks |
| VULN-035 | I | unchanged (environmental) | improved in combination: `extend` now still works after a backward move |
| VULN-037 | I | **ACCEPTED-BY-DESIGN** | standard input verified byte-identical, ABI equals the compiled ABI |
| VULN-040 | I | unchanged (environmental) | argument is **stronger**: runtime `SSTORE` = 1, the new fields are immutable |
| DOC-1 | — | **PARTIALLY CLOSED** | storage clause verified true; "only the token can call back" is **false** → VULN-113 |
| DOC-2 | — | **CLOSED** | `:15-18` |
| DOC-3 | — | **CLOSED** | `README.md:50-53`. Nit NOT fixed: README says 62 tests, `forge test` reports **67** |

**Applicability changes for the three findings excluded from the client report:** VULN-036 got **worse** — the
constructor no longer refuses the class at deploy time, so a loud deploy-time revert became a silent deploy
with a later permanent revert, while `sweep` refuses it anyway. VULN-038's documentation half is closed.
VULN-039 is **closed** by the chain binding, and nobody credited it.

## Tooling — out of scope, reported because two failures would be consequential

| ID | Issue | Responsible side |
|---|---|---|
| VULN-136 | `script:44`'s `code.length > 0` is now the **only** guard against a token the contract accepts and can never pay out; `postflight` asserts neither `maxUnlockTime` nor `tokenCodehash` | **the contract** — restore the line (item 1) |
| VULN-138 | `verify-deployed.mjs:25` hardcodes a 20-byte address slice, so it **truncates `tokenCodehash`**; all five slots are labelled by volatile solc AST ids; the report prints even on MISMATCH; nothing is asserted about the new values | the script |
| VULN-139 | the verifier never validates `out/` against `src/` and prints the unqualified verdict "the deployed runtime is this source" — while **two source versions are live** (the deployed lock runs the old one) | the script |
| VULN-140 | the Safe guide omits `TokenCodeChanged` entirely, omits `maxUnlockTime()`/`tokenCodehash()` from its read-before-you-sign list, and promises an error name on paths that revert empty | the docs |

**Verifier verdict: sound as a bytecode-identity check, unsound as the claim it prints.** A mismatched runtime
cannot pass as MATCH. But a MATCH no longer tells an operator which source it matched or whether the lock is
operable — and the Safe guide offers it as an *alternative* to explorer verification, which it is not.

## Properties re-verified against the new source

| Property | Verdict |
|---|---|
| Access control | **holds, and stronger** — chain gate ahead of identity; `DELEGATECALL`/`CREATE`/`CREATE2`/`SELFDESTRUCT` all 0; no allowance ever granted |
| `extend` monotonic + overflow-safe | **holds, and the boundary is now unreachable** rather than merely tested — there is no runtime addition left to wrap |
| `sweep` not aliasable onto the locked token (4 attacks) | **holds; three now fail earlier** — precompiles are refused by name, the 7702 alias is also caught by `:102/:103` |
| Portability | **STALE — must be rewritten.** Measured: runtime `CHAINID` **6**, `EXTCODEHASH` **1** (were 0). Still zero `PUSH0`/`MCOPY`/`TLOAD`/`TSTORE`, so the `paris` pin holds — but the floor is now **Istanbul**, not Constantinople, and `README.md:3`'s "no chain-specific opcode is used" is false → VULN-145 |
| No reachable solc 0.8.26 codegen bug | **holds** — `viaIR` off, storage is one `uint256` at slot 0, transient storage impossible under `paris` |
| "The deployed lock behaves as audited" | **stale by construction — do not carry forward.** The live lock runs the **old** source; five of the new errors have never been observed on any chain |

## Open questions — what this source cannot answer

1. **Does any intended target chain host a codeless-but-answering ERC-20?** Sets VULN-101/122 between Info and
   Medium. Settle with `eth_getCode` plus a `transfer` return-value check against that chain's ERC-20
   precompiles. (Ruled out for Robinhood Chain: Nitro precompiles carry 1 byte of code.)
2. **Does any deployed ERC-20 revert or run super-linearly in `balanceOf`?** Sets VULN-115's permanence.
3. **The target chain's chain-id stability policy** (VULN-125) and **timestamp unit/skew rules** (VULN-123).
4. **Both gating questions are still formally open.** `RESPONSE.md` marks the Q1 and Q2 answers "proposed,
   pending the owner's confirmation", so VULN-031 and VULN-033 are not closed as the review defined closure —
   and for VULN-033 the code does not close either leg.

## Still open from "Act on this now" — unchanged, not code

1. **Do not re-fund lock `0x1eFCbAb62c0B1283Ee693f2bf58777990Fd8D033`.** It runs the old source; FIX-1's
   unbounded ratchet is still live on it.
2. **Use a beneficiary Safe that survives one lost key** (2-of-3 or better).

---

*Fix-review audit of `src/TokenLock.sol` at the scope hash above. Source-level and bytecode-level; no
state-changing transaction and no live-chain query was made in this run. Evidence: 25 PoC tests in
`security-audit-fixed/tests/`, all passing. The audited repository is byte-for-byte unchanged and its suite
passes identically before and after.*
