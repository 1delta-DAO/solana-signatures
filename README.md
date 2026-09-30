# Solana Ed25519 and Secp256k1 signature verification

On-chain Ed25519 and Secp256k1 signature verification using instruction introspection.

Built for checking Solana and Ethereum signatures, with examples (see tests).

### Why and how

Solana does not have a way to implement [Ed25519](https://ed25519.cr.yp.to/) or [Secp256k1](https://github.com/gavofyork/ethereum/blob/master/secp256k1/secp256k1.c) sig verification on-chain on custom programs. That's why the [native Ed25519Program](https://docs.solana.com/developing/runtime-facilities/programs#ed25519-program) and [native Secp256k1Program](https://docs.solana.com/es/developing/runtime-facilities/programs#secp256k1-program) exist, which have a set of instructions that can, amongst other things, verify signatures for those curves.

Therefore, the way to build custom instructions that "do" sig verification is by actually sending a transaction made of (at least) two instructions, and checking that the native program instruction was sent.

In doing so, these are the possible outcomes:

- ❌ Native program instruction fails -> Custom instruction is never executed.
- ❌ Native program instruction not supplied or supplied with wrong values -> Custom instruction fails to check that the Native program instruction was sent with the proper data, therefore gets rejected.
- ✅ Native program instruction succeeds -> Custom instruction gets executed -> Custom instruction checks that the Native program instruction was sent with the proper data -> If that succeeds, we can say that Custom instruction indirectly verified the signature.

### Instruction introspection

`anchor_lang::solana_program::sysvar::instructions::load_instruction_at_checked` reads an instruction from the `Instructions Sysvar`, exposing its `program_id`, `accounts`, and `data`.
In order for us to check that that instruction was constructed properly, we need to inspect the data byte array manually.

### Building and testing

Install Anchor CLI 0.32.1, Solana CLI 2.3.x, and Yarn 1.x. The on-chain program uses Anchor's modular Solana APIs rather than the old `solana-program` dependency. The lockfile pins transitive crates compatible with Solana 2.3's SBF Rust toolchain.

The Ed25519 and Secp256k1 integration tests use only a disposable wallet and a local validator. Build the program and preload it at the declared program ID; the repository does **not** include the matching deployment keypair, so a normal `anchor test` deployment cannot preserve that ID.

```bash
yarn install --frozen-lockfile
anchor build
TEST_DIR="$(mktemp -d)"
solana-keygen new --no-bip39-passphrase --silent -o "$TEST_DIR/wallet.json"
solana-test-validator --ledger "$TEST_DIR/ledger" \
  --bpf-program DHxesXA69rUmz5AJ1CnLCQezUzQR5j7KKTwTp1zZPc9j target/deploy/signatures.so \
  --rpc-port 8899 --quiet &
VALIDATOR_PID=$!
trap 'kill "$VALIDATOR_PID"' EXIT
until solana --url http://127.0.0.1:8899 cluster-version >/dev/null 2>&1; do sleep 1; done
ANCHOR_WALLET="$TEST_DIR/wallet.json" ANCHOR_PROVIDER_URL=http://127.0.0.1:8899 \
  yarn ts-mocha -p ./tsconfig.json -t 100000 tests/*.test.ts
```

Do not use production keys or submit these test transactions to a public network. Deploying at the existing program ID outside the local validator requires the original matching deployment keypair.
