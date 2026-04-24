# Competency Test - Watch-Only Multisig

## How the single-sig watch-only import works right now

I traced through the import flow starting from `MultiFormat::try_from_string` in `rust/src/multi_format.rs` - that's the entry point when a user scans a QR code or pastes/imports a text payload. It figures out what format the input is in and wraps hardware wallet exports as `HardwareExport`.

From there it goes to `Wallet::new_from_export` in `rust/src/wallet.rs`, which calls into `try_new_persisted_from_pubport`. This is the key function - it pulls descriptors out of the `pubport::Format`, then grabs the fingerprint and xpub from those descriptors (via `.fingerprint()` and `.xpub()`). It converts everything into Cove's own `Descriptors` type, hands them to BDK to create a wallet (`into_create_params().create_wallet_no_persist()`), and persists the metadata.

The problem for multisig is that this whole path assumes a single xpub and a single fingerprint. Multisig descriptors have multiple xpubs (one per cosigner), so `.fingerprint()` and `.xpub()` don't really make sense - and the code currently returns `MultisigNotSupported` if you try.

## How BDK descriptors map to Cove's wallet model

After reading the BDK book sections on descriptors, the split is basically:

- **BDK** handles the descriptor parsing, script derivation, and address generation - all the bitcoin-level stuff
- **Cove** handles everything around that: parsing the import format, validating it, classifying what kind of wallet it is, persisting it, deduplication, and error messages for the UI

The bridge between them is `rust/src/keys.rs` - Cove takes the parsed `pubport::descriptor::Descriptors` and converts them into its own `Descriptors` type (via a `From` impl), which then wraps BDK's `ExtendedDescriptor`. The nice thing is BDK already knows how to handle multisig descriptors like `wsh(sortedmulti(...))` at the wallet level. The gap is just in Cove's import/metadata layer.

## Rust test for multisig descriptor parsing

I added a test in [`rust/src/keys.rs#L342`](rust/src/keys.rs#L342) - `test_multisig_descriptor_address_derivation` - that takes a 2-of-2 `wsh(sortedmulti(...))` descriptor, parses it through pubport into BDK, creates a wallet, and derives the first few receive addresses. Mainly wanted to verify that the pubport → Cove `Descriptors` → BDK wallet pipeline works end-to-end for multisig, and that address derivation is deterministic.

To run it:

```bash
cd rust
cargo test -p cove test_multisig_descriptor_address_derivation -- --nocapture
```

Passes with `test result: ok. 1 passed`.
