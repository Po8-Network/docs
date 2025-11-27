# Po8 Integration Test Plan (Nov 2025)

This document outlines the steps to test the end-to-end integration of the Po8 Quantum-Resistant Blockchain, including the Core Node, Web Wallet, and Dapp.

## Prerequisites
- Rust (latest stable)
- Node.js (v18+) & NPM
- Google Chrome (or Chromium-based browser)

## 1. Start the Core Node

The Core Node runs the blockchain, EVM, and JSON-RPC server.

```bash
cd po8-core
# Run in development mode (ChainID: 1337)
cargo run -p po8-node
```

**Expected Output:**
- `Starting Po8 Node on Network: Development (ChainID: 1337)`
- `Node Identity (ML-DSA-65 Public Key size: ...)`
- `Miner Address: 0x...`
- `JSON-RPC listening on http://127.0.0.1:8833/rpc`
- `Mined Block 1! ...` (Mining logs should appear every second)

## 2. Install the Web Wallet

The Web Wallet manages ML-DSA keys and signs transactions.

1.  **Build the Wallet:**
    ```bash
    cd web-wallet
    npm install
    npm run build
    ```
2.  **Load into Chrome:**
    -   Open `chrome://extensions/`
    -   Enable **Developer mode** (top right).
    -   Click **Load unpacked**.
    -   Select the `web-wallet/dist` directory.
3.  **Create Account:**
    -   Open the extension popup.
    -   Click "Create New Wallet".
    -   Copy the displayed **Address** (starts with `0x`).

## 3. Fund the Wallet

Since this is a local devnet, you need to transfer funds from the genesis miner to your new wallet.

**Using curl (or Postman):**
Replace `YOUR_WALLET_ADDRESS` with the address copied in Step 2.

```bash
curl -X POST -H "Content-Type: application/json" --data '{
  "jsonrpc": "2.0",
  "method": "send_transaction",
  "params": [{
    "sender_pk": "GENESIS_PK_HEX", 
    "recipient": "YOUR_WALLET_ADDRESS",
    "amount": "100000000000000000000",
    "nonce": 1,
    "signature": "DUMMY_SIG_FOR_DEV_OR_IMPLEMENT_CLI_TOOL"
  }],
  "id": 1
}' http://127.0.0.1:8833/rpc
```

*Note: Since the Miner controls the genesis funds, you may need to use a CLI tool or a script in `po8-core` to send this initial transaction if you don't have the Miner's private key handy outside the node logs. For MVP testing, you can modify `po8-core/node/src/main.rs` to mint funds to your specific address on startup.*

**Easier Dev Method:**
Modify `po8-core/node/src/main.rs` line ~106:
```rust
evm_engine.mint("YOUR_WALLET_ADDRESS".to_string(), ...);
```
Restart the node.

## 4. Run the Dapp

The Dapp connects to the wallet and interacts with the blockchain.

```bash
cd dapp
# Install dependencies
cd frontend
npm install
# Start Frontend
npm run dev
```

1.  Open `http://localhost:5173` in Chrome.
2.  Click **Connect Po8 Wallet**.
3.  Accept the connection request (if prompt implemented) or verify account appears.
4.  Verify the Balance matches what you funded.

## 5. Deploy Smart Contract (Hardhat)

1.  **Configure Hardhat:**
    Ensure `dapp/hardhat.config.ts` points to `http://127.0.0.1:8833`.
2.  **Deploy:**
    ```bash
    cd dapp
    npx hardhat run scripts/deploy.ts --network po8
    ```
    *Note: Hardhat uses Ethers.js, which signs with ECDSA by default. Since Po8 requires ML-DSA, standard Hardhat deployment scripts will FAIL unless we use a custom signer or the Node accepts unsigned txs for dev (not recommended).*

    **Correct Approach for Po8:**
    Use the Dapp frontend to trigger a deployment via `eth_sendTransaction`. The Wallet will sign it with ML-DSA.

## 6. Test Transaction

1.  In the Web Wallet Popup:
    -   Click **Send**.
    -   Enter a random address (e.g., `0x1234...`) and Amount `10`.
    -   Click **Confirm**.
2.  **Verify:**
    -   Check Node logs for `Received Quantum TX`.
    -   Check Wallet "Recent Activity" list.



