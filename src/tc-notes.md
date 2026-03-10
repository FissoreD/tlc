This patch integrates the **Elpi-based typeclass solver** in place of Coq’s default solver and adapts several TLC definitions to work smoothly with it.

### 1. Override Coq’s typeclass solver

The Elpi solver is imported and installed globally:

* **`src/LibEqual.v` (lines 9–10)**

  ```coq
  From elpi.apps Require Export tc.
  Elpi TC Solver Override TC.Solver All.
  ```

This ensures all subsequent typeclass resolution goes through the Elpi solver.

Additionally, the solver is configured in:

* **`src/LibList.v` (lines 16–19)**

  ```coq
  Elpi TC.AddAllClasses.
  Elpi TC Solver Override TC.Solver All.
  Elpi TC Solver Override TC.Solver Rm Proper ProperProxy subrelation RelationClasses.Reflexive.
  ```

This registers all classes with the solver and removes a few problematic ones from automatic resolution to avoid undesirable search behavior.

---

### 2. Avoid unfolding issues in aliases

Some type aliases are converted from `Definition` to `Notation` so that typeclass resolution operates directly on the underlying function types without introducing extra reducibility steps.

* **`src/LibContainerDemos.v` (line 107)**

  ```diff
  -Definition map (A B : Type) := A -> option B.
  +Notation map := LibMap.map.
  ```

* **`src/LibRelation.v` (line 19)**

  ```diff
  -Definition binary (A : Type) := A -> A -> Prop.
  +Notation binary A := (A -> A -> Prop).
  ```

---

### 3. Add Elpi rules for extensionality

Since several TLC structures are represented as functions, explicit Elpi clauses are added to forward `tc-Extensionality` goals to the underlying function type.

* **`src/LibMap.v` (lines 23–27)**

  ```coq
  Elpi Accumulate TC.Solver lp:{{
    tc-TLC.LibEqual.tc-Extensionality {{map lp:A lp:B}} R :-
      tc-TLC.LibEqual.tc-Extensionality {{lp:A -> option lp:B}} R.
  }}.
  ```

* **`src/LibMultiset.v` (lines 25–29)**

  ```coq
  Elpi Accumulate TC.Solver lp:{{
    tc-TLC.LibEqual.tc-Extensionality {{multiset lp:A}} R :-
      tc-TLC.LibEqual.tc-Extensionality {{lp:A -> Type}} R.
  }}.
  ```

* **`src/LibSet.v` (lines 241–245)**

  ```coq
  Elpi Accumulate TC.Solver lp:{{
    tc-TLC.LibEqual.tc-Extensionality {{set lp:A}} R :-
      tc-TLC.LibEqual.tc-Extensionality {{lp:A -> Prop}} R.
  }}.
  ```

These rules preserve the intended extensional reasoning for these container abstractions.

---

### 4. Replace `Hint Extern` with explicit instances

Some hint-based registrations are replaced with explicit instances, which interact more predictably with the Elpi solver.

* **`src/LibMultiset.v` (lines 66–70)**

  ```diff
  -#[global]
  -Hint Extern 1 (BagIn _ (multiset _)) => apply in_inst : typeclass_instances.
  +#[global] Existing Instance in_inst.
  ```

* **`src/LibSet.v` (lines 111–113)**

  ```diff
  -Hint Extern 1 (BagIn _ (set _)) => apply in_inst : typeclass_instances.
  +Existing Instance in_inst.
  ```

---

### 5. Minor compatibility update

* **`src/LibEqual.v` (line 868)**

  ```diff
  -From Coq.Logic Require Import JMeq.
  +From Stdlib.Logic Require Import JMeq.
  ```

This aligns the import with the modern Coq standard library layout.

---

Overall, these changes ensure that TLC’s abstractions (`set`, `map`, `multiset`, relations, etc.) remain compatible with the **Elpi typeclass resolution engine**, while preserving the intended extensional reasoning principles and improving predictability of instance resolution.
