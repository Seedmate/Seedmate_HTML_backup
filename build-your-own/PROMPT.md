I want to build my own offline tool to recover backups made with a Seedmate device,
so I do not have to trust anybody else's software. Please write it for me.

Requirements:
- One single HTML file with plain HTML, CSS and JavaScript. No libraries, no fonts,
  images or scripts from the internet, no network requests of any kind. Put this tag in
  the head: <meta http-equiv="Content-Security-Policy" content="default-src 'none';
  script-src 'unsafe-inline'; style-src 'unsafe-inline'">
- The functions I need: [WRITE HERE WHAT YOU NEED, for example: merge Seedmate Shamir
  shares; undo an obfuscation (all five methods); turn a MNEMONIC TO CARDS order back
  into words; XOR of seeds].
- Follow the specification below exactly. If something is not in it, ask me; do not
  guess.
- The BIP39 English word list goes inside the file. Check its SHA-256 when the page
  opens (value in the specification) and refuse to work if it differs.
- SHA-256: use crypto.subtle.digest or write it in the file; test it on the vectors.
- Add a "Self test" button that runs every test vector below through the same functions
  the tool uses for real input, and shows PASS or FAIL for each one and the totals. The
  tool is finished only when every vector passes.
- Show errors in plain words (wrong word, wrong checksum, two shares with the same ID,
  inputs of different sizes). Warn that wrong shares give another valid seed, not an
  error.
- Never ask me for my real words or shares, and never put real words in the code. The
  vectors are public test seeds with no funds.

Explain briefly how each function works, so I can check the code against the
specification.

The specification and the test vectors follow.
