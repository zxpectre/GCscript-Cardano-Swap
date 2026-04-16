# GCScript Issues to Report to Developer

## Pre-computed Values Required (injected via `args`)

### 1. Beacon asset names — `pairBeacon`, `offerBeacon`, `askBeacon`

Computed externally as SHA-256 hashes of specific byte sequences defined by the cardano-swaps protocol:

- `offerBeacon` = `sha256(0x01)`
- `askBeacon` = `sha256(0x02 || ask_policy_id_bytes || ask_asset_name_bytes)`
- `pairBeacon` = `sha256(0x00 || offer_policy_id_bytes || offer_asset_name_bytes || ask_policy_id_bytes || ask_asset_name_bytes)`

**Root cause**: ISL's `sha256()` only accepts UTF-8 strings, not raw byte arrays. The ADA policy ID in particular encodes as the zero byte `0x00` (not the string `"ada"`), which cannot be expressed as a UTF-8 string argument.

### 2. Datum CBOR — `datumHex`

The full `SwapDatum` encoded as CBOR hex, pre-computed offline.

**Root cause**: The `plutusData.fromJSON` step cannot accept integer values sourced from `args` macros. `args` values are always strings, so `{"int": "{get('args.priceNumerator')}"}` expands to `{"int": "1"}` — a string where an integer is required — causing the step to hang. There is no ISL function to cast a string to an integer.

---

## Encoding Problems

### 3. Script hash discrepancy — `scriptHashHex` vs on-chain policy ID

This is the most significant issue, worth filing explicitly.

**The problem**: GCScript's `plutusScript` step computes `scriptHashHex` differently from how the Cardano node (and Aiken) computes the policy ID from the same CBOR bytes.

| | Formula | Input | Result |
|---|---|---|---|
| Cardano node / Aiken | `blake2b_224(0x02 \|\| cbor_bytestring_header \|\| flat_bytes)` | `5911ed<flat>` | `c4d7d117...` |
| GCScript internal | `blake2b_224(0x02 \|\| flat_bytes)` | strips `5911ed`, hashes `flat` | `705e07...` |

When `mints[].policyId` was hardcoded to Aiken's hash `c4d7d117...`, GCScript threw `missing_script_source` at build time because its internal script registry stored the script under `705e07...`.

**The workaround**: Double-wrap the `scriptHex`. Instead of providing `5911ed<flat>` (single CBOR), provide `5911f05911ed<flat>` (outer CBOR wrapping the single-CBOR). GCScript then strips the outer wrapper, hashes `5911ed<flat>`, and arrives at `c4d7d117...` — matching both `mints[].policyId` and the on-chain policy ID the node derives.

**Questions for the developer**:
- Is double-wrapping the intended `scriptHex` format for `plutusScript`? Is there a documented format specification?
- Is `scriptHashHex` intended to be usable as a `policyId` in `mints`? If so, why does it not match the Cardano-standard hash when single-CBOR scripts are provided?

### 4. `args` inaccessible inside nested `script` blocks

`get('args.x')` and `get('cache.args.x')` both fail inside a nested `script` block's `run`. Only top-level `run` steps can access `args` directly. Steps that need `args` values must be placed at the top level, not inside nested scripts.

**Questions for the developer**:
- Is this by design? Is there a supported way to access `args` from within a nested `script` block?
