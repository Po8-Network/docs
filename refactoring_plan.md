# Po8 Network: Core Refactoring Plan (Phase 1.5)

## Strategic Re-evaluation
The previous progress report incorrectly identified Phase 1 as "Complete". A deeper audit reveals that while the *structure* of the Post-Quantum components exists, the *verification logic* for the consensus mechanism (Proof-of-Useful-Work) is mathematically incomplete. The current implementation relies on a placeholder hash rather than the verifiable Freivalds' algorithm specified in the whitepaper.

Moving to networking/gossip (Phase 2) without a verifiable consensus mechanism would built a distributed system on a broken foundation. We must first harden the core implementations.

## Objectives

### 1. True TensorChain Verification (Freivalds' Algorithm)
**Current State:** The miner computes $A \times B = C$ and returns `Hash(Sum(C))`.
**Problem:** Validators cannot verify this without re-doing the $O(N^3)$ work, defeating the purpose of "Asymmetric Verification".
**Target State:**
*   **Miner (Prover):**
    1.  Computes $C = A \times B$.
    2.  Derives challenge vector $r$ pseudo-randomly from the block header/seed.
    3.  Computes proof vector $v = C \times r$.
    4.  Returns proof $(v, \text{hash}(C))$.
*   **Validator (Verifier):**
    1.  Re-generates $A, B$ (cheap-ish generation).
    2.  Derives same $r$.
    3.  Computes $y = B \times r$ ($O(N^2)$).
    4.  Computes $z = A \times y$ ($O(N^2)$).
    5.  Asserts $z == v$.
    *   This reduces verification cost from $O(N^3)$ to $O(N^2)$, allowing CPUs to verify NPU work.

### 2. Consensus Engine Hardening
**Current State:** `verify_block` checks signatures but ignores the PoW payload.
**Target State:**
*   Integrate the Freivalds check into `verify_block`.
*   Blocks with valid signatures but invalid matrix proofs must be rejected.

---

## Implementation Tasks

1.  **`po8-miner` (C++):** Update `mlx_bridge.cpp` to perform the vector-matrix multiplication $C \times r$ after the matrix multiplication.
    *   **Status:** ✅ **Completed**. Implemented `Xoshiro256PlusPlus` and updated `mlx_bridge.cpp` to output proof vector `v`.
2.  **`po8-miner` (Rust):** Update FFI to return the proof vector (size $N \times 1$) instead of just a 32-byte hash.
    *   **Status:** ✅ **Completed**. Updated `lib.rs` and `main.rs` to handle `Vec<f32>`.
3.  **`po8-consensus`:** Implement the $O(N^2)$ verification logic (can be pure Rust for now, or use `ndarray`/`matrixmultiply` crates if needed, but pure loop is fine for MVP).
    *   **Status:** ✅ **Completed**. Implemented `Xoshiro256PlusPlus` in Rust and added `verify_tensor_chain` logic to `ConsensusEngine`.

## Next Steps
- [ ] **EVM Un-stubbing**: Restore full EVM execution once `revm` dependencies are stable.
- [ ] **Integration Testing**: Run a full mining cycle to ensure the Rust verifier accepts the C++ miner's proofs.
