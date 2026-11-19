# Seedmate backup formats: specification

Version 1. This is everything a program needs to read and write the backups that a
Seedmate device makes. The test vectors (seedmate_test_vectors.json) were computed with
the firmware's own code; a correct implementation reproduces every one of them.

## 1. Common: BIP39

- Entropy E is 128 bits (12 words) or 256 bits (24 words), as bytes.
- Bits are numbered from 0, starting with the most significant bit of the first byte.
- Checksum: the first L/32 bits of SHA-256(E) (4 bits for 12 words, 8 for 24).
- Words: E followed by the checksum, cut into groups of 11 bits, each group read
  most significant bit first, is an index 0..2047 into the BIP39 English word list.
- The word list is the official one (bitcoin/bips, bip-0039/english.txt): 2048 words,
  one per line, every line ending in a newline. SHA-256 of that file:
  2f5eed53a4727b4bf8880d8f3f199efc90e58503646d9ff8eff3a2ed3b24dbda. Check this hash in
  the program: a single wrong word gives wrong seeds without any error.
- To read words back: each word gives its index, the indexes give E and the checksum;
  refuse the words when the checksum is not the one computed from E.
- Every operation below works on E and then computes a new checksum for the result.
  The checksum of an input is checked, never reused.

## 2. Shamir secret sharing (Seedmate format, not SLIP-39)

- Each share is a normal BIP39 mnemonic (with a valid checksum) plus a share ID x,
  written next to it. IDs are 1..9; k is 2..6; n is k..9.
- Field: GF(2^8) with the reduction polynomial x^8 + x^4 + x^3 + x + 1 (0x11B, the
  AES field). Addition is XOR. 3 is a generator.
- Split, for every byte position i of E separately:
  f(x) = E[i] + c1[i]*x + c2[i]*x^2 + ... + c(k-1)[i]*x^(k-1), with k-1 random
  coefficient arrays c1..c(k-1) as long as E. Share x is the bytes f(x) for x = 1..n.
- Merge: with k shares (x_j, y_j), all x_j different, for every byte position:
  E[i] = sum over j of y_j[i] * product over m != j of x_m / (x_j + x_m)
  (Lagrange interpolation at 0; + is XOR, / is division in the field).
- A share of another split, a wrong ID or fewer shares than k give another valid seed,
  not an error. A tool should say so.

## 3. Obfuscation

Applied to E; the result gets a new checksum. L is the number of bits of E.

- Circular shift left by n bits: bit i of the result is bit (i + n) mod L of E.
  Right by n is left by L - (n mod L). Undo: shift the other way by the same n.
- NOT: every bit inverted. Undoes itself.
- Words add / subtract v: for each whole group of 11 bits from bit 0 (11 groups for
  128 bits, 23 for 256), group = (group + v) mod 2048, or (group - v) mod 2048. The bits
  after the last whole group (7 of 128, 3 of 256) do not change. Undo: the opposite
  operation with the same v.
- 1 word XOR with the word of index w: bit i of E is XORed with bit (10 - (i mod 11))
  of w, counting bit 0 as the least significant bit of w (so the 11 bits of w, most
  significant first, repeat from bit 0 of E). Undoes itself.
- Byte swap: bytes 0 and 1 are exchanged, 2 and 3, and so on. Undoes itself.

## 4. XOR of seeds (SEED WORD XOR, MIX ENTROPY)

E = E1 XOR E2 (XOR E3, XOR E4), byte by byte. All inputs have the same size.

## 5. Cards

- A card is c = suit * 13 + rank. Ranks A 2 3 4 5 6 7 8 9 T J Q K are 0..12 (T is
  ten); suits C D H S (clubs, diamonds, hearts, spades) are 0..3. Written rank then
  suit, e.g. AC = 0, TD = 22, KS = 51.
- Bits of a card: c < 32 gives the 5 bits of c; 32 <= c < 48 gives the 4 bits of c - 32;
  c >= 48 gives the 2 bits of c - 48 (most significant bit first).
- MNEMONIC TO CARDS, reading an order back: the bits of every card in order, joined,
  are E; they must add up to exactly 128 or 256 bits.
- MNEMONIC TO CARDS, making the order (needed only to get the same order as the
  device, not to recover): a beam search that keeps at most 8 paths. A path has a bit
  offset, a count of how often each card is used, the card sequence, the highest count
  (the decks needed) and a score = highest count * 256 + sequence length.
  Start with one empty path. Each round builds a new list from the paths of the list,
  in their order: a path at offset L is inserted again unchanged; otherwise, for len in
  5, 4, 2 (this order), when rem = L - offset is at least len and rem - len is neither
  1 nor 3, the next len bits give the card (len 5: their value; 4: 32 + value;
  2: 48 + value) and the path with that card added is inserted.
  Insert: the new list is sorted by score, lowest first, and holds at most 8 paths. A
  path is inserted when the list has fewer than 8 or its score is lower than the last
  one's; it goes after every path with a score lower than or equal to its own; when the
  list had 8, the last one is dropped.
  Rounds go on while any path of the list is not at offset L; the result is then the
  first path of the list.
- POKER CARDS entropy (CREATE): the bits of each card drawn, in order, until there are
  at least L bits; E is the first L bits (a last card may give bits more, not used).
  The device takes no card after L is reached.

## 6. Dice

- Raw dice (CREATE > DICE): each face gives bits: 1 = 01, 2 = 10, 3 = 11, 4 = 0, 5 = 1,
  6 = 00. Rolls are added until there are at least L bits; E is the first L bits.
- Dice hash (CREATE > DICE > STRING HASH): E is the first 16 (12 words) or 32 (24 words)
  bytes of SHA-256 of the faces written as ASCII digits, nothing else (no spaces).
  In the 0-5 mode (as Keystone) every 6 is written as 0 first. At least 50 rolls for
  12 words, 99 for 24, at most 500.

## 7. Coins and words (CREATE)

- Coins: heads is 0, tails is 1; exactly 128 or 256 tosses are the bits of E.
- Words: 12 or 24 words chosen by hand; E is the first 128 or 256 bits of their
  indexes, and the checksum is computed again (the last word may change).
