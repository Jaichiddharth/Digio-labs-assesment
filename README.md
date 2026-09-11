# Digio-labs-assesment

# NOTES

## What I built
A rule-based, initial-aware name matching module for Indian identity-document
names. Stdlib-only so it can run in a small service or batch job without
external dependencies. It exposes `score_names`, `decide`, and `explain`,
plus an evaluation CLI and a small regression test suite.

## Normalization
- Unicode NFKD + diacritic stripping.
- Lowercasing.
- Removing relation markers: S/o, D/o, W/o, son of, daughter of, wife of.
- Removing honorifics: Mr, Mrs, Ms, Dr, Shri, Smt, etc.
- Punctuation to spaces, whitespace collapse.
- Small alias map for Md / Mohd / Mohammad / Mohammed.

## Matching logic
Features:
1. Exact token F1 (multiset overlap).
2. Initial-aware fuzzy token F1.
3. Sequence ratio (order-sensitive).
4. Sorted sequence ratio (order-insensitive).

score = 0.30 * exact_f1
      + 0.30 * fuzzy_f1
      + 0.20 * sequence_ratio
      + 0.20 * sorted_sequence_ratio

Extra rules:
- Reordered exact token sets boosted to >= 0.92.
- Initial expansion boosted only when a full token also matches.
- All-initial comparisons penalized.
- Lone leading initial ("R Kumar") capped below MATCH, because it is
  ambiguous ("Rahul" / "Rajesh" / "Rakesh" / "Ravi" all collide).

## Thresholds
score >= 0.72         -> MATCH
0.58 <= score < 0.72  -> REVIEW
score < 0.58          -> NO_MATCH

MATCH is conservative because false positives are expensive in identity
verification. REVIEW routes ambiguous cases to manual review, which is the
correct place to resolve them.

## Evaluation
python -m name_matcher.evaluate data/sample_pairs.csv

Reports TP/FP/TN/FN, precision, recall, F1, accuracy, and a per-pair table.
The included sample_pairs.csv is a regression suite of 212 labelled pairs
that exercises exact repeats, trailing-initial expansions, reordered
tokens, honorifics, relation markers, Md/Mohd aliases, typos, missing
middle names, and hard negatives. It is a smoke test, not a claim of
production accuracy. In real deployment, replace it with labeled pairs
from Digio's historical verification data and tune thresholds against the
business cost of FP vs FN.

## Tradeoffs
- Common Indian middle/surnames (Kumar, Devi, Singh) are kept, because
  they are sometimes real identity tokens and removing them causes more
  harm than it prevents.
- Initials are useful but ambiguous:
  * `Rahul Kumar` vs `Rahul K`  -> MATCH (trailing initial, first name is a full anchor)
  * `R Kumar`     vs `Rajesh Kumar` -> REVIEW (leading initial, first name unknown)
  * `K S Rao`     vs `Krishna Srinivas Rao` -> MATCH (multi-initial pattern)
- Handles reordering, titles, relation markers, Md aliases, small typos.
- Does not do OCR, transliteration, or image processing.

## Limitations and how to address them

### 1. Regional scripts are not handled
The current module assumes names have already been romanized to Latin
script. This is a deliberate scope cut driven by the environment this
code was developed in: no GPU, no Indic NLP toolkit installed, and no
labelled parallel corpus available to validate transliteration quality.
Shipping a half-trained transliterator would be worse than not shipping
one, because it would silently corrupt matches and produce false
positives that are hard to debug.

**If the task were intended to be implemented correctly, the approach
would be:**

1. **Detect script at input time.**
   Use `unicodedata` block ranges to classify each string as Latin,
   Devanagari (`U+0900–U+097F`), Bengali, Tamil, Telugu, Gurmukhi,
   Gujarati, Kannada, Malayalam, Oriya, or Arabic-script (for Urdu
   names). Route Latin to the existing pipeline and non-Latin to a
   transliteration pre-processor.

2. **Transliterate with a library, not by hand.**
   Use `indic-transliteration` (Python, MIT license) which provides
   `sanscript` and per-script romanization schemes (IAST, ITRANS,
   Harvard-Kyoto, ISO 15919). For Urdu/Arabic-script names, use
   `pyarabic` or a small mapping table.

   
   from indic_transliteration import sanscript
   from indic_transliteration.sanscript import transliterate
   roman = transliterate(name, sanscript.DEVANAGARI, sanscript.ITRANS)

3. **Normalize after transliteration, not before.**
Transliteration introduces its own inconsistencies (sh vs ṣ,
ee vs i, v vs w). The existing normalize_tokens should run
after transliteration, so the alias/normalization rules apply to
romanized tokens only.

4. **Handle the many-to-one mapping.**
Indian romanization is not bijective: Sharma / Sarma,
Krishna / Krishnaa / Krsna, Choudhary / Chaudhary /
Chowdhary all refer to the same name. Add a canonicalization map
on the romanized form. This is where a nickname/alias dictionary
(see below) does double duty.

5. **Validate with a parallel corpus.**
Assemble a small labelled set of (Devanagari name, Latin name) pairs
drawn from real Digio data. Measure per-script accuracy separately,
because transliteration error rates differ across scripts. Treat
transliteration as a separate model with its own precision/recall,
not as a preprocessing detail.

6. **Fallback for unmappable characters.**
If transliteration produces empty or garbage tokens (e.g. rare
conjuncts), fall back to the raw string comparison and force the
decision to REVIEW rather than NO_MATCH.

Cost estimate if compute were available: roughly 1–2 days of
engineering work, plus a labelled corpus for validation. The
indic-transliteration library is pure Python and CPU-only, so it is
actually cheap to run — the constraint here was validation data, not
compute. The stated "lack of computational power" is really about the
absence of a validated transliteration model in this environment.

### 2. Nickname map is minimal and can be extended
The current ALIASES map only canonicalizes Mohammed variants. This is
the most common and highest-impact case in Indian documents, so it was
prioritized, but a real system needs a broader nickname map.

Nickname map can be extended in three tiers:

**Tier 1 — Hard-coded canonical map (cheap, deterministic).**
Add a static dictionary keyed by romanized token, with a canonical
value. Examples:


NICKNAMES = {
    # Mohammed cluster (already present)
    "md": "mohammed", "mohd": "mohammed", "mohammad": "mohammed",
    # North Indian short forms
    "raju": "rajesh", "raj": "rajesh", "raja": "rajesh",
    "bunty": "vikram", "vicky": "vikram",
    "pinky": "priya", "priya": "priya",
    "bablu": "rahul", "guddu": "naveen",
    "sonu": "suresh", "monu": "manoj",
    "chotu": "chandra", "bittu": "vikas",
    # Regional variants
    "krishna": "krishna", "krishn": "krishna", "krsna": "krishna",
    "ram": "rama", "raama": "rama",
    "shiv": "shiva", "siva": "shiva",
    "ganesh": "ganesh", "ganesha": "ganesh",
    "vishnu": "vishnu", "vishnu": "vishnu",
    # Bengali / South Indian
    "babu": "babu", "bablu": "bablu",
    "subbu": "subramaniam", "subu": "subramaniam",
    "ravi": "ravi", "ravindra": "ravi",
    # Urdu / Muslim
    "ali": "ali", "asal": "aslam",
    "asif": "asif", "ashif": "asif",
}
Apply it in normalize_tokens after alias substitution:


token = ALIASES.get(token, token)
token = NICKNAMES.get(token, token)

**Tier 2 — Data-driven expansion from labeled pairs.**
Once Digio has historical match decisions, mine the co-occurring token
pairs from matched documents: for every pair (name1, name2) labelled
MATCH where exactly one token differs and the other tokens are
identical, record the differing pair as a candidate nickname mapping.
Rank by frequency, keep those above a threshold (e.g. >= 50 co-occurrences),
and audit manually before adding to NICKNAMES. This turns an
open-ended curation task into a periodic batch job.

**Tier 3 — Learned embedding similarity.**
For nicknames with no explicit mapping (Raju vs Rakesh), embed each
token with a small fastText or IndicBERT model and treat tokens as
equivalent if cosine similarity exceeds a threshold. This is the
technique most likely to generalize to unseen names, but it requires
a model artifact and careful threshold calibration, so it should only
be added once Tiers 1 and 2 have saturated.

**Ordering matters:**
apply Tier 1 (cheap, deterministic) first, then
Tier 2 (still deterministic), and only fall back to Tier 3 for pairs
that survive both. Never let an embedding model override an explicit
alias, because embeddings drift with model updates and make the system
non-reproducible.

**Validation plan for the nickname map:**

Hold out 20% of labelled pairs that contain at least one nickname
variant. Measure precision/recall before and after adding mappings.

Reject any mapping that lowers precision on the hard-negative set,
even if it raises recall. In identity verification, a nickname map
that produces false positives is worse than no map.

Log every mapping used in a decision (explain() should report
aliases_applied) so operations can audit individual cases.

Future improvements
Nickname/alias dictionary for common Indian variants (see Tier 1–3 above).

Transliteration support for regional scripts (see approach above).

Learn feature weights from labeled pairs using logistic regression,
once Digio has enough decision history to fit one.

Per-document-type threshold calibration: PAN, Aadhaar, Passport,
Voter ID, and utility bills have different error profiles, so a single
global threshold is a compromise.

Use DOB / address / gender / document number when available. Name
matching alone has a ceiling; a joint model over multiple fields
would push precision higher.

Calibrate the REVIEW band by business cost: measure the human review
throughput and adjust REVIEW_THRESHOLD so that the review queue
stays within operational capacity.
