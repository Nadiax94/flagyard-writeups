# Roll Call - Crypto Writeup

## Challenge Information

- **Challenge Name:** Roll Call
- **Category:** Crypto
- **Platform:** Flagyard CTF
- **Service:** `tcp.flagyard.com:15537`

---

# Challenge Description

<img width="906" height="198" alt="لقطة الشاشة 2026-08-26 214213" src="https://github.com/user-attachments/assets/5c6d8b68-476b-403a-8529-359f31a0eeaa" />


The challenge simulates a roll call system containing 256 members.

In each round, the server randomly removes one member from the roster and asks us to identify the missing member.

The goal is to correctly identify the absent member for **16 consecutive rounds** in order to obtain the flag.

---

# Protocol Analysis

After connecting to the service, the server provides the following command:

```
handle <c>
```

The value `c` is controlled by the client.

The server returns:

- A list of encrypted identifiers for all present members.
- An encrypted value corresponding to the chosen input.

After that, we submit our answer using:

```
submit <member>
```

---

# Cryptographic Analysis

The server maps each member using the function:

```
H(m) = (SHA256("id=" + str(m)) mod p)^2 mod p
```

Then it generates a random secret value:

```
k
```

and applies it to the roster values:

```
eids = [H(m)^k mod p for m in roster]
```

It also computes:

```
deid = c^k mod p
```

where `c` is controlled by the attacker.

---

# Vulnerability Discovery

The main weakness is that the same secret exponent `k` is applied to both:

- The encrypted roster values.
- The attacker-controlled input.

Since the attacker can choose `c`, it is possible to construct predictable values and compare them with the server responses.

Additionally, the roster is sorted and contains exactly one missing member, creating a single gap that can be detected.

For the correct absent member:

```
predicted_value == deid
```

while incorrect guesses produce unrelated values.

---

# Recovering the Prime Value p

The value of `p` was not provided directly.

It was recovered using controlled queries.

A selected set of members was used:

```
S = {40,80,120,160,200}
```

A controlled value was generated:

```
c = ∏ H(m)
```

For each possible absent member `a`, the expected result was calculated.

For the correct candidate:

```
pred(a) - deid ≡ 0 (mod p)
```

Using:

```
gcd()
```

on the obtained differences allowed recovery of:

```
p =
113922386694983050382912391517437431439572639334235676622418446302583918640667
```

---

# Solving Strategy

Each round allows only three operations:

```
handle
handle
submit
```

Therefore, brute forcing all 256 members is not possible.

The solution uses two queries.

---

## Query 1

The first query uses odd member values:

```
{1,3,5,...,255}
```

The calculated predictions are compared against `deid`.

If a match is found, the absent member is identified.

---

## Query 2

If the first query does not reveal the answer, the second query uses:

```
{0,2,4,...,254}
```

This covers the remaining cases.

After identifying the missing member:

```
submit <answer>
```

is sent to complete the round.

The process is repeated for all 16 rounds.

---

# Performance Optimization

To improve performance, prefix and suffix products were used.

Instead of recalculating products for every possible candidate:

```
O(n)
```

the calculation was optimized to:

```
O(1)
```

for each candidate.

This allows every round to complete almost instantly.

---

# Solver Implementation

The final solver was written in Python using only standard libraries:

```
socket
hashlib
ast
re
sys
time
```

The solver performs:

- Connection handling.
- Sending `handle` queries.
- Identifying the missing member.
- Submitting the correct answer.
- Repeating the process for 16 rounds.

---

# Execution Result

Running:

```bash
python3 solve.py
```

Successfully completes all rounds.

<img width="1165" height="240" alt="لقطة الشاشة 2026-08-26 214259" src="https://github.com/user-attachments/assets/05a6034d-9bce-4e0e-afc3-0333aba59eff" />


Flag:

```
FlagY{0894dcefbb95e38469********}
```

(The remaining part of the flag is hidden.)

---

# Conclusion

The challenge relies on a cryptographic design weakness where the attacker can control the input to the same transformation that protects the server's data.

By exploiting the ordered roster and the mathematical properties of the construction, we were able to determine the missing member with a limited number of queries and complete all 16 rounds successfully.
