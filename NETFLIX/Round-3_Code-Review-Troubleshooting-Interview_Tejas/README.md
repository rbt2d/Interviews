# Netflix Interview — Round 3: Code Review & Troubleshooting

## Interview Details

| Field           | Info                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Date**        | March 3, 2026                                                        |
| **Duration**    | 1 hour 2 minutes                                                     |
| **Interviewer** | Tejas — Netflix ESO (Engineering Support Org), ~2 years at Netflix   |
| **Focus**       | CICD and Java application triaging and support                       |
| **Format**      | CodeSignal — 2 JavaScript code review exercises (no prior JS knowledge required) |
| **Outcome**     | **Positive** — *"This is good discussion, I agree with all of what you said"* — no explicit pass/fail stated |

---

## About the Interviewer & Format

**Tejas's day-to-day:** Reads code all day. Gets tickets from Netflix engineers about delivery/CICD/Java application issues. Reads unfamiliar code to understand what it's supposed to do, finds why it's not working, and suggests fixes. 90% code reading + support; 10% automation and knowledge sharing. Does **not** write production code.

**Format framing:** *"We are two co-workers. We both are seeing some code we have not seen before. We're just trying to understand what it does. There's no right answer. Tell me your train of thought."*

Platform: **CodeSignal** — JavaScript code (JavaScript knowledge is irrelevant; the exercise is about reading unfamiliar code and reasoning about intent vs. behavior)

---

## Exercise 1: API Call with Validation Logic

### Code Structure
Three functions:
- `execute()` — async function that makes a POST request to HTTPBin (echo service), validates response, writes to file
- `validate()` — checks response data for correctness
- `writeToFile()` — writes content to file system using the `fs` module

### Bug Identified

**Test case (line 56):** `execute()` called with `movieId: 4829581` (a number)

**Trace through validate():**
1. Does response have a `json` property? → Yes (HTTPBin echoes it back) → doesn't return false
2. `typeof response.json.movieId !== 'number'` → **This is the bug**

**Root cause:** HTTPBin echoes back the request body but `movieId` comes back as a **string** (not a number), even though it was sent as a number. The validation check `typeof movieId !== 'number'` fails — returns `false` — so `writeToFile()` is never called.

**Was this intentional?** The developer's intent was clearly to verify `movieId` is a valid numeric value — but the implementation breaks because the upstream API response format doesn't match the expected type.

### How Ramana Communicated the Fix

**Presented both options to the developer:**

> *"Hey, I found where the issue is. The `validate()` function checks if `movieId` is type `number`, but the API you're calling returns `movieId` as a string in the response. The validation always fails. There are two ways to fix this depending on what you actually need:*
> 1. *If you control the upstream service and movieId should come back as a number — fix it on the API side. The validation logic would then be correct.*
> 2. *If the API format won't change — update the validation to either coerce the type or check for numeric string values.*"

### Customer Unblocking Scenario

**Scenario:** Customer has an Excel sheet with movie IDs in Column A. Their automation reads each ID and calls this script to get the response written to a file. They're blocked because the script is failing validation.

**Tejas added a constraint:** JavaScript owner refuses to change their validation ("my validation is correct — it should be a number"). How do you unblock the customer?

**Ramana's answer:**
> *"While the bug is being sorted out between the code team and the API team, you don't have to be blocked. You can call the API directly from your automation. You're already reading movie IDs from Excel — instead of going through this script, just make the POST request to the endpoint yourself and get the response written to the file directly."*

**Tejas validated:** Agreed, and added another option: ask the JavaScript owner to add a `--no-validation` flag so the customer can skip validation entirely with a one-line config change — no code writing required.

**Key insight demonstrated:** Understand what the customer actually needs (response written to file) vs. what the implementation does (validates + writes). Find the path that achieves the customer's goal independent of the internal bug resolution timeline.

---

## Exercise 2: Topological Sort — Course Prerequisites

### Code Structure

A `foo()` function (DFS-based topological sort) for ordering courses based on prerequisites:
- `visited` set — fully processed courses
- `visiting` set — courses currently in the active DFS path (for cycle detection)
- Throws an error if a course in `visiting` is encountered again → circular dependency
- Input: course data structure with `chapterId`, `chapterName`, `prerequisites`

### Bug Identified

**The `result` array is never populated.** The function builds the traversal correctly but never pushes anything to the output.

**Fix:** After removing a course from `visiting` and adding it to `visited` (line after backtracking), push the course to `result`:

```javascript
visiting.delete(courseId);
visited.add(courseId);
result.push(courseId);  // ← missing line
```

**Why this placement:** Each course gets added to the output only after all its dependencies have been fully resolved — which is the correct topological sort order (post-order DFS).

**Tejas confirmed:** *"That's the bug, yes. Perfect."*

### Code Quality Improvements Suggested

**1. Function naming (vague → descriptive):**
- `foo()` → `getOrderedCourses()` or `visitCourse()` or `dfsTraversal()`
- Without meaningful names, any support engineer reading this must trace the entire function before understanding intent

**2. Error message quality:**
```javascript
// Current (useless in production):
throw new Error("Circular dependency")

// Suggested (actionable):
throw new Error(`Circular dependency detected for course: ${courseId}`)
```
When a ticket comes in with this error, the course ID immediately tells you where the cycle is — no guesswork.

**3. Safety check for missing prerequisite references:**
```javascript
const course = courses[courseId];
if (course === undefined) {
  throw new Error(`Course not found: ${courseId}`);
}
```
If a prerequisite references a course ID that doesn't exist in the data structure, `courses[prereqId]` returns `undefined`, and the next line throws a cryptic `TypeError` instead of a meaningful message.

**4. Empty prerequisites array check (minor):**
- An empty array is truthy in JavaScript — `if (prereqs.length > 0)` would be more precise than `if (prereqs)`, though functionally the behavior is the same since the `for...of` loop just doesn't iterate

**5. Return full course object instead of just courseId:**
```javascript
result.push(course);  // entire object, not just courseId
```
A caller building a learning path or course curriculum needs more than just an ID — they need the name and metadata too.

### Data Structure Improvements

**Current structure:**
```javascript
{
  "4": { chapterId: 4, chapterName: "Advanced JS", prerequisites: [2, 3] }
}
```

**Problem 1 — Redundant key:** `chapterId: 4` inside the object is redundant since `4` is already the key. If someone updates one but not the other, the data becomes inconsistent.

**Fix:** Remove `chapterId` from the object body — the key is the single source of truth:
```javascript
{
  "4": { chapterName: "Advanced JS", prerequisites: [2, 3] }
}
```

**Problem 2 — Type mismatch:** Object keys are strings (`"4"`) but prerequisites are numbers (`[2, 3]`). This forces the algorithm to do explicit type coercion checks.

**Fix:** Make prerequisites strings to match the key type, OR change the outer structure to use an array (numeric index):
```javascript
// Option A: string prerequisites to match string keys
{ "4": { chapterName: "Advanced JS", prerequisites: ["2", "3"] } }

// Option B: array-based (eliminates string-vs-number entirely)
[
  { chapterName: "Intro", prerequisites: [] },
  { chapterName: "Advanced JS", prerequisites: [0, 1] }
]
```

With consistent types, the `typeof` check in the algorithm becomes unnecessary — Tejas confirmed this was exactly the point.

---

## What Went Well

- **Correctly identified both bugs** — type mismatch in validation (Exercise 1) and missing `result.push()` (Exercise 2)
- **Strong "think out loud" approach** — traced execution step by step, narrated reasoning at each line
- **Customer-first unblocking** — didn't just fix the code bug; found a path for the customer to get their work done independently of the internal bug resolution timeline
- **Code quality suggestions were thorough** — function naming, error messages, null safety, return type, data structure inconsistency — all unprompted
- **Data structure analysis** — identified both the redundant key AND the type mismatch, then traced how eliminating the type inconsistency simplifies the algorithm
- **Collaborative tone** — treated it as two engineers reading unfamiliar code together, not a performance
- **Tejas's feedback:** *"I agree with all of what you said"*, *"I like that approach"*, *"That's perfect"* throughout

## What Could Have Been Better

- **Exercise 1 workaround came after prompting** — Ramana landed on the direct API call solution but only after Tejas set the full scenario (Excel sheet, blocked customer). Could have offered it earlier as a proactive option
- **Validation flag option** — Tejas suggested the `--no-validation` flag workaround; Ramana could have surfaced this independently as a quick customer-side workaround
- **Array-based data structure** was offered by Tejas as the cleaner alternative — Ramana was close but landed on string prerequisites first

---

## Hiring Signal Assessment

**Verdict: Positive — No explicit pass/fail stated**

Tejas expressed agreement with every suggestion throughout the session and said *"I agree with all of what you said"* at the end. The session ran the full hour with genuine collaborative engagement. Tejas's closing questions (what excites you at Netflix, high performer definition, how team stays current) were standard culture-fit questions — all answered well.

The role is clearly a strong fit: Ramana's framing of his work as *"detective work — reading through someone's code or config, understanding what they're doing and what they're trying to find, finding where the actual behavior diverges from their intent"* matched Tejas's description of his own day-to-day almost word for word.

---

## Tejas's Answers to Ramana's Questions

### Q1: What's the most exciting thing about working at Netflix day-to-day?
> *"I think it's the peers — the people I work with. They are amazing colleagues. Having the opportunity to work with them every day is something I look forward to."*

### Q2: What defines a high performer on your team?
> *"Everyone on my team is a high performer — there's no written list of 15 things that makes you an outperformer. It's how you gel with the team, what ideas you bring to the table. Given the fast-moving world of AI and adapting to it while working on support — I'd say it's about recognizing productivity improvements and how to improve productivity for each other. It's about daily work, honesty towards it, and how you help each other bring out the best every day."*

### Q3: How does the team stay current on internal tooling and platform changes?
> - Netflix has a **"paved path"** — productivity engineering teams document the supported update path for Java, JavaScript, and Python ecosystems clearly
> - Support engineers are the **advocates for the paved path** — front-line for adoption
> - Trainings every 15 days
> - Productivity engineering team stays up-to-date and pushes updates to support teams proactively
> - Real-world learning through tickets fills in edge cases that documentation doesn't cover

---

## Key Concepts Demonstrated

| Skill | Where Demonstrated |
|-------|-------------------|
| Reading unfamiliar code cold | Both exercises — no prior exposure |
| Bug identification | Type mismatch (Ex 1), missing result.push (Ex 2) |
| Presenting fix options without prescribing | "Here are two ways to fix this depending on what you need" |
| Customer unblocking (independent of bug fix) | Direct API call workaround |
| Code quality review | Function names, error messages, null safety, return types |
| Data structure analysis | Redundant key, type inconsistency, suggesting array-based alternative |
| Communication calibration | Framing feedback as helpful, not critical |
