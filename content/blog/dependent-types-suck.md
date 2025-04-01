+++
title = "Dependent types suck actually"
date = 2025-04-01
+++

STOP DOING DEPENDENT TYPES
- Types were not supposed to be given computation!
- YEARS of typechecking but NO real world use for going higher than GADTS
- Wanted to go higher anyway for a laugh? We had a tool for that: It was called RUNTIME VERIFICATION
- "Yes please give me `Id A x x` of something. Please give me `Vec (m + n) A` of it." -- Statements dreamed up by the UTTERLY INSANE 

Look at what computer scientists have been doing, with all the languages and typecheckers we built for them: 
(This is REAL dependent types, done by REAL computer scientists)

```agda
sq2 : ◀.F-map-iso (◀.F-map-iso (ρ≅ Iso⁻¹)) ∙Iso α≅
    ≡ (α≅ ∙Iso α≅) ∙Iso ▶.F-map-iso (λ≅ Iso⁻¹)
sq2 = ≅-path $
    α→ _ _ _ ∘ ((ρ← ⊗₁ id) ⊗₁ id)    ≡
    (ρ← ⊗₁ ⌜ id ⊗₁ id ⌝) ∘ α→ _ _ _  ≡
    (ρ← ⊗₁ id) ∘ α→ _ _ _            ≡˘
    (id ⊗₁ λ←) ∘ α→ _ _ _ ∘ α→ _ _ _ ∎
```
```lean
theorem integral_re {f : X → 𝕜} (hf : Integrable f μ) :
    ∫ x, RCLike.re (f x) ∂μ = RCLike.re (∫ x, f x ∂μ) :=
  (@RCLike.reCLM 𝕜 _).integral_comp_comm hf
```
```coq
Lemma next_spoke_ring (y : G) : y \in r -> next r y = face (spoke y).
Proof.
move=> ry; have [cycFp Up] := andP ucycFp.
rewrite -{1}[y]nodeK (next_map inj_spoke) ?rev_uniq // (next_rev Up).
rewrite -[prev p _]faceK /spoke ee -(canRL nodeK (nnn y)).
by rewrite (eqP (prev_cycle cycFp _)) // -Ep -Er.
Qed.
```

"Hello I would like `(p : f ∘ g ≡ h ∘ k)` apples please"

they have played us for absolute fools
<br><br><br><br><br><br>
<details>
  <summary>credits</summary>
  
  - Agda credit: <a href="https://1lab.dev/Cat.Displayed.BeckChevalley.html#6773">The excellent 1lab</a>
  
  - Lean credit: <a href="https://github.com/leanprover-community/mathlib4/blob/362bdac45f1c3e9307252e394f8ecfff137c4d85/Mathlib/MeasureTheory/Integral/SetIntegral.lean#L1183-L1185">Mathlib4</a>
  
  - Roqc credit: <a href="https://github.com/rocq-community/fourcolor/blob/master/theories/proof/birkhoff.v">4 color theorem proof</a>
</details>
