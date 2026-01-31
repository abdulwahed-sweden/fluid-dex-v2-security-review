# Isolated Collateral Design Notes

## Classification

**Level**: Design Observation
**Status**: Complex but correct
**Impact**: State management complexity

## Overview

The isolated collateral system allows certain tokens to be designated as
"isolated," meaning positions using them as collateral have restricted
borrowing capabilities. This analysis documents the state management
patterns and their implications.

## State Model

### NFT Configuration Flags

```
nftConfig bits:
|-- [0-159]   Owner address
|-- [160-171] Emode
|-- [172-181] Number of positions
|-- [182]     Isolated collateral flag
+-- [183-194] Isolated collateral token index
```

### Related Mappings

```solidity
_isolatedCapConfigs[isolatedTokenIndex][debtTokenIndex] = {
    maxTotalTokenRawBorrow,
    totalTokenRawBorrow
}
```

## State Transitions

### Entering Isolated Mode

When a user first supplies an isolated collateral token:

```
1. Check: No D4 (smart debt) positions exist
2. For each normal borrow position:
   a. Read token raw borrow amount
   b. Add to _isolatedCapConfigs[isolatedToken][debtToken]
   c. Verify total <= max cap
3. Set isolated flag and token index in nftConfig
```

### Exiting Isolated Mode

When the last isolated collateral is withdrawn:

```
1. Scan all positions for remaining isolated token
2. If found: remain in isolated mode
3. If not found:
   a. For each normal borrow position:
      - Subtract from _isolatedCapConfigs
   b. Clear isolated flag in nftConfig
```

## Position Type Restrictions

| Collateral Type | D4 (Smart Debt) Allowed |
|-----------------|-------------------------|
| Standard | Yes |
| Permissionless | Yes |
| Isolated | No |

This restriction is enforced at position creation:

```solidity
if ((nftConfig_ >> BITS_ISOLATED_COLLATERAL_FLAG) & X1 == 1) {
    revert();  // D4 not allowed with isolated collateral
}
```

## Complexity Analysis

### Multi-Position Scanning

The `_beforeCreatingIsolatedCollateralPosition` function iterates through
all positions when entering isolated mode:

```solidity
for (uint256 i_ = 1; i_ <= numberOfPositions_; i_++) {
    // Check for D4 positions (must revert)
    // Update isolated caps for borrow positions
}
```

### Exit Path Complexity

The `_afterIsolatedCollateralFullWithdraw` function performs two passes:

```
Pass 1: Check if isolated token exists in other positions
Pass 2: If not, update isolated caps for all borrow positions
```

## Design Observations

### Position Index Stability

The scanning functions use `positionIndexToSkip_` to exclude the position
being deleted. This is necessary because the scan occurs before the
position deletion swap operation.

### Cap Accounting Consistency

The isolated cap system maintains its own accounting separate from position
caps. Both must be updated consistently during:

- New borrow creation
- Borrow amount changes
- Position deletion
- Isolated mode entry/exit

### D3 Position Handling

D3 (smart collateral) positions may contain isolated tokens as one of
their two underlying tokens. The system handles this by checking both
token indices:

```solidity
if (token0Index == isolatedTokenIndex ||
    token1Index == isolatedTokenIndex) {
    // Position contains isolated collateral
}
```

## Recommendations

1. **State Diagram**: Consider adding a formal state diagram to documentation
   showing valid transitions

2. **Invariant Checks**: Consider adding explicit invariant verification
   in test suites for isolated cap consistency

3. **Event Emission**: Emit events when entering/exiting isolated mode
   for easier off-chain tracking

## Conclusion

The isolated collateral system correctly implements its intended restrictions.
The complexity arises from the need to maintain consistency across multiple
state locations during various operations. The implementation handles edge
cases appropriately, including partial withdrawals and multi-token positions.
