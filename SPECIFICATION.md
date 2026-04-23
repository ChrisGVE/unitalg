# unitalg — Specification

**Version:** draft-1 (2026-04-22)
**Status:** draft, pre-freeze
**Target release:** v0.1.0
**Crate kind:** algebra engine (operations on `mathcore_units` types; no catalog data, no Arc<Expr>)

This document specifies the authoritative contract for `unitalg`. It
depends on `mathcore-units` (spec version draft-1, amended 2026-04-22)
and must be read alongside that document. All section cross-references
of the form `§ N.M of mathcore-units` refer to the mathcore-units
specification at that version. The amendment updated § 2.6
(`Conversion::Logarithmic` — `LogBase` enum, `reference: Option<UnitId>`,
`reference_scale: Rational`), § 4 (ConstantSpec unit-field contract;
AU/LightYear/Parsec dual appearance documented), § 5.6 (logarithmic
catalog table rewritten with new shape), and § 6.2 (alias table now
includes `yr`, `min`, `hr`, `day`).

Once accepted, the public API shapes, error enum variants, and
algorithmic invariants described here are frozen for the v0.x series.
Additions are permitted in minor versions; removals or renames of
public items require a major version bump coordinated with thales and
mathlex.

---

## 1. Scope and non-goals

### 1.1 In scope

- Arithmetic operations on `mathcore_units::UnitExpression`: multiply,
  divide, power (integer), power (rational), additive unification.
- Structural operations: expand (composite → base units), factor (base
  units → most-specific named unit), canonical-form normalization.
- Dimension computation from a `UnitExpression` tree.
- Unit compatibility check (same dimension predicate).
- System selection heuristic (first-wins with fallback, per § 5).
- Symbolic conversion emission: produces an `Expression` fragment
  representing the conversion factor; does not evaluate numerically.
- Transcendental-argument enforcement (sin/cos/exp/log/ln must receive
  a dimensionless or angle-dimensioned unit).
- Token parsing: wraps mathcore-units alias + prefix tables and
  enforces collision-resolution rules from § 6.3 of mathcore-units.
- Error types for all failure modes.
- `no_std + alloc` support.

### 1.2 Out of scope

- Numeric evaluation of conversion factors — this is mathlex-eval's
  and thales's responsibility.
- `Arc<Expr>` interop — unitalg has no dependency on thales and never
  touches the Arc<Expr> internal representation.
- Catalog data ownership — the unit catalog, alias tables, and prefix
  tables all live in mathcore-units. unitalg queries them, does not
  duplicate them.
- General symbolic algebra over `Expression` trees — any capability
  that goes beyond unit-annotated arithmetic belongs in thales.
- Serde serialization of `UnitExpression` — defined in mathcore-units.
  unitalg may serde its own error variants, but introduces no new
  serializable structural types.
- Currency units — rejected per mathcore-units (non-constant conversion
  rates).
- Infotech units — rejected per mathcore-units.

---

## 2. Supporting types

### 2.1 `AdditiveResult`

Returned by `combine_additive`. Carries the unified output unit plus
the conversion factors (as `Expression` fragments) for each operand so
that thales or mathlex can rewrite the main Expression without numeric
evaluation.

```rust
#[derive(Debug, Clone)]
pub struct AdditiveResult {
    /// The unit to use for the combined result.
    pub unified_unit: UnitExpression,
    /// Factor to multiply the first operand's numeric value by before
    /// combining. Expression::Rational(1, 1) when operands already share
    /// the same scale.
    pub factor_for_u1: Expression,
    /// Factor to multiply the second operand's numeric value by before
    /// combining.
    pub factor_for_u2: Expression,
    /// Additive offset to apply to the first operand after scaling.
    /// Expression::Rational(0, 1) for all non-affine conversions.
    pub offset_for_u1: Expression,
    /// Additive offset to apply to the second operand after scaling.
    pub offset_for_u2: Expression,
}
```

**Affine note:** for temperatures, conversion between Celsius,
Fahrenheit, and Rankine involves both a scale factor and an additive
offset. When one or both operands carry an affine-conversion unit
(DegreeCelsius, DegreeFahrenheit, DegreeRankine), the offset fields are
nonzero symbolic values. The unified unit is the target unit selected by
`choose_system`. Downstream consumers must apply
`value_new = factor * value_old + offset` for each operand before
combining.

### 2.2 Error enums

#### `DimError`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum DimError {
    /// The two operands have incompatible dimensions for an additive
    /// operation (e.g., adding length to mass).
    IncompatibleDimensions {
        lhs: Dimension,
        rhs: Dimension,
    },
    /// A logarithmic-family unit (dB variant) cannot be combined with
    /// a unit whose logarithmic reference differs, unless one is
    /// dimensionless.
    LogarithmicReferenceMismatch {
        lhs_ref: Option<UnitId>,
        rhs_ref: Option<UnitId>,
    },
    /// System selection failed; no system covers all operands.
    SystemSelectionFailed(SysError),
}
```

#### `ConvError`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum ConvError {
    /// Source and target units have different dimensions.
    IncompatibleDimensions {
        from: Dimension,
        to: Dimension,
    },
    /// Logarithmic-to-non-logarithmic or mismatched-reference conversion
    /// is not representable as a simple Expression fragment.
    LogarithmicConversionAmbiguous {
        from_ref: Option<UnitId>,
        to_ref: Option<UnitId>,
    },
    /// The FromConstant mechanism produced a missing constant id
    /// (should not occur if catalog and mathcore-constants are in sync,
    /// but reported to surface catalog bugs).
    MissingConstant(ConstantId),
}
```

#### `SysError`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum SysError {
    /// An explicit target system was supplied but at least one unit
    /// cannot be converted to it.
    TargetSystemIncompatible {
        target: System,
        incompatible_unit: UnitExpression,
    },
    /// No system (tried SI, Imperial, None) can cover all supplied units.
    NoCompatibleSystem,
}
```

#### `ArgError`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum ArgError {
    /// Argument to a transcendental function (sin, cos, exp, log, ln)
    /// carries a non-dimensionless, non-angle unit.
    DimensionedArgument {
        function: &'static str,
        found: Dimension,
    },
}
```

#### `ParseError`

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum ParseError {
    /// Token did not match any alias or prefix-decomposition.
    UnknownToken(String),
    /// Token matched a prefix but the companion atom is absent or invalid.
    PrefixWithoutAtom { prefix: SiPrefix, rest: String },
    /// Multiple valid decompositions exist and the collision table
    /// provides no resolution. Consumers should use a more explicit form.
    AmbiguousToken(String),
    /// The token is known to be reserved or rejected in unit context
    /// (e.g., bare `c`, bare `e`, `rem`).
    ReservedToken(String),
    /// Unit parsed successfully but the unit's PrefixPolicy forbids
    /// the prefix that was applied.
    PrefixNotAllowed {
        prefix: SiPrefix,
        unit: UnitId,
    },
    /// An operator that is syntactically invalid in a unit expression
    /// (e.g., `+`, `-`, `log`, `sin`) was encountered at the given
    /// byte position in the input string.
    InvalidOperator {
        pos: usize,
        found: char,
    },
    /// A `(` was opened but never closed, or a `)` appeared without a
    /// matching `(`. `pos` is the byte offset of the offending
    /// parenthesis.
    UnmatchedParen { pos: usize },
    /// An exponent was found that is neither an integer literal nor a
    /// parenthesised rational `(num/den)`. `pos` is the byte offset of
    /// the `^` operator that introduced the exponent.
    InvalidExponent { pos: usize },
    /// The input string was empty or contained only whitespace.
    EmptyExpression,
}
```

---

## 3. Operations API

Every function listed below is `pub` and lives in the crate root
(`unitalg::<fn>`). All `UnitExpression` arguments are borrowed; return
values are owned.

### 3.1 `multiply`

```rust
pub fn multiply(
    u1: &UnitExpression,
    u2: &UnitExpression,
) -> UnitExpression
```

Computes the product of two unit expressions. Semantics:

- Combine dimension exponents additively (per § 2.2 of mathcore-units).
- Multiply scale factors.
- Result is returned in canonical form (§ 4.1).

This operation is always valid — two `UnitExpression` values can always
be multiplied, even across systems. System information is preserved in
the canonical form atoms; the caller is responsible for system selection
if desired.

**Example:**

```
multiply(Newton, Second)
  → kg·m·s⁻²  ×  s  =  kg·m·s⁻¹   (canonical)
  → (factored) Newton·Second = N·s = kg·m·s⁻¹
```

### 3.2 `divide`

```rust
pub fn divide(
    u1: &UnitExpression,
    u2: &UnitExpression,
) -> UnitExpression
```

Computes the quotient `u1 / u2`. Semantics:

- Combine dimension exponents subtractively (u1 exponent − u2 exponent
  per base).
- Divide scale factors.
- Result is returned in canonical form.

**Example:**

```
divide(Meter, Second)   →   m·s⁻¹
divide(Newton, Meter)   →   kg·s⁻²   (= Pa·m, but canonically kg·s⁻²)
```

### 3.3 `power`

```rust
pub fn power(
    u: &UnitExpression,
    exponent: i32,
) -> UnitExpression
```

Raises a unit expression to an integer power. Semantics:

- Scale each dimension exponent by `exponent`.
- Raise the scale factor to the same power.
- Exponent 0 produces a dimensionless unit (empty Dimension).
- Negative exponents are valid.

**Example:**

```
power(Meter, 2)    →  m²
power(Second, -1)  →  s⁻¹   (= Hertz dimensionally)
power(Newton, 0)   →  (dimensionless, scale 1)
```

### 3.4 `power_rational`

```rust
pub fn power_rational(
    u: &UnitExpression,
    num: i32,
    den: i32,
) -> Result<UnitExpression, DimError>
```

Raises a unit expression to a rational exponent `num/den`. Required for
noise-density contexts such as `V/√Hz` where the exponent is 1/2.

Semantics:

- Each dimension exponent `e` becomes `e * num / den`. This is exact
  only when `e * num` is divisible by `den` for every base in the
  dimension map. When the result would be fractional, return
  `Err(DimError::IncompatibleDimensions)` — fractional dimension
  exponents are not representable in `Dimension` (integer map).
- The scale factor is raised to `num/den`; this may be irrational and
  is stored symbolically as `Literal(Rational(num, den))` in the
  expression scale node.
- `den` must not be zero; caller contract (panics in debug mode).

**Example:**

```
power_rational(Watt, 1, 2)
  → dimension: L¹·M^(1/2)·T^(-3/2)  — NOT representable → Err
power_rational(SquareMeter, 1, 2)
  → dimension: L¹ = Meter             → Ok(Meter) with scale 1
```

### 3.5 `combine_additive`

```rust
pub fn combine_additive(
    u1: &UnitExpression,
    u2: &UnitExpression,
    target: Option<System>,
) -> Result<AdditiveResult, DimError>
```

Unifies two unit expressions for an additive operation (`+` or `−`).
Both operands must be dimension-compatible. Returns the factors and
offsets required to restate both operands in terms of the chosen
unified unit.

The algorithm is:

1. Compute `d1 = dimension(u1)`, `d2 = dimension(u2)`.
2. If `d1 ≠ d2`, return `Err(DimError::IncompatibleDimensions)`.
3. Check logarithmic-reference compatibility (§ 7.3); return
   `Err(DimError::LogarithmicReferenceMismatch)` if they are
   incompatible.
4. Call `choose_system(&[u1.clone(), u2.clone()], target)`. On error,
   return `Err(DimError::SystemSelectionFailed(...))`.
5. Select the target unit from the chosen system's canonical atom for
   the shared dimension.
6. Compute conversion factors and offsets for `u1 → target` and
   `u2 → target` using the `convert` emission algorithm (§ 3.9).
7. Return `AdditiveResult`.

**Example:**

```
combine_additive(Foot, Meter, None)
  d1 = d2 = {Length: 1}  ✓
  choose_system → SI (first-wins; Meter is SI)
  unified_unit = Meter
  factor_for_u1 = 0.3048   (Expression::Literal)
  factor_for_u2 = 1
  offsets = 0
```

### 3.6 `expand`

```rust
pub fn expand(u: &UnitExpression) -> UnitExpression
```

Unfolds composite named units to their base-unit decomposition.
Traverses the `UnitExpression` tree; for each `Atom`, looks up its
`UnitSpec` in the catalog and replaces it with the base-unit product
implied by its `Dimension` map and `Conversion` scale.

Semantics:

- Base units (Meter, Kilogram, Second, Ampere, Kelvin, Mole, Candela,
  Radian) are already expanded; they remain as atoms.
- A named derived unit (e.g., `Newton`) whose UnitSpec has dimension
  `{Length:1, Mass:1, Time:-2}` and `Linear { scale: 1 }` expands to
  `Kilogram · Meter · Second⁻²`.
- A prefixed atom (e.g., `kN = Kilo Newton`) expands by folding the
  prefix scale into the literal coefficient: `1000 · Kilogram · Meter ·
  Second⁻²`.
- Affine-conversion units (Celsius, Fahrenheit) **cannot** be expanded
  by simple dimensional substitution — they carry an additive offset
  that has no unit-expression representation. `expand` on an affine unit
  returns the atom unchanged and annotates it via the `Log` node with
  `reference = Some(UnitId)` to preserve offset context. Callers
  needing numeric conversion use `convert` instead.
- `FromConstant` units (SolarMass, etc.) expand to a `Binary { op: Mul,
  left: Literal(1), right: Atom { Kilogram } }` with the constant
  factor represented as `Expression::Variable(constant_symbol)` in the
  caller's expression, not inside `UnitExpression`. The unit type
  remains mass; the scale is symbolic. `expand` returns the dimensional
  atoms without the constant factor (constant factor lives in the
  `Expression` domain, not the `UnitExpression` domain).
- Logarithmic units are not further expanded — their semantic content is
  the logarithmic relationship, not a dimensional product.

The result is a product of `Atom` nodes whose `UnitId` values are all SI
base units, multiplied by `Literal` scale factors where applicable.

### 3.7 `factor`

```rust
pub fn factor(u: &UnitExpression) -> UnitExpression
```

Folds a base-unit product to the most specific named unit in the
catalog that matches exactly.

Semantics:

1. Compute `d = dimension(u)`.
2. Compute the total scale `s` of `u` relative to the canonical SI
   representation of `d`.
3. Scan the catalog for all `UnitSpec` entries whose `dimension == d`
   and whose `Conversion::Linear { scale }` satisfies `scale == s`.
4. If exactly one match: return `Atom { id: match.id, prefix: None }`.
5. If multiple matches with the same scale and dimension: apply the
   priority order — (a) explicit `priority` tag if present, (b) named
   SI derived (§ 5.2 of mathcore-units) over additional (§ 5.3) over
   legacy (§ 5.4), (c) smallest catalog index as tiebreaker.
6. If no match: return `u` unchanged (canonical form, base-unit
   product).
7. Affine-conversion units and logarithmic units are never returned by
   `factor`; those must be preserved from the input context.

**Example:**

```
factor(expand(Newton))    →   Newton
factor(kg · m · s⁻²)     →   Newton
factor(kg · m² · s⁻²)    →   Joule
factor(kg · s⁻²)          →   (no named match) → kg·s⁻²  (unchanged)
```

### 3.8 `dimension`

```rust
pub fn dimension(u: &UnitExpression) -> Dimension
```

Computes the `Dimension` (sparse exponent map) of a `UnitExpression`
tree. Pure function; no side effects.

Traversal rules:

| Node | Dimension |
|---|---|
| `Atom { id, prefix: _ }` | `catalog::lookup(id).dimension` |
| `Literal(_)` | empty (dimensionless) |
| `Binary { Mul, l, r }` | `dim(l) + dim(r)` (add exponents) |
| `Binary { Div, l, r }` | `dim(l) − dim(r)` (subtract exponents) |
| `Binary { Pow, l, r }` | `dim(l) * r` where r must be Literal integer |
| `Binary { Add, l, r }` | `dim(l)` (additive unification; l and r must be equal) |
| `Binary { Sub, l, r }` | `dim(l)` (same as Add) |
| `Unary { Neg, child }` | `dim(child)` |
| `Unary { Inv, child }` | `−dim(child)` (negate all exponents) |
| `Log { _, _ }` | empty (logarithmic result is dimensionless) |

**Zero-entry removal:** after each operation, entries with exponent 0
are removed per § 2.2 of mathcore-units.

### 3.9 `convert`

```rust
pub fn convert(
    value: Expression,
    from: &UnitExpression,
    to: &UnitExpression,
) -> Result<Expression, ConvError>
```

Emits a symbolic `Expression` fragment that represents `value`
expressed in `to` units. Does not perform numeric evaluation. The
returned `Expression` may contain `Expression::Variable(constant_symbol)`
nodes for `FromConstant` conversions (§ 8).

Pre-condition: `dimension(from) == dimension(to)`. Returns
`Err(ConvError::IncompatibleDimensions)` if not.

Conversion emission by `Conversion` variant of `from` and `to`:

#### Linear → Linear

```
result = value * (scale_from / scale_to)
```

where `scale_from` is `from`'s resolved scale (including any SiPrefix
factor) and `scale_to` is `to`'s resolved scale. Both scales are
`Rational`; the ratio is exact. Emitted as an `Expression` multiply
with a `Rational` literal.

#### Affine → Affine (or Affine → Linear, Linear → Affine)

Each unit's SI canonical value is:
```
x_si = scale * x_unit + offset
```

To convert from → to:
```
x_to = (scale_from * x_from + offset_from - offset_to) / scale_to
```

Emitted as a nested `Expression` multiply + add with `Rational`
literals. When one of the units is Linear, its offset is zero.

Example: Celsius → Fahrenheit
```
x_F = (1 * x_C + 273.15 - (273.15 - 32*5/9)) / (5/9)
     = x_C * (9/5) + 32
```

#### Logarithmic

The `Conversion::Logarithmic` variant (§ 2.6 of mathcore-units) carries
four fields: `reference: Option<UnitId>`, `reference_scale: Rational`,
`base: LogBase`, and `factor: i32`. The canonical inversion formula is:

```
x_si = reference_unit_value · reference_scale · base^(x_unit / factor)
```

where `reference_unit_value` is 1.0 when `reference` is `None`
(dimensionless plain dB or Neper), and the canonical SI value of the
referenced unit otherwise (always 1.0 for catalog units, since they
convert TO canonical SI). `base^(...)` is evaluated via the `LogBase`
variant: `LogBase::Ten` → 10, `LogBase::E` → e, `LogBase::Two` → 2.

The worked-out formulas for each catalog entry:

| UnitId | `reference` | `reference_scale` | `base` | `factor` | SI inversion |
|---|---|---|---|---|---|
| Decibel | None | 1 | Ten | 10 | dimensionless ratio; `10^(x/10)` |
| DecibelMilliwatt | Some(Watt) | 10⁻³ | Ten | 10 | `P_W = 10⁻³ · 10^(x/10)` |
| DecibelWatt | Some(Watt) | 1 | Ten | 10 | `P_W = 1 · 10^(x/10)` |
| DecibelSpl | Some(Pascal) | 2×10⁻⁵ | Ten | 20 | `p_Pa = 2×10⁻⁵ · 10^(x/20)` |
| Neper | None | 1 | E | 1 | `A = e^x` (dimensionless amplitude ratio) |

Logarithmic units cannot be converted to non-logarithmic units via a
simple symbolic factor. `convert` returns
`Err(ConvError::LogarithmicConversionAmbiguous)` when `from` or `to` is
a logarithmic unit and the counterpart is not the same reference. Within
the same logarithmic reference (e.g., dBm → dBm), the conversion is
identity; between different references of the same family (e.g., dBm →
dBW), the offset is derived from their `reference_scale` ratio:

```
x_dBW = x_dBm + factor · log10(reference_scale_dBm / reference_scale_dBW)
       = x_dBm + 10 · log10(10⁻³ / 1)
       = x_dBm − 30
```

This offset is emitted as a symbolic `Expression::Add(value, Rational(-30, 1))`.

The dBm → Watt inversion formula (non-symbolic, for reference):
```
P_W = 10⁻³ · 10^(x_dBm / 10)
```
is intentionally NOT emitted by `convert` — the exponential inversion
crosses from the unit-annotation domain into the expression domain and
belongs in thales. `convert` raises `LogarithmicConversionAmbiguous` in
this case; thales handles it at the Expression level.

#### FromConstant

```
result = value * Expression::Variable(constant_symbol(from.id))
         / Expression::Variable(constant_symbol(to.id))
```

where `constant_symbol` maps a `UnitId` to its `ConstantId`'s variable
name (e.g., `UnitId::SolarMass` → `"M_sun"`). When only one side is a
`FromConstant` unit, the other side contributes a Rational scale.
Downstream resolves the variable via `mathcore_constants::lookup(id)`.

### 3.10 `choose_system`

```rust
pub fn choose_system(
    units: &[UnitExpression],
    target: Option<System>,
) -> Result<System, SysError>
```

Selects a unit system for a collection of unit expressions. Full
algorithm in § 5.

### 3.11 `is_compatible`

```rust
pub fn is_compatible(
    u1: &UnitExpression,
    u2: &UnitExpression,
) -> bool
```

Returns `true` iff `dimension(u1) == dimension(u2)`. Dimensionless
(`Count`, `Percent`, empty Dimension) is compatible with all other
dimensionless units and with `Radian` only when the caller explicitly
handles the angle/dimensionless promotion (callers should call
`check_transcendental_argument` for transcendental functions rather than
relying on `is_compatible`).

### 3.12 `check_transcendental_argument`

```rust
pub fn check_transcendental_argument(
    u: &UnitExpression,
) -> Result<(), ArgError>
```

Returns `Ok(())` if `u` is dimensionless (empty Dimension) or has
`Dimension { Angle: 1 }` (pure angle). Returns
`Err(ArgError::DimensionedArgument)` for any other dimension.

Called by thales before applying `sin`, `cos`, `tan`, `exp`, `log`,
`ln` to a node annotated with a unit.

**Examples:**

```
check_transcendental_argument(Radian)       →  Ok(())
check_transcendental_argument(Degree)       →  Ok(())   (dimension = Angle)
check_transcendental_argument(Count)        →  Ok(())   (dimensionless)
check_transcendental_argument(Meter)        →  Err(DimensionedArgument { "sin", {Length:1} })
check_transcendental_argument(Hertz)        →  Err(DimensionedArgument { "sin", {Time:-1} })
```

### 3.13 `parse_token`

```rust
pub fn parse_token(s: &str) -> Result<UnitExpression, ParseError>
```

Parses a single unit token string into a `UnitExpression`. Delegates
to `mathcore_units::alias::lookup_alias` and
`mathcore_units::alias::lookup_with_prefix`, then enforces the
collision-resolution rules from § 6.3 of mathcore-units. Full algorithm
in § 6.

### 3.14 `parse_unit_expression`

```rust
/// Parse a composite unit expression string into a UnitExpression tree.
///
/// Accepts inputs like:
///     "m/s"
///     "kg·m²/s²"       (Unicode middle-dot as multiplication)
///     "kg*m^2/s^2"     (ASCII asterisk + caret)
///     "N·m"
///     "W/(m²·K⁴)"
///     "m^(1/2)"        (rational exponent)
///     "dB"             (bare logarithmic unit)
///
/// Grammar (restricted arithmetic subset per mathcore-units § 2.8 UnitExpression):
///     unit_expr   := term (('*' | '·' | ' ' | '/') term)*
///     term        := atom ('^' exponent)?
///     atom        := TOKEN | '(' unit_expr ')'
///     exponent    := INTEGER | '(' INTEGER '/' INTEGER ')'
///     TOKEN       := symbols resolved via parse_token
///
/// Multiplication operators accepted (all equivalent):
///     '*'  ASCII asterisk
///     '·'  U+00B7 middle dot
///     ' '  whitespace between tokens (implicit multiplication)
///
/// Division is '/'. Parentheses allowed for grouping.
/// Exponentiation '^' takes integer or rational (num/den).
/// Unicode superscripts (²³⁴) accepted as syntactic sugar for ^2, ^3, ^4.
///
/// Operators NOT accepted (rejected with ParseError):
///     '+', '-'    (unit addition is dimension-unification, not syntax)
///     'log', 'ln', 'exp'  (these are operators on UnitExpression tree,
///                           not surface syntax — user can't write
///                           "log(m)" as a unit)
///     'sin', 'cos', ...   (transcendentals never on units)
///
/// Returns a UnitExpression that is:
///     - NOT canonicalized (factor/expand are explicit follow-up calls)
///     - Semantically equivalent to the input string's composition
///
/// Collision resolution for individual tokens follows parse_token rules
/// (mathcore-units § 6.3).
pub fn parse_unit_expression(s: &str) -> Result<UnitExpression, ParseError>;
```

**Implementation note:** `parse_unit_expression` is layered directly on
top of `parse_token`. Each atomic piece of the input (an uninterrupted
run of non-operator, non-parenthesis characters after stripping Unicode
superscripts) is passed to `parse_token`, which enforces alias lookup
and collision resolution. The composite parser is responsible only for
the grammar level: it tokenizes operators and parentheses, resolves
exponent syntax, and assembles the resulting `UnitExpression` tree via
`Binary { Mul | Div | Pow, … }` nodes. This separation ensures that
adding a new unit alias to the catalog or updating a collision-resolution
rule automatically propagates to `parse_unit_expression` with no further
changes.

**Worked examples:**

```
parse_unit_expression("m/s")
  → Binary(Div, Atom(Meter), Atom(Second))

parse_unit_expression("kg·m²/s²")
  → Binary(
      Div,
      Binary(Mul, Atom(Kilogram), Binary(Pow, Atom(Meter), Literal(2))),
      Binary(Pow, Atom(Second), Literal(2))
    )

parse_unit_expression("W/(m²·K⁴)")
  → Binary(
      Div,
      Atom(Watt),
      Binary(Mul,
        Binary(Pow, Atom(Meter), Literal(2)),
        Binary(Pow, Atom(Kelvin), Literal(4))
      )
    )
```

**Error cases:**

```
parse_unit_expression("")          → Err(ParseError::EmptyExpression)
parse_unit_expression("  ")       → Err(ParseError::EmptyExpression)
parse_unit_expression("m+s")      → Err(ParseError::InvalidOperator { pos: 1, found: '+' })
parse_unit_expression("m/s)")     → Err(ParseError::UnmatchedParen { pos: 3 })
parse_unit_expression("m^1.5")    → Err(ParseError::InvalidExponent { pos: 1 })
parse_unit_expression("unkn")     → Err(ParseError::UnknownToken("unkn".to_owned()))
```

---

## 4. Algorithms

### 4.1 Canonical form

A `UnitExpression` in canonical form satisfies:

1. It is a product of `Atom` nodes and at most one `Literal` scale
   coefficient.
2. Atoms are sorted by `BaseDim` (the order defined in
   `BaseDim` discriminant order; see § 2.1 of mathcore-units).
3. Zero-exponent dimensions are absent (matching `Dimension`'s
   zero-omission rule).
4. The overall `UnitExpression` tree structure is:
   - `Literal(scale)` if scale ≠ 1, followed by
   - A chain of `Binary { Mul, Atom { ... }, Binary { Mul, ... } }` for
     positive-exponent bases, then
   - A chain of `Binary { Mul, ..., Binary { Div, 1, Atom { ... } } }` for
     negative-exponent bases. Equivalently, negative exponents use
     `Binary { Pow, Atom, Literal(-n) }`.
5. Two canonical-form `UnitExpression` values for the same unit are
   structurally identical. This is required so that equality (`PartialEq`,
   `Hash`) derived on `UnitExpression` correctly identifies equal units.

**Canonicalization function (internal):**

```
fn canonicalize(u: &UnitExpression) -> UnitExpression {
    let dim = dimension(u);
    let scale = total_scale(u);   // product of all Literal factors and prefix scales
    let mut atoms: Vec<(BaseDim, i32)> = dim.0.into_iter().collect();
    atoms.sort_by_key(|(bd, _)| *bd);
    // build the expression tree from sorted atoms + scale
    build_canonical(scale, &atoms)
}
```

`total_scale` walks the tree collecting prefix exponents (each
`SiPrefix` contributes `10^exponent`) and `Literal` nodes, computing
their product as a `Rational`.

### 4.2 Expansion algorithm

```
fn expand_atom(id: UnitId, prefix: Option<SiPrefix>) -> UnitExpression {
    let spec = catalog::lookup(id);

    // SI base units: already expanded
    if spec.dimension.0.len() == 1
       && spec.dimension.0.values().next() == Some(&1)
       && spec.conversion == Linear { scale: 1 } {
        return UnitExpression::Atom { id, prefix };
    }

    // Affine units: cannot expand dimensionally; return atom as-is
    if matches!(spec.conversion, Conversion::Affine { .. }) {
        return UnitExpression::Atom { id, prefix };
    }

    // Logarithmic units: return atom as-is
    if matches!(spec.conversion, Conversion::Logarithmic { .. }) {
        return UnitExpression::Atom { id, prefix };
    }

    // FromConstant units: the constant factor lives in the Expression
    // domain, not UnitExpression. Return the dimension atoms without
    // the constant scale.
    // (Caller incorporates the constant factor into the main Expression.)
    if matches!(spec.conversion, Conversion::FromConstant { .. }) {
        return build_dimension_product(&spec.dimension, Rational::ONE);
    }

    // Linear: compute effective scale = spec.scale * prefix.factor
    let scale = spec_scale(spec) * prefix_scale(prefix);
    build_dimension_product(&spec.dimension, scale)
}

fn expand(u: &UnitExpression) -> UnitExpression {
    match u {
        Atom { id, prefix } => expand_atom(*id, *prefix),
        Binary { op, left, right } =>
            Binary { op: *op,
                     left: Box::new(expand(left)),
                     right: Box::new(expand(right)) },
        Unary { op, child } =>
            Unary { op: *op, child: Box::new(expand(child)) },
        Literal(_) => u.clone(),
        Log { .. } => u.clone(),   // logarithmic nodes not decomposed
    }
}
```

`build_dimension_product` produces a product of SI base atoms
weighted by their exponents and the given scale factor.

### 4.3 Factoring algorithm

```
fn factor(u: &UnitExpression) -> UnitExpression {
    let d = dimension(u);
    let s = total_scale(u);

    // 1. Find all catalog entries matching (dimension, scale)
    let candidates: Vec<&UnitSpec> = catalog::all()
        .filter(|spec| spec.dimension == d)
        .filter(|spec| linear_scale(spec) == Some(s))
        .collect();

    // 2. Priority: named SI derived > SI-derived additional > SI legacy > Imperial
    // 3. Tiebreak: catalog index (smallest wins)
    if let Some(best) = candidates.into_iter().min_by_key(priority_key) {
        UnitExpression::Atom { id: best.id, prefix: None }
    } else {
        // No match: return input in canonical form
        canonicalize(u)
    }
}
```

`priority_key` returns a tuple `(group, catalog_index)` where `group`
encodes: 0 = SI derived named, 1 = SI-derived additional, 2 = SI
legacy, 3 = Imperial, 4 = everything else.

**Round-trip invariant:** for all named units `u` that carry a
`Conversion::Linear` and whose scale matches the canonical product of
their base atoms: `factor(expand(u)) == u` holds. Affine and
logarithmic units are exempt from this invariant (they do not fully
round-trip through expand/factor).

### 4.4 Dimension arithmetic

Dimension operations are applied via the traversal in § 3.8.
Implementation uses simple `BTreeMap` iteration. Addition and
subtraction of `Dimension` values are helper functions internal to
unitalg:

```rust
fn dim_add(a: &Dimension, b: &Dimension) -> Dimension {
    let mut result = a.0.clone();
    for (bd, exp) in &b.0 {
        let entry = result.entry(*bd).or_insert(0);
        *entry += exp;
        if *entry == 0 { result.remove(bd); }
    }
    Dimension(result)
}

fn dim_sub(a: &Dimension, b: &Dimension) -> Dimension {
    let mut result = a.0.clone();
    for (bd, exp) in &b.0 {
        let entry = result.entry(*bd).or_insert(0);
        *entry -= exp;
        if *entry == 0 { result.remove(bd); }
    }
    Dimension(result)
}

fn dim_scale(a: &Dimension, factor: i32) -> Dimension {
    Dimension(a.0.iter()
        .filter_map(|(bd, exp)| {
            let new_exp = exp * factor;
            if new_exp == 0 { None } else { Some((*bd, new_exp)) }
        })
        .collect())
}
```

---

## 5. System selection algorithm

### 5.1 Algorithm

```rust
pub fn choose_system(
    units: &[UnitExpression],
    target: Option<System>,
) -> Result<System, SysError>
```

**Step 1: explicit target.**
If `target = Some(sys)`:
- For every `u` in `units`, verify that a conversion path exists to
  `sys` (i.e., `u`'s atoms are either in `sys` or convertible to
  a unit in `sys` via a Linear or Affine conversion). Dimensionless
  units (`System::None`) are compatible with any target.
- If all compatible: return `Ok(sys)`.
- If any incompatible: return `Err(SysError::TargetSystemIncompatible { target: sys, incompatible_unit: u.clone() })`.

**Step 2: heuristic (no explicit target).**
- Candidate order: `[units[0].system(), System::SI, System::Imperial, System::None]`,
  deduplicated (if `units[0].system()` is SI, the list starts `[SI, Imperial, None]`).
- For each candidate system `sys` in order:
  - Check all `u` in `units` for compatibility with `sys` (same predicate as step 1).
  - If all compatible: return `Ok(sys)`.
- If no candidate covers all units: return `Err(SysError::NoCompatibleSystem)`.

**Compatibility predicate** for unit `u` with system `sys`:
- All `Atom` leaves of `u` have `spec.system == sys` OR `spec.system == System::None`.
- Atoms with `System::None` (dimensionless) are compatible with any system.

### 5.2 Worked examples

**Example A — mixed length units, no target:**
```
units = [Foot, Meter]
units[0].system() = Imperial
candidates = [Imperial, SI, None]
Imperial: Foot ✓, Meter ✗ → skip
SI: Foot (convertible to Meter? Foot has system=Imperial) → Foot ✗ → skip
  ↳ wait: compatibility here checks system tag, not conversion existence.
  Meter ✓ SI, Foot ✗ SI → Imperial fails, SI fails
No candidate covers both → Err(NoCompatibleSystem)
```

Caller (mathlex / thales) invokes `combine_additive` which calls
`choose_system`. Receiving `NoCompatibleSystem` triggers an explicit
conversion: the operand in the non-dominant system is converted, then
additive result uses the dominant system's unit.

**Example B — all SI:**
```
units = [Newton, Meter, Second]
units[0].system() = SI
candidates = [SI, Imperial, None]
SI: Newton ✓, Meter ✓, Second ✓ → Ok(SI)
```

**Example C — dimensionless + SI:**
```
units = [Radian, Meter]
Radian.system = SI (per catalog), Meter.system = SI → Ok(SI)
```

**Example D — explicit target incompatible:**
```
units = [Meter], target = Some(Imperial)
Meter.system = SI ≠ Imperial → Err(TargetSystemIncompatible { Imperial, Meter })
```

---

## 6. Tokenizer integration

### 6.1 Overview

unitalg exposes two public parsing entry points:

- `parse_token` — converts a **single atomic token** (e.g., `"km"`,
  `"kg"`, `"Hz"`) to a `UnitExpression`. This is the foundation layer.
- `parse_unit_expression` — parses a **composite unit string** (e.g.,
  `"m/s"`, `"kg·m²/s²"`, `"W/(m²·K⁴)"`) by tokenizing operators and
  parentheses, then calling `parse_token` on each atomic piece. This
  is the entry point for mathlex consumers.

Both delegate to `mathcore_units` alias and prefix tables but enforce
collision-resolution rules centrally. `parse_unit_expression` reuses
`parse_token` entirely for atom resolution; it adds no independent
alias logic.

### 6.2 `parse_token` algorithm

```
fn parse_token(s: &str) -> Result<UnitExpression, ParseError> {
    // Step 1: reject known-reserved tokens before any lookup
    if RESERVED_TOKENS.contains(s) {
        return Err(ParseError::ReservedToken(s.to_owned()));
    }

    // Step 2: direct alias match (longest first, per mathcore-units § 6.1)
    if let Some(id) = mathcore_units::alias::lookup_alias(s) {
        return Ok(UnitExpression::Atom { id, prefix: None });
    }

    // Step 3: prefix + atom decomposition
    if let Some((Some(prefix), id)) = mathcore_units::alias::lookup_with_prefix(s) {
        // Check PrefixPolicy
        let spec = catalog::lookup(id);
        if spec.applies_prefix == PrefixPolicy::None {
            return Err(ParseError::PrefixNotAllowed { prefix, unit: id });
        }
        return Ok(UnitExpression::Atom { id, prefix: Some(prefix) });
    }

    // Step 4: ambiguity check (multiple decompositions, no resolution)
    if has_multiple_decompositions(s) {
        return Err(ParseError::AmbiguousToken(s.to_owned()));
    }

    Err(ParseError::UnknownToken(s.to_owned()))
}
```

`RESERVED_TOKENS` is a static set of strings that are valid in the main
expression namespace but must not resolve as units: `"c"` (speed of
light constant), `"e"` (Euler number / electron charge), `"rem"`
(obsolete radiation unit).

### 6.3 Collision resolution (enforced by tokenizer)

The rules below implement § 6.3 of mathcore-units at the parse_token
boundary. Each row states the token, the required resolution, and the
mechanism.

| Token | Required resolution | Mechanism |
|---|---|---|
| `T` | Tesla | Direct alias wins; prefix `T` (Tera) requires companion atom |
| `TT` | `(Tera, Tesla)` | No direct alias; decomp `T` (prefix) + `T` (Tesla) |
| `G` | Gauss (in unit context) | Direct alias wins; prefix `G` (Giga) requires companion atom |
| `Gm` | `(Giga, Meter)` | No direct alias `Gm`; decomp succeeds |
| `ms` | `(Milli, Second)` | Listed as direct alias in mathcore-units; takes precedence over juxtaposition |
| `m` | Meter | Direct alias, bare, case-sensitive |
| `h` | Hour | Direct alias; Hecto requires companion atom |
| `hm` | `(Hecto, Meter)` | Decomp |
| `d` | Day | Direct alias (admitted only unambiguously per § 6.4 of mathcore-units) |
| `cm` | `(Centi, Meter)` | Decomp; Centimeter is not a standalone UnitId |
| `c` | `ReservedToken` error | Reserved: speed-of-light constant in main-expr context; not a unit |
| `e` | `ReservedToken` error | Reserved: Euler number / elementary charge |
| `rad` | Radian | Direct alias; radiation absorbed-dose `rad` dropped from catalog |
| `rem` | `ReservedToken` error | Obsolete; Sievert replaces |
| `u` | AtomicMassUnit | Direct alias; Micro prefix requires `μ` / `µ` |
| `P` | Poise | Direct alias; Peta requires companion atom |
| `R` | Roentgen | Direct alias; Ronna requires companion atom |
| `ct` | Carat | Direct alias; centi-tonne decomp absurd |
| `gr` | Grain | Direct alias; never resolves to Gram |
| `g` | Gram | Direct alias, bare |

### 6.4 Prefix admissibility enforcement

After resolving `(prefix, id)`, `parse_token` checks the catalog's
`applies_prefix` for `id`:

- `SiDecimal` → any prefix accepted.
- `SiDecimalRestricted` → only power-of-3 prefixes (Kilo, Mega, Giga,
  Tera, Peta, Exa, Zetta, Yotta, Ronna, Quetta, Milli, Micro, Nano,
  Pico, Femto, Atto, Zepto, Yocto, Ronto, Quecto). Non-power-of-3
  prefixes (Centi, Deci, Deca, Hecto) → `Err(ParseError::PrefixNotAllowed)`.
- `None` → any prefix → `Err(ParseError::PrefixNotAllowed)`.
- `Binary` → not used in v0.x; no binary prefixes accepted.

### 6.5 `parse_unit_expression` algorithm

```
fn parse_unit_expression(s: &str) -> Result<UnitExpression, ParseError> {
    // Step 1: reject empty / whitespace-only input
    if s.trim().is_empty() {
        return Err(ParseError::EmptyExpression);
    }

    // Step 2: scan for additive operators and transcendental keywords —
    // rejected immediately before any recursive descent
    for (pos, ch) in s.char_indices() {
        if ch == '+' || ch == '-' {
            return Err(ParseError::InvalidOperator { pos, found: ch });
        }
    }
    for kw in ["log", "ln", "exp", "sin", "cos", "tan"] {
        if s.contains(kw) {
            // Find position of first char of keyword
            let pos = s.find(kw).unwrap();
            return Err(ParseError::InvalidOperator {
                pos,
                found: s[pos..].chars().next().unwrap(),
            });
        }
    }

    // Step 3: recursive descent per grammar (§ 3.14)
    //   unit_expr := term (('*' | '·' | ' ' | '/') term)*
    parse_unit_expr_inner(s.trim())
}

// Recursive descent helpers (internal):
//
//   parse_unit_expr_inner(s) → splits on top-level '*', '·', ' ', '/'
//     while respecting parenthesis depth; builds Binary { Mul | Div } nodes.
//
//   parse_term(s) → splits on '^'; calls parse_atom for the base,
//     then parse_exponent for the exponent. Normalises Unicode superscripts
//     (², ³, ⁴, etc.) to their integer values before calling parse_atom.
//
//   parse_atom(s) → if surrounded by matching '(' ')': recurse via
//     parse_unit_expr_inner on the inner slice; otherwise call parse_token(s).
//
//   parse_exponent(s) → if s is a bare INTEGER: return Literal(n);
//     if s matches '(' INTEGER '/' INTEGER ')': return
//       Binary { Pow, Literal(num), Literal(den) } (rational exponent);
//     otherwise return Err(ParseError::InvalidExponent { pos }).
//
// Parenthesis balance is checked at each level; an unmatched '(' or ')'
// returns Err(ParseError::UnmatchedParen { pos }).
```

Superscript normalisation table used by `parse_term`:

| Character | Unicode | Normalised integer |
|---|---|---|
| `²` | U+00B2 | 2 |
| `³` | U+00B3 | 3 |
| `⁴` | U+2074 | 4 |
| `⁵` | U+2075 | 5 |
| `⁶` | U+2076 | 6 |
| `⁷` | U+2077 | 7 |
| `⁸` | U+2078 | 8 |
| `⁹` | U+2079 | 9 |
| `⁰` | U+2070 | 0 |
| `¹` | U+00B9 | 1 |
| `⁻` | U+207B | negates following digit |

Negative superscripts (`⁻¹`, `⁻²`) are supported: `⁻` followed by a
superscript digit produces the corresponding negative integer exponent.

---

## 7. Logarithmic unit handling

### 7.1 dB family conversions

The dB family uses `Conversion::Logarithmic { reference, reference_scale, base, factor }`
per § 2.6 of mathcore-units. `LogBase` (re-exported from mathcore-units)
is the enum governing the logarithm base:

```rust
// re-exported from mathcore_units::conversion
pub use mathcore_units::conversion::LogBase;
// LogBase::Ten  — base 10 (all dB variants)
// LogBase::E    — base e  (Neper)
// LogBase::Two  — base 2  (reserved, not used in v0.1.0 catalog)
```

The physical SI-inversion formula for each catalog entry
(`x_si = reference_unit_value · reference_scale · base^(x_unit / factor)`):

| UnitId | `reference` | `reference_scale` | `base` | `factor` | SI inversion |
|---|---|---|---|---|---|
| Decibel | None | 1 | `LogBase::Ten` | 10 | dimensionless; `10^(x/10)` (power) or `10^(x/20)` (amplitude) |
| DecibelMilliwatt | Some(Watt) | 10⁻³ | `LogBase::Ten` | 10 | `P_W = 10⁻³ · 10^(x/10)` |
| DecibelWatt | Some(Watt) | 1 | `LogBase::Ten` | 10 | `P_W = 1 · 10^(x/10)` |
| DecibelSpl | Some(Pascal) | 2×10⁻⁵ | `LogBase::Ten` | 20 | `p_Pa = 2×10⁻⁵ · 10^(x/20)` |
| Neper | None | 1 | `LogBase::E` | 1 | `A = e^x` (dimensionless amplitude ratio) |

**Power vs. amplitude factor:** `factor = 10` for power quantities
(intensity, power density, voltage squared normalized to impedance);
`factor = 20` for amplitude quantities (voltage, current, field
amplitude). The `factor` field in `Conversion::Logarithmic` encodes
this distinction per unit.

**`reference_scale` encodes the per-variant reference level** directly,
replacing any implicit "offset for milli/μPa" workaround from earlier
drafts. For variants with no physical reference (plain Decibel, Neper),
`reference` is `None` and `reference_scale` is 1.

**Neper:** uses `base: LogBase::E` (natural logarithm). The inversion
is `A = e^(x_Np)` — dimensionless amplitude ratio referenced to 1.
Conversion between Neper and dB: `1 Np = 20/ln(10) dB ≈ 8.686 dB`.
`convert(Neper, Decibel)` emits the exact symbolic factor derived from
the `LogBase::E` vs `LogBase::Ten` base ratio.

### 7.2 `combine_additive` and logarithmic units

Mixing dB variants in an additive expression is physically meaningful
only within the same logarithmic reference:

- `dBm + dBm` is not physically additive in the standard sense (power
  adds linearly, not logarithmically). However, for the purpose of
  CAS symbolic manipulation, `combine_additive(dBm, dBm, None)` is
  permitted and returns `unified_unit = dBm`, `factors = (1, 1)`,
  `offsets = (0, 0)`. It is left to the caller to recognize that the
  result is a symbolic expression, not a physical sum.
- `dBm + dBW` have the same underlying dimension (Power) but different
  logarithmic references. `combine_additive` converts `dBW` to `dBm`
  via the 30 dB offset (see § 3.9 logarithmic conversion) and returns
  the result in `dBm`.
- `dBm + Watt` mixes a logarithmic unit with a linear power unit.
  `combine_additive` returns `Err(DimError::LogarithmicReferenceMismatch)`.
  thales must unwrap the logarithmic relationship before combining.
- `Decibel + <dimensionless>` is accepted: plain Decibel is
  dimensionless; a dimensionless addend is compatible.

### 7.3 Logarithmic reference compatibility predicate

```rust
fn log_refs_compatible(
    u1: &UnitExpression,
    u2: &UnitExpression,
) -> Result<(), DimError>
```

Two unit expressions are logarithmically compatible for additive
operations if:

1. Neither is a `Log` node with a non-None `reference` (i.e., neither
   is a reference-bearing dB variant), OR
2. Both have the same `reference` (same `Option<UnitId>`), OR
3. Both have different references but the same underlying linear
   dimension (same physical quantity — convertible by an offset, see
   § 7.2).
4. One is dimensionless (`Log { reference: None, ... }` = plain
   Decibel / Neper) and the other is also dimensionless.

Any other combination → `Err(DimError::LogarithmicReferenceMismatch)`.

---

## 8. Constant-referenced conversion flow

### 8.1 Pattern

When a `UnitSpec` carries `Conversion::FromConstant { id: ConstantId }`,
the unit's scale factor is not a `Rational` — it equals the numeric
value of the physical constant identified by `id`, which is stored in
`mathcore-constants`.

unitalg's role in this chain is strictly symbolic:

1. `convert(value, from_unit, to_unit)` where `from_unit` or `to_unit`
   contains a `FromConstant` atom:
   - Emit `Expression::Variable(constant_symbol(id))` in place of the
     numeric scale.
   - The returned `Expression` contains unevaluated `Variable` nodes.
2. unitalg never calls `mathcore_constants::lookup`. It does not
   import `mathcore-constants` as a dependency.
3. Downstream consumers (thales, mathlex-eval) either:
   - Leave the variable symbolic (for exact / parametric results), or
   - Substitute by calling `mathcore_constants::lookup(id).value`.

### 8.2 Constant symbol mapping

The mapping from `ConstantId` to the string name used in
`Expression::Variable` is a static table in unitalg:

| ConstantId | Variable string |
|---|---|
| SolarMass | `"M_sun"` |
| EarthMass | `"M_earth"` |
| JupiterMass | `"M_J"` |
| LunarMass | `"M_moon"` |
| ElectronMass | `"m_e"` |
| ProtonMass | `"m_p"` |
| NeutronMass | `"m_n"` |
| SolarRadius | `"R_sun"` |
| EarthRadius | `"R_earth"` |
| SolarLuminosity | `"L_sun"` |
| AtomicMassUnit | `"u_amu"` |

These strings must match the variable names used by mathlex when
parsing expressions containing those symbols. Coordination with the
mathlex catalog is required at integration time.

### 8.3 Symbolic emission example

```
// Convert "1.5 M_sun" to kilograms
convert(
    Expression::Rational(3, 2),      // value = 1.5
    &UnitExpression::Atom { id: UnitId::SolarMass, prefix: None },
    &UnitExpression::Atom { id: UnitId::Kilogram, prefix: None },
)
→ Expression::Mul(
    Expression::Rational(3, 2),
    Expression::Variable("M_sun")    // resolved by downstream
  )
```

---

## 9. Wire format

unitalg introduces no new serde types beyond the error enums listed in
§ 2.2. Serialization of `UnitExpression`, `Dimension`, `UnitId`, and
`SiPrefix` is owned entirely by mathcore-units (per § 7 of
mathcore-units).

Error variants are serde-capable when the `serde` feature is enabled,
using the same `#[serde(tag = "kind", content = "value")]` convention
as mathcore-units. This enables structured error reporting over FFI or
RPC transports.

`AdditiveResult` contains `Expression` fields. Serde for `Expression`
is owned by mathlex. When `AdditiveResult` is serialized (e.g., for
wire transport), the enclosing system must enable both mathlex's serde
feature and unitalg's serde feature.

---

## 10. Test strategy

### 10.1 Unit tests (per module)

- Every public function has at least one positive test and at least
  one negative test (error path).
- `multiply`, `divide`: commutativity (`multiply(a, b) == multiply(b, a)`);
  associativity with three operands.
- `power`: power 0 returns dimensionless; power 1 is identity;
  negative exponent produces inverse dimension.
- `power_rational`: fractional exponents that result in integer
  dimensions succeed; fractional results fail.
- `dimension`: traversal covers all node variants.
- `is_compatible`: reflexivity, symmetry, incompatible pair.
- `check_transcendental_argument`: all six passing cases (Radian,
  Degree, ArcMinute, ArcSecond, Count, empty); representative failing
  cases (Meter, Kilogram, Hertz).

### 10.2 Catalog coverage test (CI-gated, compile-time or integration)

For every `UnitId` variant:
- `dimension(UnitExpression::Atom { id, prefix: None })` returns a
  non-panicking, valid `Dimension`.
- `expand(Atom { id, None })` completes without panic.
- `factor(expand(Atom { id, None }))` returns a valid `UnitExpression`
  (round-trip test; see § 10.3 for exact equality expectation).

### 10.3 Round-trip tests

For every named unit `id` whose `Conversion` is `Linear`:

```
assert_eq!(factor(expand(Atom { id, prefix: None })), Atom { id, prefix: None });
```

Affine units (DegreeCelsius, DegreeFahrenheit, DegreeRankine) and
logarithmic units (Decibel family, Neper) are excluded from this
assertion; they survive `expand` unchanged and `factor` does not
reconstruct them.

### 10.4 Collision audit test (CI-gated)

Iterates the full alias table (both direct aliases and every valid
prefix-atom decomposition):

1. Every token resolves to exactly one `(Option<SiPrefix>, UnitId)`.
2. No two tokens resolve to the same result via different paths unless
   they are explicit duplicate aliases.
3. `RESERVED_TOKENS` set is exhaustive: no token in the reserved list
   accidentally resolves to a valid unit via direct alias.

This test mirrors the mathcore-units CI audit (§ 6 of mathcore-units)
but tests the enforcement layer in unitalg rather than the raw tables.

### 10.5 Dimension arithmetic sanity

```
// Newton / Second = Newton · Hertz dimensionally?
let n = dimension(&Atom(Newton));     // {L:1, M:1, T:-2}
let s = dimension(&Atom(Second));     // {T:1}
let ns_dim = dim_sub(&n, &s);         // {L:1, M:1, T:-3}
let hz = dimension(&Atom(Hertz));     // {T:-1}
let n_hz = dim_add(&n, &hz);          // {L:1, M:1, T:-3}
assert_eq!(ns_dim, n_hz);             // ✓
```

### 10.6 Transcendental-argument check

```
// sin(m) rejected; sin(rad) accepted; sin(1) accepted
assert!(check_transcendental_argument(&Atom(Meter)).is_err());
assert!(check_transcendental_argument(&Atom(Radian)).is_ok());
assert!(check_transcendental_argument(&Literal(Rational::ONE)).is_ok());
```

### 10.7 Conversion emission

- Linear conversion: `convert(x, Foot, Meter)` emits
  `Mul(x, Rational(3048, 10000))` (exact).
- Affine conversion: `convert(x, DegreeCelsius, DegreeFahrenheit)`
  emits `Add(Mul(x, Rational(9, 5)), Rational(32, 1))`.
- `FromConstant` conversion: `convert(x, SolarMass, Kilogram)` emits
  `Mul(x, Variable("M_sun"))`.

### 10.8 `parse_unit_expression` tests

Tests live in `tests/parse_unit_expression.rs`.

**Positive cases — tree shape:**

```rust
// m/s → Div(Meter, Second)
assert_eq!(
    parse_unit_expression("m/s").unwrap(),
    Binary(Div, Atom(Meter), Atom(Second))
);

// kg·m²/s² — Unicode middle-dot and superscripts
assert_eq!(
    parse_unit_expression("kg·m²/s²").unwrap(),
    Binary(Div,
        Binary(Mul, Atom(Kilogram), Binary(Pow, Atom(Meter), Literal(2))),
        Binary(Pow, Atom(Second), Literal(2)))
);

// ASCII asterisk and caret equivalent to Unicode forms
assert_eq!(
    parse_unit_expression("kg*m^2/s^2").unwrap(),
    parse_unit_expression("kg·m²/s²").unwrap()
);

// Parenthesised denominator: W/(m²·K⁴)
assert_eq!(
    parse_unit_expression("W/(m²·K⁴)").unwrap(),
    Binary(Div,
        Atom(Watt),
        Binary(Mul,
            Binary(Pow, Atom(Meter), Literal(2)),
            Binary(Pow, Atom(Kelvin), Literal(4))))
);

// Rational exponent: m^(1/2)
assert_eq!(
    parse_unit_expression("m^(1/2)").unwrap(),
    Binary(Pow, Atom(Meter), Binary(Div, Literal(1), Literal(2)))
);

// Bare single token passes through parse_token unchanged
assert_eq!(
    parse_unit_expression("dB").unwrap(),
    parse_token("dB").unwrap()
);

// Implicit multiplication by whitespace: "kg m" == "kg*m"
assert_eq!(
    parse_unit_expression("kg m").unwrap(),
    parse_unit_expression("kg*m").unwrap()
);
```

**Error cases:**

```rust
assert_eq!(parse_unit_expression("").unwrap_err(),    ParseError::EmptyExpression);
assert_eq!(parse_unit_expression("   ").unwrap_err(), ParseError::EmptyExpression);
assert!(matches!(
    parse_unit_expression("m+s").unwrap_err(),
    ParseError::InvalidOperator { found: '+', .. }
));
assert!(matches!(
    parse_unit_expression("m/s)").unwrap_err(),
    ParseError::UnmatchedParen { .. }
));
assert!(matches!(
    parse_unit_expression("m^1.5").unwrap_err(),
    ParseError::InvalidExponent { .. }
));
// Unknown token propagated from parse_token
assert!(matches!(
    parse_unit_expression("unkn").unwrap_err(),
    ParseError::UnknownToken(_)
));
```

**Collision resolution propagation:** `parse_unit_expression("TT/s")`
must resolve `TT` as `(Tera, Tesla)` via `parse_token`, consistent
with § 6.3 collision table. Test uses the same expected output as the
`parse_token` collision audit.

---

## 11. Crate layout

```
unitalg/
├── Cargo.toml
├── LICENSE
├── README.md
├── CLAUDE.md
├── SPECIFICATION.md                 # this file
├── src/
│   ├── lib.rs                       # pub re-exports + crate doc
│   ├── ops.rs                       # multiply, divide, power, power_rational
│   ├── additive.rs                  # combine_additive, AdditiveResult
│   ├── structural.rs                # expand, factor, canonicalize
│   ├── dimension.rs                 # dimension(), dim_add/sub/scale helpers
│   ├── convert.rs                   # convert, conversion emission
│   ├── system.rs                    # choose_system, SysError
│   ├── check.rs                     # is_compatible, check_transcendental_argument
│   ├── parse.rs                     # parse_token, parse_unit_expression, RESERVED_TOKENS
│   ├── constants.rs                 # constant_symbol mapping table (§ 8.2)
│   └── error.rs                     # DimError, ConvError, SysError, ArgError, ParseError
└── tests/
    ├── catalog_coverage.rs          # every UnitId round-trips without panic
    ├── collision_audit.rs           # all tokens resolve unambiguously (CI-gated)
    ├── round_trip.rs                # factor(expand(u)) == u for Linear units
    ├── dimension_arithmetic.rs      # sanity checks on dim_add/sub/scale
    ├── convert.rs                   # emission correctness (linear, affine, log, const)
    ├── transcendental.rs            # check_transcendental_argument
    ├── parse_unit_expression.rs     # parse_unit_expression positive + error cases (§ 10.8)
    └── system_selection.rs          # choose_system worked examples
```

---

## 12. Versioning

- v0.1.0 ships the full API surface defined here.
- Minor versions: new public functions, new error variants (additive
  only — removing variants is breaking), extended collision table.
- Breaking changes: any public function signature change, error variant
  removal or rename, algorithmic invariant change (e.g., round-trip
  guarantee relaxed). Requires major bump coordinated with thales and
  mathlex.
- Feature flags:
  - `default = ["std"]`
  - `std`: enables `std::collections` (otherwise `alloc`-only).
  - `serde`: enables serde derives on error types.

---

## 13. Requirements summary (UA-1..UA-N)

| ID | Requirement | Severity |
|---|---|---|
| UA-1 | `multiply` and `divide` operate on `UnitExpression` by adding/subtracting dimension exponents; result in canonical form | Blocker |
| UA-2 | `power` accepts integer exponents including zero and negative | Blocker |
| UA-3 | `power_rational` accepts rational exponents; returns error when result dimension exponents are non-integer | Blocker |
| UA-4 | `combine_additive` enforces dimension equality and logarithmic-reference compatibility before returning `AdditiveResult` | Blocker |
| UA-5 | `AdditiveResult` carries factor and offset fields as `Expression` values (symbolic, not numeric) for both operands | Blocker |
| UA-6 | `expand` unfolds all `Linear`-conversion atoms to SI base products; leaves affine and logarithmic atoms unchanged | Blocker |
| UA-7 | `factor` folds base-unit products to the most specific named catalog entry; priority order enforced; tiebreak is catalog index | Blocker |
| UA-8 | `factor(expand(u)) == u` for all named units with `Conversion::Linear`; no assertion for affine or logarithmic units | Blocker |
| UA-9 | `dimension` is a pure, total function over all `UnitExpression` variants | Blocker |
| UA-10 | `convert` emits symbolic `Expression` fragments; never evaluates numerically; `FromConstant` units emit `Expression::Variable` | Blocker |
| UA-11 | Linear and affine conversion formulas are exact (`Rational` arithmetic, no floating-point) | Blocker |
| UA-12 | `choose_system` implements first-wins heuristic with explicit-target fast path; returns `SysError` when no system covers all operands | Blocker |
| UA-13 | `is_compatible` is a pure dimension-equality predicate | Required |
| UA-14 | `check_transcendental_argument` returns `Ok(())` iff dimension is empty or pure angle; all other dimensions return `Err(ArgError)` | Blocker |
| UA-15 | `parse_token` delegates to mathcore-units alias + prefix tables; enforces all § 6.3 collision resolutions; rejects reserved tokens | Blocker |
| UA-16 | Prefix admissibility enforced per `PrefixPolicy` at `parse_token` time | Blocker |
| UA-17 | Canonical form is deterministic: two equal units produce structurally identical `UnitExpression` trees | Blocker |
| UA-18 | Collision audit test runs in CI; covers full alias table + prefix decompositions; zero ambiguity tolerance | Blocker |
| UA-19 | `no_std + alloc` support; no `std`-only dependencies behind the `std` feature flag | Required |
| UA-20 | No dependency on thales, mathlex, mathlex-eval, or `Arc<Expr>` | Blocker |
| UA-21 | Logarithmic unit mixing in `combine_additive`: same reference permitted; different reference with same dimension converted via offset; linear + logarithmic rejected | Required |
| UA-22 | dBm conversion formula documented and tested: `P_W = 0.001 * 10^(x_dBm / 10)` is a thales-level expansion; unitalg emits only the offset expression for dB-to-dB conversions | Required |
| UA-23 | Constant symbol mapping table (`ConstantId` → variable string) is static, exhaustive for all `FromConstant` catalog entries, and coordinated with mathlex variable names | Required |
| UA-24 | `AdditiveResult` serde-capable (opt-in) only when both mathlex serde and unitalg serde features are enabled | Optional |
| UA-25 | Wire format for error enums uses `#[serde(tag = "kind", content = "value")]` consistent with mathcore-units | Required |
| UA-26 | `parse_unit_expression(s)` parses composite unit strings (`m/s`, `kg·m²/s²`, `W/(m²·K⁴)`) via the grammar in § 3.14, layered on `parse_token`; Unicode middle-dot (`·`) and ASCII asterisk (`*`) are interchangeable multiplication operators; implicit multiplication by whitespace is accepted; Unicode superscript digits (`²³⁴⁵⁶⁷⁸⁹`) are accepted as sugar for integer exponents; additive operators (`+`, `-`) and transcendental keywords (`log`, `ln`, `exp`, `sin`, `cos`, `tan`) are rejected with `ParseError::InvalidOperator`; unmatched parentheses return `ParseError::UnmatchedParen`; non-integer non-rational exponents return `ParseError::InvalidExponent`; empty or whitespace-only input returns `ParseError::EmptyExpression` | Blocker |

---

## 14. Resolved decisions

Decisions confirmed during spec drafting (2026-04-22):

1. **No `Arc<Expr>` dependency.** unitalg operates exclusively on
   `UnitExpression`. thales Rule 1 is upheld because units ride as
   annotations outside the Arc<Expr> computational domain.

2. **Symbolic-only conversion.** `convert` emits `Expression` fragments;
   it never calls mathcore-constants for numeric values. This keeps
   unitalg's dependency graph minimal and preserves exact symbolic
   arithmetic for downstream consumers.

3. **Affine units excluded from expand/factor round-trip.** The additive
   offset in Celsius / Fahrenheit / Rankine has no representation in
   a pure `UnitExpression` dimension product. `expand` returns these
   atoms unchanged; `factor` never reconstructs them. Callers needing
   temperature conversion always go through `convert`.

4. **Logarithmic inversion is thales's responsibility.** The dBm → Watt
   inversion (`P_W = 0.001 * 10^(x/10)`) involves an exponential
   applied to the numeric value, not merely a unit scale. This crosses
   the boundary into expression algebra; unitalg raises
   `ConvError::LogarithmicConversionAmbiguous` and thales handles the
   rewrite at the `Expression` level.

5. **`power_rational` returns error on fractional dimension exponents.**
   `Dimension` stores integer exponents exclusively (per § 2.2 of
   mathcore-units). Rational exponents that would produce fractional
   entries are rejected with `DimError::IncompatibleDimensions`. The
   primary use case (`V/√Hz`) requires callers to pre-expand units
   before applying the rational power.

6. **First-wins system selection; no heuristic blending.** System
   selection picks the first system that covers all operands. There is
   no weighted voting or partial coverage. If no single system covers
   all operands, the error is raised and the caller must explicitly
   convert at least one operand before retrying. This keeps the
   algorithm deterministic and testable.

7. **Constant symbol strings coordinated with mathlex.** The static
   mapping in § 8.2 must match the variable names mathlex uses for the
   same constants. If mathlex's variable namespace changes, unitalg's
   constant symbol table must be updated in the same version wave.
   This coordination is a required integration test (§ 10.4's broader
   scope covers it; a dedicated integration test is noted in
   `tests/collision_audit.rs`).

8. **Error enums are additive-only in minor versions.** Adding a new
   variant to `DimError`, `ConvError`, etc. is permitted in a minor
   version (consumers using `#[non_exhaustive]` patterns are
   unaffected). Removing or renaming a variant is a breaking change.
   All error enums are marked `#[non_exhaustive]` in the implementation.

9. **`combine_additive` accepts same-unit dB addition symbolically.**
   Even though `dBm + dBm` is not physically meaningful as linear
   summation, the CAS operates symbolically and does not inject physics
   interpretations. The caller (thales) decides whether to simplify or
   flag the combination for the user.

10. **No floating-point in unitalg core.** All scale factors, offsets,
    and rational exponents are represented as `Rational` (exact). The
    only path to floating-point is via mathlex-eval or thales's
    numerical evaluation passes. This is consistent with thales Rule 5
    (zero technical debt) and with the alloc-only target.

11. **MI-ISSUE-1 resolved: composite unit parsing via `parse_unit_expression`.**
    The mathlex integration spec identified that `parse_token` only handles
    single-atom tokens (`km`, `kg`, `Hz`) and is therefore insufficient for
    consumers that need to parse composite unit strings such as `m/s`,
    `kg·m²/s²`, and `W/(m²·K⁴)`. The resolution is the new public function
    `parse_unit_expression` (§ 3.14), which adds a grammar layer on top of
    `parse_token`. Individual atoms continue to route through `parse_token`
    so collision-resolution rules and prefix-admissibility enforcement are
    inherited automatically. No changes to `parse_token` are required.
    Requirement UA-26 captures this contract; tests live in
    `tests/parse_unit_expression.rs` (§ 10.8).

---

## 15. Flagged open issues

Issues identified during reading of mathcore-units SPECIFICATION.md
that may affect unitalg. Issues resolved by the foundation amendment
of 2026-04-22 are marked accordingly.

1. **RESOLVED.** `Conversion::Logarithmic.base` is now `LogBase` enum
   (`LogBase::Ten / LogBase::E / LogBase::Two`), resolving the inability
   to represent `e` as a `u32`. See mathcore-units § 2.6 (amended
   2026-04-22). unitalg imports `LogBase` via re-export from
   `mathcore_units::conversion`. No special-casing of Neper required.

2. **RESOLVED.** `Conversion::Logarithmic.reference` is now
   `Option<UnitId>`. `None` represents the dimensionless case (plain
   Decibel, Neper); `Some(id)` carries the physical reference unit for
   dBm, dBW, and dB SPL. See mathcore-units § 2.6 (amended 2026-04-22).
   The `log_refs_compatible` predicate in § 7.3 of this spec already
   used `Option<UnitId>` throughout — it is now aligned with the
   foundation type.

3. **RESOLVED.** The relationship between `UnitExpression::Log.reference`
   (§ 2.8 of mathcore-units) and `Conversion::Logarithmic.reference`
   (§ 2.6) is clarified by the amendment: both fields are
   `Option<UnitId>`. The catalog's `Conversion::Logarithmic.reference`
   is the authoritative reference unit for a given `UnitId`. The
   `UnitExpression::Log.reference` carries the same value when the
   expression tree is built from a catalog atom (i.e., it reflects the
   catalog entry). In unitalg's `log_refs_compatible` predicate (§ 7.3),
   the reference is read from the `UnitExpression::Log.reference` field
   of each operand — which is populated from the catalog at
   `UnitExpression` construction time. No override semantics exist for
   v0.1.0; the field is set, not overridden.

4. **Documentation note (intentional, not a bug).** `AstronomicalUnit`,
   `LightYear`, and `Parsec` appear in both `UnitId` (§ 2.4) and
   `ConstantId` (§ 4) of mathcore-units. This is intentional per
   mathcore-units § 4 (amended 2026-04-22): as `UnitId` the entries are
   used in unit-annotation context (`5 ly` = five lightyears of length,
   `Conversion::Linear`); as `ConstantId` they are used in
   main-expression context (`c · t / lightyear` where `lightyear` is a
   named physical constant). The two use cases are disjoint; both are
   legitimate. unitalg is unaffected: it handles the `UnitId` side via
   the catalog (`Linear` conversion) and never needs to distinguish
   whether the same quantity also exists as a `ConstantId`.

5. **RESOLVED.** `Year` alias `yr` (and `year`, `years`) is now
   explicitly listed in mathcore-units § 6.2 alias table (amended
   2026-04-22). The audit-critical note is that `y` is NOT accepted as
   a Year alias — it is the Yocto prefix (10⁻²⁴); and `a` is NOT
   accepted — it is the Atto prefix (10⁻¹⁸). The foundation § 6.2
   table now documents this explicitly. unitalg's tokenizer requires no
   additional special-casing: `yr` is a direct alias, not a
   collision-resolution case.

6. **RESOLVED.** `DecibelSpl`'s reference level is now fully specified
   via `Conversion::Logarithmic { reference: Some(Pascal), reference_scale: 2×10⁻⁵, base: LogBase::Ten, factor: 20 }`.
   The `reference_scale` field (added in the amendment to § 2.6 of
   mathcore-units) encodes the 20 μPa reference level directly as a
   rational scale on the Pascal reference unit. No separate offset field
   or implicit convention is needed. See mathcore-units § 5.6 (amended
   2026-04-22).
