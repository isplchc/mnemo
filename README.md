# Mnemo reference client

Trustless, forget-tolerant recovery of a secret from eight assigned words: the reference
implementation of **Mnemo** from *Keys You Can Afford to Forget*. No server, account, enclave, or
committee is involved; the only stored value is a public record that can be kept anywhere.

    bash run_all.sh

That one command creates `.venv/` here, installs two pinned packages, runs the unit and known-answer
tests, runs a demo enroll/recover, and times the reference KDF on this machine. The report is
`results/selftest.txt`. Nothing is written outside this directory.

## Use

    .venv/bin/python -m mnemo enroll  --data secret.bin --record record.json
    .venv/bin/python -m mnemo recover --record record.json --out secret.bin
    .venv/bin/python -m mnemo calibrate

- `enroll` prints the eight words, runs the day-scale key derivation once (the same work recovery
  needs), and writes the public record. Rehearse the words while it runs.
- `recover` asks for the words one at a time (`?` for a forgotten word; four letters suffice; a
  one-letter typo gets a suggestion), decodes them at no cost, shows the expected run time on this
  device, then runs the derivation once and decrypts. `--dry-run` stops after decoding.
- Both commands checkpoint and resume: rerun the same command after an interruption.
- `--profile demo` runs in seconds and has **no security**; it exists for trying the client.

## What it implements (paper, Algorithm 2)

Mnemo = Comp[(E, D), F, AEAD] with:

| part | choice |
|---|---|
| secret | k = 4 uniform symbols of GF(2^10) = 40 bits |
| recall code (E, D) | systematic RS[8,4] over GF(2^10) = GF(2)[x]/(x^10+x^3+1), evaluation points alpha^0..alpha^7; errors-and-erasures decoding (Berlekamp-Welch on the unerased positions), succeeds whenever 2쨌wrong + forgotten <= 4 |
| codebook | `mnemo/codebook_v1.txt`, 1024 words, symbol = line index (SHA-256 `9893db26?쫋83a`, bound into every record) |
| costly function F | s_0 = m; s_i = Argon2id(s_{i-1} ??u32(i), salt; 1 GiB, t = 4, p = 4, 32 B), i = 1..L; key = SHA-256("mnemo-v1/key" ??u32(len x) ??x ??s_L) with x = m ??salt (final-call form) |
| chain length | L = 137,142: 24 h on the reference desktop (Intel Core Ultra 7 270K Plus, 0.63 s/eval) |
| AEAD | AES-256-GCM, 96-bit random nonce; associated data = canonical JSON of the public parameters and salt |
| record | JSON: format, version, params (n, k, field, points, codebook hash, KDF), salt, nonce, ct |

L is a public parameter fixed at enrollment, so a slower device only waits longer (about 28 h on an
Intel Core i7-12700K, about 29 h on a MacBook Air M1); the attacker's cost depends on L alone. `calibrate` reports the time on the
current device.

## Codebook v1

Built by `tools/build_codebook_v1.py` (run inside the paper's `c_experiments/` folder, which holds
the SWOW and spaCy data). Candidates are the EFF large diceware list (vetted by the EFF for
memorability and offensive words), intersected with the frequency-ordered SWOW pool; a word is
accepted only if it has no SWOW association of strength >= 0.15 with an accepted word, a new
4-letter prefix, Levenshtein distance >= 2 to every accepted word, and is not an inflection or
plural of one. Measured on SWOW R1, a human substitution lands on another codeword 13.8% of the
time, against 52.5% for BIP39 (the error the code cannot detect). This list has not been tested
with users; a deployment should vet it, and may replace it (the record binds the codebook hash).

## Tests

- GF(2^10): primitivity, field axioms, polynomial division and interpolation.
- RS[8,4]: every one of the 423 erasure/error position patterns with 2e + s <= 4 decodes correctly;
  past the radius the decoder returns nothing or a codeword it can justify (never an arbitrary one);
  RS[6,4], [10,5], [12,4], [16,4] randomized.
- KDF: resuming from any checkpoint gives the same key; inputs are separated.
- End to end: exact recall, 1 forgotten + 1 wrong, 4 forgotten, 2 wrong, 3 wrong (fails safely),
  tampered ciphertext or parameters rejected.
- Known-answer vectors (`tests/vectors.json`): the same codewords, keys, and records on every
  platform.
- CLI: enroll then recover with a forgotten word and a prefix; resume from a checkpoint.

## Security notes and limits

- The record is public by design; keep several copies. Losing every copy loses the data.
- A checkpoint holds the decoded secret and chain state. It is written with mode 0600 and deleted
  at the end of the run; do not sync the `state/` folder.
- Words are printed once and never stored by the client.
- Not implemented: position-disjoint sub-codebooks and the spaced-rehearsal scheduler, both of
  which the paper treats as deployment options rather than parts of the scheme.

## License, dependencies, and attribution

The code is released under the MIT License (`LICENSE`). 
argon2-cffi 23.1.0 (reference Argon2), cryptography 43.0.3 (AES-GCM). The codebook draws on the
EFF large wordlist (Electronic Frontier Foundation, CC BY 3.0 US) and the Small World of Words
English association norms (De Deyne et al., 2019).
