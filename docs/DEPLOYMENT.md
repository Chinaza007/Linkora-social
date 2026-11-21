# Contract deployment: Testnet and Mainnet

Use this runbook to build, deploy, initialize, and verify the core Linkora contract.
The manual commands below use explicit RPC URLs and network passphrases. The existing
[`deploy_testnet.sh`](../scripts/deploy_testnet.sh) helper and its limitations are
documented separately below. For PostgreSQL, Redis, the indexer, DM relay, and analytics
oracle, use the [backend deployment guide](./deployment.md).

## Prerequisites and release inputs

Run Bash commands from the repository root. Install Rust/Cargo, the `wasm32v1-none`
target, and a Stellar CLI version compatible with the locked Soroban SDK (26.1.0).
For SDK/application builds, use Node from [`.node-version`](../.node-version) and
pnpm from [`package.json`](../package.json), currently Node 20 and pnpm 9.15.9.
Record actual tool versions in the release record rather than assuming that `latest`
is reproducible.

```bash
rustup target add wasm32v1-none
cargo --version
stellar --version
node --version
pnpm --version
git rev-parse HEAD
pnpm install --frozen-lockfile
```

Have a funded source account, independently checked treasury address, and agreed fee in
basis points (`250` means 2.5%). Testnet assets have no production value; Mainnet requires
real XLM for reserves, resource fees, and transactions. Estimate costs by simulation with
the selected RPC rather than assuming a fixed funding amount.

These instructions deploy **only** `linkora-contracts`. Token factory, referral, vesting,
and multicall are separate workspace contracts with their own deployment/configuration
requirements. In particular, factory initialization needs an uploaded token Wasm hash,
referral needs a token address, and vesting needs an admin and treasury. Do not use the
core contract ID in place of those IDs.

## Build the release artifact

Use direct Cargo commands so a failed build/test stops the process: the current
`packages/contracts/package.json` wrappers catch failures and print “skipped”.

```bash
set -euo pipefail
export CARGO_TARGET_DIR="$PWD/packages/contracts/target"
cargo test --locked --manifest-path packages/contracts/Cargo.toml -p linkora-contracts
cargo build --locked --manifest-path packages/contracts/Cargo.toml \
  -p linkora-contracts --target wasm32v1-none --release
WASM_PATH="$CARGO_TARGET_DIR/wasm32v1-none/release/linkora_contracts.wasm"
test -s "$WASM_PATH"
sha256sum "$WASM_PATH"
```

Keep that exact artifact and SHA-256 with the source commit and lockfile. The Cargo
workspace's normal output directory is `packages/contracts/target`, even when building
from the core crate directory. A checked-in Wasm or a stale crate-local `target` is not
evidence that the selected revision built successfully. If the build fails, stop before
deploying and resolve it; do not fall back to an older artifact unnoticed.

## Testnet deployment

### 1. Select network and create a deployment identity

Use a dedicated CLI configuration directory outside the checkout. Keep its permissions
restricted; generated identities contain signing material. Avoid shell tracing (`set -x`)
and never put secret keys into source control, public build variables, or command history.

```bash
NETWORK=testnet
RPC_URL=https://soroban-testnet.stellar.org
NETWORK_PASSPHRASE='Test SDF Network ; September 2015'
DEPLOY_CONFIG_DIR="$HOME/.config/linkora-deploy/testnet"
DEPLOY_IDENTITY=linkora-testnet
mkdir -p "$DEPLOY_CONFIG_DIR"
chmod 700 "$DEPLOY_CONFIG_DIR"
stellar --config-dir "$DEPLOY_CONFIG_DIR" keys generate "$DEPLOY_IDENTITY"
ADMIN_ADDRESS=$(stellar --config-dir "$DEPLOY_CONFIG_DIR" keys address "$DEPLOY_IDENTITY")
curl --fail --get https://friendbot.stellar.org \
  --data-urlencode "addr=$ADMIN_ADDRESS"
```

For an existing identity, skip generation and derive its address using `keys address`.
Friendbot is for Testnet only. Verify account funding on that network before continuing.

Set `TREASURY_ADDRESS` to your actual public treasury address, then validate the release
inputs. `TREASURY_ADDRESS` intentionally has no example destination that could be copied
into a real deployment.

```bash
: "${TREASURY_ADDRESS:?Set the independently verified public treasury address}"
FEE_BPS=250
[[ "$FEE_BPS" =~ ^[0-9]{1,5}$ ]]
(( 10#$FEE_BPS <= 10000 ))
```

### 2. Deploy, then initialize once

```bash
CONTRACT_ID=$(stellar --config-dir "$DEPLOY_CONFIG_DIR" contract deploy \
  --rpc-url "$RPC_URL" --network-passphrase "$NETWORK_PASSPHRASE" \
  --source-account "$DEPLOY_IDENTITY" --wasm "$WASM_PATH")
test -n "$CONTRACT_ID"
printf 'Contract ID: %s\n' "$CONTRACT_ID"

stellar --config-dir "$DEPLOY_CONFIG_DIR" contract invoke \
  --rpc-url "$RPC_URL" --network-passphrase "$NETWORK_PASSPHRASE" \
  --source-account "$DEPLOY_IDENTITY" --id "$CONTRACT_ID" \
  -- initialize --admin "$ADMIN_ADDRESS" \
  --treasury "$TREASURY_ADDRESS" --fee-bps "$FEE_BPS"
```

Save the deployment ID and both transaction hashes before changing client configuration.
Deployment and initialization are separate transactions: if initialization fails, the
deployed instance still exists. The core contract source has no constructor that
atomically initializes it. Keep the interval short and verify the expected `Admin` and
`Upgrader` roles immediately; do not expose an uninitialized instance to clients.

[`initialize`](../packages/contracts/contracts/linkora-contracts/src/lib.rs) requires
the proposed admin's signature, sets treasury/fee, and grants that admin both roles.
It can succeed only once. A different source and admin require the admin's authorization;
the CLI source account alone is not a substitute for all required signatures.

### Existing helper: behavior and limitations

The helper accepts these variables and `--dry-run`:

| Variable           | Default / requirement                 | Meaning                                                                                                                        |
| ------------------ | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `ADMIN_SECRET`     | Required for a real helper deployment | Deployer/admin secret key. The script substitutes a dummy when absent; that dummy is not usable.                               |
| `TREASURY_ADDRESS` | Required for a real helper deployment | Public fee recipient. The script's dummy fallback is not a valid release input.                                                |
| `FEE_BPS`          | `0`; integer 0–10000                  | Core protocol fee in basis points.                                                                                             |
| `NETWORK`          | `testnet`                             | CLI network alias passed to deploy/invoke. This is not a Mainnet approval or safety check.                                     |
| `CONTRACT_ID`      | Unset                                 | Skip build/deployment and **still invoke initialize** on this instance. It is not an upgrade or a safe repeat-deployment flag. |
| `CARGO_TARGET_DIR` | Cargo workspace default               | Cargo output location; the helper instead checks a fixed crate-local Wasm path. See below.                                     |
| `GITHUB_OUTPUT`    | Unset; supplied by Actions            | File to which the helper appends `contract_id=...`.                                                                            |
| `GITHUB_ENV`       | Unset; supplied by Actions            | File to which the helper appends `CONTRACT_ID=...`.                                                                            |

```bash
NETWORK=testnet FEE_BPS=250 bash scripts/deploy_testnet.sh --dry-run
```

This command does not submit a transaction. It is only a configuration/tool presence
check, not a key/address check, build, authorization check, or RPC simulation. The script
checks for Stellar CLI/Cargo **before** dry-run and exits successfully when either is
missing. It also uses `set -uo pipefail` without `-e`, so later commands and the success
summary can run after earlier failures. Neither a zero exit code, summary, nor Actions
output alone proves a deployment succeeded.

Other current compatibility issues:

- The helper expects
  `packages/contracts/contracts/linkora-contracts/target/wasm32v1-none/release/linkora_contracts.wasm`,
  unlike the workspace path used above. Explicitly setting `CARGO_TARGET_DIR` to that
  crate-local `target` aligns those paths if maintaining the helper.
- Its key import uses `keys add ... --secret-key "$ADMIN_SECRET"`. In current Stellar
  CLI, `--secret-key` selects an interactive/stdin secret input mode rather than accepting
  the secret as a positional argument. Use the manual identity workflow above when this
  syntax is rejected. Do not keep retrying deployment to work around a key-import error.
- The temporary CLI config directory does not inherit project-local custom network
  aliases. The helper's network name must be resolvable within its configuration.

Use the explicit manual workflow for deployment until those helper issues are resolved.
The [Testnet Actions workflow](../.github/workflows/deploy-testnet.yml) uses this helper
and then runs codegen and publishes an SDK canary. It skips deployment without configured
`ADMIN_SECRET` / `TREASURY_ADDRESS` secrets; publication additionally needs `NPM_TOKEN`.
Dispatching that workflow can therefore publish a package as well as deploying a contract.

## Mainnet deployment checklist

Complete these checks before signing transactions:

- [ ] Select the reviewed source commit, locked dependencies, tool versions, exact Wasm
      bytes/hash, and passing direct Cargo tests. Rehearse the release on Testnet.
- [ ] Review source/artifact compatibility, authorization paths, fee calculation, storage
      rent, pause/recovery controls, and a rollback/migration plan with the release owners.
- [ ] Select a Mainnet RPC provider and verify it reports the public-network passphrase.
      Provision rate limits and availability for deployment and backend operation.
- [ ] Check the signing account, treasury, fee, account funding, and intended admin/
      upgrader custody. The current direct `upgrade` entry point checks the `Upgrader`
      role; it does not implement the timelocked ceremony in the inactive
      `upgrade_hardening.rs` file. Do not assume that ceremony protects the release.
- [ ] Arrange a controlled deployment/initialization window, record both transaction
      hashes, and keep applications pointed at the previous release until verification.
- [ ] Prepare matching SDK bindings, application endpoints, indexer start ledger,
      backups, health monitoring, and rollback ownership. Follow the
      [release process](./RELEASE_PROCESS.md) for package publication separately.

Configure a new signing identity and a Mainnet RPC. Do not reuse a Testnet secret or
Friendbot. Obtain the treasury from the approved release inputs.

```bash
NETWORK=mainnet
: "${MAINNET_RPC_URL:?Set the selected HTTPS Mainnet RPC endpoint}"
RPC_URL="$MAINNET_RPC_URL"
NETWORK_PASSPHRASE='Public Global Stellar Network ; September 2015'
DEPLOY_CONFIG_DIR="$HOME/.config/linkora-deploy/mainnet"
DEPLOY_IDENTITY=linkora-mainnet
mkdir -p "$DEPLOY_CONFIG_DIR"
chmod 700 "$DEPLOY_CONFIG_DIR"
stellar --config-dir "$DEPLOY_CONFIG_DIR" keys add "$DEPLOY_IDENTITY" --secret-key
ADMIN_ADDRESS=$(stellar --config-dir "$DEPLOY_CONFIG_DIR" keys address "$DEPLOY_IDENTITY")
: "${TREASURY_ADDRESS:?Set the approved Mainnet treasury address}"
: "${FEE_BPS:?Set the approved fee in basis points}"
[[ "$FEE_BPS" =~ ^[0-9]{1,5}$ ]]
(( 10#$FEE_BPS <= 10000 ))
```

`keys add` prompts for the secret without echoing it; use an existing secured identity or
supported hardware signer instead where required by custody policy. Fund the account
with Mainnet XLM and verify its address. Run the same **deploy then initialize** command
block from the Testnet section using these Mainnet inputs and the approved `WASM_PATH`.
These commands explicitly use `--rpc-url` and `--network-passphrase`; do not mix them
with `--network` or assume a usable default Mainnet RPC endpoint.

## Post-deployment verification

Keep the source identity and matching network variables from the selected environment.
Fetch the deployed executable and compare it with the exact release artifact:

```bash
VERIFY_DIR=$(mktemp -d)
stellar --config-dir "$DEPLOY_CONFIG_DIR" contract fetch \
  --rpc-url "$RPC_URL" --network-passphrase "$NETWORK_PASSPHRASE" \
  --id "$CONTRACT_ID" --out-file "$VERIFY_DIR/deployed.wasm"
sha256sum "$WASM_PATH" "$VERIFY_DIR/deployed.wasm"
cmp "$WASM_PATH" "$VERIFY_DIR/deployed.wasm"
```

Stop rollout on a byte/hash mismatch. Do not infer the deployed hash from an old local
artifact or only from the application's configured contract ID.

Simulate configuration reads without submitting extra transactions (`--send=no` matters
because some apparent getters bump storage TTL):

```bash
for method in get_fee_bps get_treasury is_paused; do
  stellar --config-dir "$DEPLOY_CONFIG_DIR" contract invoke \
    --rpc-url "$RPC_URL" --network-passphrase "$NETWORK_PASSPHRASE" \
    --source-account "$DEPLOY_IDENTITY" --id "$CONTRACT_ID" --send=no \
    -- "$method"
done

for role in Admin Upgrader; do
  stellar --config-dir "$DEPLOY_CONFIG_DIR" contract invoke \
    --rpc-url "$RPC_URL" --network-passphrase "$NETWORK_PASSPHRASE" \
    --source-account "$DEPLOY_IDENTITY" --id "$CONTRACT_ID" --send=no \
    -- has_role --account "$ADMIN_ADDRESS" --role "\"$role\""
done
```

Expect the agreed fee, the exact treasury, `false` for global pause on a fresh instance,
and `true` for each intended role. Confirm successful transaction status and inclusion
ledger using the selected RPC/CLI or network explorer. Record source commit, Wasm hash,
network passphrase, RPC provider, contract ID, admin, treasury, fee, transaction hashes,
and first deployment/initialization ledger in the release record. Simulate an
admin-only operation with an unrelated signer and verify authorization cannot be
satisfied; do not submit a mutation as an authorization smoke test.

### Wire up consumers

| Consumer / variable                                                                             | Value and purpose                                                                                                                              |
| ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Runbook `RPC_URL`, `NETWORK_PASSPHRASE`                                                         | Explicit CLI inputs; these shell variables are not automatically service configuration.                                                        |
| Runbook `DEPLOY_CONFIG_DIR`, `DEPLOY_IDENTITY`, `ADMIN_ADDRESS`, `WASM_PATH`, `MAINNET_RPC_URL` | Local operator inputs; only the derived public address belongs in a public release record.                                                     |
| Core `CONTRACT_ID`                                                                              | Newly verified core instance; also the helper output and backend input.                                                                        |
| Web `NEXT_PUBLIC_CONTRACT_ID`                                                                   | Verified core contract ID, set at build time.                                                                                                  |
| Web `NEXT_PUBLIC_SOROBAN_RPC_URL`, `NEXT_PUBLIC_RPC_URL`                                        | Set both to the same selected RPC: different web call sites currently read different names.                                                    |
| Web `NEXT_PUBLIC_NETWORK_PASSPHRASE`                                                            | Exact Testnet or Mainnet passphrase above.                                                                                                     |
| Web `NEXT_PUBLIC_TOKEN_FACTORY_ID`                                                              | Separately deployed factory, if enabling creator token flows.                                                                                  |
| Web `NEXT_PUBLIC_INDEXER_URL`, `NEXT_PUBLIC_INDEXER_API_URL`                                    | Matching indexer origin; both names are currently read by web call sites.                                                                      |
| Indexer `STELLAR_RPC_URL`, `CONTRACT_ID`, `START_LEDGER`                                        | Matching RPC, core ID, and starting ledger at/before deployment. A cursor from another contract/network is not reusable.                       |
| Oracle `SOROBAN_RPC_URL`, `NETWORK_PASSPHRASE`, `CONTRACT_ID`                                   | Match the selected network; oracle signing secrets use the configured secret-file source.                                                      |
| Mobile `EXPO_PUBLIC_INDEXER_URL`                                                                | HTTPS origin validated by mobile app config. Contract IDs currently live in `NetworkContext.tsx` presets; replace placeholders before release. |
| Backend `DATABASE_URL`, `REDIS_URL`, `NODE_ENV`                                                 | Per-environment service configuration; use separate databases/cursors for separate networks. See backend guide for all required settings.      |

Never place `ADMIN_SECRET`, oracle seeds, or private RPC credentials in a `NEXT_PUBLIC_*`
or `EXPO_PUBLIC_*` variable. Public endpoints must be suitable for use from a client.

Build all contract artifacts explicitly before generating ABI bindings, then review and
validate generated changes before rebuilding applications:

```bash
cargo build --locked --manifest-path packages/contracts/Cargo.toml \
  --workspace --target wasm32v1-none --release
pnpm --filter @linkora/codegen generate
pnpm --filter @linkora/codegen check
pnpm --filter linkora-sdk build
```

Codegen reads local Wasm artifacts in `packages/contracts/target/wasm32v1-none/release`;
setting `CONTRACT_ID` alone does not make it fetch the deployed ABI. Check the core
artifact hash again if the workspace build recreated it. Rebuild web/mobile with the
verified configuration. Confirm indexer ingestion from the recorded ledger, expected
on-chain events and read APIs, and `/health/ready` for each deployed backend. Run a
Testnet create/read flow to exercise the client → contract → indexer path; scope any
Mainnet canary to an explicitly agreed transaction and funding limit.

## Failure, retry, and rollback

| Failure                                               | Recovery                                                                                                                                                                  |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Missing tools, build failure, missing artifact        | Stop before deployment. Install the recorded tools and rebuild; do not treat helper “skipped” output as success.                                                          |
| Insufficient balance / simulation failure             | Inspect the RPC result, signer, resource estimate, and reserve requirements. Correct the cause before signing again.                                                      |
| Submission timeout / unknown status                   | Save the transaction hash and query its status on the same network. Reconcile before retrying; another deployment can create another instance.                            |
| Deployment succeeded, initialization failed           | Retain the contract ID. Check on-chain state/transaction outcome, then retry **only initialization** with correct authorization if the instance is still uninitialized.   |
| “already initialized”                                 | Verify treasury, fee, roles, and transaction history. Do not rerun the helper as an upgrade; reject an instance initialized by an unexpected account.                     |
| Wrong RPC/passphrase, wrong contract ID, stale client | Verify the selected network and byte hash; correct and rebuild consumers before proceeding.                                                                               |
| Indexer missing events                                | Check contract ID, network, start ledger, cursor, and event-parser compatibility; preserve raw events for replay. Do not reset a production cursor without a replay plan. |

Before rollout, retain the previous verified artifact, deployment configuration, client
builds, service configuration, and database backups. An application rollback can restore
those builds/settings, but **does not reverse finalized on-chain transactions**. Repointing
clients to an older contract ID also does not migrate state or assets from the new one.

The active core `upgrade(upgrader, new_wasm_hash)` is role-gated and changes the code of
the existing instance. A code rollback requires uploading the previous approved Wasm and
a separately authorized upgrade with that hash; it is safe only if the current storage
is compatible. If storage changed, use a tested migration/forward fix instead. Pausing
is not a universal upgrade workaround: `upgrade` itself checks the Governance pause.
Verify that a recovery plan can execute under the actual pause/role state.

After any rollback or forward fix, repeat executable-hash, role, fee/treasury, client,
indexer, and service readiness checks. Keep the failed release and its transaction
records for diagnosis; do not erase evidence or describe redeployment as undoing it.

## CLI references

Check the installed CLI's `--help` against the version recorded for the release. The
official implementation documents [identity import](https://github.com/stellar/stellar-cli/blob/main/cmd/soroban-cli/src/commands/keys/add.rs),
[contract invocation and `--send`](https://github.com/stellar/stellar-cli/blob/main/cmd/soroban-cli/src/commands/contract/invoke.rs),
and [contract fetch](https://github.com/stellar/stellar-cli/blob/main/cmd/soroban-cli/src/commands/contract/fetch.rs).
