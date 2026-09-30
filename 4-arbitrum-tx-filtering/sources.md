# Sources — ArbFilteredTransactionsManager deep dive

Collected 2026-09-08. Primary only. Secondary explainers (Beosin recap, news wires) not used as ground truth.

## Official product / docs

| ID | Source | Why |
|---|---|---|
| D1 | https://docs.arbitrum.io/launch-arbitrum-chain/chain-config/sequencer/compliance-filtering | Canonical operator doc: sequencer + STF, guardian precompile, sentinel, S3 hash list, event rules, burn exception, ETH-lock caveat |
| D2 | https://docs.arbitrum.io/notices/arbos61-upgrade-notice | ArbOS 61 Elara dates; filtering disabled on One/Nova; ArbOS 60 never activated on One/Nova |
| D3 | https://blog.arbitrum.io/arbos-elara/ | Product announcement, 2026-08-20, title "Compliance Filtering, Priority Fee Support" |
| D4 | https://blog.arbitrum.io/compliance-for-the-programmable-economy/ | Marketing: policy is operator's, Arbitrum supplies enforcement; 2026-04-27 |
| D5 | https://docs.arbitrum.io/how-arbitrum-works/inside-arbitrum-nitro | Inbox → STF → outputs; Geth sandwich |
| D6 | https://docs.arbitrum.io/how-arbitrum-works/deep-dives/arbos | Sandwich, precompiles via reflection, STF lives in Go |
| D7 | https://docs.arbitrum.io/how-arbitrum-works/deep-dives/sequencer | Sequencer feed, batching, 24h force inclusion, censorship timeout |
| D8 | https://docs.arbitrum.io/how-arbitrum-works/deep-dives/transaction-lifecycle | Two submission paths; forceInclusion |
| D9 | https://docs.arbitrum.io/how-arbitrum-works/deep-dives/l1-to-l2-messaging | Delayed Inbox, aliasing, retryables |
| D10 | https://docs.arbitrum.io/how-arbitrum-works/deep-dives/stf-gentle-intro | Why STF must be deterministic (fraud proofs) |
| D11 | https://docs.arbitrum.io/arbitrum-essentials/precompiles/overview | Precompiles = native client code |
| D12 | https://docs.arbitrum.io/arbitrum-essentials/precompiles/reference | Address table; `0x74` not yet in this reference table as of fetch |
| D13 | https://docs.arbitrum.io/launch-arbitrum-chain/overview/introduction | Orbit customization; protocol-level screening section |

## Nitro source

| ID | Source | Why |
|---|---|---|
| N1 | https://github.com/OffchainLabs/nitro-precompile-interfaces/blob/main/ArbFilteredTransactionsManager.sol | ABI: add/delete/isFiltered; events; ArbOS 60+ |
| N2 | https://github.com/OffchainLabs/nitro-precompile-interfaces/blob/main/ArbOwner.sol | add/remove filterer, setTransactionFilteringFrom, setFilteredFundsRecipient; addr `0x70` |
| N3 | https://github.com/OffchainLabs/nitro-precompile-interfaces/blob/main/ArbOwnerPublic.sol | public getters at `0x6b` |
| N4 | https://github.com/OffchainLabs/nitro/blob/master/precompiles/ArbFilteredTransactionsManager.go | hasAccess via TransactionFilterers; BurnOut if unauthorized |
| N5 | https://github.com/OffchainLabs/nitro/blob/master/arbos/filteredTransactions/state.go | storage map hash→{1}; IsFilteredFree; DeleteFree |
| N6 | https://github.com/OffchainLabs/nitro/blob/master/arbos/tx_processor.go | STF: deposit redirect, retryable redirect, RevertedTxHook skip+gas burn |
| N7 | https://github.com/OffchainLabs/nitro/blob/master/precompiles/wrapper.go | FreeAccessPrecompile: filterers pay 0 gas |
| N8 | https://github.com/OffchainLabs/nitro/blob/master/precompiles/ArbOwner.go | feature gated by TransactionFilteringFromTime |
| N9 | https://github.com/OffchainLabs/nitro/blob/master/cmd/transaction-filterer/api/api.go | sentinel RPC `Filter(txHash)` → AddFilteredTransaction |
| N10 | https://github.com/OffchainLabs/nitro/blob/master/system_tests/delayed_message_filter_test.go | halt until hash registered; retryable skip auto-redeem |

## OP Stack

| ID | Source | Why |
|---|---|---|
| O1 | https://docs.optimism.io/op-stack/transactions/forced-transaction | Sequencing window 12h, max time drift 30m; deposit-only catch-up |
| O2 | https://docs.optimism.io/op-stack/bridging/deposit-flow | OptimismPortal.depositTransaction → derivation MUST include |

## Robinhood / L2Beat

| ID | Source | Why |
|---|---|---|
| R1 | https://docs.robinhood.com/chain/differences-from-ethereum/ | Sequencer screening; blocked tx "looks like it never occurred"; FCFS |
| R2 | https://github.com/l2beat/l2beat/blob/master/packages/config/src/projects/robinhood/robinhood.ts | sequencerFailure = No mechanism; force inclusion nullifiable |
| R3 | https://github.com/l2beat/l2beat/blob/master/packages/config/src/projects/_templates/orbitstack/ArbFilteredTransactionsManager/template.jsonc | Discovery template; One/Nova tracked but unused |

## Contradictions to keep visible

1. **ArbOS 60 vs 61.** Interfaces/Nitro comments: ArbOS 60. Elara activation + L2Beat: ArbOS 61. D2: ArbOS 60 never activated on One/Nova (gas-refund bugs). Treat as: code landed in 60, production train is 61 Elara.
2. **Docs D1 ETH-lock vs N6 deposit path.** D1 warns dropped L1 ETH credit can lock in the bridge. N6 redirects `to` to `filteredFundsRecipient` instead of dropping. Redirect is the implemented STF behavior; lock is a residual risk if an operator *does* drop credit.
3. **Robinhood R1 "never happened" vs N6 failed receipt.** R1 describes sequencer rejection (not included). STF path *does* produce a failed receipt, burns gas, bumps nonce. Do not collapse the two.
4. **Product name vs code.** D1: "Transaction Guardian Precompile" + "Delayed Inbox Sentinel". Code: `ArbFilteredTransactionsManager` (`0x74`) + `cmd/transaction-filterer`.
5. **D12 precompiles reference** still omits `0x74` from the summary table even though N1/N4 exist.

## Cut (not used as evidence)

News recaps (KuCoin, Coinness), arbreth.rs aggregator, Beosin secondary write-up, cryptoarepa.
