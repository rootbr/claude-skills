---
name: clean-code
description: "Cleans up recently modified code: simplify, reduce nesting, remove redundancy, enforce project style. Use this skill proactively whenever you write, refactor, or modify code — not just when the user explicitly asks. Trigger on: any code change during a feature implementation, bug fix, or refactoring session; phrases like 'clean this up', 'simplify', 'refactor'. The goal is to leave every piece of code you touch cleaner than you found it."
---

# Clean Code

## MAIN RULE: MAXIMUM SIMPLICITY

**Simple solution is ALWAYS better than complex. If you can make it simpler — do it simpler.**

---

## 1. FUNCTIONS

- **SMALL** — maximum 20 lines, better 5-10
- **ONE TASK** — function does exactly one thing
- **0-2 ARGUMENTS** — ideal 0, acceptable 2, maximum 3
- **NO FLAGS** — boolean argument = make two functions
- **NO SIDE EFFECTS** — don't change what you don't promise in the name
- **DESCRIPTIVE NAME** — function name fully explains what it does

---

## 2. NAMES

- **DESCRIPTIVE** — name explains everything without comments
- **PRONOUNCEABLE** — not `genymdhms`, but `generation_timestamp`
- **NO ENCODINGS** — not `str_name`, `m_value`, `user_obj`
- **NO MAGIC NUMBERS** — only named constants
- **SEARCHABLE** — not `7` or `86400`, but `DAYS_IN_WEEK`, `SECONDS_PER_DAY`

---

## 3. CONDITIONS

- **EXTRACT TO VARIABLES** — turn complex conditions into understandable variables
- **AVOID NEGATIONS** — `if is_valid` instead of `if not is_not_valid`
- **READS LIKE TEXT** — `can_drive = is_adult and has_license`
- **NO NESTING** — maximum 2 levels of if nesting

---

## 4. POLYMORPHISM INSTEAD OF IF/ELSE

- **DON'T USE IF/ELSE CHAINS** — use classes and polymorphism
- **AVOID SWITCH/CASE** — create class hierarchy
- **OPEN-CLOSED** — easy to add new, don't change old

---

## 5. COMMENTS

### Write code that explains itself

### Comments ONLY for:
- **Why** (not what!) — explain business decision
- **Warnings** — important gotchas
- **TODO** — temporary notes

### NEVER:
- **Don't comment out code** — delete it
- **Don't duplicate code** — `i++; // increment i`
- **Don't write obvious** — code says it all
- **Don't leave commented code** — delete it

---

## 6. CODE STRUCTURE

- **VARIABLES CLOSE TO USAGE** — not at the beginning of function
- **RELATED CODE TOGETHER** — separate groups with blank lines
- **TOP TO BOTTOM** — public methods on top, private below
- **LINES < 120 CHARACTERS**
- **ONE ABSTRACTION LEVEL** — don't mix high and low level
- **DEPENDENT FUNCTIONS NEARBY** — called function right after calling one

---

## 7. CLASSES

- **SMALL** — single responsibility (Single Responsibility Principle)
- **FEW FIELDS** — 2-5 instance variables
- **HIDE DATA** — only through methods, not directly
- **HIGH COHESION** — methods use class fields
- **NOT HYBRIDS** — either object (behavior) or data structure

---

## 8. AVOID (CODE SMELLS)

Remove immediately:

1. **Duplication** — DRY (Don't Repeat Yourself)
2. **Dead code** — delete unused
3. **Magic numbers** — name all constants
4. **Long functions** — split into small ones
5. **Many parameters** — group into object
6. **Flag arguments** — make separate functions
7. **Deep nesting** — simplify conditions
8. **Unclear names** — rename

---

## 9. DESIGN PRINCIPLES

- **CONFIGURATION ON TOP** — constants and settings at high level
- **DEPENDENCY INJECTION** — dependencies passed from outside
- **LAW OF DEMETER** — object knows only direct dependencies
- **SEPARATE THREADS** — multithreaded code separately
- **AVOID OVER-CONFIGURABILITY** — don't make everything configurable

---

## CHECKLIST BEFORE COMMIT

- [ ] Function < 20 lines?
- [ ] Function does ONE thing?
- [ ] Arguments <= 2?
- [ ] Names explain everything?
- [ ] No magic numbers?
- [ ] Conditions in variables?
- [ ] No duplication?
- [ ] Can be simplified?
- [ ] Code reads like prose?

---

## GOLDEN RULE

> **"Any fool can write code that a computer can understand.  
> Good programmers write code that humans can understand."**

**If in doubt — choose the simpler option.**
