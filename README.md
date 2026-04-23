# mathcore-unitalg

Unit algebra engine for the mathcore/thales ecosystem.

Provides:

- Arithmetic on `mathcore_units::UnitExpression`: multiplication, division,
  power, logarithmic composition
- Dimension unification, first-wins system selection, compatibility checks
- Conversion (linear, affine, logarithmic, constant-referenced) emitted
  as symbolic substitutions (not numerical evaluation)
- Canonical form: expand composite units (`N` → `kg·m·s⁻²`), then factor
  back to the most specific catalog entry on output
- Consistency validation: dimension match on `+`/`−`, dimensionless argument
  required for `sin/cos/exp/log/ln`, tokenizer-level prefix disambiguation
  (bare `T` = tesla, `TT` = tera-tesla)

## Scope

Strictly unit algebra. Not a general CAS. Operates only on
`UnitExpression` values (restricted vocabulary from `mathcore-units`:
closed set of `UnitId` atoms; operators restricted to `+, −, ×, /, ^,
log, ln, exp`).

For general symbolic mathematics over full Expression trees, use
[thales](https://github.com/ChrisGVE/thales).

## Relationship to the workspace

Subordinate member of the thales workspace. Called by:

- **mathlex** at Expression+ assembly time to validate unit annotations,
  emit conversion substitutions, and compute the factored output unit.
- **thales** at every algebraic step where units can change: addition
  with dimension check, multiplication and division (unit compose),
  power (unit scale), differentiation (divide by variable unit),
  integration (multiply by variable unit), trigonometric and
  exponential functions (dimensionless-argument enforcement), equation
  solving (preserve dimensional consistency).

Never calls thales. Never depends on `Arc<Expr>`. Always operates on
`UnitExpression`. This type lives outside Arc<Expr>'s computational
domain — it rides as an annotation, updated step-by-step by thales
calls into unitalg.

## Conversion model

Conversions are emitted as symbolic substitutions into the main
Expression, never as numerical evaluation. Example: when `1 m + 1 ft`
is seen, unitalg emits the substitution `1 ft ↦ 0.3048 m` and the
Expression tree is rewritten accordingly. The constant `0.3048` stays
symbolic so downstream consumers (thales, mathlex-eval) can recognize
its origin.

For unit conversions that depend on a physical constant (e.g., solar
mass as a mass unit), the conversion factor references the constant
by `ConstantId` — never by value. mathcore-constants owns the numeric
values.

## Status

**Pre-specification.** The authoritative spec (UA-1…UA-N) is being drafted
and will land as `SPECIFICATION.md` once `mathcore-units` freezes its
own spec. Consumers should track this repo or the thales workspace for
release readiness.

## Install

Not yet published to crates.io.

## License

MIT. See `LICENSE`.
