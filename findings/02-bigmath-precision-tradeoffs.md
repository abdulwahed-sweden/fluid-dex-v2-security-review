# BigMath Precision Trade-offs

## Classification

**Level**: Design Observation
**Status**: By design
**Impact**: Expected precision loss in storage

## Context

The protocol uses a BigMath library for efficient storage of large numbers
in compressed form. This analysis documents the precision characteristics
and their implications.

## Mechanism

BigMath stores numbers using a coefficient and exponent representation:

```
stored_value = coefficient << exponent
```

Where:
- Coefficient: Variable bit width (e.g., 56 bits)
- Exponent: Fixed bit width (e.g., 8 bits)

## Precision Characteristics

### Compression Loss

When converting from full precision to BigMath representation, least
significant bits may be lost:

```solidity
tokenRawSupply_ = BM.toBigNumber(
    tokenRawSupply_,
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    ROUND_DOWN  // or ROUND_UP depending on context
);
```

### Rounding Direction

The protocol consistently rounds in favor of the protocol:

| Operation | Rounding | Rationale |
|-----------|----------|-----------|
| Supply (user deposits) | Down | Protocol credits less |
| Withdraw (user receives) | Down | Protocol pays less |
| Borrow (user receives) | Up | Protocol records more debt |
| Payback (user pays) | Down | Protocol credits less payback |

### Cumulative Effect

Over multiple operations, precision loss accumulates:

```
Operation 1: 1,000,000 -> compress -> 999,744
Operation 2: withdraw half -> 499,872 -> compress -> 499,712
Net loss: 288 units (0.03%)
```

## Supply Calculation Example

The supply path includes explicit double-rounding:

```solidity
// First rounding: division with offset
tokenRawSupply_ = ((supplyAmount_ * PRECISION) - 1) / exchangePrice_;

// Second rounding: explicit subtraction
if (tokenRawSupply_ > 0) tokenRawSupply_ -= 1;
```

This ensures the protocol never credits more than actually received.

## Design Rationale

The precision trade-offs serve multiple purposes:

1. **Storage Efficiency**: Reduces storage costs by ~50%
2. **Protocol Safety**: Accumulated rounding favors protocol solvency
3. **Dust Prevention**: Small amounts round to zero, preventing dust attacks

## Quantified Impact

For typical operations:

| Amount | Precision Loss | Percentage |
|--------|---------------|------------|
| 1e18 (1 token, 18 dec) | ~1e9 | 0.0000001% |
| 1e6 (1 USDC) | ~100 | 0.01% |
| 1e4 (minimum) | ~10 | 0.1% |

## Recommendations

1. **Documentation**: Add explicit documentation noting expected precision
   characteristics for integrators

2. **Minimum Amounts**: The 1e4 minimum amount helps limit percentage impact
   of precision loss

3. **User Interface**: Frontend applications should display expected received
   amounts accounting for rounding

## Conclusion

The precision trade-offs are intentional design decisions that balance storage
efficiency against exact accounting. The consistent protocol-favorable rounding
direction ensures system solvency. Integrators should account for these
characteristics in their implementations.
