# Glossary

- **Signing key:** the BLS key a validator uses for attestations, proposals and sync committee messages; must be online.
- **Withdrawal credentials:** 32-byte field deciding where stake goes on exit (`0x00` BLS key, `0x01` execution address, `0x02` compounding).
- **Slashing:** protocol penalty and forced exit for provably contradictory messages (double proposal, double vote, surround vote).
- **Surround vote:** an attestation whose source-target span contains, or is contained by, a previous attestation from the same validator.
- **Slashing protection database:** local record of signed messages checked before every new signature.
- **EIP-3076:** JSON interchange format for moving slashing protection history between clients.
- **Remote signer:** separate service (e.g. Web3Signer, Dirk) that holds keys and signs on request from validator clients.
- **Doppelganger protection:** start-up check where a client watches the network for its own keys before signing.
- **DVT (distributed validator technology):** threshold-signing setup where several nodes jointly operate one validator.
- **DKG (distributed key generation):** ceremony producing key shares without the full key ever existing in one place.
- **MEV-Boost relay:** intermediary that passes builder blocks to proposers and releases the payload after the proposer signs.
- **Client diversity:** spreading stake across independent client implementations to limit the impact of a single bug.
