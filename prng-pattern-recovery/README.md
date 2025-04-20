# PRNGPattern [Custom Challenge]

📅 **Date of writeup**: April 20, 2025

> **During analysis of a closed-source game environment, I noticed a consistent arrangement of in-world elements whose positions were tightly linked to pseudorandom logic.
These placements, while indirect, revealed enough entropy to reconstruct the internal state of the RNG.
Ultimately, this enabled full prediction of hidden gameplay outcomes and recovery of PRNG-dependent secrets — including a 12-character generated password.**

# Introduction
The challenge was abstract in nature: recover a deterministic password generated on the client side. No binary was given — only observations: a fixed sequence of numeric markers shown on a static 3D scene, and knowledge of their mapping to a linear congruential PRNG state.

Reverse engineering was performed on a Lua-based game scripting layer and supplemented with custom tooling in Python to brute-force seed and pattern alignment.

Software and methods used:
- **Python 3.10**
- **Game scripting bridge (via exposed API)**
- **PRNG emulator (LCG reverse)**

---

## 🧩 Context & setup
Inside a restricted zone of the game world, several static world objects had associated 3D labels that changed across sessions.
These labels displayed strings like:
```
"Light Armor [20,8]"
"Shotgun [12]"
"Colt [18]"
"MP5 [21]"
```

All labels are client-visible and are generated once after the server starts.
They are not random junk: they map to internal server-side RNG calls, whose results are surfaced via limited labeling.

---

## 🔍 Observation: partial leakage of PRNG state

Each label corresponds to a roll of `random(seed, 22)`, with the result determining what item is shown.
Some mappings were one-to-one (`"Shotgun"` → `[12]`), others covered ranges:
```lua
label_map = {
    "Light Armor":        [0, 1],
    "Heavy Rifle":        [7, 8, 9],
    "Uzi":                [23],
    "Colt":               [12],
    "MP5":                [18],
    --[[ ... ]]
}
```

The RNG function being used (confirmed via client-side research and server source leaks) is this:
```python
def random(seed: int, max: int) -> tuple[int, int]:
    MULT = 1103515245
    INC = 12345
    MOD = 2 ** 32
    seed = (seed * MULT + INC) % MOD
    return seed, (seed >> 16) % max
```

A textbook x32 LCG.
Knowing it means perfect predictability, once seed is recovered.

---

## 🧱 Building the signature
Each world object had a fixed position. I collected their coordinates into a hardcoded list.

Then, via runtime inspection, I looped through all visible labels, extracted their (x, y, z) world positions, and matched them against known slots.

If the position matched one of the defined targets, the label was parsed:
```lua
-- pseudocode
for label in allVisibleLabels():
    if label.position ≈ knownObjectPosition:
        if label.text contains "20,8":
            signature[i] = [0, 1]
        elseif label.text contains "MP5":
            signature[i] = [21]
        ...
        else
            signature[i] = [-1]  -- unknown / wildcard
```

Unmatched or absent labels were recorded as wildcards: ([-1], 22).
These act as flexible “don't care” slots in the pattern.

Final structure:
`List[Tuple[List[int], int]]`

Each tuple: acceptable RNG values + `max` used in call.

---

## 🔍 Pattern search logic
The full RNG output is hidden — we only get ~30 filtered samples.

But that's enough to recover both:
- The initial seed
- The offset (how many .next() calls occurred before label assignment)

We simulate the PRNG forward from a guessed seed and search for the signature in a window:
```python
def find_pattern(start_seed: int, signature, area):
    seed = start_seed
    for _ in range(area[0]):
        seed, _ = random(seed, signature[0][1])
    
    stream = []
    seeds = []
    for _ in range(area[1] - area[0]):
        seed, val = random(seed, signature[0][1])
        stream.append(val)
        seeds.append(seed)

    for i in range(len(stream) - len(signature)):
        match = True
        for j in range(len(signature)):
            valid, _ = signature[j]
            if -1 not in valid and stream[i + j] not in valid:
                match = False
                break
        if match:
            return seeds[i + len(signature) - 1], i + area[0]
    
    return None, -1
```

Wildcards (`[-1]`) match any value.

---

## ✅ Match found
The actual brute-force range was small (±500 seconds from observed session start).
Result:
```shell
=> Recovered seed: 1741478463
=> Pattern offset: 97
```
Offset of 97 meant: 97 calls to .next() happened between initial seeding and first label render.
Confirmed via matching observed layouts across multiple runs.

---

## 🔐 Password generation
After placing the objects, the server generated a 12-character password using:
```python
password = ""
for _ in range(12):
    seed, val = random(seed, 25)
    password += string.ascii_lowercase[val]
```

Using recovered seed and offset:
```shell
=> password: `jnpqzxqtmkea`
```
This matched observed values during gameplay interactions requiring password input.

---

## 🧪 Validation

To ensure correctness:
- The recovered password was cross-tested in a protected in-game mechanic requiring exact input.
- Additional seeds were tested to confirm uniqueness of the match (false positives were filtered out).
- The system's randomness was validated as fully deterministic and non-noised.

---

## 📋 Notes
- No fancy RNG was used — just a textbook 32-bit LCG.
- Even without direct access to internal state, a partial leak of PRNG outputs was enough to fully reconstruct it.
- Wildcards didn't reduce precision, as long as enough fixed outputs were present (~25-30 samples).
- Reconstructing seed + offset allowed full prediction of downstream calls.

### 🧪 Dev Notes & Testing
All tooling — PRNG simulator, pattern matcher, visualization — was written in **one day**.  
The remaining **two days** were spent on extensive validation: matching known outputs, confirming seed uniqueness, tuning the pattern scanner, and testing on multiple snapshots.

---

## 📌 Summary
By extracting filtered PRNG outputs from in-game elements, we reconstructed the exact seed and call offset used during session init.

This enabled deterministic recovery of hidden, one-time generated data — including sensitive credentials — with zero server access or debugging tools.

No exploits. No code execution. Just passive observation, math, and 32-bit determinism.

### 🔐 How to prevent this
The main vulnerability is the predictability of the `seed`. A linear congruent generator (LCG) is used, which can be modeled externally and reproduce the course of generation.

To complicate the reversal one could:
- apply a **crypto-resistant source of entropy**, e.g. tied to current player coordinates, number of players in a radius, system time, etc.
- **separate seed pools**: use a separate `seed` for sensitive areas/objects that should not be externally recoverable
- do not rely directly on `random(seed, max) % x`, or encrypt the value before comparison - otherwise the pattern can be picked manually

None of this will solve the problem completely, but it **strongly increases the cost of the attack** - especially if `seed` cannot be obtained externally.
