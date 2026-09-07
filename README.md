# TradingCalendar

**Gives a pool a trading session, and closes it by raising the price of immediacy rather than by reverting.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://trading-calendar-6bp.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/TradingCalendarHook.sol`](src/hooks/TradingCalendarHook.sol)
- **Licence:** MIT

## How it works

Some assets should not be quoted around the clock at the same spread. A pool whose reference market is open for part of the day is being priced blind the rest of the time, and the liquidity sitting in it overnight is providing a free option to anyone with better information about where the asset will open. The obvious hook for this reverts outside session hours, and several published ones do exactly that.

Reverting is the wrong instrument. A pool that reverts is a pool that every router, aggregator and quoting service must special case; it fails after the user has signed; it strands liquidity providers who wanted to exit; and it converts a pricing problem into a liveness problem. Worse, it does not stop informed flow at all, it just moves it to the first second after the open, where it hits the same liquidity at the same stale price.

This hook keeps the pool open and quotable at every instant and expresses the session in the fee instead: - Inside the session, swaps pay `sessionFee`. - Outside it, swaps pay `closedFee`, which is meant to be punitive rather than prohibitive. - Across `rampSeconds` on either side of each boundary the fee moves linearly between the two, so the open and the close are gradients rather than cliffs and there is no single block worth racing to.

The ramp is the part that matters. A cliff at the open creates a race: the first swap after the boundary captures the whole overnight gap at the session spread. A ramp means the trader who wants that gap must choose between paying for it early and waiting for a lower fee while the price moves against them, which is precisely the tradeoff that makes the gap get closed gradually and by more than one participant.

Sessions are expressed in UTC seconds-of-day and may wrap midnight (`open > close` describes an overnight session). `daysMask` selects the days of the week the session runs, bit 0 being Monday. A pool with `daysMask` covering all seven days and a 24-hour session is always in session, which is a valid way to disable the calendar.

Prior art: "New York Trading Hours" and "Trading Hours" hooks revert outside a window. Continuous fee ramps around scheduled events appear in the `UniCast` design for known catalysts. Expressing a *recurring weekly calendar* as a continuous fee surface, with no revert path and no oracle, is the contribution here.

## Prior art

The published calendar hooks ("New York Trading Hours", "Trading Hours") revert outside a window. Continuous fee ramps around a single scheduled event appear in the UniCast design. Expressing a recurring weekly calendar as a continuous fee surface, with no revert path and no oracle, is the contribution here.

## Where it does not help

The calendar is a fixed weekly pattern in UTC. It does not know about holidays, half days, or daylight-saving shifts in the reference market, so a pool tracking an asset with an irregular schedule has to pick a session that is correct most weeks and accept that it is wrong on the exceptions.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    TradingCalendarHook.Config({
        openSecond: /* uint32 */ 0,
        closeSecond: /* uint32 */ 0,
        rampSeconds: /* uint32 */ 0,
        sessionFee: /* uint24 */ 0,
        closedFee: /* uint24 */ 0,
        daysMask: /* uint8 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `openSecond` | `uint32` |  |
| `closeSecond` | `uint32` |  |
| `rampSeconds` | `uint32` | seconds |
| `sessionFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `closedFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `daysMask` | `uint8` | bitmask |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `ClosedFeeBelowSessionFee()` | `closedFee` must be at least `sessionFee`; a session that is cheaper when closed is a configuration error. |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidSecondOfDay()` | A seconds-of-day field was at or above 86400. |
| `NoTradingDays()` | `daysMask` selected no days, which would close the pool permanently. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `RampTooLong()` | The ramp cannot be longer than the session or the gap between sessions it has to fit inside. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # TradingCalendar
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # dynamic-fee, schedule, rwa, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/trading-calendar
cd trading-calendar
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
