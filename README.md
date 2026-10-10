# Decode Ways: the dynamic programming pattern behind LeetCode 91, why zeros break everything, and how to say it out loud in an interview

Most people searching for "decode ways" fall into one of two groups. The first is stuck on LeetCode 91 right now, staring at a string like `"226"` and wondering why the answer is 3 and not 6. The second group solved it once, passed the problem, and three weeks later couldn't reproduce the recurrence from memory. Both groups need the same thing, and it isn't another paste of code.

Below is the full picture: what the problem actually asks, why zeros carry almost all the difficulty, four working implementations from brute force to O(1) space, the edge cases that cause wrong answers, and the pattern label you should attach to it so you can spot the next one.

## What the problem is actually asking

A message of letters A–Z is encoded into digits:


'A' -> "1", 'B' -> "2", ... 'Z' -> "26"


You get a digit string and you count how many ways it can be grouped and mapped back to letters. The encoding direction is unambiguous; decoding is not, because `"1"` and `"12"` and `"112"` can each be split in multiple valid ways.

The three examples you should be able to answer without thinking:

| Input | Output | Why |
| --- | --- | --- |
| `"12"` | 2 | `"AB"` (1, 2) and `"L"` (12) |
| `"226"` | 3 | `"BZ"` (2, 26), `"VF"` (22, 6), `"BBF"` (2, 2, 6) |
| `"06"` | 0 | `"06"` is not `"F"` — a group can't have a leading zero |

Constraints on the official version: `1 <= s.length <= 100`, the string contains only digits, it may contain leading zeros, and the test cases are generated so the answer fits in a 32-bit integer.

The `"11106"` case is worth memorizing because it isolates the leading-zero rule. It maps to `"AAJF"` (1, 1, 10, 6) and `"KJF"` (11, 10, 6). The grouping (1, 11, 06) is *not* valid, even though 06 numerically equals 6 and 6 maps to F. The string `"06"` and the string `"6"` are different inputs.

## The intuition: climbing stairs with a bouncer at the door

If you've done Climbing Stairs, you already have the skeleton. There, `f(n) = f(n-1) + f(n-2)`: from step *n* you came from one step down or two. Here, from position *i* you either consume **one digit** or **two digits** — so you inherit the count from `i-1` or from `i-2`.

The difference is that Climbing Stairs has an open door and Decode Ways has a bouncer checking IDs at each entry:

- One digit is allowed only if it's between 1 and 9. A `'0'` cannot stand alone.
- Two digits are allowed only if the pair forms a number from 10 to 26. That excludes `"00"` through `"09"` and everything from 27 up.

State definition:


f[i] = number of ways to decode the first i characters


Transitions:


f[i] += f[i-1]   if s[i-1] != '0'                        # decode one digit
f[i] += f[i-2]   if 10 <= int(s[i-2:i]) <= 26            # decode two digits


Base cases are where people quietly lose points. `f[0] = 1` — the empty prefix has exactly one decoding, the empty one. That value isn't intuitive, but it's what makes the two-digit transition work at `i = 2` without a special branch. And `f[1] = 1` if the first character isn't `'0'`, otherwise 0.

## Why zero is the entire tricky part

Strip the zeros out and this problem is a two-line recurrence. Every wrong submission I've seen traces back to one of these cases:

- **A leading zero.** `"0"` returns 0. So does `"06"`. There is no encoding that starts with zero.
- **`"10"` and `"20"`.** These are the only two-digit groups that end in zero, and they have exactly one valid decoding each — the pair itself. `"10"` is `"J"`, never `"1"` + `"0"`.
- **A zero after 3–9.** `"30"`, `"40"`, `"70"` and friends are dead. The digit can't stand alone, and 30 > 26 so it can't pair up. Any string containing a `'0'` immediately preceded by a digit in 3–9 decodes to zero ways — the whole string, not just that position.
- **`"100"`.** `"10"` then a stranded `"0"`. Result is 0.
- **`"101"`.** `"10"` then `"1"` → `"JA"`. Result is 1.

Here's the bug that eats the most time. Writing the two-digit check as:

python
if i >= 2 and int(s[i-2:i]) <= 26:


`int("06")` is `6`, which passes. So `"06"` gets treated as a valid pair and you return 1 instead of 0. You need the leading-zero guard:

python
if i >= 2 and s[i-2] != '0' and int(s[i-2:i]) <= 26:


Checking the digit directly (`s[i-1] == '1' or (s[i-1] == '2' and s[i] <= '6')`) sidesteps the whole issue, and it's also what most interviewers want to hear described out loud.

## Four implementations, from exponential to constant space

### 1. Plain recursion — O(2ⁿ)

python
def numDecodings(s: str) -> int:
    def dfs(i: int) -> int:
        if i == len(s):
            return 1              # consumed the whole string: one valid decoding
        if s[i] == '0':
            return 0              # dead end
        ways = dfs(i + 1)         # take one digit
        if i + 1 < len(s) and 10 <= int(s[i:i+2]) <= 26:
            ways += dfs(i + 2)    # take two digits
        return ways
    return dfs(0)


Correct, and it times out. In the worst case — a long run of `'1'`s — the tree branches at every position.

### 2. Memoized recursion — O(n) time, O(n) space

python
def numDecodings(s: str) -> int:
    memo = {}
    def dfs(i: int) -> int:
        if i in memo:
            return memo[i]
        if i == len(s):
            return 1
        if s[i] == '0':
            return 0
        ways = dfs(i + 1)
        if i + 1 < len(s) and 10 <= int(s[i:i+2]) <= 26:
            ways += dfs(i + 2)
        memo[i] = ways
        return ways
    return dfs(0)


Same logic, cache `dfs(i)`. There's nothing clever here; the subproblem "how many ways to decode `s[i:]`" only depends on `i`, so it should only be computed once.

### 3. Bottom-up tabulation — O(n) time, O(n) space

python
def numDecodings(s: str) -> int:
    n = len(s)
    f = [0] * (n + 1)
    f[0] = 1
    f[1] = 0 if s[0] == '0' else 1
    for i in range(2, n + 1):
        if s[i-1] != '0':
            f[i] += f[i-1]
        if s[i-2] != '0' and int(s[i-2:i]) <= 26:
            f[i] += f[i-2]
    return f[n]


This is the version AlgoMonster's editorial uses, and it's the easiest to defend in an interview because the array index lines up with "prefix length" instead of "character position."

### 4. Rolling variables — O(n) time, O(1) space

Since `f[i]` only ever reads `f[i-1]` and `f[i-2]`, two variables are enough:

python
def numDecodings(s: str) -> int:
    if not s or s[0] == '0':
        return 0
    prev2, prev1 = 1, 1
    for i in range(2, len(s) + 1):
        cur = 0
        if s[i-1] != '0':
            cur += prev1
        if s[i-2] != '0' and int(s[i-2:i]) <= 26:
            cur += prev2
        prev2, prev1 = prev1, cur
    return prev1


Write this one only after the array version works. Optimizing space before you have correct transitions is how you end up debugging two bugs at once.

## Tracing `"226"` by hand

This is the answer to "why 3 and not 6" — and it's the trace you'll be asked to reproduce on a whiteboard.

| i | Character | One-digit valid? | Two-digit valid? | f[i] |
| --- | --- | --- | --- | --- |
| 0 | — | — | — | f[0] = 1 (base) |
| 1 | `'2'` | yes → +f[0] | — | f[1] = 1 → `"2"` = B |
| 2 | `'2'` | yes → +f[1] = 1 | `"22"` ≤ 26 → +f[0] = 1 | f[2] = 2 → BB, V |
| 3 | `'6'` | yes → +f[2] = 2 | `"26"` ≤ 26 → +f[1] = 1 | f[3] = 3 → BBF, VF, BZ |

The count stays small because each position only *inherits* earlier counts. It never enumerates the strings themselves. And notice there's no multiplication anywhere: when a position has both options, you're adding two independent sets of decodings, not pairing them.

## Complexity, and what to say when asked

| Approach | Time | Space | Verdict |
| --- | --- | --- | --- |
| Recursion | O(2ⁿ) | O(n) stack | Correct but TLE |
| Memoized DFS | O(n) | O(n) cache | Fine, and easiest to derive |
| Bottom-up DP | O(n) | O(n) | The standard answer |
| Rolling variables | O(n) | O(1) | Best space, worse readability |

You iterate the string once and do constant work per index, so O(n) time is the floor — there's no way to count these groupings in less than linear time, since a change at the last character can change the total.

## Which pattern is this, exactly

If you're collecting patterns rather than solutions, Decode Ways lives in the 1-D dynamic programming family: single index state, constant-size transition, answer read off the last state. Climbing Stairs, House Robber, and Minimum Cost Climbing Stairs are the same shape with different validity rules.

AlgoMonster files Decode Ways under its Dynamic Programming pattern and publishes a full editorial walkthrough for LeetCode 91 that follows the array-based formulation above, plus a separate editorial for Decode Ways II (LeetCode 639). It's one of the 48 named patterns the platform organizes its curriculum around, which is the actual product: not a bigger problem bank, but an explicit mapping from "here's a new problem" to "here's the template that fits."

👉 [Read the Decode Ways pattern walkthrough and see how the template is presented](https://algo.monster?fpr=li69)

### The sequel: Decode Ways II

LeetCode 639 keeps the same mapping but adds `'*'` as a wildcard that can stand for any digit 1–9. The structure doesn't change; the validity checks multiply:

- A lone `'*'` contributes 9 ways (`"1"` through `"9"`).
- `"**"` contributes 15 — `11`–`19` is nine, `21`–`26` is six.
- `"*X"` contributes 2 when X ≤ 6 (so 10–16 or 20–26) and 1 when X > 6 (only 17, 18, 19).

The result is returned modulo 10⁹ + 7 because the counts explode. If you can do LeetCode 91, 639 is a bookkeeping exercise rather than a new idea — and that's the point of learning the pattern instead of memorizing one solution.

## The mistakes that actually cost people the interview

- **Only checking `<= 26`.** Covered above. `"06"` breaks it.
- **Forgetting `f[0] = 1`.** Without the empty-prefix base case, the two-digit branch at `i = 2` has nothing to add.
- **Indexing out of range.** At `i = 1` there is no `f[i-2]`. Guard with `i >= 2`.
- **Mixing up "index in the string" and "number of characters consumed."** Pick one and be consistent in both the array and your explanation.
- **Optimizing space first.** Get O(n) space correct, verify on `"226"`, `"06"`, `"0"`, `"10"`, `"27"`, `"100"`, *then* collapse to two variables.

Also be ready for the follow-up: "what if the mapping went up to 99 instead of 26?" The recurrence gains a third branch from `i-3`, and the leading-zero rules get messier. The interview question is testing whether you understand *why* the recurrence has the number of terms it does, not whether you can recite two lines.

## Practicing this pattern without grinding 500 problems

Reading one solution works until the next variation shows up. The people who get fast at these problems aren't solving more of them — they're recognizing sooner which template applies.

AlgoMonster is built around that idea. It's a text-and-diagram curriculum that names 48 patterns, gives you the template for each, and lists the problems that use it. The teaching format matters: there are no video walkthroughs at all, so if you learn by watching someone code, this isn't the right tool. The problem bank is also deliberately small compared to LeetCode's thousands — the claim isn't volume, it's structure.

Third-party reviews of the platform generally land in the same place: reviewers describe it as suited to a 4–8 week prep push at one to two hours a day, with thin coverage of system design and behavioral interviews. So if you're preparing for a senior loop that includes a design round, budget separately for that.

Where it does save you money is the billing model. There's no monthly plan and no recurring treadmill if you don't want one — you either pay for a year or pay once for lifetime access.

👉 [Check the current plan pricing and the free tier on AlgoMonster](https://algo.monster?fpr=li69)

### Current plans

AlgoMonster shows three tiers on its subscribe page. Prices move with promotions, so the number you see at checkout is the only one that counts — and third-party write-ups quote anywhere from $99 to $459 for lifetime access because those sales have changed over time.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| **Free** | Limited access to a subset of patterns and problems — enough to judge the text-and-diagram teaching style | $0 | No cost, no commitment | [Start with the free tier](https://algo.monster?fpr=li69) |
| **Pro — Yearly** | All patterns, lessons and problems; unlimited AI assistant usage; unlimited Speedrun; real interview questions from top tech companies updated monthly; content written by ex-Google engineers and competitive programmers; member discount on live bootcamps | $99/year, shown as $8.25/mo billed annually (list price $198) | Recurring yearly, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| **Pro — Lifetime** | Everything in the yearly plan plus lifetime updates to all current and future content | $189 one-time (list price $378) | Billed once | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

The break-even question is simple arithmetic. If you expect to touch interview prep more than once — a job search now, another in two or three years — the lifetime plan costs less than two annual renewals at list price. If this is a single focused push and you'll finish the curriculum in six weeks and never log in again, yearly is cheaper upfront.

One honest caveat: there's no formal free trial, so the free tier is your preview of the format. Spend a few days in it before paying. If text and diagrams don't click for you, no discount fixes that.

## FAQ

**Why is the answer for `"226"` three and not six?**
Because you're counting valid groupings, not permutations. The three groupings are (2, 2, 6), (22, 6) and (2, 26). Each grouping maps to exactly one string.

**Does `"0"` alone have a decoding?**
No. Zero has no letter, and it can't lead a two-digit group. `"0"` and `"06"` both return 0.

**Is `"10"` one way or two?**
One. It decodes as `"J"`. The split (1, 0) is invalid because `"0"` isn't mapped.

**What's the time complexity?**
O(n) with memoization or DP. Plain recursion is O(2ⁿ) because each position can branch twice.

**Is Decode Ways asked in real interviews?**
It's a long-standing medium in the dynamic programming tag and shows up in interview prep sets, which is why the pattern is worth learning properly rather than solving once and forgetting. The follow-up questions are usually about zeros or about the wildcard variant.

**Do I need LeetCode Premium to practice it?**
No. The base problem and its editorial-style explanation exist outside any paywall, and a structured pattern curriculum covers the same ground if you want the template rather than a one-off answer.

## The short version

`f[i]` counts decodings of the first `i` characters. Add `f[i-1]` when the current digit isn't `'0'`. Add `f[i-2]` when the last two digits form a number between 10 and 26, with no leading zero. Base case `f[0] = 1`. Everything hard about this problem is a zero.

Get that recurrence right on paper, trace `"226"` and `"06"` out loud, and you've got both the answer and the pattern.
