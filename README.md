# Monotonic Queue: What It Really Does, a Sliding Window Maximum Walkthrough, and How to Spot It in an Interview

Search for "monotonic queue" and you'll get a pile of LeetCode solution dumps that paste a `deque` snippet and call it a day. What they usually skip is the part that actually matters: why the pops are legal, why the total cost is O(n) instead of O(nk), and how you're supposed to recognize the pattern before you've already burned twenty minutes on a heap.

So this is the version I'd want if I were trying to actually learn it: the data structure, a full trace on the classic problem, the variants that show up in real interviews, and where to practice it if you want structured feedback instead of a comment section.

## The one-sentence definition

A monotonic queue is a double-ended queue whose values stay sorted — increasing or decreasing — from head to tail. Every element enters at the back once and leaves once, either from the back (because a bigger or smaller value just made it irrelevant) or from the front (because it fell out of the window you're tracking).

Because each element is pushed and popped at most once, maintaining it is amortized O(1) per operation, and O(n) across a whole array. [1]

The important part isn't the sorting. It's what the sort buys you: whatever sits at the head is the current maximum or minimum, readable in O(1). That's what makes the sliding window maximum problem solvable in linear time.

Terminology warning before you go further: a monotonic *stack* and a monotonic *queue* are not the same tool, even though they look nearly identical in code. A monotonic stack answers "for each element, what's the next greater or smaller one?" and pops from one end. A monotonic queue adds a second exit — the head — so elements can also expire as a window slides. Both are covered in AlgoMonster's [Monotonic Stack/Deque intro](https://algo.monster/problems/mono_stack_intro?fpr=li69), which is free to read and distinguishes the two cases explicitly. [2]

## Why the naive answer is too slow

Take the standard problem: given `nums = [1, 3, -1, -3, 5, 3, 6, 7]` and `k = 3`, return the maximum of every window of size 3. Expected output: `[3, 3, 5, 5, 6, 7]`. [3]

The brute-force version scans all k elements for each of the n − k + 1 windows, which is O(n·k). With n up to 10⁵, that's the difference between passing and a time-limit error.

The observation that fixes it is this: when a window slides, only two things change — one element leaves on the left, one arrives on the right. Everything in between is untouched. If you could keep the maximum of the middle elements cached, you'd only ever need to handle the two updates.

And here's the part worth internalizing: you don't need to keep all of them. For two indices `left < right`, `nums[left]` leaves the window earlier than `nums[right]`. So if `nums[right] >= nums[left]`, then `nums[left]` can never be the answer while both are in the window. It's dead weight. Drop it. [4]

Repeat that reasoning as you walk left to right and the survivors form a decreasing sequence — a monotonic deque.

## A full trace, step by step

The implementation stores indices, not values, because you need the index to tell whether the head has expired:

python
from collections import deque

def sliding_window_maximum(nums, k):
    q = deque()  # indices, values decreasing from head to tail
    res = []
    for i, cur in enumerate(nums):
        while q and nums[q[-1]] <= cur:
            q.pop()          # cur outlives it and is >= it: it can never be the max
        q.append(i)
        if q[0] == i - k:    # head just left the window
            q.popleft()
        if i >= k - 1:
            res.append(nums[q[0]])
    return res


Now watch the state change on `[1, 3, -1, -3, 5, 3, 6, 7]`, k = 3:

| i | nums[i] | deque (indices → values) | What happened | Window max |
| --- | --- | --- | --- | --- |
| 0 | 1 | [0→1] | first element | — |
| 1 | 3 | [1→3] | 1 ≤ 3, so index 0 popped from the back | — |
| 2 | -1 | [1→3, 2→-1] | -1 < 3, nothing popped | 3 |
| 3 | -3 | [1→3, 2→-1, 3→-3] | -3 < -1 | 3 |
| 4 | 5 | [4→5] | 5 clears everything behind it; index 1 also expired | 5 |
| 5 | 3 | [4→5, 5→3] | 3 < 5 | 5 |
| 6 | 6 | [6→6] | 6 clears the deque; index 4 expired | 6 |
| 7 | 7 | [7→7] | 7 > 6 | 7 |

Output: `[3, 3, 5, 5, 6, 7]`. [3] [5]

Two details in that trace are exactly where people lose points in interviews.

The first is that the index-1 entry at step 4 got removed twice over — once because 5 dominated it, once because the window had moved past it. Both removals are fine, but they're different reasons, and interviewers ask about the difference.

The second is the order of operations. Pop-from-back, append, then check the head for expiry. If you check expiry before appending, you can pop a head that should still be valid; if you compare values from the head instead of indices, duplicates in the input can push the window boundary off by one. The expiration check `q[0] == i - k` works precisely because the deque holds indices in increasing order [6].

## Why it's O(n), not "O(n) because magic"

Each index is appended exactly once. Each index is removed at most once — from the back when a value dominates it, or from the front when it exits the window. So the inner `while` loops together execute at most n removals across the entire run. Total work: O(n). [1] [4]

Space is O(k) for a window of size k in the sliding-window-maximum case, plus O(n − k + 1) for the output.

This amortized argument is the real answer to "why is this linear?" — and it's the single most common follow-up question, because the code contains a nested loop and the naked eye reads nested loops as quadratic.

## How it compares to the alternatives

If you're asked "why not a heap?", this table is the answer.

| Approach | Time | Space | Catch |
| --- | --- | --- | --- |
| Brute force per window | O(n·k) | O(1) | Too slow for large n |
| Max-heap with lazy deletion | O(n log k) | O(k) | Stale entries sit in the heap; you pop them later |
| Segment tree / sparse table | O(n log n) build, O(1) query | O(n) or O(n log n) | Fine, but more code than the problem deserves |
| Monotonic deque | O(n) | O(k) | Slightly more reasoning up front, much less code overall |

The heap version is genuinely competitive in practice for small k. The monotonic deque wins when k is large, when you need both max and min in the same pass, or when the window itself is the thing being optimized inside a dynamic programming recurrence.

## Where it shows up beyond sliding window maximum

Sliding window maximum is the teaching example, not the whole pattern. Places it recurs:

- **Windowed extrema with a distance constraint.** Minimum or maximum of a rolling range, where "next greater" only counts if it's within reach. AlgoMonster's intro notes that once you add a distance restriction, the stack needs a second exit and becomes a deque. [2]
- **Daily Temperatures-style variants.** Pure next-greater-element problems use the stack form, but the moment the constraint becomes "within k positions," it becomes the queue form.
- **DP optimization.** Recurrences where each state takes the min or max over the previous k states collapse from O(n·k) to O(n). This is standard in competitive programming — USACO problems like Redistricting are built on it — and it's the version most candidates have never seen. [7]
- **Streaming and real-time systems.** Rolling maximum over a fixed recent time window: think a metric that alerts on the peak of the last N samples, or surge pricing keyed to the highest demand-supply ratio in the last few minutes. Same algorithm, different vocabulary. [8]
- **Audio and signal processing.** A compressor that looks back a few milliseconds and scales the current sample by the loudest recent peak is running a sliding-window maximum. [8]

If you only learn the LeetCode 239 template, you'll recognize maybe half of these. That's the argument for learning the pattern rather than the problem.

## The bugs that actually cost you the round

Five things go wrong repeatedly:

1. **Storing values instead of indices.** It works on inputs without duplicates and breaks on the first repeated value, because you can't tell whether the head is still in the window.
2. **Using `<` where you need `<=`.** With `nums[q[-1]] < cur` you keep equal values in the deque, which is harmless for correctness but wastes memory and complicates the expiry check. Pick one and be consistent.
3. **Expiring from the back instead of the front.** Old elements leave through the head. Values get evicted through the tail. Mixing those up produces an answer that's correct on the first window and wrong after.
4. **Forgetting the empty-deque guard.** `nums[q[0]]` on an empty deque is an index error waiting for the right input.
5. **Emitting results before the first window is full.** The condition is `i >= k - 1`, not `i >= k`.

None of these are conceptual failures. They're all off-by-one or ordering mistakes, which is exactly why this pattern rewards repetition more than reading.

## Practicing this pattern without random grinding

The failure mode for monotonic queue isn't usually "I don't understand deques." It's "I wrote it six weeks ago from a memorized template and now I can't reconstruct the expiry condition."

That's a structural problem, and it's what [AlgoMonster](https://algo.monster?fpr=li69) is built around — it teaches patterns as reusable templates with decisions you can re-derive, rather than as 3,000 problems to grind. The relevant material for this topic:

- The [Monotonic Stack/Deque intro](https://algo.monster/problems/mono_stack_intro?fpr=li69) covers the definition, the push modification, and the two application families: next greater/smaller element, and windowed max/min.
- The [Sliding Window Maximum walkthrough](https://algo.monster/problems/sliding_window_maximum?fpr=li69) builds from brute force to max-heap to monotonic deque in that order, with complexity analysis at each step. That progression is useful precisely because it shows *why* the deque exists instead of asserting it.
- The [problem flowchart](https://algo.monster/flowchart?fpr=li69) maps questions like "need nearest greater/smaller bounds?" and "compute a max/min over a moving range?" to the monotonic stack or deque branch — a decision procedure for recognition, which is the part that fails under interview pressure. [9]

Beyond that, the platform includes unlimited AI assistant usage for hints when you're stuck, a Speedrun mode for timed review, and monthly-updated questions from top tech companies. Bonus courses (a CS foundations course and a build-an-LLM course) come with the subscription; partner-developed courses like Quant Interview Prep are sold separately. [10]

## AlgoMonster plans and current pricing

There's no month-to-month option — it's annual or lifetime. [10]

| Plan | What you get | Price | Billing | Sign up |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to test whether the text-and-diagram format works for you | $0 | — | [Start on the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; member discount on live bootcamps | $99/year, displayed at $8.25/mo (list price $198) | Billed annually, cancel or upgrade anytime | [See the Pro Yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan, plus lifetime updates and internal referral consideration for qualified candidates | $189 one-time (list price $378) | Billed once | [See the Pro Lifetime plan](https://algo.monster/subscribe?fpr=li69) |

A word of caution about numbers you'll find elsewhere. Several third-party articles quote $299/year, or a $459 lifetime price, or a "$99–150 lifetime" range. Those don't match what the official subscribe page displays, and the official page is the one that charges your card. Pricing here moves with sales, so check the page rather than a coupon blog — some of those articles also list promo codes claiming an extra 10% stack, and I can't verify those codes actually apply at checkout.

One more limitation worth knowing before you pay: AlgoMonster has no video content. Every explanation is text plus interactive diagrams. If you learn by watching someone else code, this will feel like a wall.

## Is it worth paying for, for this specific topic?

If you're stuck on monotonic queue, the free tier plus the two pages linked above will get you through sliding window maximum, and that may be all you need. The honest math changes if you're preparing across the full pattern set: the annual plan at $99 works out to roughly $0.28/day for 12 months of access, and the lifetime plan at $189 costs less than one year of the annual list price.

For someone with an interview in 4–8 weeks, the annual plan covers it. For someone who interviews every couple of years when they want a new job, the lifetime option is the one that doesn't expire between searches.

What it won't do is simulate interview pressure. There's no timer running and no one probing your reasoning — which means the monotonic queue template you write here still needs to survive a live 45-minute screen on its own. Plan for that separately.

👉 [Check the current AlgoMonster pricing and start on the free plan](https://algo.monster?fpr=li69)
