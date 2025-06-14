# Issue M-1: CollateralTokens can't be soft-removed 

Source: https://github.com/sherlock-audit/2025-05-usual-eth0-judging/issues/54 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
0xPhantom2, Afriaudit, SamuelTroyDomi, TessKimy, jasonxiale

### Summary

Quoting from the readme:
>CollateralTokens are not removable by design, they can however be soft-removed by changing their pricefeed / upgrade.

And after asking the dev team about the detail of soft-removal:
![Image](https://sherlock-files.ams3.digitaloceanspaces.com/gh-images/f2b0b96b-b4fe-4162-bd87-ff3dbba8eae2)
 
As the image shows, the project will set  oracle price to 0 if the collateral token is removed.
However, changing the oracle price to 0 will block `DaoCollateral.swap` and `DaoCollateral.swapWithPermit`

### Root Cause

Both `DaoCollateral.swap` and `DaoCollateral.swapWithPermit` will call [$.eth0.mint(msg.sender, wadAmountInETH0)](https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/daoCollateral/DaoCollateral.sol#L376) to mint ETH0 token. and in [Eth0.mint](https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/token/Eth0.sol#L129-L166), the function will calculate the total collateral backing in ETH in [Eth0.sol#L146-L161](https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/token/Eth0.sol#L146-L161) by loop all the collateral tokens, and in [Eth0.sol#L149](https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/token/Eth0.sol#L149), the collateral token's price will be queried.
```solidity
128     /// @inheritdoc IEth0
129     function mint(address to, uint256 amount) public {
...
146         uint256 wadCollateralBackingInETH = 0;
147         for (uint256 i = 0; i < collateralTokens.length;) {
148             address collateralToken = collateralTokens[i];
>>> each collateral token's price will be queried here
149             uint256 collateralTokenPriceInETH = uint256(oracle.getPrice(collateralToken));
150             uint8 decimals = IERC20Metadata(collateralToken).decimals();
...
161         }
...
166     }
```

However according to `ClassicalOracle._latestRoundData's` implementation in [ClassicalOracle.sol#L81](https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/oracles/ClassicalOracle.sol#L81), **if the returned price is zero, the function will revert**
```solidity
 71     function _latestRoundData(address token) internal view override returns (uint256, uint256) {
...
 80         (, int256 answer,, uint256 updatedAt,) = priceAggregatorProxy.latestRoundData();
>>> the function will revert here
 81         if (answer <= 0) revert OracleNotWorkingNotCurrent();
...
 90     }
```



### Internal Pre-conditions

CollateralTokens is removed

### External Pre-conditions

None

### Attack Path

1. Collateral token is removed.
2. calling `DaoCollateral.swap`

### Impact

Both `DaoCollateral.swap` and `DaoCollateral.swapWithPermit` will revert after collateral token is soft-removed

### PoC

_No response_

### Mitigation



# Issue M-2: Incorrect Rounding in `stETH` Price Calculation May Break Treasury Invariant 

Source: https://github.com/sherlock-audit/2025-05-usual-eth0-judging/issues/176 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
TessKimy, xiaoming90

### Summary

The protocol relies on an invariant that the total supply of `ETH0` must always be fully backed by an equal or greater ETH-equivalent value of collateral held in the treasury. However, rounding behavior in the `stEthPerToken` function used for pricing `wstETH` rounds down by design. This rounding is not compensated for in the `Usual` contract’s redemption logic, potentially allowing redemptions that break the protocol's core invariant.


### Root Cause

The `wstETH` token uses `stEthPerToken()` to determine the ETH-equivalent price of `wstETH`. This function will call `getPooledEthByShares` function in `stETH` and always rounds *down*, which means the returned price may be slightly less than the true market value:

```solidity
    function getPooledEthByShares(uint256 _sharesAmount) public view returns (uint256) {
        return _sharesAmount
            .mul(_getTotalPooledEther())
            .div(_getTotalShares());
    }
```

In [`Usual`](https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/utils/normalize.sol#L83) , this rounded-down price is used during redemptions in the denominator of the collateral amount calculation:

```solidity
function wadTokenAmountForPrice(uint256 wadStableAmount, uint256 wadPrice, uint8 tokenDecimals)
    internal
    pure
    returns (uint256)
{
    return Math.mulDiv(wadStableAmount, 10 ** tokenDecimals, wadPrice, Math.Rounding.Floor);
}
```

This leads to a slightly larger `wstETH` amount being withdrawn than what is truly equivalent in ETH value. Over time, these rounding losses can accumulate and breach the following invariant:


> ETH0 total supply must be less than or equal to the ETH-equivalent value of collateral held.

### Internal Pre-conditions

1. Round down should happen

### External Pre-conditions

No need

### Attack Path

* Assume:

  * `stEthPerToken()` = 1.199999 ETH (rounded down in stETH side)
  * Actual fair price = 1.200000 ETH
* Redemption of 1 `ETH0`:

  * `wadTokenAmountForPrice(1e18, 1.199999e18)` returns slightly more than it would if 1.200000 were used.
  * User receives slightly more `wstETH` than fair ETH value.

Even if each instance of rounding is minor, these errors may accumulate over time, allowing users to redeem more than the true backing.

### Impact

* **Invariant Violation**: Treasury can end up holding less ETH-equivalent value than the outstanding `ETH0` supply.

This rounding gap must be explicitly accounted for in redemption logic to maintain invariant safety guarantees. This situation is also causing insolvency due to breaking 1:1 ratio.


### Mitigation

Add +1 in redemption:

```diff
    function wadTokenAmountForPrice(uint256 wadStableAmount, uint256 wadPrice, uint8 tokenDecimals)
        internal
        pure
        returns (uint256)
    {
+        return Math.mulDiv(wadStableAmount, 10 ** tokenDecimals, wadPrice + 1, Math.Rounding.Floor);
-        return Math.mulDiv(wadStableAmount, 10 ** tokenDecimals, wadPrice, Math.Rounding.Floor);
    }
```

# Issue M-3: Lack of on-chain deviation check for LST can lead to loss of assets 

Source: https://github.com/sherlock-audit/2025-05-usual-eth0-judging/issues/179 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
0xSlowbug, 0xlemon, Chonkov, Drynooo, Ironsidesec, Olugbenga, TessKimy, j3x, moray5554, xiaoming90

### Summary

-

### Root Cause

-

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

For Liquid Staking Token (LST) collateral supported by the protocol, the protocol cannot enable the depeg check as LST collateral is not a stablecoin (1 wstETH does not peg to 1 ETH). Currently, one (1) wstETH is worth 1.2 ETH. As time goes on, the price of a single wstETH will increase to 1.3, 1.5, 2.0 ETH, and so on. Thus, `TokenOracle.isStablecoin` parameter of LST must always be set to `false` for wstETH and other LST because the current depeg check is only designed for collateral pegged 1:1 to USD or ETH (aka stablecoin/stableasset). In short, the current depeg check does NOT work for LST that accrued value and is not pegged to a specific value (e.g., 1 ETH).

As a result, the problem is that there is no on-chain threshold/deviation check mechanism for wstETH and LST within the in-scope contracts in the current implementation or codebase that will automatically pause contracts or revert the transaction when the LST's price crosses a certain threshold or deviation that might indicate something has gone wrong.

The [Contest's README](https://github.com/sherlock-audit/2025-05-usual-eth0-xiaoming9090?tab=readme-ov-file#q-are-there-any-off-chain-mechanisms-involved-in-the-protocol-eg-keeper-bots-arbitrage-bots-etc-we-assume-these-mechanisms-will-not-misbehave-delay-or-go-offline-unless-otherwise-specified) mentioned that it will monitor the Chainlink's stETH/ETH to detect any black-swan event where the price deviates significantly and will automatically pause the `DaoCollateral`. However, that will not be effective and will not work.

When the Usual's monitoring bot detects an anomaly (e.g., LST's exchange rate gets manipulated upwards) in the price from the on-chain Chainlink's stETH/ETH price feed, it submits a transaction to pause the DAOCollateral. However, malicious users can always front-run Usual's pause transaction and arbitrage/drain/exploit the DAOCollateral before the pause transaction is executed.

There are various ways to arbitrage/drain/exploit the DAOCollateral to steal assets due to the lack of a deviation check:

- This can happen whenever there is a gap in the LIDO's wstETH exchange rate and market price. Assuming the LIDO's wstETH exchange rate has a slight delay in updating and reports a price higher than the on-chain market price. This can happen due to many reasons, such as the market reacting faster as they are already aware of the mass slashing before the oracle validators collect and push an update of the price to LIDO.
- As a result, malicious users can purchase discounted wstETH from the market and swap it for ETH0, and they will receive more ETH0 than expected. Next, they can swap the ETH0 for other LST collateral (e.g., rETH) supported by `DAOCollateral` OR hold it until the LIDO's wstETH exchange rate reflects the latest and correct lower price and then swap their ETH0 back to wstETH. 
- This basically turns Usual protocol into an exit liquidity venue when an adverse event or depeg event happens to a supported LST. Since it is a zero-sum game, the swapper's gain is Usual protocol's loss.

The following is the detailed scenario:

Assume two (2) Liquid Staking Token (LST) is supported by the protocol: wstETH and rETH. Currently, the price of wstETH is fetched from an exchange rate oracle, which is fetched directly from LIDO's wstETH `stEthPerToken()` function.

Assume that LIDO validators have a major issue or outrage, resulting in a mass slashing of LIDO validators. The market price of wstETH drops significantly instantly as traders know that the backing ETH is about to decrease a lot (or already has).

However, the problem with using an exchange rate oracle is that there is a delayed response to adverse events. The oracle price (`stEthPerToken()`) does not change immediately. Under normal circumstances, Lido’s oracle finalizes a new report every 24 hours (225 Ethereum Consensus Layer epochs). Refer to [LIDO's documentation](https://docs.lido.fi/contracts/accounting-oracle/#report-cycle).

Assume the current price of collateral is as follows:

- wstETH = 1.2 ETH
- rETH = 1.2 ETH

There is a major slashing event on the LIDO validator. The market price of wstETH drops to 1 wstETH = 0.5 ETH, but the exchange rate from Lido's oracle or wstETH `stEthPerToken()` function still reflects 1 wstETH = 1.2 ETH.

Bob quickly buys 100 wstETH from the market at a value of 50 ETH (a price of 0.5 ETH). He then swaps it for 120 ETH0 because DaoCollateral uses an exchange rate oracle where the price is still 1 wstETH = 1.2 ETH.

Within the same transaction, redeem 120 ETH0 for 120 ETH worth of another LST (rETH).

Through the arbitration process, Bob earned a total of 70 ETH.

```solidity
120 ETH - 50 ETH (Cost) = 70 ETH (Profit)
```

Bob will bundle a transaction within the same block, coupled with front-running/high priority fee, and flash-loan to ensure that the exploit succeeds before the DAOCollateral is paused.

https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/daoCollateral/DaoCollateral.sol#L61

https://github.com/sherlock-audit/2025-05-usual-eth0/blob/main/eth0-protocol/src/oracles/ClassicalOracle.sol#L21

### Impact

Loss of assets, as mentioned in the scenario above.

### PoC

_No response_

### Mitigation

Some on-chain deviation validation must be implemented for ETH0 to guard against this risk. As mentioned earlier, monitoring the price off-chain, and reacting by submitting pause or countermeasure transaction on-chain is not going to work because there is no guarantee that the transaction will be executed immediately. Anyone can front-run the transaction, and the arrangement of transactions is decentralized and not guaranteed in any manner on Ethereum.

The following are some of the possible solutions:

**Solution 1 - Cross-check with market price**

Automatically pauses if the wstETH exchange rate deviates from the market price (wstETH Chainlink's price feed) by certain threshold. For instance, within the `AbstractOracle.getPrice()` or `AbstractOracle._checkDepegPrice()` function, it can implement logic to perform a deviation check.

Assume Chainlink's stETH or wstETH price feed reports a market price of 0.5 due to a mass slashing or a major hack event observed on LIDO validators. The Chainlink reported market price dropped from earlier 1.2 to the current 0.5.

The deviation check will compare the exchange rate/price fetches from LIDO's wstETH contract, and it will detect that some abnormal or black-swan events have occurred and automatically revert the transaction. Note that the LIDO's wstETH contract might not always reflect the latest market price due to some delay in updating the oracle.

**Solution 2 - Cross-check with expected price**

This approach is adopted by AAVE (See [here](https://etherscan.io/address/0xB4aB0c94159bc2d8C133946E7241368fc2F2a010#code#F24#L221)). LST is expected to grow at a rate of around 5% annually. Thus, if the exchange rate returned from the LIDO's wstETH contract deviates from this, it might indicate that the exchange rate might be manipulated or something has gone wrong, and counter-measures need to be done (in AAVE case, it will cap the price to the `maxRatio`.)

