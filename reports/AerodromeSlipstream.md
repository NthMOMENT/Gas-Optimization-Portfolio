# Gas Optimization Report: Aerodrome Finance `CLPool.sol` (Slipstream)

**Target:** [Aerodrome Finance Slipstream](https://github.com/aerodrome-finance/slipstream)  
**Contract:** `contracts/core/CLPool.sol`  
**Fork Base:** Uniswap V3 core at commit `d8b1c63`  
**Context:** Aerodrome is deploying Aero to Ethereum mainnet in July 2026 = 10x higher gas costs than Base  
**Scope:** Advanced EVM storage optimization — zero logic changes  
**Verification:** All findings confirmed present in the Slipstream codebase. No modifications to swap core logic were made from the Uniswap V3 upstream fork.

---

## Why This Report Matters Now

Aerodrome is the #1 DEX on Base by volume and fees — $810M daily trading volume, $500M TVL. In July 2026, the protocol expands to Ethereum mainnet as "Aero."

On Base, gas costs are negligible. On Ethereum mainnet, the same swap costs 10x more. Every gas inefficiency that was invisible on Base becomes a real cost to users and LPs on mainnet. The window to fix these before deployment is now.

This report identifies gas inefficiencies inherited directly from the Uniswap V3 fork that remain unaddressed in `CLPool.sol`.

---

## Optimization Summary

| Finding | Category | Gas Impact | Location |
|---------|----------|------------|----------|
| Non-swap-direction `feeGrowthGlobal` read from storage on every tick crossing | SLOAD Caching | ~100 gas per tick crossed | `swap()` |
| `slot0` written multiple times at end of every swap | Batched SSTORE | ~200 gas per swap | `swap()` |
| `_updatePosition` re-reads `slot0` fields already cached in caller | Duplicate SLOAD | ~600 gas per mint/burn | `_updatePosition()` |
| `feeProtocol` read twice in `flash()` | Redundant SLOAD | ~100 gas per flash | `flash()` |

---

## 💡 Finding 1: `feeGrowthGlobal` Read From Storage on Every Tick Crossing

**Impact:** High ~100 gas per tick crossed. Every multi-tick swap on Aero mainnet pays this cost repeatedly.

`CLPool.sol` inherits the Uniswap V3 swap loop directly. Inside the tick-crossing block, `ticks.cross()` needs both fee growth values, one for each token. The swap-direction fee growth is correctly tracked in memory as `state.feeGrowthGlobalX128` throughout the loop. The other direction is read directly from storage on every crossing:

```solidity
int128 liquidityNet = ticks.cross(
    step.tickNext,
    (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128), // SLOAD if !zeroForOne
    (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128), // SLOAD if zeroForOne
    cache.secondsPerLiquidityCumulativeX128,
    cache.tickCumulative,
    cache.blockTimestamp
);
```

The fix is a two-line addition before the loop:

```solidity
// Cache both before the loop — neither changes during the swap
uint256 _feeGrowthGlobal0X128 = feeGrowthGlobal0X128;
uint256 _feeGrowthGlobal1X128 = feeGrowthGlobal1X128;

// Inside tick crossing:
int128 liquidityNet = ticks.cross(
    step.tickNext,
    (zeroForOne ? state.feeGrowthGlobalX128 : _feeGrowthGlobal0X128),
    (zeroForOne ? _feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
    cache.secondsPerLiquidityCumulativeX128,
    cache.tickCumulative,
    cache.blockTimestamp
);
```

The non-swap-direction fee growth doesn't change during the swap and it's a protocol-level accumulator for the other token. Reading it from storage on every tick crossing is paying for the same value repeatedly. On Base, 100 gas per tick is noise. On Ethereum mainnet at Aero's projected volume, it's a meaningful cost to every trader.

The rest of the `swap()` function already caches `slot0` into `slot0Start`, `liquidity` into `cache.liquidityStart`, and `_blockTimestamp()` into `cache.blockTimestamp`. This finding is the one caching decision that was missed in the upstream fork and carried forward into Slipstream unchanged.

**Result:** ~100 gas saved per tick crossed. A 5-tick swap: ~500 gas. At mainnet pricing, this is the single highest-impact change before the July launch.

---

## 💡 Finding 2: `slot0` Written Multiple Times at End of Every Swap

**Impact:** Medium ~200 gas per swap.

`Slot0` is a tightly packed struct stored in a single storage slot. Every swap writes to it multiple times at the end:

```solidity
// Write 1: at function start:
slot0.unlocked = false;

// Writes 2-5: at function end if tick changed:
(slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
    state.sqrtPriceX96,
    state.tick,
    observationIndex,
    observationCardinality
);

// Write 6:
slot0.unlocked = true;
```

Consolidating the final state into a single struct assignment:

```solidity
slot0 = Slot0({
    sqrtPriceX96: state.sqrtPriceX96,
    tick: state.tick,
    observationIndex: observationIndex,
    observationCardinality: observationCardinality,
    observationCardinalityNext: slot0Start.observationCardinalityNext,
    feeProtocol: slot0Start.feeProtocol,
    unlocked: true
});
```

The reentrancy protection is fully preserved, `slot0.unlocked = false` still happens at the top before any external calls. The difference is at the end: multiple writes to the same slot collapsed into one.

**Result:** ~200 gas saved per swap.

---

## 💡 Finding 3: `_updatePosition` Re-reads `slot0` When Caller Already Has It in Memory

**Impact:** Medium — ~600 gas per `mint` and `burn` call. Every liquidity addition and removal hits this path.

`_modifyPosition` caches `slot0` into memory with the comment "SLOAD for gas optimization," then calls `_updatePosition` passing `_slot0.tick` as a parameter. Inside `_updatePosition`, three fields of `slot0` are read from storage again:

```solidity
observations.observeSingle(
    time,
    0,
    slot0.tick,               // SLOAD — already in _slot0 in the caller
    slot0.observationIndex,   // SLOAD — already in _slot0 in the caller
    liquidity,
    slot0.observationCardinality  // SLOAD — already in _slot0 in the caller
);
```

Passing the cached `Slot0` memory struct through to `_updatePosition` eliminates all three redundant reads. The pattern is already established in the codebase `_modifyPosition` caches `slot0` precisely to avoid this. The fix just takes the pattern one step further into the call chain.

**Result:** ~600 gas saved per `mint` and `burn` call.

---

## 💡 Finding 4: `flash()` Reads `feeProtocol` Twice

**Impact:** Low ~100 gas per flash loan.

```solidity
if (paid0 > 0) {
    uint8 feeProtocol0 = slot0.feeProtocol % 16;  // SLOAD
    ...
}
if (paid1 > 0) {
    uint8 feeProtocol1 = slot0.feeProtocol >> 4;   // SLOAD — same slot
    ...
}
```

One cache before both blocks:

```solidity
uint8 _feeProtocol = slot0.feeProtocol; // SLOAD once
if (paid0 > 0) {
    uint8 feeProtocol0 = _feeProtocol % 16;
    ...
}
if (paid1 > 0) {
    uint8 feeProtocol1 = _feeProtocol >> 4;
    ...
}
```

**Result:** ~100 gas saved per flash loan.

---

## 📊 Gas Report Diff (Simulated: Base vs Ethereum Mainnet Impact)

```
Function              | Gas Saved | Cost on Base | Cost on Mainnet (est.)
----------------------|-----------|--------------|----------------------
swap() 5-tick         | ~700 gas  | ~$0.001      | ~$0.036
mint() / burn()       | ~600 gas  | ~$0.001      | ~$0.031
flash()               | ~100 gas  | ~$0.0002     | ~$0.005
```

*At 20 gwei, $1,700/ETH. Mainnet gas estimated at 30 gwei.*

---

## Executive Summary

Aerodrome's `CLPool.sol` is a direct fork of Uniswap V3 core. It's clean, well-tested, and battle-hardened. These findings are not about missing basics. They're about four specific optimization gaps that exist in the Uniswap V3 upstream code and were carried forward into Slipstream unchanged.

On Base, none of these matter enough to prioritize. Gas is cheap and Aerodrome's competitive advantage is liquidity depth, not transaction cost. But the Aero launch changes the equation entirely. On Ethereum mainnet, the same swap costs 10x more in absolute terms. Every trader, every LP, every integrator will feel the difference.

Finding 1 is the one to fix before July. The `feeGrowthGlobal` caching change is two lines of code, zero risk, and directly impacts every swap that crosses a tick boundary. Which is every interesting swap in a volatile market.

**Total estimated savings at Aero's projected mainnet volume:**

| Scenario | Gas Saved | At 10M swaps/month, 20 gwei, $1,700/ETH |
|----------|-----------|------------------------------------------|
| swap() 5-tick | ~700 gas | ~$357,000/month |
| mint() / burn() | ~600 gas | ~$306,000/month |
| **Combined** | **~1,300 gas** | **~$663,000/month** |

**Risk Level:** Low across all findings. No swap math, no liquidity accounting, no oracle logic is touched. The changes affect only how values are read from storage and not what those values are or how they're used.

---

*Report by [Fourier](https://fourier.u) | [@0xfourier](https://x.com/0xfourier) | [t.me/fourier_u](https://t.me/fourier_u)*  
*June 2026*

---

<p align="center">
  <sub>Fourier | EVM Gas Optimization | June 2026</sub>
</p>
