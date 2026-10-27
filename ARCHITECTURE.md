# ARCHITECTURE.md — Zeltra Protocol Code Mapping & Architecture

Dokumen ini memetakan struktur kode riil, crate, dan hirarki direktori di seluruh workspace **Zeltra Protocol**.

---

## 1. Repository Layout & Code Mapping

```text
/workspaces/Zeltra-Protocol/
├── zeltra-contracts/                 — Smart Contract Arbitrum Stylus WASM (EIP-2537 BLS12-381)
│   ├── src/lib.rs                    — Entrypoint #[entrypoint] contract Zeltra:
│   │                                   • Inflow: deposit_direct_note, deposit_direct_weth, deposit_direct_native_eth
│   │                                   • Cross-Chain Inflow: ccip_receive (Chainlink CCIP Programmable Inbound Deposit DEC-055)
│   │                                   • Outflow/Transfer: spend_joinsplit (Strict 3-Mode), spend_private_note
│   │                                   • Solvensi: check_asset_solvency_invariant, check_liability_invariant
│   │                                   • Nullifier & MMR: is_nullifier_spent, note_tree_root, mmr_leaf_count, etc.
│   ├── src/deposit.rs                — Logika deposit direct note ERC20 & native ETH auto-wrap (_mmr_insert)
│   ├── src/ccip_deposit.rs           — Chainlink CCIP inbound programmable token deposit receiver & MMR minting (DEC-055)
│   ├── src/spend.rs                  — Logika eksekusi Strict 3-Mode JoinSplit & private spend
│   ├── src/merkle.rs                 — Akumulator on-chain Merkle Mountain Range (_mmr_insert, canonical peak bagging, ring-buffer bounded history DEC-053)
│   ├── src/storage.rs                — Stylus sol_storage! Zeltra layout (MMR state, accepted roots ring-buffer DEC-053, nullifiers, multi-asset)
│   ├── src/groth16_note_verifier.rs  — Verifier Groth16 BLS12-381 untuk PrivateNoteCircuit (16 public inputs, sanitasi titik kurva DEC-052) via precompile 0x0c & 0x0f
│   ├── src/groth16_joinsplit_verifier.rs — Verifier Groth16 BLS12-381 untuk JoinSplitCircuit (19 public inputs, sanitasi titik kurva DEC-052) via precompile 0x0c & 0x0f
│   ├── src/verification.rs          — Wrapper pemanggilan host precompile EIP-2537 (call_msm_g1 0x0c, call_pairing 0x0f)
│   ├── src/vault.rs                  — Akuntansi solvensi multi-aset terisolasi (_check_asset_solvency_invariant, total_assets)
│   ├── src/views.rs                  — View/getter functions untuk status on-chain
│   ├── src/governance.rs             — Pause, unpause, dan pengaturan parameter darurat
│   ├── src/events.rs                 — Event NoteCommitmentAppended (memo 96B), CCIPDepositExecuted, JoinSplitSpendExecuted, dll.
│   ├── src/poseidon_w5_constants.rs  — Konstanta round constants & MDS matrix Poseidon width-5 in-contract
│   ├── src/interfaces.rs, types.rs, constants.rs, helpers.rs — Interface ERC-20 / CCIP Router, tipe EVM & fungsi sanitasi titik G1/G2 (DEC-052)
│   ├── src/bin/contract_interop.rs   — Binary interop contract CLI
│   └── abi/IZeltra.sol               — Antarmuka Solidity ABI
│
├── zeltra-core/                      — Primitif Kriptografi & Sirkuit ZK (Arkworks 0.6.0 BLS12-381)
│   ├── src/lib.rs                    — Re-export BLS12-381, Poseidon, EVM, fees, MMR, dan sirkuit
│   ├── src/note_circuit.rs           — PrivateNoteCircuit (1-in-1-out Groth16, 16 public inputs, two-limb quote hash, DEC-035B)
│   ├── src/joinsplit_circuit.rs      — JoinSplitCircuit (Universal 2-in-2-out Groth16, 19 public inputs, companion constraints DEC-046, DeniabilityPermutationGadget DEC-051)
│   ├── src/selective_disclosure_circuit.rs — SelectiveDisclosureCircuit (CEX compliance Groth16 dengan Verifier Binding & sk-bound owner_key SEC-002, DEC-040A)
│   ├── src/circuit_gadgets.rs        — Reusable R1CS gadgets (flags_packed multi-asset, range checks, Poseidon gadgets)
│   ├── src/note.rs                   — Note primitives (PrivateNoteV1 dengan asset_id, NotePlaintext)
│   ├── src/keys.rs                   — Derivasi kunci ZSA (SpendingKey, NullifierKey, ViewingKey, IncomingViewingKey)
│   ├── src/memo.rs                   — Committing AEAD 96B memo (EtH-KS transform, ChaCha20 + HMAC-SHA256, 1B View Tag, DEC-048)
│   ├── src/memo_ct.rs                — Constant-Time Flag Oblivious Evaluation (CT-FOE), branchless canonical Fr, zeroize (DEC-049)
│   ├── src/merkle.rs                 — Flat Merkle Mountain Range (FlatMerkleMountainRange, Peter Todd indexing DEC-045)
│   ├── src/poseidon.rs               — Poseidon width-3 & width-5, Grain-128 LFSR parameter generator (DEC-021)
│   ├── src/jacobian.rs               — Mesin kalkulasi Augmented Jacobian M(a) untuk uji kekakuan tangen (DEC-047)
│   ├── src/formal_tests.rs           — Test suite formal Augmented Jacobian & verifikasi rank finansial (DEC-047)
│   ├── src/accounting.rs             — State machine ContractAccounting & validasi invariant matematis
│   ├── src/fees.rs                   — Kalkulasi fee (0 bps deposit, 0 bps transfer, 45 bps flat public spend DEC-028, 15% execution overhead buffer DEC-054)
│   ├── src/curve_opt.rs              — Fast Jacobian point tripling 3P, Eisenstein tau-endomorphism (DEC-050) & endomorfisme Argo S (DEC-052)
│   ├── src/evm.rs                    — Sanitasi deterministik titik G1/G2 EIP-2537 5-Gate & serialisasi EVM (DEC-052)
│   ├── src/serialization.rs          — Helper serialisasi data arkworks
│   ├── tests/deniability_tests.rs    — Suite pengujian KAT & adversarial boundary DEC-051 (9/9 tests passed)
│   └── src/bin/                      — Binary CLI / benchmarks (bench.rs, bench_curve.rs, bench_deniability.rs, flat_mmr_interop.rs, zsa_interop.rs)
│
├── zeltra-node/                      — Stateless Single-Sequencer Relayer Node (Axum 0.8)
│   ├── src/main.rs                   — Server boot, background worker orchestration, rate limiter
│   ├── src/lib.rs                    — Library exports untuk relayer testing & integration
│   ├── src/router.rs                 — Axum routes definition (termasuk spend, quote, mmr, status, ccip, x402)
│   ├── src/state.rs, config.rs, dto.rs — AppState, network configuration, DTOs
│   ├── src/handlers/                 — HTTP Endpoint Handlers:
│   │   ├── health.rs                 — /health, /live, /ready & metrik makro solvensi
│   │   ├── quote.rs                  — /api/v1/quote (Signed EIP-712 execution fee quote, zero-custody overhead DEC-054)
│   │   ├── spend.rs                  — /api/v1/spend-joinsplit & /api/v1/spend-private-note
│   │   ├── ccip.rs                   — /api/v1/ccip/deposit/:id, /api/v1/ccip/spend/:id, /api/v1/ccip/status/:id (DEC-055 & DEC-056)
│   │   ├── mmr.rs                    — /api/v1/mmr/proof/:leaf_index, /api/v1/mmr/leaves, /api/v1/notes/sync
│   │   ├── tx_status.rs              — /api/tx-status
│   │   └── x402.rs                   — /api/x402/verify (ZK-UTXO HTTP 402 Facilitator gateway)
│   ├── src/database/                 — Modul Database SQLite + SQLCipher AES-256 (WAL mode):
│   │   ├── mod.rs, types.rs          — Connection pool, migrations, schema definitions
│   │   ├── spends.rs, quotes.rs      — Queue spend ingress & quote storage (ephemeral pruning DEC-053)
│   │   ├── mmr.rs, nullifiers.rs     — Leaf cache & status nullifier on-chain
│   │   ├── ccip.rs                   — Tracking pesan CCIP inbound & outbound (DEC-055 & DEC-056)
│   │   ├── relayer_tx.rs             — Transaksi broadcast & nonce tracking
│   │   └── idempotency.rs            — Idempotency deduplication guard
│   ├── src/validation/               — Preflight validation (spend_preflight.rs: skalar kanonikal, sanitasi kurva & nullifier check DEC-052)
│   ├── src/evm_client.rs             — Alloy client untuk broadcast L2 tx, gas bumping, CCIP Router calls, dan receipt tracking
│   ├── src/mmr_indexer.rs            — Background worker pengindeks event NoteCommitmentAppended, CCIPDepositExecuted & memo
│   ├── src/mempool_watchdog.rs       — Background worker pemulih lease stalled & pemantau nullifier
│   ├── src/execution_fee_claimer.rs  — Background worker klaim akumulasi execution fee dari smart contract
│   ├── src/sanctions_sync.rs         — Background worker sinkronisasi harian daftar OFAC SDN
│   ├── src/ccip_monitor.rs           — Background daemon terpadu: monitor lifecycle CCIP inbound deposit & outbound spend (DEC-056)
│   ├── src/batch_processor.rs        — Pemrosesan batch transaksi relayer dengan on-chain proof enforcement sebelum CCIP dispatch (DEC-056)
│   └── src/bin/                      — Binary CLI (inpool_interop.rs, relayer_test_server.rs, mmr_bench.rs, query_db.rs)
│
├── zeltra-sdk/                       — Client SDK & Private Note Wallet (WASM + Native)
│   ├── src/lib.rs                    — Re-exports, memory zeroization guards (secure_zeroize)
│   ├── src/coin_selection.rs         — Stochastic Knapsack multi-asset coin selection & dust threshold checks
│   ├── src/storage.rs                — 2PC crash-safe note store terenkripsi Argon2id + ChaCha20Poly1305
│   ├── src/client.rs                 — HTTP client komunikasi ke relayer endpoints
│   ├── src/eip712.rs                 — EIP-712 signer untuk validasi quote relayer (executionOverheadFee DEC-054)
│   ├── src/zk_wasm.rs, fee_wasm.rs   — Prover WASM in-browser runner dengan Web Worker lifecycle (DEC-041)
│   ├── src/wasm_types.rs             — Serializable DTOs untuk bindings JS/TS
│   ├── src/wallet/                   — Core Private Note Wallet:
│   │   ├── mod.rs                    — Re-exports & Wallet interfaces
│   │   └── note_wallet/              — Modular Wallet Implementation:
│   │       ├── mod.rs                — PrivateNoteWallet struct & core state
│   │       ├── deposit.rs            — Alur deposit direct note (USDC & WETH/ETH) & CCIP inbound prep
│   │       ├── joinsplit.rs          — In-pool shielded transfer & mode 2/3 proof generation
│   │       ├── spend.rs              — Public spend & outflow generation
│   │       ├── self_broadcast.rs     — Jalur Direct Self-Broadcast (unhosted wallet, 0% relayer fee, DEC-054)
│   │       ├── sync.rs               — Sinkronisasi MMR & View Tag fast scanner (recover_notes_batch DEC-045C)
│   │       ├── compliance.rs         — Integrasi pembuatan bukti Selective Disclosure (DEC-040)
│   │       ├── keys.rs               — Manajemen kunci wallet & derive child keys
│   │       ├── state_2pc.rs          — Two-phase commit safe state transitions
│   │       └── types.rs              — Tipe data internal note wallet
│   ├── src/compliance/               — Modul Kepatuhan & Audit B2B (DEC-040 & DEC-052):
│   │   ├── argo.rs                   — Argo Groth16 invalidity encoding codec & O(1) fraud slashing (DEC-052)
│   │   ├── exporter.rs               — Ekspor laporan audit double-entry (CSV/JSON)
│   │   ├── verifier.rs               — Mesin verifikasi independen CEX (Binance/Coinbase)
│   │   └── models.rs                 — Model data compliance & challenge-response
│   ├── src/x402/                     — Pure ZK-UTXO HTTP 402 Client Protocol (DEC-042):
│   │   ├── codec.rs                  — Header parser & serializer untuk X-402-Invoice / Authorization
│   │   └── types.rs                  — Struct invoice, payment receipt, status
│   ├── js/                           — JavaScript / TypeScript WASM Web Worker Bridge:
│   │   ├── index.ts, index.js        — Entrypoint JS SDK package
│   │   ├── prover_client.ts/.js      — Client wrapper pengelola worker
│   │   └── zeltra_prover_worker.js   — Web Worker background thread (DEC-041)
│   ├── tests/side_channel_defense_tests.rs — Matriks uji adversarial 5 kategori & evaluasi timing Dudect (|t| < 4.5) (DEC-049)
│   └── src/bin/                      — Benchmark & bridge (bench_e2e.rs, prover_worker_bridge.rs, dudect_ct_foe.rs)
│
├── zeltra-cli/                       — Pure ZK CLI Utility Tool
│   ├── src/main.rs, cli.rs           — Parser Clap: GenerateWallet, CreateDepositNote, InspectMmr, VerifyProof, Compliance
│   ├── src/lib.rs                    — Library CLI exports
│   └── src/commands/                 — Subcommand handlers:
│       ├── zk.rs                     — Pembuatan wallet, deposit note, inspeksi MMR, verifikasi proof
│       └── compliance.rs             — Ekspor audit compliance & verifikasi bukti CEX
│
├── scripts/                          — Automation, Testing & Verification Suites
│   ├── start_relayer.sh              — Daemon runner relayer sequencer tunggal (start|status|stop|restart)
│   ├── deploy_testnet.sh             — Deployment contract Stylus ke Arbitrum Sepolia
│   ├── test_dec052_argo_verification.py — Suite verifikasi aljabar & empiris gerbang subgroup Argo DEC-052
│   ├── test_dec048_committing_aead.py — Suite verifikasi adversarial EtH-KS committing AEAD DEC-048 (11/11 passed)
│   ├── test_real_case_dec043c_inpool_wallet.py — Test suite Python independen in-pool lifecycle (61/61 passed)
│   ├── test_dec046_dec047_jacobian_soundness.py — Suite verifikasi formal Jacobian rank DEC-046 & DEC-047
│   ├── test_real_case_relayer_sdk_dec034c.py — Test suite real-case integrasi Relayer-SDK dual asset
│   ├── test_real_case_compliance_dec040b.py — Test suite integrasi audit compliance CEX
│   ├── test_wasm_prover_worker_lifecycle.mjs — Test suite in-browser worker lifecycle DEC-041
│   ├── benchmark_mmr_bottleneck.py   — Benchmark throughput & latency MMR engine DEC-045
│   └── check_balance.py              — Utility pengecekan saldo on-chain
│
├── research/dec/                     — 76 Architecture Decision Records (DEC-001 s/d DEC-056)
├── docs/                             — Documentation (bisnis.md, roadmap.md, X402_INTEGRATION_STRATEGY.md, analisis/)
├── WHITEPAPER.md                     — Living Technical Specification & Architecture Whitepaper
└── todo.md                           — Master active checklist (pending tasks only)
```

---

## 2. Core Stack

* **Smart Contract:** Rust Stylus SDK 0.10.7, Alloy 2.0, Arbitrum Nitro WASM, EIP-2537 BLS12-381 Precompiles (`0x0c` MSM + `0x0f` 4-pairing), Chainlink CCIP v2.
* **Cryptography & Circuits:** Arkworks 0.6.0 (`ark-bls12-381`, `ark-groth16`, `ark-r1cs-std`), Poseidon hash (width-3 & width-5), Grain-128 LFSR.
* **Relayer Node:** Axum 0.8, Tokio, SQLite + SQLCipher AES-256 (`bundled-sqlcipher`), Alloy EVM client.
* **Client SDK:** Dual Native/WASM, Web Worker disposable lifecycle, Argon2id + ChaCha20Poly1305, Direct Self-Broadcast safe harbor.

---

## 3. Matriks Evolusi Arsitektur Terkini (DEC-046 s/d DEC-056)

| DEC | Judul / Domain | Inovasi Kunci | Komponen Terdampak |
|---|---|---|---|
| **DEC-046** | *Null-Branch Witness Attack Defense* | Gerbang pendamping kuadratik deterministik (`in2_is_dummy * (owner, rho, r, index) == 0`) meniadakan degrees of freedom pada percabangan dummy. | `zeltra-core/src/joinsplit_circuit.rs` |
| **DEC-047** | *Augmented Jacobian Diagnostics* | Rank verification $M(a) = \begin{pmatrix} J(a) \\ E_F \end{pmatrix}$ membuktikan $r_T = 0$ (tangent rigidity) di 6 keluarga saksi adversarial. | `zeltra-core/src/jacobian.rs`, `formal_tests.rs` |
| **DEC-048** | *Committing AEAD (EtH-KS)* | Generic transform `EtH-KS` dengan Key Separation RCR-PRG & CR-PRF mengeliminasi kerentanan partitioning oracle (`AUD-21`). | `zeltra-core/src/memo.rs`, `zeltra-sdk/src/wallet/note_wallet/sync.rs` |
| **DEC-049** | *CT-FOE & Leakage-Resilient Decapsulation* | Evaluasi bendera validitas note bebas-cabang via `subtle::Choice` & sanitasi skalar Fr branchless (Dudect Welch t-test $\|t\| < 4.5$). | `zeltra-core/src/memo_ct.rs`, `zeltra-sdk/tests/side_channel_defense_tests.rs` |
| **DEC-050** | *Fast Jacobian Point Tripling & Endomorphism* | Fast point tripling $3P$ dan dekomposisi Eisenstein tau-endomorphism pada BLS12-381 untuk percepatan kalkulasi kurva. | `zeltra-core/src/curve_opt.rs` |
| **DEC-051** | *Nopenena Cryptographic Deniability* | `DeniabilityPermutationGadget` dengan dummy note injection untuk penyangkalan kriptografis pada transfer in-pool. | `zeltra-core/src/joinsplit_circuit.rs`, `tests/deniability_tests.rs` |
| **DEC-052** | *Argo Groth16 Invalidity & Curve Point Sanitation* | Enkoding ketidakabsahan bukti Groth16 $O(1)$ fraud slashing dan sanitasi 5-Gate titik kurva EIP-2537. | `zeltra-core/src/evm.rs`, `zeltra-contracts/src/groth16_*_verifier.rs` |
| **DEC-053** | *ammBoost State Growth Control* | Ring-buffer eviction (size 100) pada accepted MMR roots on-chain dan pembersihan receipt transaksi sementara di relayer SQLite. | `zeltra-contracts/src/merkle.rs`, `zeltra-node/src/database/` |
| **DEC-054** | *Zero-Custody Safe Harbor & Direct Self-Broadcast* | Jalur unhosted wallet self-broadcast (`self_broadcast.rs`) tanpa relayer (`execution_fee = 0`), eliminasi total terminologi broker (`markup` $\to$ `executionOverheadFee`). | `zeltra-sdk/src/wallet/note_wallet/self_broadcast.rs`, `zeltra-node/src/handlers/quote.rs` |
| **DEC-055** | *Cross-Chain CCIP Programmable Inbound Deposit* | Entrypoint `ccip_receive` di Stylus contract untuk token transfer programmable Chainlink CCIP langsung minting leaf MMR tanpa fee deposit (0 bps). | `zeltra-contracts/src/ccip_deposit.rs`, `zeltra-node/src/handlers/ccip.rs` |
| **DEC-056** | *Cross-Chain CCIP Spend Sanitation & Unified Monitor* | Eliminasi dead code Stylus spend, penegakan pembakaran nullifier & verifikasi proof di L2 sebelum CCIP dispatch, dan daemon terpadu `ccip_monitor.rs`. | `zeltra-node/src/ccip_monitor.rs`, `zeltra-contracts/src/spend.rs` |

---

## 4. Invariants & Security Gates (Non-Negotiable)

1. **Siloed Asset Solvency:**
   $$\text{Balance}(a) \ge \text{user\_note\_liabilities}[a] + \text{accrued\_execution\_fee\_liabilities}[a] + \text{realized\_protocol\_fees}[a]$$
   Ditegakkan setiap mutasi state di `zeltra-contracts/src/vault.rs`.
2. **Fee Flat Uniformity (DEC-028 & DEC-054):**
   - Deposit Inflow = **0 bps** (0.00% immutable).
   - In-Pool Mode 2 Shielded Transfer = **0 bps** (0.00% immutable).
   - Public Outflow Mode 3 Spend = **45 bps** (0.45% flat).
   - Relayer Execution Overhead = **15%** buffer di atas gas aktual (`DEFAULT_EXECUTION_OVERHEAD_BPS = 1500`), atau **0 bps** bila direct self-broadcast.
3. **Zero Degrees of Freedom (DEC-046):**
   Setiap percabangan kondisional dummy wire R1CS wajib terikat kuadratik kanonikal (`cond * (val - canonical) == 0`).
4. **Jacobian Tangent Rigidity (DEC-047):**
   Setiap modifikasi sirkuit wajib mempertahankan $r_T = 0$ di seluruh 6 keluarga saksi adversarial (*Witness-Locality Sweep*).
5. **Two-Phase Commit (2PC) Note State Safety (DEC-024 / DEC-038):**
   Wallet state tidak pernah menandai note terbelanja sebelum on-chain transaction receipt terkonfirmasi valid.
