# Zeltra Protocol: A High-Throughput Shielded Settlement Rail and ZK-UTXO Privacy Layer on Arbitrum Stylus

**Living Technical Specification & Architecture Whitepaper**  
*Version:* `1.0.0-RC (Dynamic / Pre-Testnet Freeze)`  
*Date:* October 2026  
*Authors:* Zeltra Protocol Architecture & Cryptography Working Group  
*Codebase Reference:* [`Zeltra-Protocol`](file:///workspaces/Zeltra-Protocol/) (`zeltra-core`, `zeltra-contracts`, `zeltra-node`, `zeltra-sdk`, `zeltra-cli`)

---

## Abstract

This paper specifies the architecture, cryptographic foundations, and state machine of **Zeltra Protocol**, a high-throughput, non-custodial privacy and confidential settlement rail deployed on Arbitrum Stylus (WASM). Zeltra couples a pure Zero-Knowledge Unspent Transaction Output (ZK-UTXO) state model over the BLS12-381 pairing-friendly elliptic curve with direct execution via host precompiles (EIP-2537: `0x0c` MSM and `0x0f` 4-pairing). 

Transactions are executed across a strict 3-mode Universal 2-in-2-out JoinSplit circuit with quadratic companion constraints ensuring zero unconstrained degrees of freedom (Zero-DoF) and provable tangent space rigidity ($r_T = 0$). Inflow into the shielded accumulator achieves zero friction (0 bps deposit fee) via direct Merkle Mountain Range (MMR) insertion. Circular in-pool velocity is unencumbered by rent extraction (0 bps protocol fee), while public outflow withdrawal is subject to a flat 45 bps protocol fee. Note payloads are encrypted via a 96-byte committing Authenticated Encryption with Associated Data (AEAD) memo ($EtH-KS$ transform) paired with a 1-byte View Tag and branchless Constant-Time Flag Oblivious Evaluation (CT-FOE). Finally, regulatory defensibility is achieved through an unhosted direct self-broadcast safe harbor and single-note CEX selective disclosure proofs.

---

## 1. Introduction & Motivation

### 1.1 The Account-State Privacy Problem
Public account-based blockchains expose complete transaction graphs, account balances, and counterparty topologies. While pseudonymous, statistical heuristic clustering, graph analysis, and maximal extractable value (MEV) searchers render financial operations on public ledgers vulnerable to surveillance, front-running, and competitive intelligence leakage.

Previous attempts to introduce confidential transactions on EVM-based blockchains suffer from three fundamental bottlenecks:
1. **Computational Inefficiency:** Executing bilinear pairing checks (`e(P, Q)`) and large-field arithmetic over 256-bit scalar fields in standard EVM bytecode incurs prohibitive gas costs (typically $350{,}000 - 900{,}000$ gas per verification).
2. **State Bloat & Tree Rebalancing Overhead:** Classical fixed-depth Merkle trees (e.g., depth-20 or depth-32 binary trees) demand continuous hashing during insertion, imposing an unbounded state accumulation burden on validator nodes.
3. **Regulatory Fragility:** Pure mixer designs lacking cryptographic provenance tools lead to blanket regulatory sanctioning, whereas centralized custodian relays expose operators to money transmitter liabilities under 18 U.S.C. § 1960.

### 1.2 The Zeltra Approach
Zeltra addresses these challenges through a unified systems engineering approach:
* **Arbitrum Stylus WASM Host Precompiles:** Zeltra executes contract state transitions in compiled WebAssembly (WASM) using Rust, delegating heavy BLS12-381 curve arithmetic to EIP-2537 host precompiles (`0x0c` and `0x0f`), reducing on-chain verification overhead by an order of magnitude.
* **Dual Sovereign Multi-Asset Rails:** Native support for both USDC (Asset ID 1, 6-decimal micro-units) and WETH / Native ETH (Asset ID 0, 9-decimal Gwei-normalized units) with on-contract payable auto-wrapping.
* **Flat Merkle Mountain Range (MMR) Accumulator:** An append-only logarithmic accumulator that eliminates tree rebalancing overhead, augmented with bounded ring-buffer root eviction (DEC-053).
* **Autonomous AI Agent Micropayments:** A pure ZK-UTXO HTTP 402 Facilitator gateway enabling machine-to-machine sub-cent API settlement without user session state.

---

## 2. Cryptographic Foundations & Notation

### 2.1 Curve & Field Parameters
Zeltra is constructed over the Barreto-Naehrig/Barreto-Lynn-Scott pairing-friendly elliptic curve **BLS12-381**:
* **Base Field ($\mathbb{F}_q$):** $q = 0x1a0111ea397fe69a4b1ba7b640a326e49061d94294013d07ca39a79b9746b267e88f82f494f2d1b0f87efcae96e1f61$ (381-bit prime).
* **Scalar Field ($\mathbb{F}_r$):** $r = 0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001$ (255-bit prime, order of $\mathbb{G}_1, \mathbb{G}_2, \mathbb{G}_T$).
* **Pairing:** $e: \mathbb{G}_1 \times \mathbb{G}_2 \to \mathbb{G}_T$.

### 2.2 Algebraic Hash Primitives
All circuit-internal commitments, nullifiers, and Merkle tree compressions utilize the **Poseidon** algebraic sponge hash function:
* **$\text{Poseidon-W3}$:** Width $t = 3$ ($\alpha = 5$, 8 full rounds, 56 partial rounds) for nullifiers and challenge digests.
* **$\text{Poseidon-W5}$:** Width $t = 5$ ($\alpha = 5$, 8 full rounds, 60 partial rounds) for 5-element note commitment evaluation.
* **Grain-128 LFSR:** Round constants and Maximum Distance Separable (MDS) matrices are deterministically generated via the canonical Grain-128 LFSR stream (DEC-021) to prevent trapdoors.

### 2.3 Note Structure & Commitments
A confidential note $\mathbf{n}$ is represented by the 5-tuple:
$$\mathbf{n} = (\text{value}, \text{owner\_key}, \rho, r, \text{asset\_id})$$
where:
* $\text{value} \in [0, 2^{64}-1]$ represents the asset denomination.
* $\text{owner\_key} \in \mathbb{F}_r$ is the public spending authorization key.
* $\rho \in \mathbb{F}_r$ is a unique diversifier ensuring nullifier uniqueness.
* $r \in \mathbb{F}_r$ is cryptographic blinding randomness sampled uniformly from $\mathbb{F}_r^*$.
* $\text{asset\_id} \in \{0, 1\}$ denotes the asset type ($0 = \text{WETH}, 1 = \text{USDC}$).

The note commitment $cm \in \mathbb{F}_r$ is computed via:
$$cm = \text{Poseidon-W5}(\text{value}, \text{owner\_key}, \rho, r, \text{asset\_id})$$

The note nullifier $nf \in \mathbb{F}_r$ is computed from the nullifier key $nk$:
$$nf = \text{Poseidon-W3}(nk, \rho)$$

### 2.4 Key Hierarchy & Address Derivation
Zeltra defines a 4-tier cryptographic key hierarchy:
1. **Spending Key ($sk \in \mathbb{F}_r$):** Master spending secret held strictly by the client.
2. **Nullifier Key ($nk \in \mathbb{F}_r$):** Derived via $nk = \text{Poseidon-W3}(sk, \text{TAG}_{\text{null}})$.
3. **Owner Key ($\text{owner\_key} \in \mathbb{F}_r$):** Derived via $\text{owner\_key} = \text{Poseidon-W3}(sk, \text{TAG}_{\text{owner}})$.
4. **Full Viewing Key ($fvk$):** The pair $(nk, \text{owner\_key})$, permitting complete transaction history auditability without spending authority.
5. **ZSA Public Address:** Encoded in BIP-350 Bech32m with human-readable prefix `zs1`, packing $(\text{owner\_key}, pk_{\text{ivk}})$.

---

## 3. Circuit Architecture & Soundness Formulations

Zeltra operates using **Arkworks 0.6.0 Groth16** SNARKs over BLS12-381. Verification in contract memory asserts:
$$e(-A, B) \cdot e(\alpha, \beta) \cdot e(L, \gamma) \cdot e(C, \delta) = 1_{\mathbb{G}_T}$$
where $L = \sum_{i=0}^{l} x_i \cdot [\gamma^{-1}(\beta u_i(x) + \alpha v_i(x) + w_i(x))]_1$.

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      R1CS JOIN-SPLIT CIRCUIT                           │
 │                                                                        │
 │  [Input Notes: In_1, In_2]                 [Output Notes: Out_1, Out_2]│
 │            │                                            │              │
 │            ▼                                            ▼              │
 │  ┌───────────────────┐                        ┌───────────────────┐    │
 │  │ Poseidon-W5 (cm)  │                        │ Poseidon-W5 (cm)  │    │
 │  └─────────┬─────────┘                        └─────────▲─────────┘    │
 │            │                                            │              │
 │            ▼                                            │              │
 │  ┌──────────────────────────────────────────────────────┴─────────┐    │
 │  │              VALUE CONSERVATION CONSTRAINT                      │    │
 │  │  Val_1 + Val_2 == Payout + P_Fee + Exec_Fee + Out_1 + Out_2    │    │
 │  └─────────────────────────────────┬───────────────────────────────┘    │
 │                                    │                                    │
 │                                    ▼                                    │
 │  ┌────────────────────────────────────────────────────────────────┐    │
 │  │           ZERO-DoF COMPANION CONSTRAINTS (DEC-046)             │    │
 │  │  in2_is_dummy * (owner_2 - 0) == 0                             │    │
 │  │  in2_is_dummy * (rho_2 - 0) == 0;  (1 - has_change) * val == 0 │    │
 │  └─────────────────────────────────┬──────────────────────────────┘    │
 │                                    │                                    │
 │                                    ▼                                    │
 │  ┌────────────────────────────────────────────────────────────────┐    │
 │  │            19 SEMANTIC PUBLIC INPUTS BINDING                   │    │
 │  │ [root, nf_1, nf_2, cm_1, cm_2, recipient, amounts, fees, ...]  │    │
 │  └────────────────────────────────────────────────────────────────┘    │
 └────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Universal JoinSplit Circuit (`19 Public Inputs`)
The universal 2-in-2-out JoinSplit circuit (`zeltra-core/src/joinsplit_circuit.rs`) governs note minting, consolidation, in-pool transfers, and public spend outflows. It is parameterized by exactly 19 public inputs:

$$\mathbf{x}_{\text{JoinSplit}} = \begin{pmatrix}
1. & \text{root} & 8. & \text{protocol\_fee} & 15. & \text{expiry} \\
2. & \text{nullifier\_1} & 9. & \text{execution\_fee} & 16. & \text{flags\_packed} \\
3. & \text{nullifier\_2} & 10. & \text{max\_execution\_fee} & 17. & \text{is\_rollover} \\
4. & \text{out\_cm\_1} & 11. & \text{quote\_hash\_hi} & 18. & \text{has\_change\_1} \\
5. & \text{out\_cm\_2} & 12. & \text{quote\_hash\_lo} & 19. & \text{has\_change\_2} \\
6. & \text{recipient} & 13. & \text{chain\_id} & & \\
7. & \text{merchant\_amount} & 14. & \text{contract\_address} & &
\end{pmatrix}$$

#### Value Conservation Gate
The arithmetic circuit enforces strict multi-asset balance conservation over $\mathbb{F}_r$:
$$\text{val}_{\text{in},1} + \text{val}_{\text{in},2} = \text{merchant\_amount} + \text{protocol\_fee} + \text{execution\_fee} + \text{val}_{\text{out},1} + \text{val}_{\text{out},2}$$
subject to intra-asset congruence constraints:
$$\text{asset\_id}_{\text{in},1} = \text{asset\_id}_{\text{in},2} = \text{asset\_id}_{\text{out},1} = \text{asset\_id}_{\text{out},2}$$

### 3.2 Quadratic Companion Constraints & Zero-DoF (DEC-046)
Conditional branches (such as dummy input note 2 or inactive change outputs) in naive R1CS formulations introduce unconstrained degrees of freedom. An adversary can malleate witness values without altering the public statement.

Zeltra strictly enforces canonical companion constraints across all conditional wires:
$$\begin{aligned}
\text{in2\_is\_dummy} \cdot (\text{owner\_key}_2 - 0) &= 0 \\
\text{in2\_is\_dummy} \cdot (\rho_2 - 0) &= 0 \\
\text{in2\_is\_dummy} \cdot (r_2 - 0) &= 0 \\
\text{in2\_is\_dummy} \cdot (\text{index}_2 - 0) &= 0 \\
(1 - \text{has\_change}_1) \cdot (\text{out\_val}_1 - 0) &= 0 \\
(1 - \text{has\_change}_2) \cdot (\text{out\_val}_2 - 0) &= 0
\end{aligned}$$
When a branch is deactivated, its algebraic degree of freedom collapses deterministically to the unique canonical constant zero.

### 3.3 Tangent Rigidity & Augmented Jacobian Soundness (DEC-047)
To formally prevent under-constrained witness vulnerabilities, the circuit system is evaluated against the Augmented Jacobian matrix:
$$M(a) = \begin{pmatrix} J(a) \\ E_F \end{pmatrix} \in \mathbb{F}_r^{(m + k) \times n}$$
where $J(a) = \left. \frac{\partial f_i}{\partial z_j} \right|_a$ represents the Jacobian of R1CS polynomials, and $E_F$ projects financial target signals. Tangent rigidity demands that the financial target nullity satisfies:
$$r_{T_{\text{financial}}} = 0$$
across the complete 6-family adversarial witness sweep (FAM-1: Standard witness, FAM-2: Single active branch, FAM-3: Dual inactive branch, FAM-4: Zero boundary, FAM-5: Field overflow boundary $r-1$, FAM-6: Non-canonical selector perturbation).

---

## 4. Committing AEAD Note Protocol (DEC-048 & DEC-049)

### 4.1 Wire Format (96-Byte Calldata Memo)
To achieve transaction graph detachment without inflating on-chain state storage, encrypted note plaintexts are emitted as event data (`NoteCommitmentAppended`) in calldata. The payload is packaged into a fixed 96-byte envelope:

```text
 0        1                               33                              80              96
 ┌────────┬───────────────────────────────┬───────────────────────────────┬───────────────┐
 │View Tag│  Ephemeral Public Key (EPK)   │   Encrypted Ciphertext (C)    │ Commitment Tag│
 │ (1 B)  │           (32 B)              │            (47 B)             │    (16 B)     │
 └────────┴───────────────────────────────┴───────────────────────────────┴───────────────┘
```

### 4.2 Key Separation & $EtH-KS$ Transform
To eliminate partitioning oracle vulnerabilities (where invalid ciphertexts leak viewing key bits), Zeltra applies the Encrypt-then-Hash Key Separation ($EtH-KS$) transform:
1. Diffie-Hellman Key Exchange: $S = \text{X25519}(\text{ephemeral\_sk}, pk_{\text{ivk}})$.
2. RCR-PRG Key Separation:
   $$(K_{\text{enc}}, K_{\text{cmt}}, N_7) \leftarrow \text{HKDF-SHA256}(S, \text{salt} = \text{asset\_id})$$
3. Encryption: $C = \text{ChaCha20}(K_{\text{enc}}, N_7, \text{plaintext})$ where $\text{plaintext} = \text{value} \parallel \text{owner\_key} \parallel \text{asset\_id} \parallel r_{\text{seed}}$.
4. Cryptographic Commitment Tag: $T = \text{HMAC-SHA256}(K_{\text{cmt}}, N_7 \parallel \text{associated\_data} \parallel C)_{[0..16]}$.

### 4.3 Constant-Time Flag Oblivious Evaluation (CT-FOE DEC-049)
Client note decapsulation executes branchlessly using constant-time conditional selectors (`subtle::Choice`) over field element representations. This eliminates timing leakage side-channels during scanning (mitigating Flag DPA and Guessing DPA), satisfying Welch's t-test bound $|t| < 4.5$ under the Dudect differential testing framework.

---

## 5. On-Chain Settlement on Arbitrum Stylus

### 5.1 Host Precompile Interface (EIP-2537)
Smart contract execution (`zeltra-contracts/src/`) is implemented in Rust targeting `wasm32-unknown-unknown` under the Arbitrum Stylus SDK 0.10.7. Bilinear pairings are executed by invoking native Nitro host precompiles:
* `0x0c`: Multi-Scalar Multiplication on $\mathbb{G}_1$ (`call_msm_g1`).
* `0x0f`: Bilinear 4-Pairing Check (`call_pairing`).

### 5.2 Strict 3-Mode JoinSplit State Transitions
The contract entrypoint `spend_joinsplit` processes proofs according to three mutually exclusive execution modes:

```text
                               ┌─────────────────────────┐
                               │   spend_joinsplit(...)  │
                               └────────────┬────────────┘
                                            │
                       ┌────────────────────┼────────────────────┐
                       ▼                    ▼                    ▼
             ┌───────────────────┐┌───────────────────┐┌───────────────────┐
             │      MODE 1       ││      MODE 2       ││      MODE 3       │
             │  MINT / ROLLOVER  ││  IN-POOL TRANSFER ││   PUBLIC SPEND    │
             ├───────────────────┤├───────────────────┤├───────────────────┤
             │• recipient == 0x0 ││• recipient == 0x0 ││• recipient != 0x0 │
             │• merchant_val == 0││• merchant_val == 0││• merchant_val > 0 │
             │• protocol_fee = 0 ││• protocol_fee = 0 ││• protocol_fee=45bp│
             │• Payout: None     ││• Dual 96B Memos   ││• Change Note Emit │
             │• Liquidity stays  ││• Zero Public Flow ││• ERC-20 Transfer  │
             └───────────────────┘└───────────────────┘└───────────────────┘
```

### 5.3 Bounded State Growth Control (DEC-053)
Following the state suppression principles of *ammBoost* (IACR 2024/1021), Zeltra bounds historical Merkle Mountain Range root storage to $O(1)$. Smart contracts record root freshness in a circular ring-buffer:
$$\text{slot} = \text{root\_history\_index} \pmod{100}$$
Old roots displaced after 100 insertions are evicted from the active verification mapping, preventing contract storage exhaustion while preserving an ample multi-hour reorg window.

### 5.4 Siloed Multi-Asset Solvency Invariant
After every balance-mutating function, the contract enforces the solvency invariant across each independent asset $a \in \{0, 1\}$:
$$\text{ContractBalance}(a) \ge \text{user\_note\_liabilities}[a] + \text{accrued\_execution\_fee\_liabilities}[a] + \text{realized\_protocol\_fees}[a]$$
Any execution resulting in a deficit triggers an immediate transaction revert (`SOLVENCY_VIOLATION`).

---

## 6. Institutional Compliance & Regulatory Safe Harbor

### 6.1 Selective Disclosure Circuit (SEC-002 & DEC-040)
To enable legitimate institutional compliance (e.g., Binance or Coinbase deposit provenance checks) without revealing global net worth or spending authorization, Zeltra defines the `SelectiveDisclosureCircuit`:
$$\mathcal{R}_{\text{Disclosure}} = \left\{ (root, nf, val, asset, H_{\text{chal}}), (sk, nk, \rho, r) : \begin{aligned}
\text{owner} &= \text{Poseidon-W3}(sk, \text{TAG}_{\text{owner}}) \\
nf &= \text{Poseidon-W3}(nk, \rho) \\
cm &= \text{Poseidon-W5}(val, \text{owner}, \rho, r, asset) \in \text{MMR}(root) \\
H_{\text{chal}} &= \text{Poseidon-W3}(\text{owner}, \text{nonce}, \text{target\_address})
\end{aligned} \right\}$$
The cryptographic constraint $\text{owner\_key} = \text{Poseidon}(sk, \text{TAG})$ (SEC-002) binds the spending key directly to the note provenance without exposing $sk$ to the verifier.

### 6.2 Zero-Custody Unhosted Safe Harbor (DEC-054)
Under DEC-054, Zeltra eliminates broker/dealer intermediary definitions:
1. **Direct Self-Broadcast (`self_broadcast.rs`):** Users may encode calldata locally (`encode_joinsplit_calldata`) and broadcast directly to the Stylus contract via standard personal RPCs. The execution fee is exactly 0 bps (`execution_fee = 0`), paying only L2 base gas.
2. **Terminology Purge:** All broker-dealer terminology (`markup`) has been excised from protocol code and EIP-712 typed schemas in favor of `executionOverheadFee` (15% buffer strictly bounding gas volatility).

---

## 7. Protocol Economics & Fee Schedule

Zeltra operates on a volume-over-TVL economic velocity model:

| Transaction Flow | Protocol Fee Rate | Relayer Execution Fee | Economic Rationalization |
|---|---|---|---|
| **Direct MMR Inflow** | **0 bps (0.00%)** | $0$ (User pays deposit gas) | Zero-friction entry to maximize liquidity velocity |
| **Cross-Chain CCIP Inflow** | **0 bps (0.00%)** | Router Network Gas | Seamless cross-chain onboarding (DEC-055) |
| **In-Pool Transfer (Mode 2)** | **0 bps (0.00%)** | Quoted L2 Gas + 15% Overhead | Unrestricted circular peer-to-peer velocity |
| **Epoch Rollover (Mode 1)** | **0 bps (0.00%)** | Quoted L2 Gas + 15% Overhead | Free note consolidation and dust prevention |
| **Public Outflow Spend (Mode 3)** | **45 bps (0.45%)** | Quoted L2 Gas + 15% Overhead | Protocol monetization upon fiat off-ramp |
| **Direct Self-Broadcast** | **45 bps (Spend only)**| **0 bps (0.00%)** | Unhosted non-custodial safe harbor (DEC-054) |

*Dust Thresholds:* Standalone rollovers enforce minimum value boundaries to protect relayer gas solvency: $\$2.00$ USDC ($2{,}000{,}000$ micro-units) and $0.001$ WETH ($1{,}000{,}000$ Gwei).

---

## 8. Empirical Benchmarks & Testnet Evaluation Metrics

> **Methodological Note:** In accordance with Zeltra's engineering standards, all production marketing claims are gated on empirical measurements. The performance profile is segmented into *Local Cryptographic Micro-Benchmarks* (verified on development hardware) and the *Arbitrum Sepolia Testnet Evaluation Framework* (slated for deployment during Phase 4 / Gate G).

### 8.1 Local Cryptographic Micro-Benchmarks
*Measurements executed on Linux x86_64, AMD EPYC / Intel Xeon 3.2GHz, Arkworks 0.6.0:*

| Primitif / Modul | Parameter / Ukuran | Latensi / Throughput | Status Verifikasi |
|---|---|---|---|
| **Poseidon-W5 Hash** | 5 inputs over $\mathbb{F}_r$ | $< 15\ \mu\text{s}$ | Verified (`bench.rs`) |
| **In-Memory Flat MMR Proof** | 1,000 leaves | $43.4\ \mu\text{s}$ ($12.2\times$ speedup) | Verified (`benchmark_results_dec045.json`) |
| **In-Memory Flat MMR Proof** | 5,000 leaves | $173.1\ \mu\text{s}$ | Verified (`benchmark_results_dec045.json`) |
| **MMR Node Ingestion** | Batch Append | $1{,}869\ \text{leaves/sec}$ | Verified (`benchmark_results_dec045.json`) |
| **Committing AEAD Decrypt** | 96-byte memo | $8.2\ \mu\text{s}$ | Verified (`memo_committing_tests.rs`) |
| **CT-FOE Branchless Scan** | 1-byte View Tag Filter | $99.6\%\ \text{rejection}$ at $0.03\ \mu\text{s}$ | Verified (`test_dec048_committing_aead.py`) |
| **Augmented Jacobian Rigidity** | 6-family witness sweep | $r_T = 0$ (Nullity verified) | Verified (`formal_tests.rs`, 26/26 passed) |
| **Groth16 JoinSplit Prover** | 2-in-2-out R1CS | $\sim 2.8 - 4.1\ \text{s}$ (Native CPU) | Verified (`joinsplit_circuit.rs`) |

### 8.2 Arbitrum Sepolia Testnet Framework (Phase 4 / Gate G Live Targets)
The following matrix defines the benchmark targets to be certified with on-chain receipts upon deployment of `zeltra-contracts` to Arbitrum Sepolia:

```text
[TESTNET BENCHMARK TARGET MATRIX — TO BE FROZEN AT GATE G]
┌─────────────────────────────────┬───────────────────┬───────────────────┐
│ Operation Metric                │ Stylus WASM (Est.)│ Legacy EVM (Sol.) │
├─────────────────────────────────┼───────────────────┼───────────────────┤
│ deposit_direct_note (USDC)      │ ~ 42,000 gas      │ ~ 115,000 gas     │
│ deposit_direct_native_eth (WETH)│ ~ 48,000 gas      │ ~ 125,000 gas     │
│ ccip_receive (Inbound Mint)     │ ~ 85,000 gas      │ ~ 220,000 gas     │
│ spend_joinsplit (Mode 2 P2P)    │ ~ 145,000 gas     │ ~ 680,000 gas     │
│ spend_joinsplit (Mode 3 Outflow)│ ~ 178,000 gas     │ ~ 740,000 gas     │
│ EIP-2537 4-Pairing Check (0x0f) │ ~ 72,000 gas      │ ~ 340,000 gas     │
│ E2E Relayer Turnaround Latency  │ < 1,200 ms        │ > 4,500 ms        │
└─────────────────────────────────┴───────────────────┴───────────────────┘
```

---

## Appendix D: Gas, Latency, and Proving Benchmark Standards

Angka performa hanya dapat dianggap sebagai benchmark protokol apabila berasal dari eksekusi yang dapat direproduksi (*reproducible execution*). Benchmark kriptografi lokal, estimasi simulator, dan transaksi on-chain harus dilaporkan sebagai kategori yang terpisah secara tegas.

### D.1 Matriks Tolok Ukur Utama
| Metrik | Unit / Pelaporan | Bukti yang Wajib Disertakan |
|---|---|---|
| **Deposit USDC / WETH / Native ETH** | `gasUsed`; L1 data fee; biaya total ETH / USD | Chain ID, tx hash, block number, receipt, effective gas price |
| **In-Pool Transfer (Mode 2 JoinSplit 2-in/2-out)** | `gasUsed`; calldata bytes; biaya per transfer | Proof Groth16 dan tx receipt on-chain yang berhasil |
| **Public Spend (1-in/1-out PrivateNote)** | `gasUsed`; L1 fee; L2 execution fee | Receipt + jenis aset + recipient address |
| **Public Spend (Mode 3 JoinSplit 2-in/2-out)** | `gasUsed`; L1 fee; L2 execution fee | Receipt + jumlah note input/output |
| **Groth16 Prover Native / WASM** | median / p95 / p99 (ms), peak memory (MB) | CPU/browser spec, git commit SHA, setup artifacts, jumlah iterasi |
| **Groth16 Verifier Native / Host** | latensi native (ms); EVM gas untuk on-chain | Membedakan verifikasi native dan precompile host EIP-2537 (`0x0c`, `0x0f`) |
| **MMR Append, Proof, & Wallet Scan** | median / p95 / p99 (ms), throughput (leaves/s) | Jumlah leaf, spesifikasi perangkat, jumlah sampel |
| **Relayer End-to-End Latency** | Quote submission $\to$ Mempool $\to$ Confirmation | RPC endpoint, concurrency (RPS), success rate, target chain |

### D.2 Standar Pelaporan Gas
1. **Jumlah Sampel Minimum:** Gunakan sekurang-kurangnya 30 transaksi sukses per jenis operasi dengan kondisi root / ukuran pool yang dinyatakan secara transparan; laporkan nilai median, p95, minimum–maksimum, dan fee riil.
2. **Pemisahan Komponen Biaya Arbitrum:** Pecah biaya eksekusi L2 (`executionGasUsed`) dan komponen data L1 (`l1DataFee`) bila didukung receipt/API yang dipakai. Dilarang keras menyamakan `gasUsed` dengan gas price atau biaya total transaksi.
3. **Rumus Praktis:**
   $$\text{Transaction Fee} = (\text{Gas Used} \times \text{Effective Gas Price}) + \text{L1 Data Fee}$$
   Konversi ke denominasi USD memerlukan pencatatan harga ETH/USDC pada waktu pengukuran disertai sumber oracle dan timestamp Unix.

### D.3 Instrumen Pengukuran dalam Codebase
* `zeltra-core/src/bin/bench.rs`: Pengukuran Poseidon W3/W5, Flat MMR, JoinSplit Groth16 prove/verify pada harness CPU lokal.
* `zeltra-sdk/src/bin/bench_e2e.rs`: Pengukuran persiapan deposit, MMR proof generation, dan verifikasi PrivateNote.
* `scripts/benchmark_mmr_bottleneck.py`: Pengukuran beban throughput MMR, latensi endpoint HTTP Axum (p50/p95/p99), konkurensi cache, dan efisiensi wallet scanning 1-byte view tag.

---

## Appendix E: Format Hasil Benchmark Publik (Pre-Testnet Certification)

Saat hasil transaksi testnet tersedia di Arbitrum Sepolia, publikasi resmi wajib menggunakan format tabel ringkas berikut. **Sel yang belum mempunyai receipt valid sengaja dikosongkan (`—`) agar pembaca tidak mengira angka proyeksi sebagai data lapangan:**

| Operasi Transaksi | Median Gas | p95 Gas | Median Fee (ETH) | Referensi Tx Hash (Arbitrum Sepolia) |
|---|---|---|---|---|
| **Deposit Direct Note (USDC)** | — | — | — | *Pending Gate G Deployment* |
| **Deposit Direct Native ETH (WETH)** | — | — | — | *Pending Gate G Deployment* |
| **CCIP Inbound Deposit (`ccip_receive`)** | — | — | — | *Pending Gate G Deployment* |
| **Shielded In-Pool Transfer (Mode 2 JoinSplit)** | — | — | — | *Pending Gate G Deployment* |
| **Public Outflow Spend (1-in/1-out Note)** | — | — | — | *Pending Gate G Deployment* |
| **Public Outflow Spend (Mode 3 JoinSplit 2-in/2-out)** | — | — | — | *Pending Gate G Deployment* |

### Reproducibility Manifest Minimum
Setiap publikasi hasil benchmark wajib menyertakan manifes reproduksi berikut:
* Git Commit SHA: `[HEAD commit hash]`
* Target Chain ID: `421614` (Arbitrum Sepolia)
* Smart Contract Address: `[Stylus Contract Address]`
* Contract Bytecode Hash / Verification Digest
* Groth16 Verifying Key Digest (`JoinSplitCircuit` & `PrivateNoteCircuit`)
* Proving Hardware Spec (CPU, RAM, Core count) & Client Environment (OS / Browser WASM version)
* Batch Size & Jumlah Sampel Transaksi ($N \ge 30$)
* Raw CSV Dataset & Daftar Tx Hashes publik yang dapat diverifikasi di Arbiscan.

---

## 9. Conclusion

Zeltra Protocol demonstrates that zero-knowledge financial privacy does not require sacrificing throughput, composability, or legal defensibility. By coupling the mathematical rigour of BLS12-381 ZK-UTXO circuits with the native execution efficiency of Arbitrum Stylus and Chainlink CCIP v2, Zeltra establishes an industrial-grade settlement rail capable of powering high-velocity retail commerce, institutional liquidity, and autonomous machine-to-machine economies.

---

## 10. References

1. **Jens Groth (2016):** *"On the Size of Pairing-based Non-interactive Arguments"*, Advances in Cryptology – EUROCRYPT 2016.
2. **Lorenzo Grassi et al. (2019):** *"Poseidon: A New Hash Function for Zero-Knowledge Proof Systems"*, USENIX Security Symposium.
3. **Dr. Zachary J. Williamson (2018):** *"The AZTEC Protocol: Anonymous Zero-knowledge Transactions with Efficient Communication"*, Aztec Protocol Specification.
4. **Alexey Pertsev, Roman Semenov, Roman Storm (2019):** *"Tornado Cash Privacy Solution"*, Version 1.4.
5. **RAILGUN Project (2021):** *"RAILGUN: On-Chain Private Transactions & Smart Contract Interactions"*, Technical Whitepaper.
6. **Kazuhiko Minematsu, Akiko Inoue (2026):** *"EtH-KS: Generic Transform for Committing AEAD with Key Separation"*, IACR Cryptology ePrint Archive, Report 2026/240.
7. **Duyên Pay, Thomas Peters, François-Xavier Standaert (2026):** *"In Mid-Stream: Removing the FO-Transform Helps against Leakage but is not Enough"*, IACR Communications in Cryptology, Vol. 3, No. 1.
8. **Nicolas Michel, Mohamed E. Najd, Ghada Almashaqbeh (2024/2025):** *"ammBoost: State Growth Control for AMMs"*, IEEE Transactions on Services Computing / IACR ePrint 2024/1021.
9. **Peter Todd (2016):** *"Merkle Mountain Ranges: An append-only commitment scheme for blockchain state"*, Bitcoin Development Mailing List.
10. **EIP-2537 (2020):** *"Precompile for BLS12-381 curve operations"*, Ethereum Improvement Proposals.
