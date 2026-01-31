# Contracts Analyzed

## Money Market Core

| File | Path | Lines |
|------|------|-------|
| Base Main | `moneyMarket/core/base/main.sol` | 286 |
| Base Helpers | `moneyMarket/core/base/helpers.sol` | 41 |
| Operate Main | `moneyMarket/core/operateModule/main.sol` | 122 |
| Operate Helpers | `moneyMarket/core/operateModule/helpers.sol` | 582 |
| Callback Main | `moneyMarket/core/callbackModule/main.sol` | 482 |
| Callback Helpers | `moneyMarket/core/callbackModule/helpers.sol` | 226 |
| Liquidate Main | `moneyMarket/core/liquidateModule/main.sol` | ~700 |
| Common Helpers | `moneyMarket/core/other/helpers.sol` | 1068 |

## DEX V2 Core

| File | Path | Lines |
|------|------|-------|
| DEX Main | `dexV2/base/core/main.sol` | 294 |
| DEX Helpers | `dexV2/base/core/helpers.sol` | 280 |

## Libraries

| File | Path | Lines |
|------|------|-------|
| Operation Control | `libraries/operationControl.sol` | 30 |
| Pending Transfers | `libraries/pendingTransfers.sol` | 218 |
| Reentrancy Lock | `libraries/reentrancyLock.sol` | 18 |
| BigMath Minified | `libraries/bigMathMinified.sol` | ~200 |
| Safe Transfer | `libraries/safeTransfer.sol` | ~100 |

## Liquidity Layer

| File | Path | Lines |
|------|------|-------|
| User Module | `liquidity/userModule/main.sol` | 1161 |

## Interfaces

| File | Path |
|------|------|
| IFluidLiquidity | `liquidity/interfaces/iLiquidity.sol` |
| IFluidDexV2 | `dexV2/interfaces/iDexV2.sol` |
| IDexV2Callbacks | `dexV2/interfaces/iDexV2Callbacks.sol` |

## Total Lines Analyzed

Approximately 5,500 lines of Solidity code.

## Compiler Version

All contracts use Solidity 0.8.29 with the following considerations:
- Built-in overflow/underflow protection
- Custom errors for gas-efficient reverts
- Assembly blocks for gas optimization
- Transient storage (EIP-1153) support
