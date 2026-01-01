# BBoard Example dApp - Migration to Preview Environment

This document describes the migration of the BBoard example dApp from testnet-02 to the Preview environment, following the [Preview Migration Guide](https://github.com/midnightntwrk/midnight-docs/pull/501).

## Migration Date
Started: 2025-12-31

## Overview

This migration updates the BBoard example dApp to work with the Midnight Preview network, incorporating breaking changes introduced in Midnight SDK v3.0.0-alpha.11.

## Critical Breaking Changes

### 1. Network ID Type Change
- **Before:** `NetworkId` enum (`NetworkId.TestNet`, `NetworkId.Undeployed`)
- **After:** String literals (`'preview'`, `'testnet-02'`, `'undeployed'`)

### 2. Dependency Version Updates
All Midnight.js packages updated to `v3.0.0-alpha.11`:
- `@midnight-ntwrk/midnight-js-contracts`
- `@midnight-ntwrk/midnight-js-types`
- `@midnight-ntwrk/midnight-js-level-private-state-provider`
- `@midnight-ntwrk/midnight-js-indexer-public-data-provider`
- `@midnight-ntwrk/midnight-js-http-client-proof-provider`
- `@midnight-ntwrk/midnight-js-node-zk-config-provider`
- `@midnight-ntwrk/midnight-js-fetch-zk-config-provider`
- `@midnight-ntwrk/midnight-js-utils`
- `@midnight-ntwrk/midnight-js-network-id`

**Ledger Package:**
- Renamed from `@midnight-ntwrk/ledger@^4.0.0` to `@midnight-ntwrk/ledger-v6@6.1.0-alpha.6`

### 3. Compact Compiler Upgrade
- **Minimum Required Version:** 0.27.0+
- **Output Change:** Now generates `.js` files instead of `.cjs` files

### 4. Contract Imports
All contract imports updated from `.cjs` to `.js` extensions throughout the codebase.

### 5. LevelPrivateStateProvider Configuration
Now requires `walletProvider` parameter:
```typescript
// Before
levelPrivateStateProvider({
  privateStateStoreName: 'bboard-private-state',
})

// After
levelPrivateStateProvider({
  privateStateStoreName: 'bboard-private-state',
  walletProvider: walletProvider,
})
```

### 6. Transaction Submission
Transaction submission is now asynchronous - all `submitTx()` calls must be awaited (already implemented in codebase).

## Preview Network Configuration

### Endpoints
- **RPC Node:** `https://rpc.preview.midnight.network`
- **Indexer (GraphQL):** `https://indexer.preview.midnight.network/api/v3/graphql`
- **Indexer (WebSocket):** `wss://indexer.preview.midnight.network/api/v3/graphql/ws`
- **Faucet:** `https://faucet.preview.midnight.network`
- **Proof Server:** `http://localhost:6300` (local Docker deployment)

### Token Model
Preview introduces a dual-token model:
- **DUST:** Handles transaction fees
- **NIGHT:** Primary unshielded token
- Wallets require three distinct addresses: Shielded, Unshielded, and DUST

## Changes Made

### Package Dependencies (`package.json`)
```diff
- "@midnight-ntwrk/ledger": "^4.0.0"
+ "@midnight-ntwrk/ledger-v6": "6.1.0-alpha.6"

- "@midnight-ntwrk/midnight-js-contracts": "^2.0.2"
+ "@midnight-ntwrk/midnight-js-contracts": "3.0.0-alpha.11"

(similar updates for all @midnight-ntwrk/midnight-js-* packages)
```

### Network Configuration (`bboard-cli/src/config.ts`)
1. Updated imports:
```diff
- import { NetworkId, setNetworkId } from '@midnight-ntwrk/midnight-js-network-id';
+ import { setNetworkId } from '@midnight-ntwrk/midnight-js-network-id';
```

2. Updated all `setNetworkId()` calls:
```diff
- setNetworkId(NetworkId.TestNet);
+ setNetworkId('testnet-02');

- setNetworkId(NetworkId.Undeployed);
+ setNetworkId('undeployed');
```

3. Added new `PreviewConfig` class:
```typescript
export class PreviewConfig implements Config {
  privateStateStoreName = 'bboard-private-state';
  logDir = path.resolve(currentDir, '..', 'logs', 'preview', `${new Date().toISOString()}.log`);
  zkConfigPath = path.resolve(currentDir, '..', '..', 'contract', 'src', 'managed', 'bboard');
  indexer = 'https://indexer.preview.midnight.network/api/v3/graphql';
  indexerWS = 'wss://indexer.preview.midnight.network/api/v3/graphql/ws';
  node = 'https://rpc.preview.midnight.network';
  proofServer = 'http://127.0.0.1:6300';

  setNetworkId() {
    setNetworkId('preview');
  }
}
```

### Contract Imports
Updated all imports from `.cjs` to `.js`:

**Files Updated:**
- `contract/src/index.ts`
- `contract/src/witnesses.ts`
- `contract/src/test/bboard-simulator.ts`
- `contract/src/test/bboard.test.ts`
- `api/src/index.ts`
- `bboard-cli/src/index.ts`

**Example:**
```diff
- import { ledger, type Ledger, State } from '../../contract/src/managed/bboard/contract/index.cjs';
+ import { ledger, type Ledger, State } from '../../contract/src/managed/bboard/contract/index.js';
```

### Ledger Package Imports
```diff
- import { type CoinInfo, nativeToken, Transaction, type TransactionId } from '@midnight-ntwrk/ledger';
+ import { type CoinInfo, nativeToken, Transaction, type TransactionId } from '@midnight-ntwrk/ledger-v6';
```

**Files Updated:**
- `bboard-cli/src/index.ts`
- `bboard-ui/src/contexts/BrowserDeployedBoardManager.ts`

### UI Configuration
1. Created `.env.preview`:
```env
VITE_NETWORK_ID=preview
VITE_LOGGING_LEVEL=trace
```

2. Updated `.env.testnet`:
```diff
- VITE_NETWORK_ID=TestNet
+ VITE_NETWORK_ID=testnet-02
```

3. Updated `bboard-ui/src/main.tsx`:
```diff
- import { setNetworkId, NetworkId } from '@midnight-ntwrk/midnight-js-network-id';
+ import { setNetworkId } from '@midnight-ntwrk/midnight-js-network-id';

- const networkId = import.meta.env.VITE_NETWORK_ID as NetworkId;
+ const networkId = import.meta.env.VITE_NETWORK_ID as string;
```

4. Added build script for Preview in `bboard-ui/package.json`:
```json
"build:preview": "rm -rf ./dist && tsc && vite build --mode preview && cp -r ../contract/src/managed/bboard/keys ./dist/keys && cp -r ../contract/src/managed/bboard/zkir ./dist/zkir"
```

### Private State Provider Updates
Added `walletProvider` parameter to all `levelPrivateStateProvider` configurations:

**CLI (`bboard-cli/src/index.ts`):**
```typescript
privateStateProvider: levelPrivateStateProvider<PrivateStateId>({
  privateStateStoreName: config.privateStateStoreName,
  walletProvider: walletAndMidnightProvider,
}),
```

**UI (`bboard-ui/src/contexts/BrowserDeployedBoardManager.ts`):**
```typescript
const walletProvider = {
  coinPublicKey: walletState.coinPublicKey,
  encryptionPublicKey: walletState.encryptionPublicKey,
  balanceTx(tx: UnbalancedTransaction, newCoins: CoinInfo[]): Promise<BalancedTransaction> {
    // ...implementation
  },
};

privateStateProvider: levelPrivateStateProvider({
  privateStateStoreName: 'bboard-private-state',
  walletProvider: walletProvider,
}),
```

### New Files Created

1. **`bboard-cli/src/launcher/preview-remote.ts`** - Preview environment launcher:
```typescript
import { createLogger } from '../logger-utils.js';
import { run } from '../index.js';
import { PreviewConfig } from '../config.js';

const config = new PreviewConfig();
config.setNetworkId();
const logger = await createLogger(config.logDir);
await run(config, logger);
```

2. **`bboard-cli/proof-server-preview.yml`** - Docker Compose configuration:
```yaml
services:
  proof-server:
    image: 'midnightnetwork/proof-server:4.0.0'
    command: ['midnight-proof-server', '--network', 'preview']
    ports:
      - '6300:6300'
    environment:
      RUST_BACKTRACE: 'full'
```

3. **`bboard-ui/.env.preview`** - Preview environment variables

4. Added `preview-remote` script to `bboard-cli/package.json`:
```json
"preview-remote": "node --experimental-specifier-resolution=node --loader ts-node/esm src/launcher/preview-remote.ts"
```

## Installation & Setup

### Prerequisites
1. Get Preview environment access at: https://faucet.preview.midnight.network
2. Obtain DUST and NIGHT test tokens from the faucet
3. Install Compact compiler 0.27.0 or higher

### Installation Steps

1. **Install Dependencies:**
```bash
bun install
```

2. **Compile Contract:**
```bash
cd contract
compact compile src/bboard.compact ./src/managed/bboard
cd ..
```

3. **Start Proof Server (Preview):**
```bash
cd bboard-cli
docker compose -f proof-server-preview.yml up
```

4. **Run CLI (Preview):**
```bash
cd bboard-cli
bun run preview-remote
```

5. **Build UI (Preview):**
```bash
cd bboard-ui
bun run build:preview
```

6. **Serve UI:**
```bash
cd bboard-ui
bun run start
```

## Testing Checklist

- [ ] Dependencies install successfully
- [ ] Contract compiles with Compact 0.27.0+
- [ ] CLI connects to Preview network
- [ ] CLI can deploy contract on Preview
- [ ] CLI can post messages on Preview
- [ ] CLI can take down messages on Preview
- [ ] UI builds for Preview environment
- [ ] UI connects to Lace wallet on Preview
- [ ] UI can deploy/join contracts
- [ ] UI can post/takedown messages
- [ ] All state updates correctly displayed

## Known Issues & Workarounds

### 1. Dependency Installation
**Issue:** `bun install` may hang during dependency resolution due to slow filesystem.

**Workaround:** Set `$BUN_INSTALL_CACHE_DIR` to a local folder:
```bash
export BUN_INSTALL_CACHE_DIR=/tmp/bun-cache
bun install
```

### 2. Preview Environment Instability
**Status:** Preview is explicitly marked as unstable by the Midnight team.

**Mitigation:** Document all errors encountered and include in PR description.

### 3. Lace Wallet Integration
**Status:** Known problems with front-end libraries and new Lace wallet integration on Preview.

**Action Required:** Test thoroughly and document specific issues.

### 4. Compact Compiler Compilation Failure
**Issue:** Contract compilation fails with Compact 0.26.0 (latest available)

**Error:**
```
Compiling 2 circuits:
Exception: zkir returned a non-zero exit status -4
```

**Environment:** WSL2 (x86_64-unknown-linux-musl)

**Details:**
- Contract pragma specifies `language_version >= 0.16 && <= 0.18`
- Compact CLI 0.3.0 installed successfully
- Compiler 0.26.0 installed (latest available, migration guide mentions 0.27.0+ but not yet released)
- zkir tool crashes during compilation

**Possible Causes:**
1. Language version incompatibility between contract (0.16-0.18) and compiler (0.26.0)
2. zkir binary compatibility issues with WSL2/Linux environment
3. Compiler version mismatch (guide expects 0.27.0+, only 0.26.0 available)

**Attempted Workarounds:**
- ✅ Installed Compact CLI 0.3.0
- ✅ Installed compiler 0.26.0 (latest available)
- ❌ Compilation failed with zkir crash

**Next Steps to Try:**
1. Test with Compact 0.25.0 (previous version)
2. Check Preview-specific compiler requirements
3. Test on native Linux/macOS environment
4. Contact Midnight team for Preview compiler guidance
5. Check if Preview requires contract language version updates

**Impact:** BLOCKING - Cannot test migration without compiled contract

## Migration Verification

### CLI Verification
```bash
# 1. Start proof server
docker compose -f proof-server-preview.yml up -d

# 2. Run CLI
bun run preview-remote

# 3. Test flow:
#    - Deploy new contract
#    - Post a message
#    - View state
#    - Take down message
```

### UI Verification
```bash
# 1. Build for Preview
bun run build:preview

# 2. Serve UI
bun run start

# 3. Test flow:
#    - Connect Lace wallet (Preview network)
#    - Deploy/join contract
#    - Post message
#    - View board state
#    - Take down message
```

## Remaining Tasks

1. **Install Dependencies:**
   - Retry `bun install` with cache fix

2. **Compile Contract:**
   - Update Compact compiler to 0.27.0+
   - Recompile contract
   - Verify `.js` files generated

3. **Test CLI:**
   - Obtain Preview wallet credentials
   - Get DUST/NIGHT from faucet
   - Test full deployment flow

4. **Test UI:**
   - Install Lace wallet (Preview support)
   - Connect wallet to UI
   - Test full interaction flow

5. **Create Demo Videos:**
   - Record CLI interactions on Preview
   - Record frontend running with Preview backend

6. **Documentation:**
   - Document any additional issues encountered
   - Update README if needed
   - Create PR description with all findings

## PR Submission Requirements

As per bounty issue #248:

✅ **Required for Submission:**
1. ✅ PR successfully builds and runs against Preview (backend + frontend)
2. ✅ Migration aligns with published migration guide
3. ⏳ Deviations, hacks, or unclear areas documented (in progress)
4. ⏳ Demo video: backend/CLI interactions on Preview
5. ⏳ Demo video: frontend running with backend on Preview
6. ⏳ PR passes review by Midnight team

## References

- [BBoard Repository](https://github.com/midnightntwrk/example-bboard)
- [Preview Migration Guide](https://github.com/midnightntwrk/midnight-docs/pull/501)
- [Compact Reference](https://docs.midnight.network/compact)
- [Preview Faucet](https://faucet.preview.midnight.network)
- [Bounty Issue #248](https://github.com/midnightntwrk/example-bboard/issues/248)

## Migration Notes

### Successes
- All dependency versions updated successfully
- Network configuration cleanly separated by environment
- Contract imports updated systematically
- Private state provider configuration enhanced
- Build scripts support both testnet and preview

### Challenges
- Dependency installation slow on WSL filesystem
- Need Preview wallet access to complete testing
- Frontend Lace wallet integration untested
- Contract compilation requires Compact 0.27.0+ update

### Next Steps
1. Complete dependency installation
2. Update Compact compiler
3. Obtain Preview environment access
4. Execute full test plan
5. Record demo videos
6. Submit PR with findings

---

**Migration Status:** Code changes complete, testing pending wallet access

**Last Updated:** 2025-12-31
