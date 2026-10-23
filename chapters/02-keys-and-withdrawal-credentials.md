# Validator Keys, Withdrawal Credentials and Key Storage

Every control in validator operations depends on knowing exactly which key does what, where it lives and who can use it. This chapter separates the hot signing key from the cold withdrawal authority, explains how they are derived, and describes storage options from a plain keystore file to hardware-backed signers.

## Key hierarchy

Ethereum validator keys are BLS12-381 keys derived from a single BIP-39 mnemonic using EIP-2333 (tree derivation) and the paths defined in EIP-2334. For validator index `i` in the derivation, the withdrawal key sits at `m/12381/3600/i/0` and the signing key at `m/12381/3600/i/0/0`. Signing keys are usually exported as EIP-2335 keystores: JSON files that encrypt the secret with a password using scrypt or PBKDF2.

| Item | Derived from | Used for | Where it should live |
|---|---|---|---|
| Mnemonic | Generated offline | Re-deriving all keys | Offline, split or sealed backup |
| Signing key | Mnemonic, path `.../0/0` | Attestations, proposals, sync committee | Online signer only |
| BLS withdrawal key | Mnemonic, path `.../0` | One-time change from `0x00` to `0x01` | Offline |
| Execution withdrawal address | Separate wallet | Receiving withdrawals; EL-triggered exits (EIP-7002) | Multisig or hardware wallet |

## Withdrawal credentials

The 32-byte withdrawal credential committed in the deposit decides who controls the stake on exit:

- **`0x00`**: points at the hash of a BLS withdrawal public key. Since the Capella upgrade (2023) these can be converted once to `0x01` by signing a `BLSToExecutionChange` message with the BLS withdrawal key.
- **`0x01`**: points at an execution-layer address. Withdrawals are swept there automatically.
- **`0x02`**: introduced with Pectra (EIP-7251), also points at an execution address but allows the effective balance to compound above 32 ETH.

Two consequences matter for security. First, the `0x00` to `0x01` change is first-come, first-served and irreversible. If an attacker learns your mnemonic before you convert, they can point withdrawals at their own address, and you cannot undo it. Operators with legacy `0x00` validators should convert early from an air-gapped machine. Second, with EIP-7002 an execution withdrawal address can trigger a validator exit through a system contract. This makes the withdrawal address more powerful than before: compromise it and an attacker can both exit your validators and receive the funds. Treat it as a treasury key and use a multisig.

## Generating keys safely

Generate keys with a maintained, audited tool on a machine that has never been and will never be online. Widely used options include the `ethstaker-deposit-cli` (the community continuation of the original `staking-deposit-cli`), Wagyu Key Gen (a graphical wrapper) and `ethdo` from Attestant for more advanced workflows. Verify downloads with published checksums or signatures before use.

```bash
# On an offline machine, after verifying the release checksum
./deposit new-mnemonic \
  --num_validators 4 \
  --chain hoodi \
  --execution_address 0xYourSafeAddressHere
# Produces validator_keys/keystore-m_12381_3600_<i>_0_0-*.json
# and deposit_data-*.json; check flag names against your version's --help
```

Setting `--execution_address` at generation time gives `0x01` credentials from the start, avoiding the `0x00` conversion race entirely.

## Storage options for the signing key

The signing key must be usable around the clock, so the question is how to minimise exposure while staying online.

1. **Keystore files on the validator client host.** Simple, common, and the key is decrypted in the client's memory. Protect the password file with file permissions and keep the host minimal.
2. **Remote signer.** Keys live in a separate service such as Web3Signer or Dirk; the validator client asks it to sign. This isolates keys from the internet-facing beacon stack and centralises slashing protection (Chapter 3).
3. **Remote signer backed by a vault or HSM.** Web3Signer can load keys from HashiCorp Vault, AWS Secrets Manager, Azure Key Vault and similar stores. Note that most cloud HSMs do not natively support BLS12-381, so in many designs the vault stores the key and the signer still decrypts it in memory. Read the documentation for what the backend actually protects.
4. **Threshold or distributed signing.** The key is split across machines or operators so no one host holds it (Chapter 5).

None of these prevents slashing on its own. A perfectly isolated key in an HSM will still be slashed if two validator clients both send it conflicting messages and nothing checks history.

## Worked example

*Execution status: the commands, code and numbers in this example were not executed in this book's CI; they are illustrative. Verify locally on a devnet, testnet or fork before relying on them.*

Inspect key derivation and credentials on a local devnet with no real funds.

1. Generate a throwaway mnemonic and four keys with `--chain` set to a testnet and `--execution_address` set to a test address you control.
2. Open one keystore JSON. Confirm the `path` field reads `m/12381/3600/0/0/0`, note the `pubkey`, and check that `crypto.kdf.function` is `scrypt` or `pbkdf2`.
3. Open `deposit_data-*.json` and find `withdrawal_credentials`. It should start with `01`, followed by 11 zero bytes, then your 20-byte address.
4. Start a Kurtosis devnet (`kurtosis run github.com/ethpandaops/ethereum-package`) and query a validator's credentials via the beacon API: `curl -s localhost:<bn-port>/eth/v1/beacon/states/head/validators/0 | jq .data.validator.withdrawal_credentials`.
5. If your devnet keys use `0x00` credentials, generate a `BLSToExecutionChange` with your tool of choice (for example `ethdo validator credentials set`) and submit it on the devnet. Watch the credential flip after inclusion, then try to submit a second change to a different address and observe that it is rejected.

## Common pitfalls

- Generating keys on an internet-connected laptop "just this once".
- Leaving validators on `0x00` credentials while the mnemonic sits in a password manager synced to the cloud.
- Using a single externally owned account as the `0x01` address now that it can also trigger exits.
- Storing keystore passwords in the same repository or secret as the keystores.
- Assuming an HSM or vault backend also provides slashing protection.

## Exercises

1. Derive the same signing key twice from one test mnemonic with two different tools and confirm the public keys match.
2. Write a short script that parses `deposit_data` files and flags any validator whose credentials are not `0x01` or `0x02` or that point at an unexpected address.
3. Configure Web3Signer on a devnet to load one keystore from a local HashiCorp Vault dev server and sign attestations for it.

## Further reading

- EIP-2333, EIP-2334, EIP-2335 (BLS key derivation, paths and keystores)
- EIP-7002 (execution layer triggerable exits) and EIP-7251 (increase max effective balance)
- ethstaker-deposit-cli: <https://github.com/eth-educators/ethstaker-deposit-cli>
- ethdo: <https://github.com/wealdtech/ethdo>
- Related books: **mpc-wallet-security**, **dprk-crypto-threat-operations**
