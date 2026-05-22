# Circ 1.5 → Maude 3.5.1 Refactoring Report

**Project:** Refactoring `circ.maude` (Circ prover v1.5, originally targeting Maude 2.4) to run
correctly under Maude 3.5.1.

**Outcome:** All 25 example files in the `examples/` directory prove successfully with no errors.

---

## Background

Circ is a theorem prover for coinductive and inductive properties of programs written in Maude.
The original `circ.maude` (v1.5, 2010) depended on Full Maude 2.4 APIs, module sorts, and
bubble-parsing conventions that changed significantly in Maude 3.x. This report documents every
modification made to achieve compatibility, explaining the root cause and the fix for each.

The two primary targets for modification are:
- **`circ.maude`** — the prover itself
- **`examples/...`** — example specification files (Maude 2.x syntax that needed updating)

A secondary target is:
- **`full-maude351.maude`** — Francisco Durán's Full Maude for Maude 3.5.1 (one minor addition)

---

## Modification 1: Import rename in `fmod CONTAINERS`

**File:** `circ.maude`, `fmod CONTAINERS`

**Change:**
```maude
--- Before:
protecting VIEW-MAP-SET-APPL-ON-TERM .

--- After:
protecting FM-RENAMING-SET-APPL-ON-TERM .
```

**Motivation:** In Full Maude 3.5.1, the module providing `applySubst`, `setOps`, `addEqs`, and
`META-LEVEL` access was renamed from `VIEW-MAP-SET-APPL-ON-TERM` to
`FM-RENAMING-SET-APPL-ON-TERM`. Without this rename, circ.maude would not load.

---

## Modification 2: `@DeclList@` vs `@FDeclList@` sort in `fmod CIRC-SPEC-LANG-SORTS`

**File:** `circ.maude`, `fmod CIRC-SPEC-LANG-SORTS`

**Change:**
```maude
--- Before:
subsort @CircDeclX@ < @CDeclList@ .

--- After:
subsort @CircDeclX@ < @DeclList@ .
```

(for each circ-specific declaration sort `@CircDeclX@`)

**Motivation:** In Full Maude 3.5.1, the top-level bubble sort for module declarations changed
from `@CDeclList@` (Maude 2.x) to `@DeclList@` (Maude 3.x). Circ's custom declaration sorts
(`@GrdEqDeclList@`, `@SimpRlDeclList@`, etc.) must be subsorts of this top-level sort so they
can appear inside `theory ... endtheory` declarations.

---

## Modification 3: Bubble sort names in `fmod CIRC-SPEC-LANG-SIGN`

**File:** `circ.maude`, `fmod CIRC-SPEC-LANG-SIGN`

**Change:** Switched from the generic `@Bubble@` to specialized bubble sorts: `@RBubble@`,
`@EqLBubble@`, `@RlLBubble@`, `@RCBubble@`; added `including FM-STATEMENTS`.

**Motivation:** Full Maude 3.5.1 uses distinct bubble sorts for different syntactic positions
(left-hand-side of equations, right-hand-side of rules, conditions, etc.). The old Maude 2.x
code used generic `@Bubble@` for all positions. Using the specialized sorts allows Full Maude's
bubble-solving infrastructure to distinguish syntactic roles correctly, and including
`FM-STATEMENTS` brings in the required grammar rules.

At the meta-level, this also means bubble term constructors change from `'bubble[T]` to
`'rllbubble[T]`, `'rcbubble[T]`, `'rbubble[T]`, `'eqlbubble[T]` respectively, and all
pattern-matches on bubble terms in `circ.maude`'s rules were updated accordingly.

---

## Modification 4: Grammar imports in `fmod CIRC-CMD-LANG-SIGN`

**File:** `circ.maude`, `fmod CIRC-CMD-LANG-SIGN`

**Change:** Added `including FM-COMMANDS . including FM-SIGNATURE .`

**Motivation:** In Maude 3.5.1, the Full Maude command and signature grammar is split into
dedicated modules. Circ's command language must import these to participate in Full Maude's
parsing pipeline for user commands like `(theory ... endtheory)` and `(coinduction .)`.

---

## Modification 5: Removal of non-existent import in `fmod META-CIRC-LANG-SIGN`

**File:** `circ.maude`, `fmod META-CIRC-LANG-SIGN`

**Change:**
```maude
--- Before:
protecting UNIT .
protecting META-FULL-MAUDE-SIGN .

--- After:
protecting META-FULL-MAUDE-SIGN .
```

**Motivation:** The `UNIT` module does not exist in Maude 3.5.1. Only `META-FULL-MAUDE-SIGN`
is needed to bring in the meta-level grammar for Full Maude units.

---

## Modification 6: Class identifier type in `fmod CIRC-DATABASE-HANDLING`

**File:** `circ.maude`, `fmod CIRC-DATABASE-HANDLING` and `mod CIRC-PROVER`

**Change:**
```maude
--- Before:
CIRCDataBase < DatabaseClass .
var X@Database : DatabaseClass .

--- After:
CIRCDataBase < Cid .
var X@Database : Cid .
```

**Motivation:** In Maude 3.5.1's object-oriented module system, class identifiers have sort `Cid`
(Class ID), not `DatabaseClass`. Using `DatabaseClass` caused a sort error; switching to `Cid`
restores correct object matching in the `mod CIRC-PROVER` rules.

---

## Modification 7: `getHeader` equations for CTheory

**File:** `circ.maude`, `mod CIRC-UNIT`

**Change:** Added two equations:
```maude
eq getHeader(theory ME is IL sorts SS . SSDS OPDS MAS EqS RlS DDL SCDL SRDL ESDL GEDL CEDL endtheory) = ME .
eq getHeader(theory ME{PDL} is IL sorts SS . SSDS OPDS MAS EqS RlS DDL SCDL SRDL ESDL GEDL CEDL endtheory) = ME{PDL} .
```

**Motivation:** Full Maude's `evalPreModule3` calls `getHeader(PU)` to determine the module name
for inserting flat and internal modules into the database. The `getHeader` function (from
`FM-UNIT`) had no equation for circ's custom `theory ... endtheory` constructor. Without it,
`evalPreModule3` produced a stuck term, causing `fillInternal` to fail silently and the
`[parseTheory]` rule to never fire — resulting in completely blank output when the user typed
`(theory ... endtheory)`.

The fix adds the missing `getHeader` equations so the module name is correctly extracted and all
database insertions succeed.

---

## Modification 8: `addCoFreezingSorts` — remove equations with undeclared variables

**File:** `circ.maude`, `fmod EQ-MANAGER`

**Change:** Removed four pattern-matching equations for module types (`omod`, `oth`, `theory`,
`unitError`) that used variables not declared in the module's `var` section. Added a single
catch-all:
```maude
eq addCoFreezingSorts(M) = M [owise] .
```

**Motivation:** In Maude 3.5.1, using an undeclared variable in an equation's left-hand side is
a hard error (the equation is rejected at load time), whereas Maude 2.x was more permissive.
The `omod`, `oth`, and `theory` module types either do not need co-freezing sorts (circ's
coinduction only operates on functional/system modules) or are not supported by this function.
The `[owise]` equation safely passes these types through unchanged.

---

## Modification 9: VDS added to `solveBubbles` for `SimpRlDeclList` (srl/csrl)

**File:** `circ.maude`, `mod CIRC-UNIT`, `solveBubbles` for `SimpRlDeclList`

**Change:** In all four equations handling `SimpRlDeclList` (two success cases and two error
cases for `csrl`/`srl`), changed the `metaParse` calls from 3-argument to 4-argument form by
adding `VDS`:

```maude
--- Before:
/\ RP  := metaParse(M'', '`( QL '`), '@Condition@)
/\ RP1 := metaParse(M'', '`( QL1 '`), '@Condition@)
/\ RP2 := metaParse(M'', '`( QL2 '`), '@Condition@)

--- After:
/\ RP  := metaParse(M'', VDS, '`( QL '`), '@Condition@)
/\ RP1 := metaParse(M'', VDS, '`( QL1 '`), '@Condition@)
/\ RP2 := metaParse(M'', VDS, '`( QL2 '`), '@Condition@)
```

**Motivation:** In Maude 3.5.1, the 3-argument `metaParse(M, QIL, Sort)` cannot resolve
variable names that appear without explicit sort annotations (such as `E1`, `E2`, `F1`, `F2` in
`csrl` LHS and RHS bubbles). Without the variable declarations (`VDS`), `metaParse` returns
`noParse` for any expression containing unqualified variable names. This caused the entire
`solveBubbles` condition to fail, which propagated as `srlError` → `unitError` → the flat module
was never inserted into the database → `[initialize]` could not fire.

The 4-argument `metaParse(M, VDS, QIL, Sort)` uses the variable declarations stored in `VDS`
(retrieved during module processing from the Full Maude database) to resolve variable names
correctly.

---

## Modification 10: VDS added to `solveBubbles` for `GrdEqDeclList` (geq)

**File:** `circ.maude`, `mod CIRC-UNIT`, `solveBubbles` for `GrdEqDeclList`

**Change:** Same 3-arg → 4-arg `metaParse` upgrade as Modification 9, applied to the four
`metaParse` calls in the `GrdEqDeclList` equations (2 calls in the success case, 2 in the error
case):

```maude
--- Before:
/\ RP  := metaParse(M'', '@wrapper '`( QL '`), '@@@)
/\ RP' := metaParse(M'', '@wrapper '`( QL' '`), '@@@)

--- After:
/\ RP  := metaParse(M'', VDS, '@wrapper '`( QL '`), '@@@)
/\ RP' := metaParse(M'', VDS, '@wrapper '`( QL' '`), '@@@)
```

**Motivation:** Same root cause as Modification 9. The `geq` (guarded equation) declarations
use variables in LHS and RHS expressions, e.g.:
```maude
geq hd(merge(S1, S2)) =
    hd(S1) if hd(S1) < hd(S2) = true []
    ...
```
Without VDS, `S1` and `S2` could not be resolved as Stream variables, causing parse failure.

---

## Modification 11: VDS added to `solveBubbles` for `CaseEqDeclList` (cases)

**File:** `circ.maude`, `mod CIRC-UNIT`, `solveBubbles` for `CaseEqDeclList`

**Change:** Same 3-arg → 4-arg `metaParse` upgrade applied to the four `metaParse` calls in the
`CaseEqDeclList` equations.

**Motivation:** Same root cause as Modifications 9 and 10. `cases` declarations also use
variable names in their pattern and condition expressions.

---

## Modification 12: `eMetaPrettyPrint` → `eMetaPrettyPrintEq`

**File:** `circ.maude`, approximately line 4613

**Change:**
```maude
--- Before:
eMetaPrettyPrint(M, EqS)

--- After:
eMetaPrettyPrintEq(M, EqS)
```

**Motivation:** In Full Maude 3.5.1, the function for pretty-printing equation sets was renamed
from `eMetaPrettyPrint` to `eMetaPrettyPrintEq`. Using the old name caused a missing-equation
error (stuck term), producing garbled or absent output when printing proven properties.

---

## Modification 13: `full-maude351.maude` — Add `pr LOOP-MODE . pr LEXICAL .`

**File:** `full-maude351.maude`, `fmod FULL-MAUDE`

**Change:** Added two imports at the top of the `FULL-MAUDE` module:
```maude
pr LOOP-MODE .
pr LEXICAL .
```

**Motivation:** Circ uses Maude's loop mode (`[in, out, state]` interface) for its
read-eval-print loop. In Maude 3.5.1, `LOOP-MODE` and `LEXICAL` are not transitively imported
by `FULL-MAUDE` by default. Without them, the `[in]` and `[out]` rules in `mod CIRC-INTERFACE`
could not fire, so no user input was processed after startup.

---

## Modification 14 (example file): Explicit parentheses in CFP-EX2 and CFP-EX3 `csrl` conditions

**File:** `examples/coinduction/processes/context-free-processes.maude`

**Change:** In the CFP-EX2 theory, added explicit parentheses around Bool-valued sub-conditions
in two `csrl` declarations:

```maude
--- Before:
csrl   (E1 + E2) = F1
    => (E1 = F1)
    if (  | E1 | = | F1 |
        /\ | E2 | == | F1 | = false
        /\ noPlus(F1) = true
       )
  .

--- After:
csrl   (E1 + E2) = F1
    => (E1 = F1)
    if (  | E1 | = | F1 |
        /\ (| E2 | == | F1 | = false)
        /\ (noPlus(F1) = true)
       )
  .
```

(Same change applied to the third `csrl` in CFP-EX2, and the single `csrl` in CFP-EX3.)

**Motivation:** This is a parsing ambiguity introduced by the interaction of two features in
Maude 3.5.1's meta-level parser:

1. **`Bool < @Condition@` subsort**: `addInfoConds` (Full Maude's condition augmentation
   function) adds a subsort relation `Bool < @Condition@`, allowing Boolean terms to appear
   directly in conditions.

2. **`gather('& '&)` on `_=_`**: The condition equality operator is declared as
   `op '_=_ : Universal Universal -> @Condition@ [gather('& '&) prec(71)]`. The `gather('& '&)`
   attribute means it can gather arguments of *any* precedence (there is no upper bound).

Together, these two features mean that an expression like:

```
| E2 | == | F1 | = false
```

has two valid parses:
- **Intended:** `(| E2 | == | F1 |) = false`  — where `_==_` compares two Int? values, then `= false` is a condition equality
- **Alternative:** `| E2 | == (| F1 | = false)` — where `| F1 | = false` is an `@Condition@` (via `Bool < @Condition@`), then `_==_` applies with `gather('& '&)` gathering it

In Maude 2.x, this ambiguity either did not arise (different parse infrastructure) or was
resolved differently. In Maude 3.5.1, `metaParse` returns `ambiguity(...)` instead of a
`ResultPair`, causing the condition `RP :: ResultPair` to fail, which cascaded:
`srlError` → `unitError` → flat module not stored → `[initialize]` never fired.

The fix wraps each Bool sub-condition in explicit parentheses, making it syntactically
unambiguous which parse is intended.

---

## Modification 15 (example file): natstream.maude `add cgoal` condition

**File:** `examples/coinduction/natstream/natstream.maude`

**Change:**
```maude
--- Before:
(add cgoal toBits(merge(S1:Stream, S2:Stream)) = ones if
     isSorted(S1:Stream) = true /\
     isSorted(S2:Stream) = true .)

--- After (this session):
(add cgoal toBits(merge(S1:Stream, S2:Stream)) = ones if
     (isSorted(S1:Stream) = true) /\
     (isSorted(S2:Stream) = true) .)
```

Note: the LHS/RHS had already been partially updated in a prior session to use
`S1:Stream`/`S2:Stream` (Maude 3.5.1 tokenizes `S1:Stream` as three tokens `S1`, `:`,
`Stream`). This session focused on fixing the condition.

**Motivation (two sub-issues):**

**Sub-issue A — Variable name resolution without sort annotation:**
In Maude 3.5.1, `metaParse(M, QIL, Sort)` (3-argument form, used by `parseCondition`) can
recognize the three-token sequence `NAME : SORT` as a variable term IF `SORT` is a sort in
module `M`. Without the explicit `:Stream` annotation, the single token `S1` is not identifiable
as a Stream variable — no parse exists, and an "Error: No parse for ..." message is produced.

This is the same reason all `add goal`/`add cgoal` commands in other example files already use
`N:Nat`, `L:List`, etc.: Maude 3.5.1 requires the three-token `NAME : SORT` pattern for
variable annotation.

**Sub-issue B — Bool/`@Condition@` conjunction ambiguity:**
The expression `isSorted(S1:Stream) = true /\ isSorted(S2:Stream) = true` is ambiguous for the
same reason as Modification 14: the `_=_` operator has `gather('& '&)` so it can consume its
right argument at any precedence, and `Bool < @Condition@` makes the conjunction operator
`_/\_` applicable in two ways:

- Parse 1 (intended): `(isSorted(S) = true) /\ (isSorted(S') = true)` — the `_/\_` for
  `@Condition@` arguments
- Parse 2 (spurious): `isSorted(S) = (true /\ isSorted(S') = true)` — where `true /\ isSorted(S')` is `Bool`, then `= ...` is `@Condition@`

Adding explicit parentheses around each equality condition eliminates the ambiguity: the args to
`/\` are clearly of sort `@Condition@` (result type of `_=_`), and only `_/\_ : @Condition@
@Condition@ -> @Condition@` applies.

---

## Summary Table

| # | File | What Changed | Root Cause |
|---|------|-------------|-----------|
| 1 | `circ.maude` | Import rename: `VIEW-MAP-SET-APPL-ON-TERM` → `FM-RENAMING-SET-APPL-ON-TERM` | Module renamed in Full Maude 3.5.1 |
| 2 | `circ.maude` | Bubble sort hierarchy: `@CDeclList@` → `@DeclList@` | Hierarchy changed in Full Maude 3.5.1 |
| 3 | `circ.maude` | Specialized bubble sorts `@RBubble@`, `@RlLBubble@`, `@RCBubble@`, `@EqLBubble@`; add `FM-STATEMENTS` | Generic `@Bubble@` split into specialized sorts in Full Maude 3.5.1 |
| 4 | `circ.maude` | Add `FM-COMMANDS`, `FM-SIGNATURE` imports | Required grammar modules in Full Maude 3.5.1 |
| 5 | `circ.maude` | Remove non-existent `protecting UNIT .` | `UNIT` module does not exist in Maude 3.5.1 |
| 6 | `circ.maude` | `DatabaseClass` → `Cid` | OO class ID sort renamed in Maude 3.5.1 |
| 7 | `circ.maude` | Add `getHeader` for CTheory | Missing equation caused `evalPreModule3` to produce stuck term → silent failure of `[parseTheory]` |
| 8 | `circ.maude` | `addCoFreezingSorts`: remove equations with undeclared vars, add `[owise]` | Maude 3.5.1 rejects undeclared variables in equations (hard error) |
| 9 | `circ.maude` | `solveBubbles`/`SimpRlDeclList`: 3-arg → 4-arg `metaParse` with VDS | Maude 3.5.1 3-arg `metaParse` cannot resolve unqualified variable names → srl/csrl declarations broken |
| 10 | `circ.maude` | `solveBubbles`/`GrdEqDeclList`: 3-arg → 4-arg `metaParse` with VDS | Same as #9; `geq` declarations broken |
| 11 | `circ.maude` | `solveBubbles`/`CaseEqDeclList`: 3-arg → 4-arg `metaParse` with VDS | Same as #9; `cases` declarations broken |
| 12 | `circ.maude` | `eMetaPrettyPrint` → `eMetaPrettyPrintEq` | Function renamed in Full Maude 3.5.1 |
| 13 | `full-maude351.maude` | Add `pr LOOP-MODE . pr LEXICAL .` | Required for Circ's read-eval-print loop interface |
| 14 | `context-free-processes.maude` | Explicit parens around Bool sub-conditions in `csrl` IF clauses | `Bool < @Condition@` + `gather('& '&)` on `_=_` creates ambiguity in Maude 3.5.1 |
| 15 | `natstream.maude` | Sort annotations `S1:Stream` + parens in `add cgoal` condition | (A) Maude 3.5.1 requires `NAME : SORT` 3-token form; (B) same Bool/Condition ambiguity as #14 |

---

## Key Maude 3.5.1 Compatibility Insights

**Insight 1 — Variable annotation format:** In Maude 3.5.1, the token `N:Nat` is lexed as three
separate tokens `N`, `:`, `Nat`. The 3-arg `metaParse` recognizes the three-token pattern
`NAME : SORT` as a variable annotation (if `SORT` is a sort in the module). User-written
specifications must use `N:Nat` (or with spaces: `N : Nat`) rather than expecting single-token
variable names to be resolved automatically.

**Insight 2 — VDS required for 4-arg `metaParse` in bubble solving:** When circ-specific
declarations (`geq`, `srl`, `csrl`, `cases`) are parsed during module processing, the
expressions in their LHS/RHS/condition bubbles may contain variable names without sort
annotations (e.g., `E1`, `E2` declared by `var E1 E2 : Pexp .`). These require the 4-arg
`metaParse(M, VDS, QIL, Sort)` form, where `VDS` is retrieved via `getVars(ModuleName, DB)`.
The 3-arg form silently fails for such expressions in Maude 3.5.1.

**Insight 3 — Bool/Condition ambiguity from `addInfoConds`:** Full Maude's `addInfoConds`
function adds `subsort Bool < @Condition@` when `Bool` is present, enabling Boolean terms in
conditions. Combined with `_=_ : Universal Universal -> @Condition@ [gather('& '&)]` (which
can absorb arbitrarily-precedenced right arguments), any conjunction of the form
`boolExpr = true /\ boolExpr' = true` becomes ambiguous. Explicit parentheses around each
Boolean equality eliminate the ambiguity and are required in Maude 3.5.1.
