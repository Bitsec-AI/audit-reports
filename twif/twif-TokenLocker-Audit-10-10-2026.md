# TokenLock — Remediation Verification

|                             |                                                                                                                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Original review**         | `CLIENT-REPORT.md` v1.1 — 19 findings + DOC-1/2/3 against `src/TokenLock.sol` at 84 lines, sha256 `ff3366fa…f242d`                                                             |
| **Remediated source**       | `src/TokenLock.sol` — 132 lines, sha256 `4bc9d4f0d2da44848bb878dfe6fac7993e6d6c5ff1b04a76e0e846407796cfb4` (HEAD `a06fe5d`)                                                    |
| **Vendor response checked** | `docs/audit-2026-10/RESPONSE.md`, `docs/audit-2026-10/REPLY-2026-10-06.md` — every claim treated as a claim to verify                                                          |
| **Method**                  | Full re-audit of the remediated source from scratch, bytecode disassembly, then a re-check of the two follow-up commits (`cee797a`, `ba75320`) and the docs commits after them |
| **Build**                   | solc 0.8.26 · evm `paris` · optimizer 200 · `bytecode_hash = none` · OZ 5.6.1                                                                                                  |
| **Vendor suite**            | `forge test` **73/73 pass**                                                                                                                                                    |
| **Client acknowledgment**   | Every open item below acknowledged and accepted by the client, 2026-10-08                                                                                                      |
| **Report version**          | 1.4 — 2026-10-08                                                                                                                                                               |

---

## Summary

**All findings are addressed: each one is fixed, accepted by design, environmental, or acknowledged by the
client.** No issue lets anyone take funds or shorten the lock. Every Medium finding is closed, either in code
or by a deployment policy (Q1 and Q2) that the client has acknowledged. The acknowledged items are lower
severity. They can only cause loss of access through a token that misbehaves, a deployment mistake, or a
chain renumbering. **None of these applies to TWIF deployed with the official
script.**

Four properties hold at any re-entry depth:

- Only the beneficiary can act.
- `unlockTime` only moves later.
- The locked token cannot leave before `unlockTime`.
- Every payment goes to the immutable beneficiary.

| Status                                   | Original findings                                          |
| ---------------------------------------- | ---------------------------------------------------------- |
| **Fixed**                                | VULN-001, 002, 003, 004, 011, 021, 022, 025 · DOC-2, DOC-3 |
| **Accepted by design**                   | VULN-012, 015, 016, 037                                    |
| **Environmental, no code change needed** | VULN-035, 040                                              |
| **Acknowledged by client**               | VULN-013, 017, 026, 031, 033 · DOC-1                       |

**What the vendor got right.** The highest-priority item in the original review was `extend`'s unbounded
ratchet. It is **closed by a better fix than we recommended**: an immutable `maxUnlockTime` anchored at
deployment. The follow-up commits applied both remaining fixes we proposed, word for word:

- **`cee797a`:** the constructor again requires the token to have code, before the `balanceOf` probe.
- **`ba75320`:** `withdraw` checks the token-code pin before the time gate.

---

## Per-finding status

| ID           | Fix shipped                                                        | Status                             | Notes                                                                                                                                                                                                                                                                                                            |
| ------------ | ------------------------------------------------------------------ | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **VULN-001** | `code.length` guard **and** `balanceOf` probe, `:66-68`            | **Fixed**                          | Refuses EOAs, inert EIP-7702 designators, stubs, a Safe pasted as the token, and codeless accounts that answer with a word (precompiles)                                                                                                                                                                         |
| **VULN-002** | `_deliver` measures delivery, `:125-131`                           | **Fixed**                          | Uses the tolerant form we asked for (`delivered == 0` on a non-zero request), so fee-on-transfer tokens still work                                                                                                                                                                                               |
| **VULN-003** | `BeneficiaryIsToken`, `:70`                                        | **Fixed**                          | FIX-3 word for word                                                                                                                                                                                                                                                                                              |
| **VULN-004** | `NotAContract` before any call, `:107`                             | **Fixed**                          |                                                                                                                                                                                                                                                                                                                  |
| **VULN-011** | immutable `maxUnlockTime`, `:30/:71/:75/:115`                      | **Fixed, better than recommended** | Cumulative and anchored to a fixed point. `invariant_unlockNeverPassesTheCeiling`                                                                                                                                                                                                                                |
| **VULN-012** | same                                                               | **Accepted by design**             | The delay is now bounded at deployment + 3650 days. While `beneficiary` is immutable, no code change can do more. Documented in the README                                                                                                                                                                       |
| **VULN-013** | same                                                               | **Acknowledged**                   | The backward-clock leg is fixed. A lock deployed exactly at the ceiling can never be extended. `withdraw` still works                                                                                                                                                                                            |
| **VULN-015** | not adopted, with reasoning                                        | **Accepted by design**             | The `Locked` event makes a lock's terms auditable from outside                                                                                                                                                                                                                                                   |
| **VULN-016** | documented                                                         | **Accepted by design**             |                                                                                                                                                                                                                                                                                                                  |
| **VULN-017** | `Locked(token, beneficiary, unlockTime, maxUnlockTime)`, `:39/:79` | **Acknowledged**                   | The event omits `deployChainId` and `tokenCodehash`                                                                                                                                                                                                                                                              |
| **VULN-021** | `NotSweepable`, `:106`                                             | **Fixed**                          | FIX-4                                                                                                                                                                                                                                                                                                            |
| **VULN-022** | header `:15-18`, README                                            | **Fixed**                          | Fixed as far as code allows                                                                                                                                                                                                                                                                                      |
| **VULN-025** | `:106` checked on every call                                       | **Fixed**                          |                                                                                                                                                                                                                                                                                                                  |
| **VULN-026** | `:106`                                                             | **Acknowledged**                   | The `sweep(beneficiary)` route is fixed. A callback-capable token can still re-enter through a contract beneficiary. That re-entry can inflate the reported `Withdrawn` figure, but it cannot move funds                                                                                                         |
| **VULN-031** | `tokenCodehash` pin, checked first in `withdraw`, `:35/:77/:93-94` | **Acknowledged**                   | Shipped as recommended. The pin can no longer be sprung by a stranger, and a lock with a broken pin is now visible before funding. It does not detect a proxy-implementation upgrade (the README says to lock only non-upgradeable tokens). The Q1 answer ("no pre-Cancun chains") is acknowledged by the client |
| **VULN-033** | `deployChainId`, `:32/:76/:83`                                     | **Acknowledged**                   | The binding closes fork and replay. A deliberate second deployment on another chain is covered by the deploy script's chain and beneficiary checks and by the Q2 policy, which the client has acknowledged                                                                                                       |
| **VULN-035** | —                                                                  | **Environmental**                  | `extend` still works after a backward clock move                                                                                                                                                                                                                                                                 |
| **VULN-037** | standard JSON input published                                      | **Accepted by design**             | Regenerated for the current source                                                                                                                                                                                                                                                                               |
| **VULN-040** | —                                                                  | **Environmental**                  | The runtime makes a single `SSTORE` (`unlockTime`). The new fields are immutable                                                                                                                                                                                                                                 |
| **DOC-1**    | header rewritten, `:10-18`                                         | **Acknowledged**                   | The storage clause is true. The sentence "the only contract that can call back is the token" should be corrected: a token's `transfer` can call the beneficiary                                                                                                                                                  |
| **DOC-2**    | header                                                             | **Fixed**                          |                                                                                                                                                                                                                                                                                                                  |
| **DOC-3**    | README                                                             | **Fixed**                          | Ceiling, 2-of-3 requirement and threshold caveat are all stated                                                                                                                                                                                                                                                  |

---

## Owner actions

1. **Do not re-fund lock `0x1eFCbAb62c0B1283Ee693f2bf58777990Fd8D033`.** It runs the original source, so the
   unbounded `extend` ratchet is still live on it. Deploy new locks from `4bc9d4f0…`.
2. **Use a beneficiary Safe that survives one lost key** (2-of-3 or better).
3. **Record the Q1 and Q2 answers as confirmed** in `RESPONSE.md`. The client has acknowledged them, but the
   file still marks them "proposed".

---

## Reproduce

```bash
cd source/TokenLock && forge build && forge test        # 73/73
shasum -a 256 src/TokenLock.sol                         # 4bc9d4f0…6cfb4
```

_Remediation verification of `src/TokenLock.sol` at sha256 `4bc9d4f0…6cfb4` against `CLIENT-REPORT.md` v1.1.
Source-level and bytecode-level only. No state-changing transaction and no live-chain query was made.
Working papers are in `security-audit-fixed/`._
