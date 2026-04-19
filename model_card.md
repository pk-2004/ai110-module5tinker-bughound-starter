# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

Describe the workflow in your own words (plan → analyze → act → test → reflect).  
Include what is done by heuristics vs what is done by Gemini (if enabled).

---
The heuristic model using regex to match keywords with substitute code. Gemini mode sends the entire code to the LLM asking for issues through JSON. The Gemini mode is much more accurate while the heuristic mode is quicker. 
## 3) Inputs and outputs

**Inputs:**

- What kind of code snippets did you try?
- What was the “shape” of the input (short scripts, functions, try/except blocks, etc.)?

The input is small python code with in the samples file. 

**Outputs:**

- What types of issues were detected?
- What kinds of fixes were proposed?
- What did the risk report show?

---
Issues include lack of robustness, type error cases, and logic handiling issues. If logging.info(, will replace it with print(. Gemini mode will give a much more wider range of proposal. Score, level, and autofix were showed for input. For example with cleanish.py, score was 55-60, level was medium, and autofix was no. 

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:

- What does the rule check?
- Why might that check matter for safety or correctness?
- What is a false positive this rule could cause?
- What is a false negative this rule could miss?

---

Rule 1 — Return statement removed

- Whether the original had return anywhere but the fixed code does not.
-  Removing a return silently changes the function to return None, breaking any caller that uses the result.
-  A fix that refactors a function to modify state in-place (e.g. appending to a list without returning it). The fix is valid, but the rule still fires and deducts 30 points.
-  A fix that keeps return inside an unreachable branch like if False: return value. The string check passes, but the function still returns None at runtime.

Rule 2 — New function calls introduced
- Whether the fix calls any function not present in the original, using a regex scan for word( patterns.
-  A new call like isinstance or os.path.exists assumes that function is available. The agent cannot see the rest of the codebase to verify this.
-  A fix that renames a call . The regex treats logging and info as brand-new calls and deducts 15 points, even though this is the intended replacement.
-  A fix that reuses an existing function name with completely different arguments. The call name matches so no penalty fires, even though behavior changed.

## 5) Observed failure modes

Provide at least **two** examples:

1. A time BugHound missed an issue it should have caught  
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary  

For each, include the snippet (or describe it) and what went wrong.

---

1. Missed issue — comments-only input

```python
# This is a comment
# Another comment
# No actual code here
```

The LLM flagged this as an Informational / Design issue, but the risk assessor only handles high, medium, and low severities. Informational fell through with zero score deduction, so the risk report showed score=100, level=low, and autofix=true — even though an issue was found. BugHound expressed full confidence on a file that had no executable code at all.

2. Risky/unnecessary fix — cleanish.py

```python
import logging

def add(a, b):
    logging.info("Adding two numbers")
    return a + b
```

Gemini rewrote this clean 5-line function into 20+ lines. The original function had no bugs. The fix changed observable behavior. This is a false positive fix on code that should have been left alone.

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- What did Gemini detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

---
The heuristic model using regex to match keywords with substitute code. Gemini mode sends the entire code to the LLM asking for issues through JSON. The Gemini mode is much more accurate while the heuristic mode is quicker. Heursitcs caught syntax issues that too the ones that were hardcoded. Gemini was able to catch semantic mistakes. The proposed fixes with heuristics would not makes sense at times while Gemini suggested correct fixes, but sometimes complicated ones. The risk scorer did not agree with my intuition,since a higher score was good. 

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- What trigger would you add?
- Where would you implement it (risk_assessor vs agent workflow vs UI)?
- What message should the tool show the user?

---

If the proposed fix is significantly longer than the original length of the file. I would say if the difference of lines is proportianally larger than some factor then refuse to auto-fix. I would implement it in the agent workflow. The message should say something along the lines of proposed fixes seem to be much more complicated size wise than original . 

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

Examples:

- A better output format and parsing strategy
- A new guardrail rule + test
- A more careful “minimal diff” policy
- Better detection of changes that alter behavior

Write your idea clearly and briefly.

---

I would probably first change the scoring logic, meaning higher score is bad and lower score is better. I would also tinker with the scoring by reducing the magnitudes score updates to minimize bias. 