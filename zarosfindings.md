# Part 2 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect calculation in CreditDelegationBranch::withdrawUsdTokenFromMarket allows attacker mint any amount of usdz](#H-01)
    - ### [H-02. Incorrect weight assignment in Vault::updateVaultAndCreditDelegationWeight leads to overleveraging vault positions and insolvency](#H-02)
    - ### [H-03. Lack of credit capacity update from VaultRouterBranch::deposit causes DOS in CreditDelegationBranch::depositcreditformarket](#H-03)
    - ### [H-04.  Discrepancy in how Market::wethRewardChangeX18 overallocates weth rewards per vault which allows users to withdraw more weth rewards than they should be allocated](#H-04)
    - ### [H-05. Unclaimed Rewards Are Lost During Re-Staking](#H-05)
    - ### [H-06. Vault's debt can never be settled as Vault::marketsRealizedDebtUsd is always returns zero](#H-06)
    - ### [H-07. CreditDelegationBranch::settlevaultsdebt incorrectly swaps token from marketmakingengine and not directly from Zlpvault which breaks protocol intended functionality](#H-07)
    - ### [H-08. Incorrect swap amount in CreditDelegationBranch::settleVaultsDebt improperly inflates the tokens to swap leading to DOS or/and oversettling vault debt](#H-08)
    - ### [H-09. Locked credit capacity enforcement can be broken leading to vault inadequacy to cover debt position](#H-09)
    - ### [H-10. Incorrect Handling of Redeem Fee Discount in VaultRouterBranch::getIndexTokenSwapRate which leads to a redeem fees being double counted for users who require ](#H-10)
- ## Medium Risk Findings
    - ### [M-01. Incorrect Logic in MarketMakingEngineConfigurationBranch::configureFeeRecipient can cause DOS when attempting to change share value of an existing address](#M-01)
    - ### [M-02.   Attacker can grief vaultrouterbranch::deposit via direct asset token transfers](#M-02)
    - ### [M-03. Hardcoded Fee Discount Flag Prevents Dynamic Whitelisting for Redeem Fee Discounts](#M-03)
- ## Low Risk Findings
    - ### [L-01. Incorrect Naming Convention and Description for Market::wethRewardPerVaultShare](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Zaros

### Dates: Jan 20th, 2025 - Feb 6th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 10
- Medium: 3
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect calculation in CreditDelegationBranch::withdrawUsdTokenFromMarket allows attacker mint any amount of usdz            



## Summary

Market::getCreditCapacityUsd function located in CreditDelegationBranch::withdrawUsdTokenFromMarket, which incorrectly calculates the credit capacity of a market by adding the total debt to the delegated credit instead of subtracting it. This flaw leads to an overestimation of the market's credit capacity, potentially allowing the system to operate in an unsafe state where it cannot cover its liabilities. This issue is particularly severe because it affects the core logic of credit delegation and debt management in the protocol and allows an attacker to mint an amount of usdz that the protocol cannot back 1:1 with usdc which breaks a key protocol invariant.

## Vulnerability Details

CreditDelegationBranch::withdrawUsdTokenFromMarket contains the following code:

````solidity
 /// @notice Mints the requested amount of USD Token to the caller and updates the market's
    /// debt state.
    /// @dev Called by a registered engine to mint USD Token to profitable traders.
    /// @dev USD Token association with an engine's user happens at the engine contract level.
    /// @dev We assume `amount` is part of the market's reported unrealized debt.
    /// @dev Invariants:
    /// The Market of `marketId` MUST exist.
    /// The Market of `marketId` MUST be live.
    /// @param marketId The engine's market id requesting USD Token.
    /// @param amount The amount of USD Token to mint.
    function withdrawUsdTokenFromMarket(uint128 marketId, uint256 amount) external onlyRegisteredEngine(marketId) {
        // loads the market's data and connected vaults
        Market.Data storage market = Market.loadLive(marketId);
        uint256[] memory connectedVaults = market.getConnectedVaultsIds();

        // once the unrealized debt is distributed update credit delegated
        // by these vaults to the market
        Vault.recalculateVaultsCreditCapacity(connectedVaults);

        // cache the market's total debt and delegated credit
        SD59x18 marketTotalDebtUsdX18 = market.getTotalDebt();
        UD60x18 delegatedCreditUsdX18 = market.getTotalDelegatedCreditUsd();

        // calculate the market's credit capacity
        SD59x18 creditCapacityUsdX18 = Market.getCreditCapacityUsd(delegatedCreditUsdX18, marketTotalDebtUsdX18);

        // enforces that the market has enough credit capacity, if it's a listed market it must always have some
        // delegated credit, see Vault.Data.lockedCreditRatio.
        // NOTE: additionally, the ADL system if functioning properly must ensure that the market always has credit
        // capacity to cover USD Token mint requests. Deleverage happens when the perps engine calls
        // CreditDelegationBranch::getAdjustedProfitForMarketId.
        // NOTE: however, it still is possible to fall into a scenario where the credit capacity is <= 0, as the
        // delegated credit may be provided in form of volatile collateral assets, which could go down in value as
        // debt reaches its ceiling. In that case, the market will run out of mintable USD Token and the mm engine
        // must settle all outstanding debt for USDC, in order to keep previously paid USD Token fully backed.
        if (creditCapacityUsdX18.lte(SD59x18_ZERO)) {
            revert Errors.InsufficientCreditCapacity(marketId, creditCapacityUsdX18.intoInt256());
        }

        // uint256 -> UD60x18
        // NOTE: we don't need to scale decimals here as it's known that USD Token has 18 decimals
        UD60x18 amountX18 = ud60x18(amount);

        // prepare the amount of usdToken that will be minted to the perps engine;
        // initialize to default non-ADL state
        uint256 amountToMint = amount;

        // now we realize the added usd debt of the market
        // note: USD Token is assumed to be 1:1 with the system's usd accounting
        if (market.isAutoDeleverageTriggered(delegatedCreditUsdX18, marketTotalDebtUsdX18)) {
            // if the market is in the ADL state, it reduces the requested USD
            // Token amount by multiplying it by the ADL factor, which must be < 1
            UD60x18 adjustedUsdTokenToMintX18 =
                market.getAutoDeleverageFactor(delegatedCreditUsdX18, marketTotalDebtUsdX18).mul(amountX18);

            amountToMint = adjustedUsdTokenToMintX18.intoUint256();
            market.updateNetUsdTokenIssuance(adjustedUsdTokenToMintX18.intoSD59x18());
        } else {
            // if the market is not in the ADL state, it realizes the full requested USD Token amount
            market.updateNetUsdTokenIssuance(amountX18.intoSD59x18());
        }

        // loads the market making engine configuration storage pointer
        MarketMakingEngineConfiguration.Data storage marketMakingEngineConfiguration =
            MarketMakingEngineConfiguration.load();

        // mint USD Token to the perps engine
        UsdToken usdToken = UsdToken(marketMakingEngineConfiguration.usdTokenOfEngine[msg.sender]);
        usdToken.mint(msg.sender, amountToMint);

        // emit an event
        emit LogWithdrawUsdTokenFromMarket(msg.sender, marketId, amount, amountToMint);
    }

The vulnerability is located in the getCreditCapacityUsd function, which is used in the above function to calculate the market's credit capacity.  See Market::getCreditCapacityUsd below:

```solidity
 /// @notice Returns a market's credit capacity in USD based on its delegated credit and total debt.
    /// @param delegatedCreditUsdX18 The market's credit delegated by vaults in USD.
    /// @param totalDebtUsdX18 The market's unrealized + realized debt in USD.
    /// @return creditCapacityUsdX18 The market's credit capacity in USD.
    function getCreditCapacityUsd(
        UD60x18 delegatedCreditUsdX18,
        SD59x18 totalDebtUsdX18
    )
        internal
        pure
        returns (SD59x18 creditCapacityUsdX18)
    {
        creditCapacityUsdX18 = delegatedCreditUsdX18.intoSD59x18().add(totalDebtUsdX18);
    }
```


The incorrect logic is as follows:

```solidity
SD59x18 creditCapacityUsdX18 = Market.getCreditCapacityUsd(delegatedCreditUsdX18, marketTotalDebtUsdX18);
````

Incorrect Logic:
Market::getCreditCapacityUsd function currently adds the total debt (marketTotalDebtUsdX18) to the delegated credit (delegatedCreditUsdX18). This is mathematically incorrect because:

Credit Capacity should represent the remaining capacity of the market to take on additional debt.

The correct formula should be:

Credit Capacity = Delegated Credit - Total Debt

## Impact

Overestimation of Credit Capacity:

The current implementation overestimates the market's credit capacity, making it appear as though the market has more capacity to take on debt than it actually does.

For example:

Delegated Credit = 1000 USD

Total Debt = 300 USD

Correct Credit Capacity = 1000 - 300 = 700 USD

Bugged Credit Capacity = 1000 + 300 = 1300 USD

Unsafe System State:

The system may allow withdrawals or minting of USD tokens even when the market does not have sufficient credit capacity to cover its liabilities.

This could lead to insolvency if the market's debt exceeds its delegated credit.

Inconsistency with Vault::\_updateCreditDelegations:

Vault::\_updateCreditDelegations function correctly calculates credit capacity by subtracting total debt from delegated credit. This inconsistency further highlights the bug in getCreditCapacityUsd. See below:

```solidity
 /// @notice Updates the vault's credit delegations to its connected markets, using the provided cache of connected
    /// markets ids.
    /// @dev This function assumes that the connected markets ids cache is up to date with the stored markets ids. If
    /// this invariant resolves to false, the function will not work as expected.
    /// @dev We assume self.totalCreditDelegationWeight is always greater than zero, as it's verified during
    /// configuration.
    /// @param self The vault storage pointer.
    /// @param connectedMarketsIdsCache The cached connected markets ids.
    /// @param shouldRehydrateCache Whether the connected markets ids cache should be rehydrated or not.
    /// @return rehydratedConnectedMarketsIdsCache The potentially rehydrated connected markets ids cache.
    function _updateCreditDelegations(
        Data storage self,
        uint128[] memory connectedMarketsIdsCache,
        bool shouldRehydrateCache
    )
        private
        returns (uint128[] memory rehydratedConnectedMarketsIdsCache, SD59x18 vaultCreditCapacityUsdX18)
    {
        rehydratedConnectedMarketsIdsCache = new uint128[](connectedMarketsIdsCache.length);
        // cache the vault id
        uint128 vaultId = self.id;

        // cache the connected markets length
        uint256 connectedMarketsConfigLength = self.connectedMarkets.length;

        // loads the connected markets storage pointer by taking the last configured market ids uint set
        EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1];

        // loop over each connected market id that has been cached once again in order to update this vault's
        // credit delegations
        for (uint256 i; i < connectedMarketsIdsCache.length; i++) {
            // rehydrate the markets ids cache if needed
            if (shouldRehydrateCache) {
                rehydratedConnectedMarketsIdsCache[i] = connectedMarkets.at(i).toUint128();
            } else {
                rehydratedConnectedMarketsIdsCache[i] = connectedMarketsIdsCache[i];
            }

            // loads the memory cached market id
            uint128 connectedMarketId = rehydratedConnectedMarketsIdsCache[i];

            // load the credit delegation to the given market id
            CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, connectedMarketId);

            // cache the previous credit delegation value
            UD60x18 previousCreditDelegationUsdX18 = ud60x18(creditDelegation.valueUsd);

            // cache the latest credit delegation share of the vault's credit capacity
            uint128 totalCreditDelegationWeightCache = self.totalCreditDelegationWeight;

            if (totalCreditDelegationWeightCache != 0) {
                // get the latest credit delegation share of the vault's credit capacity
                UD60x18 creditDelegationShareX18 =
                    ud60x18(creditDelegation.weight).div(ud60x18(totalCreditDelegationWeightCache));

                // stores the vault's total credit capacity to be returned
                vaultCreditCapacityUsdX18 = getTotalCreditCapacityUsd(self);

                // if the vault's credit capacity went to zero or below, we set its credit delegation to that market
                // to zero
                UD60x18 newCreditDelegationUsdX18 = vaultCreditCapacityUsdX18.gt(SD59x18_ZERO)
                    ? vaultCreditCapacityUsdX18.intoUD60x18().mul(creditDelegationShareX18)
                    : UD60x18_ZERO;

                // calculate the delta applied to the market's total delegated credit
                UD60x18 creditDeltaUsdX18 = newCreditDelegationUsdX18.sub(previousCreditDelegationUsdX18);

                // loads the market's storage pointer and update total delegated credit
                Market.Data storage market = Market.load(connectedMarketId);
                market.updateTotalDelegatedCredit(creditDeltaUsdX18);

                // if new credit delegation is zero, we clear the credit delegation storage
                if (newCreditDelegationUsdX18.isZero()) {
                    creditDelegation.clear();
                } else {
                    // update the credit delegation stored usd value
                    creditDelegation.valueUsd = newCreditDelegationUsdX18.intoUint128();
                }
            }
```

The correct calculation is where the vaultCreditCapacityUsdX18 is calculated using Vault::getTotalCreditCapacityUsd. See function below:

```solidity
 /// @notice Returns the vault's total credit capacity allocated to the connected markets.
    /// @dev The vault's total credit capacity is adjusted by its the credit ratio of its underlying collateral asset.
    /// @param self The vault storage pointer.
    /// @return creditCapacityUsdX18 The vault's total credit capacity in USD.
    function getTotalCreditCapacityUsd(Data storage self) internal view returns (SD59x18 creditCapacityUsdX18) {
        // load the collateral configuration storage pointer
        Collateral.Data storage collateral = self.collateral;

        // fetch the zlp vault's total assets amount
        UD60x18 totalAssetsX18 = ud60x18(IERC4626(self.indexToken).totalAssets());

        // calculate the total assets value in usd terms
        UD60x18 totalAssetsUsdX18 = collateral.getAdjustedPrice().mul(totalAssetsX18);

        // calculate the vault's credit capacity in usd terms
        creditCapacityUsdX18 = totalAssetsUsdX18.intoSD59x18().sub(getTotalDebt(self));
    }
```

As seen above, the credit capacity is calculated with Credit Capacity = Delegated Credit - Total Debt .

This vulnerability has a critical impact on the protocol's financial stability and security. Specifically:

Financial Losses:

The protocol may allow withdrawals or minting of USD tokens beyond the market's actual capacity, leading to undercollateralization and potential insolvency.

System Instability:

The incorrect credit capacity calculation could cause the system to operate in an unsafe state, where it cannot cover its liabilities.

Exploitation Risk:

Malicious actors could exploit this bug to drain funds from the protocol by artificially inflating the market's credit capacity.

## Proof Of Code (POC)

Running the following test will show the exploit:

```solidity
 function test_creditcapacitywrongcalculation( 
        uint256 marketId,
        uint256 amount,
        uint256 amounttodepositcredit,
        uint128 vaultId,
        uint64 anyamount
    )
        external
    {
         amounttodepositcredit = bound({ x: amount, min: 1, max: type(uint32).max });

        PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);
       vaultId = uint128(bound(vaultId, INITIAL_VAULT_ID, FINAL_VAULT_ID));
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);

        vm.assume(fuzzVaultConfig.asset != address(usdc));

        deal({ token: address(fuzzVaultConfig.asset), to: address(fuzzMarketConfig.engine), give: amounttodepositcredit*10 });
        changePrank({ msgSender: address(fuzzMarketConfig.engine) });

        //c first user deposits credit into the market
        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, amounttodepositcredit);

        uint256 mmBalance = IERC20(fuzzVaultConfig.asset).balanceOf(address(marketMakingEngine));


        uint128 depositFee = uint128(vaultsConfig[fuzzVaultConfig.vaultId].depositFee);

        _setDepositFee(depositFee, fuzzVaultConfig.vaultId);

       //c all code below makes the deposit into the vault that checks realized debt
        address user = users.naruto.account;
        deal(fuzzVaultConfig.asset, user, 100e18);

        // perform the deposit
        vm.startPrank(user);
        marketMakingEngine.deposit(vaultId, 1e18, 0, "", false);
        vm.stopPrank();

        vm.prank(address(fuzzMarketConfig.engine));
         marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, amounttodepositcredit); 

        SD59x18 marketdebt = marketMakingEngine.workaround_getTotalMarketDebt(fuzzMarketConfig.marketId);
        console.log(marketdebt.unwrap());
       
      uint256 anyamount = bound(uint256(anyamount), 1, type(uint64).max);

        vm.prank(address(fuzzMarketConfig.engine));
         marketMakingEngine.withdrawUsdTokenFromMarket(fuzzMarketConfig.marketId, anyamount); 

          UD60x18 totaldelegatedcreditusd =  marketMakingEngine.workaround_getTotalDelegatedCreditUsd(fuzzMarketConfig.marketId); 
         console.log(totaldelegatedcreditusd.unwrap());

         SD59x18 expectedcreditcapacity = totaldelegatedcreditusd.intoSD59x18().sub(marketdebt);

          SD59x18 actualcreditcapacity = totaldelegatedcreditusd.intoSD59x18().add(marketdebt);

        // it should deposit credit for market
        assert(expectedcreditcapacity.unwrap() != actualcreditcapacity.unwrap());
     
        
    }
```

## Tools Used

Manual Review, Foundry

## Recommendations

Update Market::getCreditCapacityUsd as follows:

```solidity
 /// @notice Returns a market's credit capacity in USD based on its delegated credit and total debt.
    /// @param delegatedCreditUsdX18 The market's credit delegated by vaults in USD.
    /// @param totalDebtUsdX18 The market's unrealized + realized debt in USD.
    /// @return creditCapacityUsdX18 The market's credit capacity in USD.
    function getCreditCapacityUsd(
        UD60x18 delegatedCreditUsdX18,
        SD59x18 totalDebtUsdX18
    )
        internal
        pure
        returns (SD59x18 creditCapacityUsdX18)
    {
        creditCapacityUsdX18 = delegatedCreditUsdX18.intoSD59x18().sub(totalDebtUsdX18);
    }
```

## <a id='H-02'></a>H-02. Incorrect weight assignment in Vault::updateVaultAndCreditDelegationWeight leads to overleveraging vault positions and insolvency            



## Summary

A critical vulnerability exists in the Vault::updateVaultAndCreditDelegationWeight, where the weight assigned to each connected market is incorrectly set to the total assets of the vault instead of a proportional share. This results in the total delegated credit across all markets exceeding the vault's total assets, leading to overleveraging and potential insolvency.

## Vulnerability Details

Vault::updateVaultAndCreditDelegationWeight is called  in Vault::`Vault::recalculateVaultsCreditCapacity` and its job is to set weights of each connected market proportional to the total weight of the vault. See function below:

```solidity
  /// @notice Update the vault and credit delegation weight
    /// @param self The vault storage pointer.
    /// @param connectedMarketsIdsCache The cached connected markets ids.
    function updateVaultAndCreditDelegationWeight(
        Data storage self,
        uint128[] memory connectedMarketsIdsCache
    )
        internal
    {
        // cache the connected markets length
        uint256 connectedMarketsConfigLength = self.connectedMarkets.length;

        // loads the connected markets storage pointer by taking the last configured market ids uint set
        EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1];

        // get the total of shares
        uint128 newWeight = uint128(IERC4626(self.indexToken).totalAssets());

        for (uint256 i; i < connectedMarketsIdsCache.length; i++) {
            // load the credit delegation to the given market id
            CreditDelegation.Data storage creditDelegation =
                CreditDelegation.load(self.id, connectedMarkets.at(i).toUint128());

            // update the credit delegation weight
            creditDelegation.weight = newWeight;
        }

        // update the vault weight
        self.totalCreditDelegationWeight = newWeight;
    }
```

The function assigns the total assets of the vault (newWeight) to each connected market individually:

```solidity
creditDelegation.weight = newWeight;
```

If there are multiple connected markets, each market is assigned the full weight of the vault, leading to an overestimation of the total delegated credit.

Example:
Vault Total Assets = 1000 wETH
Connected Markets = 2
Each market is assigned a weight of 1000 wETH.

Total delegated credit = 1000 + 1000 = 2000wETH (exceeds the vault's total assets).

## Impact

Overleveraging: The total delegated credit across all markets exceeds the vault's total assets, putting the system in an overleveraged and unsafe state.

Financial Instability: If one or more markets face losses, the vault may not have enough assets to cover the losses, leading to insolvency.

Exploitation Risk: Malicious actors could exploit this bug to overleverage the system, resulting in significant financial losses.

Loss of User Funds: Users who deposit collateral into the vault may lose their funds if the vault becomes insolvent due to overleveraging.

## Proof Of Code (POC)

I ran the following POC to prove the exploit:

```solidity
function testFuzz_marketweightsharescalculationerror(
        uint256 vaultId1,
        uint256 marketId1,
        uint256 vaultId2,
        uint256 marketId2
    )
        external
    {
        vm.stopPrank();
        VaultConfig memory fuzzVaultConfig1 = getFuzzVaultConfig(vaultId1);
         vm.assume(fuzzVaultConfig1.asset != address(usdc));
        PerpMarketCreditConfig memory fuzzMarketConfig1 = getFuzzPerpMarketCreditConfig(marketId1);


         VaultConfig memory fuzzVaultConfig2 = getFuzzVaultConfig(vaultId2);
         vm.assume(fuzzVaultConfig2.asset != address(usdc));
        PerpMarketCreditConfig memory fuzzMarketConfig2 = getFuzzPerpMarketCreditConfig(marketId2);
        vm.assume(fuzzMarketConfig1.marketId != fuzzMarketConfig2.marketId);
        vm.assume(fuzzVaultConfig1.vaultId != fuzzVaultConfig2.vaultId);

        //c set up 2 vaults and connect them to 2 markets
        uint256[] memory marketIds = new uint256[](2);
        marketIds[0] = fuzzMarketConfig1.marketId;
        marketIds[1] = fuzzMarketConfig2.marketId;

        uint256[] memory vaultIds = new uint256[](2);
        vaultIds[0] = fuzzVaultConfig1.vaultId;
        vaultIds[1] = fuzzVaultConfig2.vaultId;

        vm.prank(users.owner.account);
        marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);

        
       
          address user = users.naruto.account;
        deal(fuzzVaultConfig1.asset, user, 100e18);
        deal(fuzzVaultConfig2.asset, user, 100e18);

        //c deposit assets into both vaults so that each vaults credit capacity is 1e18 which gives each market a totalcreditdelegatedweight of 2e18 which is where the bug is.

        //c have to deposit twice because updates from Vault::recalculateVaultsCreditCapacity when called in VaultRouterBranch::deposit is called at the start so the deposited collateral isnt reflected . could have also called CreditDelegationBranch::updateVaultCreditCapacity but I chose this way instead

        vm.startPrank(user);
        marketMakingEngine.deposit(fuzzVaultConfig1.vaultId, 1e18, 0, "", false);
        marketMakingEngine.deposit(fuzzVaultConfig1.vaultId, 1e18, 0, "", false);
        marketMakingEngine.deposit(fuzzVaultConfig2.vaultId, 1e18, 0, "", false);
        marketMakingEngine.deposit(fuzzVaultConfig2.vaultId, 1e18, 0, "", false);
        vm.stopPrank();
       
        //c deposit credit of 2e18 into both markets so totaldelegatedcredit of both markets is 4e18 which is more than the credit capacity of both vaults. this proves that the market is allocated more credit than the vault can handle which leaves zaros in an overleveraged position
        deal({ token: address(fuzzVaultConfig1.asset), to: address(fuzzMarketConfig1.engine), give: 10e18 });
        deal({ token: address(fuzzVaultConfig2.asset), to: address(fuzzMarketConfig2.engine), give: 10e18 });
        vm.prank( address(fuzzMarketConfig1.engine));
        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig1.marketId, fuzzVaultConfig1.asset, 2e18);
        vm.prank( address(fuzzMarketConfig2.engine));
        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig2.marketId, fuzzVaultConfig2.asset, 2e18);


        //c get the totalmarketdebt of each market in usd
        uint256 totalmarketdebt;
        for(uint i = 0; i < marketIds.length; i++){
            SD59x18 marketdebt = marketMakingEngine.workaround_getTotalMarketDebt(uint128(marketIds[i]));
            console.log(marketdebt.unwrap());
            totalmarketdebt += uint256(marketdebt.unwrap());
        }
        console.log(totalmarketdebt);

        /*c get the total asset value in usd. to do this, i included the following function in vault.sol.
        //c for testing purposes
    function getTotalAssetsUsd(Data storage self) internal view returns (UD60x18 totalAssetsUsdX18) {
        // load the collateral configuration storage pointer
        Collateral.Data storage collateral = self.collateral;

        // fetch the zlp vault's total assets amount
        UD60x18 totalAssetsX18 = ud60x18(IERC4626(self.indexToken).totalAssets());

        // calculate the total assets value in usd terms
         totalAssetsUsdX18 = collateral.getAdjustedPrice().mul(totalAssetsX18);

        I also included this function in VaultRouterBranch.sol:
        //c for testing purposes
    function gettotalassetsusd(uint128 vaultId) external view returns (uint256) {
        Vault.Data storage vault = Vault.loadLive(vaultId);
        return vault.getTotalAssetsUsd().intoUint256();
    }   

    After registering the selector of this function in TreeProxyUtils.sol, it should work as expected


        */
        uint256 totalassetvalue;
        for (uint i = 0; i< vaultIds.length; i++){
            uint256 assetvalueusd = marketMakingEngine.gettotalassetsusd(uint128(vaultIds[i]));
        totalassetvalue += assetvalueusd;
        }
        console.log(totalassetvalue);

        //c shows that the total usd value of the debt after perpengine has called CreditDelegationBranch::depositCreditForMarket is more than the total usd value of the assets in the vaults which proves that zaros will be overleveraged
        assert(totalmarketdebt > totalassetvalue);

        //c I have not included the call to CreditDelegationBranch::withdrawUsdTokenFromMarket because as I explained in my previous finding, there is a bug also in that function that allows any user to bypass the creditcapacity check and withdraw any amount of usdz so that wouldn't necessarily prove anything. 


}
```

## Tools Used

Manual Review, Foundry

## Recommendations

Update Vault::updateVaultAndCreditDelegationWeight function to assign a proportional share of the vault's total assets to each connected market. This can be achieved by dividing the vault's total assets by the number of connected markets.

```solidity
 function updateVaultAndCreditDelegationWeight(
        Data storage self,
        uint128[] memory connectedMarketsIdsCache
    )
        internal
    {
        // cache the connected markets length
        uint256 connectedMarketsConfigLength = self.connectedMarkets.length;

        // loads the connected markets storage pointer by taking the last configured market ids uint set
        EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1];

        // get the total of shares
        uint128 newWeight = uint128(IERC4626(self.indexToken).totalAssets());


          // Calculate the weight per market
          uint128 weightPerMarket = newWeight / uint128(connectedMarketsIdsCache.length);

          for (uint256 i; i < connectedMarketsIdsCache.length; i++) {
         // Load the credit delegation to the given market id
         CreditDelegation.Data storage creditDelegation =
         CreditDelegation.load(self.id, connectedMarkets.at(i).toUint128());

        // Update the credit delegation weight
          creditDelegation.weight = weightPerMarket;
          }

       // update the vault weight
        self.totalCreditDelegationWeight = newWeight;
    }
```

Add a check to ensure that the number of connected markets is greater than zero before dividing.

```solidity
require(connectedMarketsIdsCache.length > 0, "No connected markets");
```

## <a id='H-03'></a>H-03. Lack of credit capacity update from VaultRouterBranch::deposit causes DOS in CreditDelegationBranch::depositcreditformarket            



## Summary

In the VaultRouterBranch::deposit function, the Vault::recalculateVaultsCreditCapacity function is called before the deposited collateral is accounted for in the vault's total assets. This results in incorrect credit delegation calculations, causing the depositCreditForMarket function to fail even when the vault has sufficient collateral. This bug disrupts the protocol's core functionality, preventing perp engine from depositing credit into markets.

## Vulnerability Details

The vulnerability is located in the VaultRouterBranch::deposit function, specifically in the sequence of operations where Vault::recalculateVaultsCreditCapacity is called before the deposited collateral is added to the vault's total assets. See  VaultRouterBranch::deposit:

```solidity
/// @notice Deposits a given amount of collateral assets into the provided vault in exchange for index tokens.
    /// @dev Invariants involved in the call:
    /// The total deposits MUST not exceed the vault after the deposit.
    /// The number of received shares MUST be greater than or equal to minShares.
    /// The number of received shares MUST be > 0 even when minShares = 0.
    /// The Vault MUST exist.
    /// The Vault MUST be live.
    /// If the vault enforces fees then calculated deposit fee must be non-zero.
    /// No tokens should remain stuck in this contract.
    /// @param vaultId The vault identifier.
    /// @param assets The amount of collateral to deposit, in the underlying ERC20 decimals.
    /// @param minShares The minimum amount of index tokens to receive in 18 decimals.
    /// @param referralCode The referral code to use.
    /// @param isCustomReferralCode True if the referral code is a custom referral code.
    function deposit(
        uint128 vaultId,
        uint128 assets,
        uint128 minShares,
        bytes memory referralCode,
        bool isCustomReferralCode
    )
        external
    {
        if (assets == 0) revert Errors.ZeroInput("assets");

        // load the mm engine configuration from storage
        MarketMakingEngineConfiguration.Data storage marketMakingEngineConfiguration =
            MarketMakingEngineConfiguration.load();

        // enforce whitelist if enabled
        address whitelistCache = marketMakingEngineConfiguration.whitelist;
        if (whitelistCache != address(0)) {
            if (!Whitelist(whitelistCache).verifyIfUserIsAllowed(msg.sender)) {
                revert Errors.UserIsNotAllowed(msg.sender);
            }
        }

        // fetch storage slot for vault by id, vault must exist with valid collateral
        Vault.Data storage vault = Vault.loadLive(vaultId);
        if (!vault.collateral.isEnabled) revert Errors.VaultDoesNotExist(vaultId);

        // define context struct and get vault collateral asset
        DepositContext memory ctx;
        ctx.vaultAsset = vault.collateral.asset;

        // prepare the `Vault::recalculateVaultsCreditCapacity` call
        uint256[] memory vaultsIds = new uint256[]();
        vaultsIds[0] = uint256(vaultId);

        // recalculates the vault's credit capacity
        // note: we need to update the vaults credit capacity before depositing new assets in order to calculate the
        // correct conversion rate between assets and shares, and to validate the involved invariants accurately
        Vault.recalculateVaultsCreditCapacity(vaultsIds);

        // load the referral module contract
        ctx.referralModule = IReferral(marketMakingEngineConfiguration.referralModule);

        // register the given referral code
        if (referralCode.length != 0) {
            ctx.referralModule.registerReferral(
                abi.encode(msg.sender), msg.sender, referralCode, isCustomReferralCode
            );
        }

        // cache the vault assets decimals value for gas savings
        ctx.vaultAssetDecimals = vault.collateral.decimals;

        // uint256 -> ud60x18 18 decimals
        ctx.assetsX18 = Math.convertTokenAmountToUd60x18(ctx.vaultAssetDecimals, assets);

        // cache the deposit fee
        ctx.vaultDepositFee = ud60x18(vault.depositFee);

        // if deposit fee is zero, skip needless processing
        if (ctx.vaultDepositFee.isZero()) {
            ctx.assetsMinusFees = assets;
        } else {
            // otherwise calculate the deposit fee
            ctx.assetFeesX18 = ctx.assetsX18.mul(ctx.vaultDepositFee);

            // ud60x18 -> uint256 asset decimals
            ctx.assetFees = Math.convertUd60x18ToTokenAmount(ctx.vaultAssetDecimals, ctx.assetFeesX18);

            // invariant: if vault enforces fees then calculated fee must be non-zero
            if (ctx.assetFees == 0) revert Errors.ZeroFeeNotAllowed();

            // enforce positive amount left over after deducting fees
            ctx.assetsMinusFees = assets - ctx.assetFees;
            if (ctx.assetsMinusFees == 0) revert Errors.DepositTooSmall();
        }

        // transfer tokens being deposited minus fees into this contract
        IERC20(ctx.vaultAsset).safeTransferFrom(msg.sender, address(this), ctx.assetsMinusFees);

        // transfer fees from depositor to fee recipient address
        if (ctx.assetFees > 0) {
            IERC20(ctx.vaultAsset).safeTransferFrom(
                msg.sender, marketMakingEngineConfiguration.vaultDepositAndRedeemFeeRecipient, ctx.assetFees
            );
        }

        // increase vault allowance to transfer tokens minus fees from this contract to vault
        address indexTokenCache = vault.indexToken;
        IERC20(ctx.vaultAsset).approve(indexTokenCache, ctx.assetsMinusFees);

        // then perform the actual deposit
        // NOTE: the following call will update the total assets deposited in the vault
        // NOTE: the following call will validate the vault's deposit cap
        // invariant: no tokens should remain stuck in this contract
        ctx.shares = IERC4626(indexTokenCache).deposit(ctx.assetsMinusFees, msg.sender);

        // assert min shares minted
        if (ctx.shares < minShares) revert Errors.SlippageCheckFailed(minShares, ctx.shares);

        // invariant: received shares must be > 0 even when minShares = 0; no donation allowed
        if (ctx.shares == 0) revert Errors.DepositMustReceiveShares();

        // emit an event
        emit LogDeposit(vaultId, msg.sender, ctx.assetsMinusFees);
    }
```

The incorrect logic lies in the sequence of operations where in VaultRouterBranch::deposit, Vault::recalculateVaultsCreditCapacity is called before the deposited collateral is transferred to the vault.This means the vault's total assets used in credit capacity calculations including Vault::updateVaultAndCreditDelegationWeight and especially Vault::\_updateCreditDelegations where market::totaldelegatedcreditusd is updated do not include the newly deposited collateral.

Vault::recalculateVaultsCreditCapacity function updates the credit delegation for connected markets based on the vault's current total assets (excluding the new deposit).As a result, the Market::totalDelegatedCreditUsd value is not updated to reflect the new deposit, leading to incorrect credit capacity calculations.

CreditDelegation::depositCreditForMarket function checks if Market::totalDelegatedCreditUsd is greater than zero before allowing credit deposits. Due to the incorrect credit delegation calculations, this check fails even when the vault has sufficient collateral, causing the function to revert with Errors.NoDelegatedCredit.

## Proof Of Code (POC)

```Solidity
function test\_doswhenuserhasdepositedbutdelegatedmarketcreditis0(
uint256 vaultId,
uint256 marketId
) external {

  vm.stopPrank();
    VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
     vm.assume(fuzzVaultConfig.asset != address(usdc));
    PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);


    //c set up 1 vault and 1 market
    uint256[] memory marketIds = new uint256[](1);
    marketIds[0] = fuzzMarketConfig.marketId;
   

    uint256[] memory vaultIds = new uint256[](1);
    vaultIds[0] = fuzzVaultConfig.vaultId;
    

    vm.prank(users.owner.account);
    marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);


    //c set delegatedmarketcredit to 0 using workaround
 marketMakingEngine.workaround_updateMarketTotalDelegatedCreditUsd(fuzzMarketConfig.marketId, 0);


    //c deposit assets into vault to increase the vaults credit capacity to 1e18 as well as market::totaldelegatedcreditusd to 1e18
      address user = users.naruto.account;
    deal(fuzzVaultConfig.asset, user, 100e18);
  vm.startPrank(user);
    marketMakingEngine.deposit(fuzzVaultConfig.vaultId, 1e18, 0, "", false);
    vm.stopPrank();



    //c DOS occurs here due to the lack of correct update of totaldelegatedcreditusd. market has delegated credit but credit capacity was not updated successfully
    vm.prank(address(fuzzMarketConfig.engine));
    vm.expectRevert(abi.encodeWithSelector(Errors.NoDelegatedCredit.selector, fuzzMarketConfig.marketId));
     marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, 1e17); 
```

}

## Impact

Disruption of Core Functionality: Perp Engine cannot deposit credit into markets, even when the vault has sufficient collateral. This prevents the protocol from functioning as intended, disrupting trading and liquidity provision. Traders and liquidity providers may be unable to execute trades or earn rewards, leading to potential financial losses.

## Tools Used

Manual Review, Foundry

## Recommendations

Update CreditDelegationBranch::depositcreditformarket as follows:

```solidity
function depositCreditForMarket(
    uint128 marketId,
    address collateralAddr,
    uint256 amount
)
    external
    onlyRegisteredEngine(marketId)
{
    if (amount == 0) revert Errors.ZeroInput("amount");

    // Loads the collateral's data storage pointer, must be enabled
    Collateral.Data storage collateral = Collateral.load(collateralAddr);
    collateral.verifyIsEnabled();

    // Loads the market's data storage pointer, must have delegated credit
    Market.Data storage market = Market.loadLive(marketId);

    // Get the connected vaults for the market
    uint256[] memory connectedVaults = market.getConnectedVaultsIds();

    // Recalculate the credit capacity for the connected vaults
    Vault.recalculateVaultsCreditCapacity(connectedVaults);

    // Ensure the market has delegated credit after recalculating
    if (market.getTotalDelegatedCreditUsd().isZero()) {
        revert Errors.NoDelegatedCredit(marketId);
    }

    // uint256 -> UD60x18 scaling decimals to zaros internal precision
    UD60x18 amountX18 = collateral.convertTokenAmountToUd60x18(amount);

    // Caches the usdToken address
    address usdToken = MarketMakingEngineConfiguration.load().usdTokenOfEngine[msg.sender];

    // Caches the usdc
    address usdc = MarketMakingEngineConfiguration.load().usdc;

    // Note: storage updates must occur using zaros internal precision
    if (collateralAddr == usdToken) {
        // If the deposited collateral is USD Token, it reduces the market's realized debt
        market.updateNetUsdTokenIssuance(unary(amountX18.intoSD59x18()));
    } else {
        if (collateralAddr == usdc) {
            market.settleCreditDeposit(address(0), amountX18);
        } else {
            // Deposits the received collateral to the market to be distributed to vaults
            // to be settled in the future
            market.depositCredit(collateralAddr, amountX18);
        }
    }

    // Transfers the margin collateral asset from the registered engine to the market making engine
    // NOTE: The engine must approve the market making engine to transfer the margin collateral asset, see
    // PerpsEngineConfigurationBranch::setMarketMakingEngineAllowance
    // Note: transfers must occur using token native precision
    IERC20(collateralAddr).safeTransferFrom(msg.sender, address(this), amount);

    // Emit an event
    emit LogDepositCreditForMarket(msg.sender, marketId, collateralAddr, amount);
}
```

Vault::recalculateVaultsCreditCapacity call ensures that the Market::totalDelegatedCreditUsd value is updated to reflect the latest vault balances. This prevents the NoDelegatedCredit error because the market will have the correct delegated credit value. If this is the first deposit into a vault, Vault::recalculateVaultsCreditCapacity call ensures that the market's delegated credit is updated to reflect the new deposit. This allows the CreditDelegationBranch::depositCreditForMarket function to proceed without reverting.

The recommendation helps maintains consistency across functions. CreditDelegationBranch::withdrawUsdTokenFromMarket function already includes this logic, so adding it to CreditDelegationBranch::depositCreditForMarket ensures consistency across similar operations. This makes the codebase more predictable and easier to maintain. Also, if multiple deposits or withdrawals occur in quick succession, the new solution ensures that the credit delegation state is always up-to-date before processing any credit-related operations. This reduces the risk of race conditions or stale state issues.

## <a id='H-04'></a>H-04.  Discrepancy in how Market::wethRewardChangeX18 overallocates weth rewards per vault which allows users to withdraw more weth rewards than they should be allocated            



## Summary

There is a logical discrepancy in how Market::wethRewardChangeX18 is computed in Market::getVaultAccumulatedValues. Unlike the other reward values (realized debt, unrealized debt, and USDC credit), Market::wethRewardChangeX18 is not multiplied by the vault’s share of total delegated credit (vaultCreditShareX18). Additionally, the code does not check whether the vault has previously distributed rewards (i.e., if lastVaultDistributedWethRewardPerShareX18 is zero) before calculating changes.

## Vulnerability Details

Incorrect Reward Calculation:

The function correctly multiplies other reward deltas (realizedDebtChangeUsdX18, unrealizedDebtChangeUsdX18, usdcCreditChangeX18) by the vault’s credit share (vaultCreditShareX18) as the function is called getVaultAccumulatedValues and the natspec is as follows:

```solidity
/// @notice Calculates the latest debt, usdc credit and weth reward values a vault is entitled to receive from the
    /// market since its last accumulation event.
```

However, wethRewardChangeX18 is calculated only as:

```solidity
wethRewardChangeX18 = ud60x18(self.wethRewardPerVaultShare)
    .sub(lastVaultDistributedWethRewardPerShareX18);
```

without factoring in vaultCreditShareX18. This discrepancy calculates the wethRewardChange for the market instead of the reward the vault is entitled to.

No Check for Zero Distribution:

The other reward calculations do something like:

```solidity

realizedDebtChangeUsdX18 = !lastVaultDistributedRealizedDebtUsdPerShareX18.isZero()
  ? ...
  : SD59x18_ZERO;

```

This condition bypasses calculations if the vault has not previously distributed any rewards. By contrast, wethRewardChangeX18 has no similar check, which may lead to incorrect initial distribution logic.

This leads to further discrepancies as the following lines are in Vault::\_recalculateConnectedMarketsState:

```solidity
 // prevent division by zero
            if (!market.getTotalDelegatedCreditUsd().isZero()) {
                // get the vault's accumulated debt, credit and reward changes from the market to update its stored
                // values
                (
                    ctx.realizedDebtChangeUsdX18,
                    ctx.unrealizedDebtChangeUsdX18,
                    ctx.usdcCreditChangeX18,
                    ctx.wethRewardChangeX18
                ) = market.getVaultAccumulatedValues( 
                    ud60x18(creditDelegation.valueUsd), 
                    sd59x18(creditDelegation.lastVaultDistributedRealizedDebtUsdPerShare),
                    sd59x18(creditDelegation.lastVaultDistributedUnrealizedDebtUsdPerShare),
                    ud60x18(creditDelegation.lastVaultDistributedUsdcCreditPerShare),
                    ud60x18(creditDelegation.lastVaultDistributedWethRewardPerShare)
                );
            }

            // if there's been no change in any of the returned values, we can iterate to the next
            // market id
            if (
                ctx.realizedDebtChangeUsdX18.isZero() && ctx.unrealizedDebtChangeUsdX18.isZero()
                    && ctx.usdcCreditChangeX18.isZero() && ctx.wethRewardChangeX18.isZero()
            ) {
                continue;
            }
         

  // update the vault's state by adding its share of the market's latest state variables
            vaultTotalRealizedDebtChangeUsdX18 = vaultTotalRealizedDebtChangeUsdX18.add(ctx.realizedDebtChangeUsdX18);
            vaultTotalUnrealizedDebtChangeUsdX18 =
                vaultTotalUnrealizedDebtChangeUsdX18.add(ctx.unrealizedDebtChangeUsdX18);
            vaultTotalUsdcCreditChangeX18 = vaultTotalUsdcCreditChangeX18.add(ctx.usdcCreditChangeX18);
            vaultTotalWethRewardChangeX18 = vaultTotalWethRewardChangeX18.add(ctx.wethRewardChangeX18);

```

Market::getVaultAccumulatedValues is called and it returns ctx.wethRewardChangeX18. The final line is the one to focus on as it adds ctx.wethRewardChangeX18 to vaultTotalWethRewardChangeX18 which represents the total weth reward change of the vault. This value is incorrect as I explained above as ctx.wethRewardChangeX18 is the total weth reward change for the market and not per vault.

Furthermore Vault::recalculateVaultsCreditCapacity is as follows:

```solidity
  function recalculateVaultsCreditCapacity(uint256[] memory vaultsIds) internal {
        for (uint256 i; i < vaultsIds.length; i++) {
            // uint256 -> uint128
            uint128 vaultId = vaultsIds[i].toUint128();

            // load the vault storage pointer
            Data storage self = load(vaultId);

            // make sure there are markets connected to the vault
            uint256 connectedMarketsConfigLength = self.connectedMarkets.length;
            if (connectedMarketsConfigLength == 0) continue;

            // loads the connected markets storage pointer by taking the last configured market ids uint set
            EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1];

            // cache the connected markets ids to avoid multiple storage reads, as we're going to loop over them twice
            // at `_recalculateConnectedMarketsState` and `_updateCreditDelegations`
            uint128[] memory connectedMarketsIdsCache = new uint128[]());

            // update vault and credit delegation weight
            updateVaultAndCreditDelegationWeight(self, connectedMarketsIdsCache);

            // iterate over each connected market id and distribute its debt so we can have the latest credit
            // delegation of the vault id being iterated to the provided `marketId`
            (
                uint128[] memory updatedConnectedMarketsIdsCache,
                SD59x18 vaultTotalRealizedDebtChangeUsdX18,
                SD59x18 vaultTotalUnrealizedDebtChangeUsdX18,
                UD60x18 vaultTotalUsdcCreditChangeX18,
                UD60x18 vaultTotalWethRewardChangeX18
            ) = _recalculateConnectedMarketsState(self, connectedMarketsIdsCache, true);

            // gas optimization: only write to storage if values have changed
            //
            // updates the vault's stored unsettled realized debt distributed from markets
            if (!vaultTotalRealizedDebtChangeUsdX18.isZero()) {
                self.marketsRealizedDebtUsd = sd59x18(self.marketsRealizedDebtUsd).add(
                    vaultTotalRealizedDebtChangeUsdX18
                ).intoInt256().toInt128();
            }

            // updates the vault's stored unrealized debt distributed from markets
            if (!vaultTotalUnrealizedDebtChangeUsdX18.isZero()) {
                self.marketsUnrealizedDebtUsd = sd59x18(self.marketsUnrealizedDebtUsd).add(
                    vaultTotalUnrealizedDebtChangeUsdX18
                ).intoInt256().toInt128();
            }

            // adds the vault's total USDC credit change, earned from its connected markets, to the
            // `depositedUsdc` variable
            if (!vaultTotalUsdcCreditChangeX18.isZero()) {
                self.depositedUsdc = ud60x18(self.depositedUsdc).add(vaultTotalUsdcCreditChangeX18).intoUint128();
            }

            // distributes the vault's total WETH reward change, earned from its connected markets
            if (!vaultTotalWethRewardChangeX18.isZero() && self.wethRewardDistribution.totalShares != 0) {
                SD59x18 vaultTotalWethRewardChangeSD59X18 =
                    sd59x18(int256(vaultTotalWethRewardChangeX18.intoUint256()));
                self.wethRewardDistribution.distributeValue(vaultTotalWethRewardChangeSD59X18);
            }

            // update the vault's credit delegations
            (, SD59x18 vaultNewCreditCapacityUsdX18) =
                _updateCreditDelegations(self, updatedConnectedMarketsIdsCache, false);

            emit LogUpdateVaultCreditCapacity(
                vaultId,
                vaultTotalRealizedDebtChangeUsdX18.intoInt256(),
                vaultTotalUnrealizedDebtChangeUsdX18.intoInt256(),
                vaultTotalUsdcCreditChangeX18.intoUint256(),
                vaultTotalWethRewardChangeX18.intoUint256(),
                vaultNewCreditCapacityUsdX18.intoInt256()
            );
        }
    }
```

In this function, the inflated vault::vaultTotalWethRewardChangeX18 variable is returned from vault::\_recalculateConnectedMarketsState and is then used to call ` self.wethRewardDistribution.distributeValue(vaultTotalWethRewardChangeSD59X18);`. See Distribution::distributeValue below:

```solidity
function distributeValue(Data storage self, SD59x18 value) internal {
        if (value.eq(SD59x18_ZERO)) {
            return;
        }

        UD60x18 totalShares = ud60x18(self.totalShares);

        if (totalShares.eq(UD60x18_ZERO)) {
            revert Errors.EmptyDistribution();
        }

        SD59x18 deltaValuePerShare = value.div(totalShares.intoSD59x18());

        self.valuePerShare = sd59x18(self.valuePerShare).add(deltaValuePerShare).intoInt256();
    }

```

Distribution::distributeValue takes the inflated weth rewards returned in vault::vaultTotalWethRewardChangeX18 and distributes it between all shareholders which allows actors who hold shares of a particular vault to claim fees for the entire market which prevents actors with other vault shares connected to the same market from claiming their fees from the market.

## Proof Of Code (POC)

```solidity
function testFuzz_vaultinflatedwethrewards(
uint128 assetsToDepositA,
uint128 assetsToDepositB,
uint256 vaultId1,
uint256 vaultId2,
uint256 marketId1,
uint128 marketFees,
uint256 adapterIndex
)
external
{
vm.stopPrank();
VaultConfig memory fuzzVaultConfig1 = getFuzzVaultConfig(vaultId1);
vm.assume(fuzzVaultConfig1.asset != address(usdc));
PerpMarketCreditConfig memory fuzzMarketConfig1 = getFuzzPerpMarketCreditConfig(marketId1);
     VaultConfig memory fuzzVaultConfig2 = getFuzzVaultConfig(vaultId2);
     vm.assume(fuzzVaultConfig2.asset != address(usdc));
    vm.assume(fuzzVaultConfig1.vaultId != fuzzVaultConfig2.vaultId);

    //c set up 2 vaults and connect them to 1 market
    uint256[] memory marketIds = new uint256[](2);
    marketIds[0] = fuzzMarketConfig1.marketId;
    marketIds[1] = fuzzMarketConfig1.marketId;

    uint256[] memory vaultIds = new uint256[](2);
    vaultIds[0] = fuzzVaultConfig1.vaultId;
    vaultIds[1] = fuzzVaultConfig2.vaultId;

     vm.prank(users.owner.account);
    marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);


   //c 2 users are configured and deposit into each vault
    address userA = users.naruto.account;
    address userB = users.sasuke.account;
   uint128 assetsToDepositA =
        uint128(bound(assetsToDepositA, calculateMinOfSharesToStake(fuzzVaultConfig1.vaultId), fuzzVaultConfig1.depositCap / 2));
    fundUserAndDepositInVault(userA, fuzzVaultConfig1.vaultId, assetsToDepositA);

    uint128 assetsToDepositB =
        uint128(bound(assetsToDepositB, calculateMinOfSharesToStake(fuzzVaultConfig2.vaultId), fuzzVaultConfig2.depositCap / 2));
    fundUserAndDepositInVault(userB, fuzzVaultConfig2.vaultId, assetsToDepositB);


    //c both users stake assets in respective vaults
    vm.startPrank(userA);
    marketMakingEngine.stake(fuzzVaultConfig1.vaultId, uint128(IERC20(fuzzVaultConfig1.indexToken).balanceOf(userA)));
    vm.stopPrank();
    vm.startPrank(userB);
    marketMakingEngine.stake(fuzzVaultConfig2.vaultId, uint128(IERC20(fuzzVaultConfig2.indexToken).balanceOf(userB)));
    vm.stopPrank();

    // sent WETH market fees from PerpsEngine -> MarketEngine
    marketFees = uint128(bound(marketFees, calculateMinOfSharesToStake(fuzzVaultConfig1.vaultId), fuzzVaultConfig1.depositCap / 2));
    deal(fuzzVaultConfig1.asset, address(fuzzMarketConfig1.engine), marketFees);
    vm.prank(address(fuzzMarketConfig1.engine));
    marketMakingEngine.receiveMarketFee(fuzzMarketConfig1.marketId, fuzzVaultConfig1.asset, marketFees);
    assertEq(IERC20(fuzzVaultConfig1.asset).balanceOf(address(marketMakingEngine)), marketFees);

    
    if(fuzzVaultConfig1.asset != marketMakingEngine.workaround_getWethAddress()) {IDexAdapter adapter = getFuzzDexAdapter(adapterIndex);
    vm.startPrank(address(fuzzMarketConfig1.engine));
    marketMakingEngine.convertAccumulatedFeesToWeth(fuzzMarketConfig1.marketId, fuzzVaultConfig1.asset, adapter.STRATEGY_ID(), bytes(""));
    vm.stopPrank(); 
    }


//c compare the total weth reward of the market to the weth reward of the user shows that user can claim the total weth reward of the market. couldnt get assertapproxeqabs to work so i used console.log to check the values. if you can get it working, please do. if not, view the logs to see the comparison. narutorewardamount and totalwethrewardmarket have a difference of 1 due to some rounding errors I guess but that is insignificant but it will cause using raw assertEq to fail.
uint256 totalwethrewardmarket = marketMakingEngine.workaround\_gettotalWethReward(fuzzMarketConfig1.marketId);
console.log(totalwethrewardmarket);
SD59x18 narutorewardamount = marketMakingEngine.exposed\_getActorValueChange(fuzzVaultConfig1.vaultId, bytes32(uint256(uint160(userA))));
console.log(narutorewardamount.unwrap());
SD59x18 sasukerewardamount = marketMakingEngine.exposed\_getActorValueChange(fuzzVaultConfig2.vaultId, bytes32(uint256(uint160(userB))));
console.log(sasukerewardamount.unwrap());
/* StdAssertions.assertApproxEqAbs(int256(totalwethrewardmarket) == narutorewardamount.unwrap(), 10); */


// staker from vault 1 claims rewards of entire market successfully
vm.startPrank(userA);
marketMakingEngine.claimFees(fuzzVaultConfig1.vaultId); 


}

```

## OPTIONAL ADDON THAT MAY BE NEEDED IF RUNNING INTO WORKAROUND ERRORS WHEN RUNNING POC

Note that I added the following workarounds to VaultHarness.sol and MarketHarness.sol to get values I needed and I may have used them for POC's so if some of the tests do not work due to workaround functions not being found, add the following functions to VaultHarness.sol:

```solidity
       function workaround_CreditDelegation_getweight(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);

    return creditDelegation.weight;
  }

   function workaround_Vault_getTotalCreditDelegationWeight(
        uint128 vaultId
    )
        external view returns (uint128)
    {
        Vault.Data storage vaultData = Vault.load(vaultId);
       return vaultData.totalCreditDelegationWeight ;
    }

     function workaround_CreditDelegation_getlastVaultDistributedRealizedDebtUsdPerShare(uint128 vaultId, uint128 marketId) external view returns (int128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedRealizedDebtUsdPerShare;}

    function workaround_CreditDelegation_setvalueUsd(uint128 vaultId, uint128 marketId, uint128 valueUsd) external {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    creditDelegation.valueUsd = valueUsd;
    }

    function workaround_CreditDelegation_getlastVaultDistributedUnrealizedDebtUsdPerShare(uint128 vaultId, uint128 marketId) external view returns (int128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedUnrealizedDebtUsdPerShare;}

    function workaround_CreditDelegation_getlastVaultDistributedUsdcCreditPerShare(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedUsdcCreditPerShare;}

    function workaround_CreditDelegation_getlastVaultDistributedWethRewardPerShare(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedWethRewardPerShare;}
```

    Add the following functions to MarketHarness.sol:

```solidity
     function workaround_gettotalWethReward(uint128 marketId) external view returns (uint256) {
        Market.Data storage market = Market.load(marketId);
        return market.wethRewardPerVaultShare;
    }

    function workaround_getrealizedDebtUsdPerVaultShare(uint128 marketId) external view returns (int128) {
        Market.Data storage market = Market.load(marketId);
        return market.realizedDebtUsdPerVaultShare;
    }
```

After registering the selectors of these functions in TreeProxyUtils.sol and increasing the bytes array size, it should work as expected and return the correct values

## Impact

Overestimation of Rewards: If a vault has a small share of the total delegated credit, it will receive an inflated reward that does not correspond to its actual contribution.

Reward Theft: Malicious actors could create or manipulate vaults with minimal delegated credit to claim rewards intended for the entire market.

Loss of Funds: Legitimate users with larger shares of delegated credit may lose out on rewards they are entitled to, as these rewards are siphoned off by malicious actors.

## Tools Used

Manual Review, Foundry

## Recommendations

Include vaultCreditShareX18 Multiplication:

Align wethRewardChangeX18 with the other per-share calculations:

```solidity
wethRewardChangeX18 = !lastVaultDistributedWethRewardPerShareX18.isZero()
    ? (
        ud60x18(self.wethRewardPerVaultShare)
        .sub(lastVaultDistributedWethRewardPerShareX18)
    ).mul(vaultCreditShareX18)
    : UD60x18_ZERO;
````

This ensures that the vault only receives the portion of WETH that corresponds to its share of total delegated credit.

Add Zero-Check for Initial Distribution:

Mirror the pattern used for realized/unrealized debt and USDC credit so that if lastVaultDistributedWethRewardPerShareX18 == 0, the function returns zero for the vault’s WETH change (or handles first-time logic appropriately).

## <a id='H-05'></a>H-05. Unclaimed Rewards Are Lost During Re-Staking            



## Summary

Users lose unclaimed rewards whenever they stake new tokens. Specifically, the Distribution::\_updateLastValuePerShare called in Distribution::accumulateActor(actorId) and Distribution::setActorShares which are both called in VaultRouterBranch::stake at stake time resets the user’s reward checkpoint (Distribution::lastValuePerShare) to the current global Distribution::valuePerShare before the user claims prior rewards. This zeroes out any accrued but unclaimed WETH rewards.

## Vulnerability Details

VaultRouterBranch::stake calls wethRewardDistribution.accumulateActor(actorId) before updating the user’s share balance.
As explained in the above summary, internally, Distribution::accumulateActor calls Distribution::\_updateLastValuePerShare, which sets actor.lastValuePerShare = self.valuePerShare.

By resetting Distribution::lastValuePerShare to the current global Distribution::valuePerShare, any difference between the actor’s old lastValuePerShare and the current valuePerShare (i.e., pending rewards) is effectively discarded. When the user later attempts to call FeeDistributionBranch::claimFees(), they find no rewards left to claim.

## Proof Of Code (POC)

```solidity

function testFuzz\_restakingerasespreviousgeneratedfees(
uint128 assetsToDepositA,
uint256 vaultId1,
uint256 vaultId2,
uint256 marketId1,
uint128 marketFees,
uint256 adapterIndex
)
external
{
vm.stopPrank();

    //c configure vaults and markets
    VaultConfig memory fuzzVaultConfig1 = getFuzzVaultConfig(vaultId1);
     vm.assume(fuzzVaultConfig1.asset != address(usdc));
    PerpMarketCreditConfig memory fuzzMarketConfig1 = getFuzzPerpMarketCreditConfig(marketId1);


     VaultConfig memory fuzzVaultConfig2 = getFuzzVaultConfig(vaultId2);
     vm.assume(fuzzVaultConfig2.asset != address(usdc));
    vm.assume(fuzzVaultConfig1.vaultId != fuzzVaultConfig2.vaultId);

   
    uint256[] memory marketIds = new uint256[](1);
    marketIds[0] = fuzzMarketConfig1.marketId;
    

    uint256[] memory vaultIds = new uint256[](1);
    vaultIds[0] = fuzzVaultConfig1.vaultId;
    

     vm.prank(users.owner.account);
    marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);


   //c deposit asset into vault
    address userA = users.naruto.account;
   uint128 assetsToDepositA =
        uint128(bound(assetsToDepositA, calculateMinOfSharesToStake(fuzzVaultConfig1.vaultId), fuzzVaultConfig1.depositCap / 2));
    fundUserAndDepositInVault(userA, fuzzVaultConfig1.vaultId, assetsToDepositA*2);


    //c user stakes index token to get reward share
    vm.startPrank(userA);
    marketMakingEngine.stake(fuzzVaultConfig1.vaultId, (uint128(IERC20(fuzzVaultConfig1.indexToken).balanceOf(userA)))/2);
    vm.stopPrank();
    

    //c send WETH market fees from PerpsEngine -> MarketEngine
    marketFees = uint128(bound(marketFees, calculateMinOfSharesToStake(fuzzVaultConfig1.vaultId), fuzzVaultConfig1.depositCap / 2));
    deal(fuzzVaultConfig1.asset, address(fuzzMarketConfig1.engine), marketFees);
    vm.prank(address(fuzzMarketConfig1.engine));
    marketMakingEngine.receiveMarketFee(fuzzMarketConfig1.marketId, fuzzVaultConfig1.asset, marketFees);
    assertEq(IERC20(fuzzVaultConfig1.asset).balanceOf(address(marketMakingEngine)), marketFees);

    //c if the asset is not weth, convert the fees to weth
    if(fuzzVaultConfig1.asset != marketMakingEngine.workaround_getWethAddress()) {IDexAdapter adapter = getFuzzDexAdapter(adapterIndex);
    vm.startPrank(address(fuzzMarketConfig1.engine));
    marketMakingEngine.convertAccumulatedFeesToWeth(fuzzMarketConfig1.marketId, fuzzVaultConfig1.asset, adapter.STRATEGY_ID(), bytes(""));
    vm.stopPrank(); 
    }

    //c get user allocation of fees after fees have been deposited
    SD59x18 prenarutorewardamount = marketMakingEngine.exposed_getActorValueChange(fuzzVaultConfig1.vaultId, bytes32(uint256(uint160(userA))));
    console.log(prenarutorewardamount.unwrap());

    //c user restakes the index token to get more rewards
    vm.startPrank(userA);
    marketMakingEngine.stake(fuzzVaultConfig1.vaultId, uint128(IERC20(fuzzVaultConfig1.indexToken).balanceOf(userA)));
    vm.stopPrank();

    //c get user allocation of fees after restaking
    SD59x18 postnarutorewardamount = marketMakingEngine.exposed_getActorValueChange(fuzzVaultConfig1.vaultId, bytes32(uint256(uint160(userA))));
    console.log(postnarutorewardamount.unwrap());
    

    //c users previous rewards are erased even though they havent claimed them using FeesDistributionBranch::claimFees
    vm.startPrank(userA);
    vm.expectRevert(Errors.NoFeesToClaim.selector);
    marketMakingEngine.claimFees(fuzzVaultConfig1.vaultId); 

   
} 
 
```

## Impact

Loss of Funds: Users can lose accrued rewards that should rightly belong to them

## Tools Used

Manual Review, Foundry

## Recommendations

Whenever Distribution::lastValuePerShare is updated,  calculate how many rewards the user has already accrued based on actor's Distribution::oldLastValuePerShare and the current global Distribution::valuePerShare.

rewardSoFar =(currentValuePerShare−actor.lastValuePerShare)×actor.shares

Add this value to Distribution::pendingReward. Rather than “burning” these rewards by simply updating lastValuePerShare, save them in the actor’s pendingReward. This effectively acknowledges that the tokens belong to the actor, but the actor hasn’t officially claimed them yet. Once the old, unclaimed rewards are set aside in pendingReward,  update the actor’s lastValuePerShare to the latest global Distribution::valuePerShare. This allows new rewards to start accumulating after the user’s stake change without losing prior entitlements.

## <a id='H-06'></a>H-06. Vault's debt can never be settled as Vault::marketsRealizedDebtUsd is always returns zero            



## Summary

Due to an overly aggressive guard clause in Market::getVaultAccumulatedValues, creditdelegationbranch::lastVaultDistributedRealizedDebtUsdPerShare is never updated from its initial zero value. As a result, subsequent accumulation events always calculate a zero change, leading to incorrect debt distributions across connected markets.

## Vulnerability Details

A key function in the zaros protocol is Vault::recalculateVaultsCreditCapacity. This function is used to update the vaults credit capacity which updates a host of state variables that are key for other functions to calculate accurately. See below:

```solidity
/// @notice Recalculates the latest credit capacity of the provided vaults ids taking into account their latest
    /// assets and debt usd denonimated values.
    /// @dev We use a `uint256` array because a market's connected vaults ids are stored at a `EnumerableSet.UintSet`.
    /// @dev We assume this function's caller checks that connectedMarketsIdsCache > 0.
    /// @param vaultsIds The array of vaults ids to recalculate the credit capacity.
    // todo: check where we're messing with the `continue` statement
    function recalculateVaultsCreditCapacity(uint256[] memory vaultsIds) internal {
        for (uint256 i; i < vaultsIds.length; i++) {
            // uint256 -> uint128
            uint128 vaultId = vaultsIds[i].toUint128();

            // load the vault storage pointer
            Data storage self = load(vaultId);

            // make sure there are markets connected to the vault
            uint256 connectedMarketsConfigLength = self.connectedMarkets.length;
            if (connectedMarketsConfigLength == 0) continue;

            // loads the connected markets storage pointer by taking the last configured market ids uint set
            EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1];

            // cache the connected markets ids to avoid multiple storage reads, as we're going to loop over them twice
            // at `_recalculateConnectedMarketsState` and `_updateCreditDelegations`
            uint128[] memory connectedMarketsIdsCache = new uint128[](connectedMarkets.length());

            // update vault and credit delegation weight
            updateVaultAndCreditDelegationWeight(self, connectedMarketsIdsCache);

            // iterate over each connected market id and distribute its debt so we can have the latest credit
            // delegation of the vault id being iterated to the provided `marketId`
            (
                uint128[] memory updatedConnectedMarketsIdsCache,
                SD59x18 vaultTotalRealizedDebtChangeUsdX18,
                SD59x18 vaultTotalUnrealizedDebtChangeUsdX18,
                UD60x18 vaultTotalUsdcCreditChangeX18,
                UD60x18 vaultTotalWethRewardChangeX18
            ) = _recalculateConnectedMarketsState(self, connectedMarketsIdsCache, true);

            // gas optimization: only write to storage if values have changed
            //
            // updates the vault's stored unsettled realized debt distributed from markets
            if (!vaultTotalRealizedDebtChangeUsdX18.isZero()) {
                self.marketsRealizedDebtUsd = sd59x18(self.marketsRealizedDebtUsd).add(
                    vaultTotalRealizedDebtChangeUsdX18
                ).intoInt256().toInt128();
            }

            // updates the vault's stored unrealized debt distributed from markets
            if (!vaultTotalUnrealizedDebtChangeUsdX18.isZero()) {
                self.marketsUnrealizedDebtUsd = sd59x18(self.marketsUnrealizedDebtUsd).add(
                    vaultTotalUnrealizedDebtChangeUsdX18
                ).intoInt256().toInt128();
            }

            // adds the vault's total USDC credit change, earned from its connected markets, to the
            // `depositedUsdc` variable
            if (!vaultTotalUsdcCreditChangeX18.isZero()) {
                self.depositedUsdc = ud60x18(self.depositedUsdc).add(vaultTotalUsdcCreditChangeX18).intoUint128();
            }

            // distributes the vault's total WETH reward change, earned from its connected markets
            if (!vaultTotalWethRewardChangeX18.isZero() && self.wethRewardDistribution.totalShares != 0) {
                SD59x18 vaultTotalWethRewardChangeSD59X18 =
                    sd59x18(int256(vaultTotalWethRewardChangeX18.intoUint256()));
                self.wethRewardDistribution.distributeValue(vaultTotalWethRewardChangeSD59X18);
            }

            // update the vault's credit delegations
            (, SD59x18 vaultNewCreditCapacityUsdX18) =
                _updateCreditDelegations(self, updatedConnectedMarketsIdsCache, false);

            emit LogUpdateVaultCreditCapacity(
                vaultId,
                vaultTotalRealizedDebtChangeUsdX18.intoInt256(),
                vaultTotalUnrealizedDebtChangeUsdX18.intoInt256(),
                vaultTotalUsdcCreditChangeX18.intoUint256(),
                vaultTotalWethRewardChangeX18.intoUint256(),
                vaultNewCreditCapacityUsdX18.intoInt256()
            );
        }
    }
```

This function calls Market::getVaultAccumulatedValues. See below:

```solidity

    /// @notice Calculates the latest debt, usdc credit and weth reward values a vault is entitled to receive from the
    /// market since its last accumulation event.
    /// @param self The market storage pointer.
    /// @param vaultDelegatedCreditUsdX18 The vault's delegated credit in USD.
    /// @param lastVaultDistributedRealizedDebtUsdPerShareX18 The vault's last distributed realized debt per credit
    /// share.
    /// @param lastVaultDistributedUnrealizedDebtUsdPerShareX18 The vault's last distributed unrealized debt per
    /// credit share.
    /// @param lastVaultDistributedUsdcCreditPerShareX18 The vault's last distributed USDC credit per credit share.
    /// @param lastVaultDistributedWethRewardPerShareX18 The vault's last distributed WETH reward per credit share.
    function getVaultAccumulatedValues(
        Data storage self,
        UD60x18 vaultDelegatedCreditUsdX18,
        SD59x18 lastVaultDistributedRealizedDebtUsdPerShareX18,
        SD59x18 lastVaultDistributedUnrealizedDebtUsdPerShareX18,
        UD60x18 lastVaultDistributedUsdcCreditPerShareX18,
        UD60x18 lastVaultDistributedWethRewardPerShareX18
    )
        internal
        view
        returns (
            SD59x18 realizedDebtChangeUsdX18,
            SD59x18 unrealizedDebtChangeUsdX18,
            UD60x18 usdcCreditChangeX18,
            UD60x18 wethRewardChangeX18
        )
    {
        // calculate the vault's share of the total delegated credit, from 0 to 1
        UD60x18 vaultCreditShareX18 = vaultDelegatedCreditUsdX18.div(getTotalDelegatedCreditUsd(self));

        // calculate the vault's value changes since its last accumulation
        // note: if the last distributed value is zero, we assume it's the first time the vault is accumulating
        // values, thus, it needs to return zero changes

        realizedDebtChangeUsdX18 = !lastVaultDistributedRealizedDebtUsdPerShareX18.isZero()
            ? sd59x18(self.realizedDebtUsdPerVaultShare).sub(lastVaultDistributedRealizedDebtUsdPerShareX18).mul(
                vaultCreditShareX18.intoSD59x18()
            )
            : SD59x18_ZERO;

        unrealizedDebtChangeUsdX18 = !lastVaultDistributedUnrealizedDebtUsdPerShareX18.isZero()
            ? sd59x18(self.unrealizedDebtUsdPerVaultShare).sub(lastVaultDistributedUnrealizedDebtUsdPerShareX18).mul(
                vaultCreditShareX18.intoSD59x18()
            )
            : SD59x18_ZERO;

        usdcCreditChangeX18 = !lastVaultDistributedUsdcCreditPerShareX18.isZero()
            ? ud60x18(self.usdcCreditPerVaultShare).sub(lastVaultDistributedUsdcCreditPerShareX18).mul(
                vaultCreditShareX18
            )
            : UD60x18_ZERO;

        // TODO: fix the vaultCreditShareX18 flow to multiply by `wethRewardChangeX18`
        wethRewardChangeX18 = ud60x18(self.wethRewardPerVaultShare).sub(lastVaultDistributedWethRewardPerShareX18);
    }
```

The bug occurs for 2 reason. Market::getVaultAccumulatedValues has the following line:

```solidity
  realizedDebtChangeUsdX18 = !lastVaultDistributedRealizedDebtUsdPerShareX18.isZero()
            ? sd59x18(self.realizedDebtUsdPerVaultShare).sub(lastVaultDistributedRealizedDebtUsdPerShareX18).mul(
                vaultCreditShareX18.intoSD59x18()
            )
            : SD59x18_ZERO;
```

This checks if lastVaultDistributedRealizedDebtUsdPerShareX18 is zero and, if so, returns a zero change for the realized debt calculation. The initial state of lastVaultDistributedRealizedDebtUsdPerShare is zero and so when Vault::recalculateVaultsCreditCapacity is first called, this value returns 0.

However, Vault::\_recalculateConnectedMarketsState has the following code block:

```solidity
 if (
                ctx.realizedDebtChangeUsdX18.isZero() && ctx.unrealizedDebtChangeUsdX18.isZero()
                    && ctx.usdcCreditChangeX18.isZero() && ctx.wethRewardChangeX18.isZero()
            ) {
                continue;
            } 
// update the vault's state by adding its share of the market's latest state variables
            vaultTotalRealizedDebtChangeUsdX18 = vaultTotalRealizedDebtChangeUsdX18.add(ctx.realizedDebtChangeUsdX18);
            vaultTotalUnrealizedDebtChangeUsdX18 =
                vaultTotalUnrealizedDebtChangeUsdX18.add(ctx.unrealizedDebtChangeUsdX18);
            vaultTotalUsdcCreditChangeX18 = vaultTotalUsdcCreditChangeX18.add(ctx.usdcCreditChangeX18);
            vaultTotalWethRewardChangeX18 = vaultTotalWethRewardChangeX18.add(ctx.wethRewardChangeX18); 

            // update the last distributed debt, credit and reward values to the vault's credit delegation to the
            // given market id, in order to keep next calculations consistent
            creditDelegation.updateVaultLastDistributedValues(
                sd59x18(market.realizedDebtUsdPerVaultShare),
                sd59x18(market.unrealizedDebtUsdPerVaultShare),
                ud60x18(market.usdcCreditPerVaultShare),
                ud60x18(market.wethRewardPerVaultShare)
            );
        }
    }
```

This checks if  ctx.realizedDebtChangeUsdX18.isZero() && ctx.unrealizedDebtChangeUsdX18.isZero()&& ctx.usdcCreditChangeX18.isZero() && ctx.wethRewardChangeX18.isZero() are all zero which in the first instance they will be due to the condition in Market::getVaultAccumulatedValues. In this case, it skips the iteration. By skipping the iteration, it misses out on a key function call which is creditDelegation.updateVaultLastDistributedValues. This function updates all state variables in creditdelegation.data with their appropriate data especially with updating market.realizedDebtUsdPerVaultShare which is used to update lastVaultDistributedRealizedDebtUsdPerShareX18 in the function.

Since this value is not updated in creditdelegation.data, lastVaultDistributedRealizedDebtUsdPerShareX18 will remain 0 whenever Vault::recalculateVaultsCreditCapacity is next called. In fact, no matter how many times it is called, lastVaultDistributedRealizedDebtUsdPerShareX18 will be 0. This has additional effects as what this means is that whenever creditdelegationbranch::settlevaultsdeposit is called to settle vaults debt, the vaults debt will always be zero. The key line in the function is:

```solidity
ctx.vaultUnsettledRealizedDebtUsdX18 = vault.getUnsettledRealizedDebt();
```

See vault.getUnsettledRealizedDebt:

```solidity
 function getUnsettledRealizedDebt(Data storage self)
        internal
        view
        returns (SD59x18 unsettledRealizedDebtUsdX18)
    {
        unsettledRealizedDebtUsdX18 =
            sd59x18(self.marketsRealizedDebtUsd).add(unary(ud60x18(self.depositedUsdc).intoSD59x18()));
    }
```

Vault::marketsRealizedDebtUsd is calculated with the following line in Vault::recalculateVaultsCreditCapacity:

```solidity
 if (!vaultTotalRealizedDebtChangeUsdX18.isZero()) {
                self.marketsRealizedDebtUsd = sd59x18(self.marketsRealizedDebtUsd).add(
                    vaultTotalRealizedDebtChangeUsdX18
                ).intoInt256().toInt128();
```

vaultTotalRealizedDebtChangeUsdX18 is calculated from the following line in Vault::\_recalculateconnectedmarketsstate:

```solidity
 vaultTotalRealizedDebtChangeUsdX18 = vaultTotalRealizedDebtChangeUsdX18.add(ctx.realizedDebtChangeUsdX18);
```

and finally, realizedDebtChangeUsdX18 is calculated in Market::getVaultAccumulatedValues as follows:

```solidity
realizedDebtChangeUsdX18 = !lastVaultDistributedRealizedDebtUsdPerShareX18.isZero() 
            ? sd59x18(self.realizedDebtUsdPerVaultShare).sub(lastVaultDistributedRealizedDebtUsdPerShareX18).mul(
                vaultCreditShareX18.intoSD59x18()
            )
            : SD59x18_ZERO;
```

This is how the vault debt maps to Market::realizedDebtUsdPerVaultShare which is the variable we identified that was never updated from 0. As a result, the vault debt is never updated from 0 so whenever creditdelegationbranch::settlevaultsdebt is called, the vault debt will always be 0 even when the vault should be allocated debt.

## Proof Of Code (POC)

```solidity
function test_vaultdebtisneverupdated(
        uint128 vaultId,
        uint128 assetsToDeposit,
        uint128 marketId
    )
        external
    {
        vm.stopPrank();
       //c configure vaults and markets
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
         vm.assume(fuzzVaultConfig.asset != address(usdc));
        PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);

       
        uint256[] memory marketIds = new uint256[](1);
        marketIds[0] = fuzzMarketConfig.marketId;
        

        uint256[] memory vaultIds = new uint256[](1);
        vaultIds[0] = fuzzVaultConfig.vaultId;

        vm.prank(users.owner.account);
        marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);

        // ensure valid deposit amount
        address userA = users.naruto.account;
        assetsToDeposit = 1e14;
        //c need to deposit twice due to recalculatevaultcreditcapacity not being called in vaultrouterbranch::deposit which is a bug I reported
        fundUserAndDepositInVault(userA, fuzzVaultConfig.vaultId, assetsToDeposit);
        deal(fuzzVaultConfig.asset, userA, 100e18);
        vm.prank(userA);
         marketMakingEngine.deposit(fuzzVaultConfig.vaultId, assetsToDeposit, 0, "", false);
        

        deal(fuzzVaultConfig.asset, address(fuzzMarketConfig.engine), 100e18);

        //c perp engine deposits credit into market 
       vm.prank(address(fuzzMarketConfig.engine));
        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, fuzzVaultConfig.depositCap*2);
        
        //c recalculatevaultcreditcapacity is not called in depositcreditformarket which is a bug I reported so i have to call vaultrouterbranch::deposit again to update the credit capacity of the vault
        address userB = users.sasuke.account;
        deal(fuzzVaultConfig.asset, userB, 100e18);
        vm.startPrank(userB);
        marketMakingEngine.deposit(fuzzVaultConfig.vaultId, assetsToDeposit, 0, "", false);
        vm.stopPrank();

        //c proof that marketdebt has been updated
        SD59x18 totalmarketdebt = marketMakingEngine.workaround_getTotalMarketDebt(fuzzMarketConfig.marketId);
        console.log(totalmarketdebt.unwrap());

        //c proof that the realizeddebtpervault
        int128 realizeddebtpervaultshare = marketMakingEngine.workaround_getrealizedDebtUsdPerVaultShare(fuzzMarketConfig.marketId);
        console.log(realizeddebtpervaultshare);

       
        //c this value should match the realizeddebtpervaultshare value because _recalculateConnectedMarketsState calls creditDelegation.updateVaultLastDistributedValues and this function sets the lastvaultdistributedrealizeddebtusdpershare to the realizeddebtpervaultshare. This isnt the case and it returns 0 which is a bug
        int128 lastvaultdistributedrealizeddebtusdpershare = marketMakingEngine.workaround_CreditDelegation_getlastVaultDistributedRealizedDebtUsdPerShare(fuzzVaultConfig.vaultId, fuzzMarketConfig.marketId);
        console.log(lastvaultdistributedrealizeddebtusdpershare);

        assert (realizeddebtpervaultshare != lastvaultdistributedrealizeddebtusdpershare);

        //c on attempting to settle the vaults debt, creditdelegationbranch::settlevaultsdebt calls recalculatevaultcreditcapacity which should update the vaults debt but vault debt is never updated and although the vault should have debt from the creditdelegationbranch::depositcreditformarket call earlier, the value isnt updated and the vault debt will always be 0
         vm.prank(address(perpsEngine));
        marketMakingEngine.settleVaultsDebt(vaultIds); 

         int128 vaultdebt = marketMakingEngine.workaround_getVaultDebt(fuzzVaultConfig.vaultId);
        console.log(vaultdebt);
        assertEq(vaultdebt, 0);
}
```

## OPTIONAL ADDON THAT MAY BE NEEDED IF RUNNING INTO WORKAROUND ERRORS WHEN RUNNING POC

Note that I added the following workarounds to VaultHarness.sol and MarketHarness.sol to get values I needed and I may have used them for POC's so if some of the tests do not work due to workaround functions not being found, add the following functions to VaultHarness.sol:

```solidity
       function workaround_CreditDelegation_getweight(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);

    return creditDelegation.weight;
  }

   function workaround_Vault_getTotalCreditDelegationWeight(
        uint128 vaultId
    )
        external view returns (uint128)
    {
        Vault.Data storage vaultData = Vault.load(vaultId);
       return vaultData.totalCreditDelegationWeight ;
    }

     function workaround_CreditDelegation_getlastVaultDistributedRealizedDebtUsdPerShare(uint128 vaultId, uint128 marketId) external view returns (int128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedRealizedDebtUsdPerShare;}

    function workaround_CreditDelegation_setvalueUsd(uint128 vaultId, uint128 marketId, uint128 valueUsd) external {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    creditDelegation.valueUsd = valueUsd;
    }

    function workaround_CreditDelegation_getlastVaultDistributedUnrealizedDebtUsdPerShare(uint128 vaultId, uint128 marketId) external view returns (int128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedUnrealizedDebtUsdPerShare;}

    function workaround_CreditDelegation_getlastVaultDistributedUsdcCreditPerShare(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedUsdcCreditPerShare;}

    function workaround_CreditDelegation_getlastVaultDistributedWethRewardPerShare(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedWethRewardPerShare;}
```

```Solidity
Add the following functions to MarketHarness.sol:
```

```solidity
     function workaround_gettotalWethReward(uint128 marketId) external view returns (uint256) {
        Market.Data storage market = Market.load(marketId);
        return market.wethRewardPerVaultShare;
    }

    function workaround_getrealizedDebtUsdPerVaultShare(uint128 marketId) external view returns (int128) {
        Market.Data storage market = Market.load(marketId);
        return market.realizedDebtUsdPerVaultShare;
    }
```

After registering the selectors of these functions in TreeProxyUtils.sol and increasing the bytes array size, it should work as expected and return the correct values

## Impact

Due to the guard clause preventing updates to the credit delegation branch's state, the vault's debt accumulation mechanism fails to capture any changes from connected markets. This results in:

Inaccurate Debt Accounting: The lastVaultDistributedRealizedDebtUsdPerShare remains at its initial zero value, causing the calculated debt changes to always be zero. As a consequence, the vault’s total debt never updates to reflect the actual market conditions.

Settlement Failures: When the system attempts to settle the vault’s debt (via functions such as settlevaultsdebt), it relies on these accumulated values. With the debt always recorded as zero, the settlement process will be flawed, potentially causing imbalances in user accounts and misallocations of funds.

## Tools Used

Manual Review, Foundry

## Recommendations

Remove the Guard Clause in Vault::recalculateVaultsCreditCapacity:
The root cause of the issue is the following code block:

```solidity

if (
    ctx.realizedDebtChangeUsdX18.isZero() && ctx.unrealizedDebtChangeUsdX18.isZero()
    && ctx.usdcCreditChangeX18.isZero() && ctx.wethRewardChangeX18.isZero()
) {
    continue;
}
```

Removing this block will prevent the function from skipping the update to creditDelegation.updateVaultLastDistributedValues. This change will allow the per-share debt values (and other related metrics) to be updated correctly, even when the computed change appears to be zero on the first accumulation event.

In place of the removed guard clause, introduce logic that explicitly compares the current market debt values (as obtained by market.getRealizedDebtUsd() and market.getUnrealizedDebtUsd()) with their last distributed counterparts stored in the credit delegation. Update the credit delegation only if there is a meaningful change. For example, add a conditional check such as:

```solidity
bool hasDebtChanged = 
    !market.getRealizedDebtUsd().eq(creditDelegation.lastVaultDistributedRealizedDebtUsdPerShare) ||
    !market.getUnrealizedDebtUsd().eq(creditDelegation.lastVaultDistributedUnrealizedDebtUsdPerShare);

if (!hasDebtChanged) {
    // No change detected, skip update.
    continue;
}
```
This delta check must be implemented so that updates are only skipped when there is absolutely no change in the market’s debt values relative to the last distributed values, ensuring that even an initial update (where the values might be zero) is correctly recorded and also preventing the same debt being redistributed to vaults everytime Vault:recalculatevaultscreditcapacity is called.

Review Initialization of Last Distributed Values: Evaluate the initialization logic for lastVaultDistributedRealizedDebtUsdPerShare (and the corresponding variables for unrealized debt, USDC credit, and WETH reward). Ensure that the first accumulation event correctly captures and sets these values so that subsequent changes can be computed accurately.

## <a id='H-07'></a>H-07. CreditDelegationBranch::settlevaultsdebt incorrectly swaps token from marketmakingengine and not directly from Zlpvault which breaks protocol intended functionality            



## Summary

CreditDelegationBranch::settlevaultsdebt is intended to swap a vault’s own assets for USDC to settle its debt. However, due to an incorrect parameter in the swap call, the function swaps assets from the marketmakingengine (address(this)) instead of the vault’s associated zlpvault contract. This discrepancy prevents the vault’s actual assets from being used in the settlement process, leading to inaccurate debt settlement.

## Vulnerability Details

The natspec of CreditDelegationBranch::settlevaultsdebt says the following:
"/// @notice Settles the given vaults' debt or credit by swapping assets to USDC or vice versa.
/// @dev Converts ZLP Vaults unsettled debt to settled debt by:
///     - If in debt, swapping the vault's assets to USDC.
///     - If in credit, swapping the vault's available USDC to its underlying assets.
/// exchange for their assets."

The natspec documentation clearly states that the vault’s assets should be swapped for USDC (if the vault is in debt) or vice versa (if in credit). However, the implementation erroneously uses address(this) when calling the swap function. This implies that the swap operation is performed using tokens from the current contract rather than from the vault’s designated asset account.

This is not intended logic and is further proven by looking at CreditDelegationBranch::convertMarketsCreditDepositsToUsdc. See below:

```solidity
/// @notice Converts assets deposited as credit to a given market for USDC.
    /// @dev USDC accumulated by swaps is stored at markets, and later pushed to its connected vaults in order to
    /// cover an engine's usd token.
    /// @dev The keeper must ensure that the market has received fees for the given asset before calling this
    /// function.
    /// @dev The keeper doesn't need to settle all deposited assets at once, it can settle them in batches as needed.
    /// @param marketId The market identifier.
    /// @param assets The array of assets deposited as credit to be settled for USDC.
    /// @param dexSwapStrategyIds The identifier of the dex swap strategies to be used.
    /// @param paths Used when the keeper wants to perform a multihop swap using one of the swap strategies.
    function convertMarketsCreditDepositsToUsdc(
        uint128 marketId,
        address[] calldata assets,
        uint128[] calldata dexSwapStrategyIds,
        bytes[] calldata paths
    )
        external
        onlyRegisteredSystemKeepers
    {
        // revert if the arrays have different lengths
        if (assets.length != dexSwapStrategyIds.length || assets.length != paths.length) {
            // we ignore in purpose the error params here
            revert Errors.ArrayLengthMismatch(0, 0);
        }

        // load the market's data storage pointer
        Market.Data storage market = Market.loadExisting(marketId);

        // working area
        ConvertMarketsCreditDepositsToUsdcContext memory ctx;

        for (uint256 i; i < assets.length; i++) {
            // revert if the market hasn't received any fees for the given asset
            (bool exists, uint256 creditDeposits) = market.creditDeposits.tryGet(assets[i]);
            if (!exists) revert Errors.MarketDoesNotContainTheAsset(assets[i]);
            if (creditDeposits == 0) revert Errors.AssetAmountIsZero(assets[i]);

            // cache usdc address
            address usdc = MarketMakingEngineConfiguration.load().usdc;

            // creditDeposits in zaros internal precision so convert to native token decimals
            ctx.creditDepositsNativeDecimals =
                Collateral.load(assets[i]).convertUd60x18ToTokenAmount(ud60x18(creditDeposits));

            // convert the assets to USDC; both input and outputs in native token decimals
            uint256 usdcOut = _convertAssetsToUsdc(
                dexSwapStrategyIds[i], assets[i], ctx.creditDepositsNativeDecimals, paths[i], address(this), usdc
            );

            // sanity check to ensure we didn't somehow give away the input tokens
            if (usdcOut == 0) revert Errors.ZeroOutputTokens();

            // settles the credit deposit for the amount of USDC received
            // updating storage so convert from native token decimals to zaros internal precision
            market.settleCreditDeposit(assets[i], Collateral.load(usdc).convertTokenAmountToUd60x18(usdcOut));

            // emit an event
            emit LogConvertMarketCreditDepositsToUsdc(marketId, assets[i], creditDeposits, usdcOut);
        }
    }

```

This function is used to convert market credit which are tokens deposited by the perp engine using CreditDelegationBranch::depositcreditformarket to usdc. As a result, it is evident that CreditDelegationBranch::settlevaultsdebt intended function is to swap assets from the vaults associated zlp vault to usdc. Assets in the zlp vault are deposited via VaultRouterBranch::deposit and the function contains the following lines:

```solidity
 // increase vault allowance to transfer tokens minus fees from this contract to vault
        address indexTokenCache = vault.indexToken;
        IERC20(ctx.vaultAsset).approve(indexTokenCache, ctx.assetsMinusFees);

        // then perform the actual deposit
        // NOTE: the following call will update the total assets deposited in the vault
        // NOTE: the following call will validate the vault's deposit cap
        // invariant: no tokens should remain stuck in this contract
        ctx.shares = IERC4626(indexTokenCache).deposit(ctx.assetsMinusFees, msg.sender);

```

These lines deposit the assets that user has deposited into the zlp vault. As a result, these tokens are no longer in the marketmakingengine contract. They are stored in the zlp vault. As a result, when CreditDelegationBranch::settlevaultsdebt, for it to work as intended, it must swap tokens directly from the associated zlp vault for usdc which currently is not the case.

The call:

```solidity
ctx.usdcOut = _convertAssetsToUsdc(
    vault.swapStrategy.usdcDexSwapStrategyId,
    ctx.vaultAsset,
    ctx.swapAmount,
    vault.swapStrategy.usdcDexSwapPath,
    address(this), // Incorrect: should target the vault's asset source
    ctx.usdc
);
```

mistakenly directs the swap to pull tokens from address(this) (the current contract) instead of from the vault's actual asset holdings.
Because the vault’s assets are not swapped as intended, the resulting USDC is not correctly allocated. This leads to an incorrect update of the vault’s unsettled debt—ultimately causing the debt to remain unaddressed when the settlement function is called.


## Impact

The vault’s actual assets are not used for the swap, meaning the vault’s debt remains unsettled or is settled using an incorrect source. This discrepancy results in the vault’s debt not being properly reduced. Since the marketmakingengine contract makes the swap, it ends up with less creditdeposits than intended. If using the marketmakingengine's tokens to perform the swap was intended behaviour, then Market::creditdeposits should have been updated to reflect that there are less credit deposits in the contract and more usdc which was done in CreditDelegationBranch::convertMarketsCreditDepositsToUsdc with the following line:

```solidity

            // settles the credit deposit for the amount of USDC received
            // updating storage so convert from native token decimals to zaros internal precision
            market.settleCreditDeposit(assets[i], Collateral.load(usdc).convertTokenAmountToUd60x18(usdcOut));

```

This was not done in CreditDelegationBranch::settlevaultsdebt which further indicates that the marketmakingengine's creditdeposits are not intended to be used in CreditDelegationBranch::settlevaultsdebt.

## Tools Used

Manual Review

## Recommendations

Correct the Swap Parameter:
Update the swap call in the debt settlement function to ensure that it pulls the vault’s assets from the correct contract. Replace
address(this) with the appropriate vault asset source address so that the swap operation utilizes the vault’s own tokens for settling its debt.

Ensure Proper Asset Approval: Before executing the swap, the ZLP vault must explicitly approve the MarketMakingEngine to spend all its tokens. This approval is critical to guarantee that the MarketMakingEngine can access and swap the vault’s assets as intended.

Review Asset Flow and Design: Verify that the vault’s assets are managed in a separate contract as originally designed, and update the debt settlement logic accordingly. Confirm that the asset flow during swaps matches the specifications outlined in the natSpec documentation.

## <a id='H-08'></a>H-08. Incorrect swap amount in CreditDelegationBranch::settleVaultsDebt improperly inflates the tokens to swap leading to DOS or/and oversettling vault debt            



## Summary

A bug in the CreditDelegationBranch::settlevaultsdeposit results in an incorrect calculation of the swap amount when the vault is in debt. Instead of converting the vault’s unsettled debt from its USD representation to the vault asset’s native token amount, the function mistakenly uses the USDC collateral conversion. This leads to an inaccurate swap amount when attempting to exchange the vault’s assets for USDC, potentially resulting in an improper settlement of the vault’s debt.

## Vulnerability Details

In the debt branch of the settleVaultsDebt function, when a vault is in debt (ctx.vaultUnsettledRealizedDebtUsdX18.lt(SD59x18\_ZERO)), the swap amount is calculated by calling:

```solidity
ctx.swapAmount = calculateSwapAmount(
    dexSwapStrategy.dexAdapter,
    ctx.usdc,
    ctx.vaultAsset,
    usdcCollateralConfig.convertSd59x18ToTokenAmount(ctx.vaultUnsettledRealizedDebtUsdX18.abs())
);
```

CreditDelegationBranch::calculateSwapAmount gets the expected output from a swap. See below:

```solidity
function calculateSwapAmount(
        address dexAdapter,
        address assetIn,
        address assetOut,
        uint256 vaultUnsettledDebtUsdAbs
    )
        public
        view
        returns (uint256 amount)
    {
        // calculate expected asset amount needed to cover the debt
        amount = IDexAdapter(dexAdapter).getExpectedOutput(assetIn, assetOut, vaultUnsettledDebtUsdAbs);
    }
```

This calls BaseAdapter::getExpectedOutput:

```solidity
 function getExpectedOutput(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    )
        public
        view
        returns (uint256 expectedAmountOut)
    {
        // fail fast for zero input
        if (amountIn == 0) revert Errors.ZeroExpectedSwapOutput();

        // get token prices
        UD60x18 priceTokenInX18 = IPriceAdapter(swapAssetConfigData[tokenIn].priceAdapter).getPrice();
        UD60x18 priceTokenOutX18 = IPriceAdapter(swapAssetConfigData[tokenOut].priceAdapter).getPrice();

        // convert input amount from native to internal zaros precision
        UD60x18 amountInX18 = Math.convertTokenAmountToUd60x18(swapAssetConfigData[tokenIn].decimals, amountIn);
        //c this function assumes that the token decimals is always <= 18,and with all the token supported by zaros, this condition is met. it then converts the amountIn to 18 decimals

        // calculate the expected amount out in native precision of output token
        expectedAmountOut = Math.convertUd60x18ToTokenAmount(
            swapAssetConfigData[tokenOut].decimals, amountInX18.mul(priceTokenInX18).div(priceTokenOutX18)
        );
        
        // revert when calculated expected output is zero; must revert here
        // otherwise the subsequent slippage bps calculation will also
        // return a minimum swap output of zero giving away the input tokens
        if (expectedAmountOut == 0) revert Errors.ZeroExpectedSwapOutput();
    }

```

which returns the expectedoutput from the swap to ctx.swapAmount in CreditDelegationBranch::settleVaultsDebt. ctx.swapAmount is then passed as the assetAmount variable into CreditDelegation::\_convertAssetsToUsdc which is the function that performs the swap.

```solidity
// swap the vault's assets to usdc in order to cover the usd denominated debt partially or fully
                // both input and output in native precision
                ctx.usdcOut = _convertAssetsToUsdc(
                    vault.swapStrategy.usdcDexSwapStrategyId,
                    ctx.vaultAsset,
                    ctx.swapAmount,
                    vault.swapStrategy.usdcDexSwapPath,
                    address(this),
                    ctx.usdc
                );

```

## Proof Of Code (POC)

```solidity
 function test_settlevaultdebtdoesnotperformasintended( uint128 vaultId,
        uint128 assetsToDeposit,
        uint128 marketId,
        uint128 adapterIndex
    )
        external
    {
        vm.stopPrank();
       //c configure vaults and markets
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
         vm.assume(fuzzVaultConfig.asset != address(usdc));
        PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);

       
        uint256[] memory marketIds = new uint256[](1);
        marketIds[0] = fuzzMarketConfig.marketId;
        

        uint256[] memory vaultIds = new uint256[](1);
        vaultIds[0] = fuzzVaultConfig.vaultId;

        vm.prank(users.owner.account);
        marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);

        // ensure valid deposit amount
        address userA = users.naruto.account;
        assetsToDeposit = fuzzVaultConfig.depositCap/2;
        deal(fuzzVaultConfig.asset, userA, 100e18);
        vm.prank(userA);
         marketMakingEngine.deposit(fuzzVaultConfig.vaultId, assetsToDeposit, 0, "", false);
        
        //c fund engine with tokens to depositcreditformarket
        deal(fuzzVaultConfig.asset, address(fuzzMarketConfig.engine), 100e18);

        //c perp engine deposits credit into market
       vm.prank(address(fuzzMarketConfig.engine));
        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, 1e8);

        //c recalculatevaultcreditcapacity is not called in depositcreditformarket which is a bug I reported so i have to call vaultrouterbranch::deposit again to update the credit capacity of the vault. could have also called CreditDelegationBranch::updateVaultCreditCapacity but I chose this way instead
        address userB = users.sasuke.account;
        deal(fuzzVaultConfig.asset, userB, 100e18);
        vm.startPrank(userB);
        marketMakingEngine.deposit(fuzzVaultConfig.vaultId, assetsToDeposit, 0, "", false);
        vm.stopPrank();
       
        IDexAdapter adapter = getFuzzDexAdapter(adapterIndex);
        
        vm.startPrank(users.owner.account);
        marketMakingEngine.updateVaultSwapStrategy(
            fuzzVaultConfig.vaultId, "", "", adapter.STRATEGY_ID(), adapter.STRATEGY_ID()
        );
        vm.stopPrank();

         uint256 marketcreditdeposit = marketMakingEngine.workaround_getMarketCreditDeposit(fuzzMarketConfig.marketId, fuzzVaultConfig.asset);
        console.log(marketcreditdeposit);

        vm.prank(address(perpsEngine));
        //c attempt to settlevaultdebt but this will revert with ERC20InsufficientBalance as the usdc representation to the vault asset’s native token amount is not the correct value to use to perform the swap

        /*c to run this test, I commented out the following lines in vault::recalculateVaultsCreditCapacity:
            if (
                    ctx.realizedDebtChangeUsdX18.isZero() && ctx.unrealizedDebtChangeUsdX18.isZero()
                        && ctx.usdcCreditChangeX18.isZero() && ctx.wethRewardChangeX18.isZero()
                ) {
                    continue;
                }
        If this block is not commented out, the vault debt will always return 0 due to the access guard issue which I have reported
        */

        //c attempt to settlevaultdebt but this will revert with ERC20InsufficientBalance as the usdc representation of the vault asset’s native token amount is not the correct value to use to perform the swap. to view the logs, run  forge test --mt test_settlevaultdebtdoesnotperformasintended -vvvv

        vm.expectRevert();
        marketMakingEngine.settleVaultsDebt(vaultIds);

    }
```
## OPTIONAL ADDON THAT MAY BE NEEDED IF RUNNING INTO WORKAROUND ERRORS WHEN RUNNING POC

Note that I added the following workarounds to VaultHarness.sol and MarketHarness.sol to get values I needed and I may have used them for POC's so if some of the tests do not work due to workaround functions not being found, add the following functions to VaultHarness.sol:

```solidity
       function workaround_CreditDelegation_getweight(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);

    return creditDelegation.weight;
  }

   function workaround_Vault_getTotalCreditDelegationWeight(
        uint128 vaultId
    )
        external view returns (uint128)
    {
        Vault.Data storage vaultData = Vault.load(vaultId);
       return vaultData.totalCreditDelegationWeight ;
    }

     function workaround_CreditDelegation_getlastVaultDistributedRealizedDebtUsdPerShare(uint128 vaultId, uint128 marketId) external view returns (int128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedRealizedDebtUsdPerShare;}

    function workaround_CreditDelegation_setvalueUsd(uint128 vaultId, uint128 marketId, uint128 valueUsd) external {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    creditDelegation.valueUsd = valueUsd;
    }

    function workaround_CreditDelegation_getlastVaultDistributedUnrealizedDebtUsdPerShare(uint128 vaultId, uint128 marketId) external view returns (int128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedUnrealizedDebtUsdPerShare;}

    function workaround_CreditDelegation_getlastVaultDistributedUsdcCreditPerShare(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedUsdcCreditPerShare;}

    function workaround_CreditDelegation_getlastVaultDistributedWethRewardPerShare(uint128 vaultId, uint128 marketId) external view returns (uint128) {
    CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
    return creditDelegation.lastVaultDistributedWethRewardPerShare;}
```

    Add the following functions to MarketHarness.sol:

```solidity
     function workaround_gettotalWethReward(uint128 marketId) external view returns (uint256) {
        Market.Data storage market = Market.load(marketId);
        return market.wethRewardPerVaultShare;
    }

    function workaround_getrealizedDebtUsdPerVaultShare(uint128 marketId) external view returns (int128) {
        Market.Data storage market = Market.load(marketId);
        return market.realizedDebtUsdPerVaultShare;
    }
```

After registering the selectors of these functions in TreeProxyUtils.sol and increasing the bytes array size, it should work as expected and return the correct values

## Impact

Inaccurate Debt Settlement: The improper conversion means the swap amount for the vault’s assets is miscalculated, resulting in the vault's debt being settled incorrectly. Over time, this can cause the vault's recorded debt to diverge from its actual debt, leading to accounting errors.

Financial Discrepancies: An inaccurate swap may leave the vault with either an excess or deficit of USDC relative to its debt position. Such imbalances can negatively impact the vault’s credit capacity and overall financial integrity.

## Tools Used

Manual Review, Foundry

## Recommendations

Instead of using the USDC collateral conversion result directly as the token input for the swap, modify the logic so that ctx.swapAmount is used to determine the expected output amount for slippage purposes. In practice, this means:

Continue to compute ctx.swapAmount using calculateSwapAmount as the expected output amount to pass to the dex adapter as a value to prevent slippage from being too high.

When executing the swap via \_convertAssetsToUsdc, pass the value obtained from:

```solidity
usdcCollateralConfig.convertSd59x18ToTokenAmount(ctx.vaultUnsettledRealizedDebtUsdX18.abs())
```

as the input token amount, rather than using ctx.swapAmount.

## <a id='H-09'></a>H-09. Locked credit capacity enforcement can be broken leading to vault inadequacy to cover debt position            



## Summary

VaultRouterBranch:: compares the delta (difference) between the vault’s credit capacity before and after the redeem to a locked credit threshold, rather than ensuring the vault’s final credit capacity remains above that threshold. As a result, users may be able to redeem assets even if it causes the vault’s remaining credit capacity to fall below the required locked level, potentially destabilizing the protocol’s financial guarantees.

## Vulnerability Details

The following check is made in VaultRouterBranch::redeem :

```solidity
if (
    ctx.creditCapacityBeforeRedeemUsdX18
        .sub(vault.getTotalCreditCapacityUsd())
        .lte(ctx.lockedCreditCapacityBeforeRedeemUsdX18.intoSD59x18())
) {
    revert Errors.NotEnoughUnlockedCreditCapacity();
}
```
the code is comparing the difference in credit capacity (beforeRedeem - afterRedeem) to the locked capacity threshold (ctx.lockedCreditCapacityBeforeRedeemUsdX18) to attempt to make sure that if the credit capacity delta is greater than the locked credit capacity before the state transition the function reverts . In the code block, this isnt what happens as it ends up checking if the credit capacity delta is less than or equal to the locked credit capacity before the state transition.

This is one of a two fold issue. The second issue is with the underlying logic because  the intended goal is to ensure that after redeeming, the vault’s remaining credit capacity stays above the locked threshold.
The intended design intends the vault always to maintain at least a certain amount of credit capacity (i.e., lockedCreditCapacity), as per the natspec which defines Vault::lockedcreditratio as:
```
/// @param lockedCreditRatio The configured ratio that determines how much of the vault's total assets can't be
    /// withdrawn according to the Vault's total debt, in order to secure the credit delegation system.
```
Due to the flawed comparison, a malicious or opportunistic user might be able to redeem an amount that causes the vault’s final credit capacity to drop below its required locked threshold without triggering an error or reverting.

Locked credit capacity is calculated in Vault::getLockedCreditCapacityUsd. See below:

```solidity
function getLockedCreditCapacityUsd(Data storage self)
        internal
        view
        returns (UD60x18 lockedCreditCapacityUsdX18)
    {
        SD59x18 creditCapacityUsdX18 = getTotalCreditCapacityUsd(self);
        lockedCreditCapacityUsdX18 = creditCapacityUsdX18.lte(SD59x18_ZERO)
            ? UD60x18_ZERO
            : creditCapacityUsdX18.intoUD60x18().mul(ud60x18(self.lockedCreditRatio));
    }

```
The total credit capacity is calculated by multiplying the total credit capacity by the lockedcreditratio set by the protocol when configuring the vault via Vault::update.

## Proof Of Code (POC)

```solidity
function test_lockedcreditrationotmaintained (uint128 vaultId,
        uint256 assetsToDeposit,
        uint128 marketId
    )
        external
    {
        vm.stopPrank();
       //c configure vaults and markets
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
         vm.assume(fuzzVaultConfig.asset != address(wBtc)); //c to avoid overflow issues
         vm.assume(fuzzVaultConfig.asset != address(usdc)); //c to log market debt
        PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);

       
        uint256[] memory marketIds = new uint256[](1);
        marketIds[0] = fuzzMarketConfig.marketId;
        

        uint256[] memory vaultIds = new uint256[](1);
        vaultIds[0] = fuzzVaultConfig.vaultId;

        vm.prank(users.owner.account);
        marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);
         Vault.UpdateParams memory params = Vault.UpdateParams({
            vaultId: fuzzVaultConfig.vaultId,
            depositCap: 10e18,
            withdrawalDelay: fuzzVaultConfig.withdrawalDelay,
            isLive: true,
            lockedCreditRatio: 0.5e18
        });


        //c update lockedcreditratio of vault by updating vault configuration
         vm.prank( users.owner.account );
        marketMakingEngine.updateVaultConfiguration(params);

        //c get updated deposit cap of vault
        uint128 depositCap = marketMakingEngine.workaround_Vault_getDepositCap(fuzzVaultConfig.vaultId);
        console.log(depositCap);

        //c user deposits into vault
        address userA = users.naruto.account;
        assetsToDeposit = depositCap;
        fundUserAndDepositInVault(userA, fuzzVaultConfig.vaultId, uint128(assetsToDeposit));

        //c perp engine deposits credit into market to incur debt
        deal(fuzzVaultConfig.asset, address(fuzzMarketConfig.engine), 100e18);
        vm.prank(address(fuzzMarketConfig.engine));
        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, fuzzVaultConfig.depositCap);
       

        uint128 userVaultShares = uint128(IERC20(fuzzVaultConfig.indexToken).balanceOf(userA));
        

        // initiate the withdrawal
        vm.startPrank(userA);
        marketMakingEngine.initiateWithdrawal(fuzzVaultConfig.vaultId, userVaultShares);

        // fast forward block.timestamp to after withdraw delay has passed
        skip(fuzzVaultConfig.withdrawalDelay + 1);

        
        //c get totalcreditcapacity before redeem

        /*c to run these tests, I added the following functions to vaultrouterbranch:
        //c for testing purposes

    function getLockedCreditCapacityUsd1(uint128 vaultId)
        external
        view
        returns (UD60x18 lockedCreditCapacityUsdX18)
    {
        // load the vault storage pointer
        Vault.Data storage self = Vault.loadLive(vaultId);
        SD59x18 creditCapacityUsdX18 = getTotalCreditCapacityUsd1(vaultId);
        lockedCreditCapacityUsdX18 = creditCapacityUsdX18.lte(SD59x18.wrap(0))
            ? UD60x18.wrap(0)
            : creditCapacityUsdX18.intoUD60x18().mul(ud60x18(self.lockedCreditRatio));
    }

    //c for testing purposes
    function getTotalCreditCapacityUsd1(uint128 vaultId) public view returns (SD59x18 creditCapacityUsdX18) {
        // load the vault storage pointer
        Vault.Data storage self = Vault.loadLive(vaultId);
        // load the collateral configuration storage pointer
        Collateral.Data storage collateral = self.collateral;

        // fetch the zlp vault's total assets amount
        UD60x18 totalAssetsX18 = ud60x18(IERC4626(self.indexToken).totalAssets());
        
        // calculate the total assets value in usd terms
        UD60x18 totalAssetsUsdX18 = collateral.getAdjustedPrice().mul(totalAssetsX18);
        // calculate the vault's credit capacity in usd terms
        
        creditCapacityUsdX18 = totalAssetsUsdX18.intoSD59x18().sub(Vault.getTotalDebt(self));
        
    }


        */


        SD59x18 totalcreditcapacitypreredeem =  marketMakingEngine.getTotalCreditCapacityUsd1(fuzzVaultConfig.vaultId);
        console.log("totalcreditcapacitypreredeem",totalcreditcapacitypreredeem.unwrap());

        //c get lockedcreditcapacity 
        UD60x18 lockedcreditcapacity = marketMakingEngine.getLockedCreditCapacityUsd1(fuzzVaultConfig.vaultId);
        console.log("lockedcreditcapacity", lockedcreditcapacity.unwrap());

        //c user redeem all their assets successfully
        marketMakingEngine.redeem(fuzzVaultConfig.vaultId, 1, 0);
        vm.stopPrank();

        //c the total credit capacity of the vault will now be less than the locked credit capacity due to the withdrawal which is not intended behaviour

        SD59x18 totalcreditcapacitypostredeem =  marketMakingEngine.getTotalCreditCapacityUsd1(fuzzVaultConfig.vaultId);
        console.log("totalcreditcapacitypostredeem",totalcreditcapacitypostredeem.unwrap());

        assert(uint256(totalcreditcapacitypostredeem.unwrap()) < lockedcreditcapacity.unwrap());
        assertEq(totalcreditcapacitypostredeem.unwrap(), 0);

    }

```

## Impact

Systemic Risk: Once the vault’s post-redeem credit capacity falls below the locked threshold, the protocol may be unable to honor future credit obligations or maintain sufficient reserves, putting the entire system at risk.
Financial Losses: If the vault cannot cover its obligations, users or other stakeholders could suffer substantial losses.
Exploitation Potential: Attackers may repeatedly exploit this bug to drain vault capacity, leading to destabilization and potential insolvency for the protocol.

## Tools Used
Manual Review, Foundry

## Recommendations

Compare Final Capacity to Locked Threshold:
Replace the capacity-delta check with a direct comparison between the vault’s new (post-redeem) total credit capacity and the locked threshold. For example:

```solidity
if (vault.getTotalCreditCapacityUsd()).lt(ctx.lockedCreditCapacityBeforeRedeemUsdX18.intoSD59x18())) {
    revert Errors.NotEnoughUnlockedCreditCapacity();
}
```


## <a id='H-10'></a>H-10. Incorrect Handling of Redeem Fee Discount in VaultRouterBranch::getIndexTokenSwapRate which leads to a redeem fees being double counted for users who require             



## Summary

The bug occurs in the way the redeem fee is applied during the calculation of expected asset output for share redemption. In the VaultRouterBranch::getIndexTokenSwapRate function, when the shouldDiscountRedeemFee flag is true, the redeem fee is subtracted from the computed preview assets out. However, in the overall redeem flow, the redeem fee is later subtracted from the expected assets. This double‑discounting (or misapplication) of the fee results in an incorrect expected asset value, causing users to receive less assets than they should. The bug is compounded by the fact that if the fee should be waived (i.e. not charged), the discount logic should instead add back the fee—effectively canceling the subtraction that occurs in the redeem function—rather than subtracting it.

## Vulnerability Details

Misapplied Fee Discount in VaultRouterBranch::getIndexTokenSwapRate:

The function getIndexTokenSwapRate computes a preliminary value, previewAssetsOut, based on the vault’s net credit capacity and share ratio.When shouldDiscountRedeemFee is true, the code subtracts the fee:

```solidity
previewAssetsOut = ud60x18(previewAssetsOut)
    .sub(ud60x18(previewAssetsOut).mul(ud60x18(vault.redeemFee)))
    .intoUint256();
```

This logic subtracts the fee from the preview assets, which then returns a lower asset amount. However, in VaultRouterBranch::redeem, after receiving expectedAssetsX18 from VaultRouterBranch::getIndexTokenSwapRate, the redeem fee is again applied by subtracting a fraction of expectedAssetsX18:

```solidity
ctx.expectedAssetsMinusRedeemFeeX18 = ctx.expectedAssetsX18.sub(
    ctx.expectedAssetsX18.mul(ud60x18(ctx.redeemFee))
);
This results in the fee being effectively applied twice if shouldDiscountRedeemFee is true.
```

If shouldDiscountRedeemFee is true, it indicates that the expected asset output should not be reduced by the fee at this stage (or should be adjusted such that later fee subtraction in the redeem function results in the correct net value). In other words, the fee should be added back (or not discounted) in getIndexTokenSwapRate so that the final redeem calculation subtracts the fee exactly once.

The current code subtracts the fee in getIndexTokenSwapRate, and then subtracts it again in the redeem flow, leading to an excessive fee deduction. As a consequence, users redeeming their shares receive fewer assets than expected. This miscalculation undermines user trust and can result in significant financial discrepancies, particularly in high‑volume or high‑value redemptions.

## Proof Of Code (POC)

```solidity
function test_redeemfeesubtractedtwice(uint128 vaultId,
        uint256 assetsToDeposit,
        uint128 marketId
    )
        external
    {
        vm.stopPrank();
       //c configure vaults and markets
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
         vm.assume(fuzzVaultConfig.asset != address(wBtc)); //c to avoid overflow issues
         vm.assume(fuzzVaultConfig.asset != address(usdc)); //c to log market debt
        PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);

       
        uint256[] memory marketIds = new uint256[](1);
        marketIds[0] = fuzzMarketConfig.marketId;
        

        uint256[] memory vaultIds = new uint256[](1);
        vaultIds[0] = fuzzVaultConfig.vaultId;

        vm.prank(users.owner.account);
        marketMakingEngine.connectVaultsAndMarkets(marketIds, vaultIds);
         Vault.UpdateParams memory params = Vault.UpdateParams({
            vaultId: fuzzVaultConfig.vaultId,
            depositCap: 10e18,
            withdrawalDelay: fuzzVaultConfig.withdrawalDelay,
            isLive: true,
            lockedCreditRatio: 0.5e18
        });


        //c update lockedcreditratio of vault by updating vault configuration
         vm.prank( users.owner.account );
        marketMakingEngine.updateVaultConfiguration(params);

        //c get updated deposit cap of vault
        uint128 depositCap = marketMakingEngine.workaround_Vault_getDepositCap(fuzzVaultConfig.vaultId);
        console.log(depositCap);

        //c user deposits into vault
        address userA = users.naruto.account;
        assetsToDeposit = depositCap;
        fundUserAndDepositInVault(userA, fuzzVaultConfig.vaultId, uint128(assetsToDeposit));

    uint128 userVaultShares = uint128(IERC20(fuzzVaultConfig.indexToken).balanceOf(userA));

        //c get predicted amount of assets out
        UD60x18 assetsOut  = marketMakingEngine.getIndexTokenSwapRate(fuzzVaultConfig.vaultId, userVaultShares, true);
        console.log(assetsOut.unwrap());

        
        // initiate the withdrawal
        vm.startPrank(userA);
        marketMakingEngine.initiateWithdrawal(fuzzVaultConfig.vaultId, userVaultShares);

        // fast forward block.timestamp to after withdraw delay has passed
        skip(fuzzVaultConfig.withdrawalDelay + 1);

        
        //c get totalcreditcapacity before redeem

        /*c 
        to run this test, I changed the following line in VaultRouterBranch::redeem to be able to set VaultRouterBranch::getIndexTokenSwapRate to true as there is currently no logic for any address to be able to be discounted from fees which is a seperate issue I reported

         RedeemContext memory ctx;
        ctx.shares = withdrawalRequest.shares;
        ctx.expectedAssetsX18 = getIndexTokenSwapRate(vaultId, ctx.shares, true);

        */


        //c user redeem all their assets successfully
        marketMakingEngine.redeem(fuzzVaultConfig.vaultId, 1, 0);
        vm.stopPrank();

        //c if fee should be discounted then the expectedassets out should be the same as the assets that user A redeems from the vault but it will be less which is proven by the assert below
        uint256 userAbal = IERC20(fuzzVaultConfig.asset).balanceOf(userA);
        assert(userAbal != assetsOut.unwrap());
        

    }
```

## Impact

Financial Loss: Users will receive less than the expected amount of underlying assets upon redemption due to the double application of the redeem fee.
Protocol Imbalance: The overall accounting within the vault may become inconsistent if the redeem fee is applied incorrectly, potentially affecting credit capacity calculations and subsequent financial operations.

## Tools Used
Manual Review, Foundry

## Recommendations

Correct Fee Discount Logic in getIndexTokenSwapRate:

Modify the fee application in getIndexTokenSwapRate so that when shouldDiscountRedeemFee is true, the redeem fee is added to (or not subtracted from) previewAssetsOut. This ensures that the final redeem fee is applied only once in the redeem function.

```solidity
if (shouldDiscountRedeemFee) {
    // Instead of subtracting, add back the fee percentage (or do not discount at all)
    previewAssetsOut = ud60x18(previewAssetsOut)
        .add(ud60x18(previewAssetsOut).mul(ud60x18(vault.redeemFee)))
        .intoUint256();
}
```
Note: The exact arithmetic may need to be adjusted to correctly cancel the later subtraction in the redeem function.


Reevaluate the overall redeem flow to ensure that the redeem fee is applied exactly once. Consider centralizing the fee adjustment logic either in getIndexTokenSwapRate or in the redeem function to avoid duplicated adjustments. Ensure that the shouldDiscountRedeemFee flag correctly reflects whether the fee should be waived or applied, and document the intended behavior clearly.
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Incorrect Logic in MarketMakingEngineConfigurationBranch::configureFeeRecipient can cause DOS when attempting to change share value of an existing address            



## Summary

There is a logical error in the MarketMakingEngineConfigurationBranch::configureFeeRecipient  that causes unwanted reverts when attempting to decrease an existing fee recipient’s share. The function checks the new share against the maximum configurable protocol fee limit before subtracting the old share. As a result, legitimate updates to an existing recipient’s share can incorrectly revert.

## Vulnerability Details

The bug occurs in the following lines in MarketMakingEngineConfigurationBranch::configureFeeRecipient :

```solidity
if (
    totalFeeRecipientsSharesX18.add(ud60x18(share)).gt(
        ud60x18(Constants.MAX_CONFIGURABLE_PROTOCOL_FEE_SHARES)
    )
) {
    revert Errors.FeeRecipientShareExceedsLimit();
}

```

This check adds the new share to marketmakingengineconfiguration::totalFeeRecipientsShares before subtracting the old share value. The total should first be decreased by the old share before the new share is added. Otherwise, a legitimate update (from a higher share to a lower share) will revert unnecessarily.

The order of operations in the share calculation is incorrect. The function does not account for the fact that the old share value should first be subtracted from the total fee recipients’ shares before verifying that the new share fits under the maximum limit.

## Impact

This bug can block legitimate reconfigurations of fee shares. Administrators may be prevented from decreasing a fee recipient’s share if the sum of current total shares plus the new share appears to exceed the limit—even when the final configuration would be valid.  This can halt necessary protocol governance or maintenance procedures, which may include Adjusting fee splits as market conditions change or correcting an over-allocated fee distribution.

## Proof Of Code (POC)
```solidity

function test_configurefeerecipientbug() external {
vm.stopPrank();

    vm.startPrank({ msgSender: users.owner.account });
    marketMakingEngine.configureFeeRecipient(users.sasuke.account, 0.6e18);
    vm.expectRevert(Errors.FeeRecipientShareExceedsLimit.selector);
    marketMakingEngine.configureFeeRecipient(users.sasuke.account, 0.4e18);
}
 
```

## Tools Used

Manual Review, Foundry

## Recommendations

```solidity

function configureFeeRecipient(address feeRecipient, uint256 share) external onlyOwner {
        // revert if protocolFeeRecipient is set to zero
        if (feeRecipient == address(0)) revert Errors.ZeroInput("feeRecipient");

        // load market making engine configuration data from storage
        MarketMakingEngineConfiguration.Data storage marketMakingEngineConfiguration =
            MarketMakingEngineConfiguration.load(); 

       (, uint256 oldFeeRecipientShares) = marketMakingEngineConfiguration.protocolFeeRecipients.tryGet(feeRecipient);

        // update protocol total fee recipients shares value
        if (oldFeeRecipientShares > 0) {
 UD60x18 totalFeeRecipientsSharesX18 = ud60x18(marketMakingEngineConfiguration.totalFeeRecipientsShares);
           UD60x18 previewtotalFeeRecipientsSharesX18 =  totalFeeRecipientsSharesX18.sub(ud60x18(oldFeeRecipientShares))

            if (
               previewtotalFeeRecipientsSharesX18.add(ud60x18(share)).gt(
                    ud60x18(Constants.MAX_CONFIGURABLE_PROTOCOL_FEE_SHARES)
                )
                
            ) {
                revert Errors.FeeRecipientShareExceedsLimit();
            }
            if (oldFeeRecipientShares > share) {
                marketMakingEngineConfiguration.totalFeeRecipientsShares -=
                    (oldFeeRecipientShares - share).toUint128();
                
            } else {
                marketMakingEngineConfiguration.totalFeeRecipientsShares +=
                    (share - oldFeeRecipientShares).toUint128();
              
            }
        } else {
            marketMakingEngineConfiguration.totalFeeRecipientsShares += share.toUint128();
        }

        // update protocol fee recipient
        marketMakingEngineConfiguration.protocolFeeRecipients.set(feeRecipient, share);

        // emit event LogConfigureFeeRecipient
        emit LogConfigureFeeRecipient(feeRecipient, share);
    }

```

The proposed fix properly reorders the logic to subtract the old fee recipient share before adding the new share, avoiding unwanted reverts. By using a “preview” total (subtracting oldFeeRecipientShares from the existing total) and then adding the new share for the limit check, you ensure no valid configuration is blocked when reducing a fee recipient’s share. This solution also maintains the integrity of the maximum fee constraint.

## <a id='M-02'></a>M-02.   Attacker can grief vaultrouterbranch::deposit via direct asset token transfers            



## Summary

A malicious actor can grief legitimate users from depositing into a vault using vaultrouterbranch::deposit by directly sending asset tokens to the vault contract, effectively “filling” the vault from the outside and preventing new deposits. Although this does not result in stolen funds, it creates a denial of service for depositors trying to use the vault normally.

## Vulnerability Details

VaultRouterBranch::deposit contains the following line which allows users deposit into a vault:

```solidity
 // then perform the actual deposit
        // NOTE: the following call will update the total assets deposited in the vault
        // NOTE: the following call will validate the vault's deposit cap
        // invariant: no tokens should remain stuck in this contract
        ctx.shares = IERC4626(indexTokenCache).deposit(ctx.assetsMinusFees, msg.sender);

```

This calls ZlpVault::deposit which in turns calls ERC4626Upgradeable::deposit:

```solidity
 /** @dev See {IERC4626-deposit}. */
    function deposit(uint256 assets, address receiver) public virtual returns (uint256) {
        uint256 maxAssets = maxDeposit(receiver);
        if (assets > maxAssets) {
            revert ERC4626ExceededMaxDeposit(receiver, assets, maxAssets);
        }

        uint256 shares = previewDeposit(assets);
        _deposit(_msgSender(), receiver, assets, shares);

        return shares;
    }

```

The bug is located in ZlpVault::maxDeposit :

```solidity

    /// @notice Returns the maximum amount of assets that can be deposited into the ZLP Vault, taking into account the
    /// configured deposit cap.
    /// @dev Overridden and used in ERC4626.
    /// @return maxAssets The maximum amount of depositable assets.
    function maxDeposit(address) public view override returns (uint256 maxAssets) {
        // load the zlp vault storage pointer
        ZlpVaultStorage storage zlpVaultStorage = _getZlpVaultStorage();
        // cache the market making engine contract
        IMarketMakingEngine marketMakingEngine = IMarketMakingEngine(zlpVaultStorage.marketMakingEngine);

        // get the vault's deposit cap
        uint128 depositCap = marketMakingEngine.getDepositCap(zlpVaultStorage.vaultId);

        // cache the vault's total assets
        uint256 totalAssetsCached = totalAssets(); 

        // underflow check here would be redundant
        unchecked {
            // we need to ensure that depositCap > totalAssets, otherwise, a malicious actor could grief deposits by
            // sending assets directly to the vault contract and bypassing the deposit cap
            maxAssets = depositCap > totalAssetsCached ? depositCap - totalAssetsCached : 0;
        }
        
    }
```

ZlpVault::maxDeposit() function in the vault contract, which calculates the remaining allowable deposit based on depositCap - totalAssets().
The vault’s logic only checks totalAssets() against depositCap, but does not handle externally transferred tokens. Attackers can bypass the normal deposit() flow by sending tokens directly to the vault contract. Once the vault’s actual balance (totalAssets()) meets or exceeds depositCap, maxDeposit() returns 0, blocking legitimate depositors from adding new assets.

## Proof Of Code (POC)

```solidity
function test_griefingdeposits(
        uint256 vaultId,
        uint256 amount
    )
        external
    {
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
       
        //c user who will grief deposits
        address userA = users.naruto.account;
        deal(fuzzVaultConfig.asset, userA, 100e18);
        

        //c user trying to deposit into vault
        address userB = users.sasuke.account;
        deal(fuzzVaultConfig.asset, userB, 100e18);
     

        //c get deposit cap of vault
        uint128 depositCap = vaultsConfig[fuzzVaultConfig.vaultId].depositCap;
       

        //c get the deposit fee of the vault
        uint128 depositFee = uint128(vaultsConfig[fuzzVaultConfig.vaultId].depositFee);

        

         amount =bound(amount, calculateMinOfSharesToStake(fuzzVaultConfig.vaultId), fuzzVaultConfig.depositCap / 2);
        //c get assets sent to vault index token 
         uint128 depositfeeforamount = uint128(amount)*depositFee/1e18;
        uint256 assetminusfee = amount - uint256(depositfeeforamount);
         

        //c attempt to grief deposits
        vm.startPrank(userA);
        IERC20(fuzzVaultConfig.asset).transfer(fuzzVaultConfig.indexToken, depositCap);
        vm.stopPrank();


        //c check totalassets of vault
        uint256 totalAssets = IERC4626(fuzzVaultConfig.indexToken).totalAssets();
        console.log(totalAssets);

        //c user attempts to deposits into vault
        vm.startPrank(userB);
        vm.expectRevert(abi.encodeWithSelector(ERC4626Upgradeable.ERC4626ExceededMaxDeposit.selector,userB,assetminusfee,0));
        marketMakingEngine.deposit(fuzzVaultConfig.vaultId, uint128(amount), 0, "", false);
        vm.stopPrank(); 

        
    }
```

## Impact

Denial-of-Service: Users are unable to deposit once an attacker inflates the vault’s balance, rendering the deposit function unusable. This directly undermines the vault’s usability and could erode user trust in the protocol’s deposit cap mechanism.

## Tools Used

Manual Review, Foundry

## Recommendations

Recommendation: Implement a Change-Detection Mechanism for ERC4626Upgreadeable::totalAssets()

Introduce a variable (e.g., ZlpVault::lastTotalAssets) that stores the vault’s previously observed total assets. Whenever ZlpVault::maxDeposit() is called:

Compute Current totalAssets(): Call ERC4626Upgreadeable::totalAssets() to get the current balance.

Compare totalAssets() with ZlpVault::lastTotalAssets.
If there is a difference, check whether the vault’s total share supply (e.g., totalSupply()) has also changed by a corresponding amount based on the share price formula. If the share supply does not reflect this change, assume the variance in totalAssets() is caused by an external transfer (i.e., not due to a legitimate deposit transaction).

If an external transfer is detected, ignore the externally inflated amount by using lastTotalAssets for deposit cap calculations (so malicious tokens sent directly to the contract do not reduce the user’s allowable deposit). Otherwise, use the newly computed ERC4626Upgradeable::totalAssets() for correct deposit cap tracking.

Once a legitimate deposit is verified (i.e., the share supply changed accordingly), update lastTotalAssets to the new totalAssets() value. This approach ensures that only valid deposit transactions (those which increase both assets and shares) will raise the vault’s internal “cap usage,” while malicious external transfers are effectively disregarded when calculating maxDeposit().

## <a id='M-03'></a>M-03. Hardcoded Fee Discount Flag Prevents Dynamic Whitelisting for Redeem Fee Discounts            



## Summary

VaultRouterBranch::getIndexTokenSwapRate is called with the shouldDiscountRedeemFee parameter hardcoded as false in every function in the zaros contracts. This design flaw prevents the protocol from dynamically granting discounted redeem fees to specific addresses. Once deployed, there is no mechanism for selectively applying fee discounts—effectively locking out any potential for whitelisting addresses that should benefit from reduced fees.

## Vulnerability Details


In the redeem flow for VaultRouterBranch::redeem, the getIndexTokenSwapRate function is invoked with the shouldDiscountRedeemFee parameter set to a constant false boolean. This hardcoding means that the redeem fee discount logic is not configurable per user or address:

```solidity
ctx.expectedAssetsX18 = getIndexTokenSwapRate(vaultId, ctx.shares, true);
```
// (For proof-of-concept purposes this may have been temporarily set, but once deployed no dynamic control exists.)

Without dynamic control over the fee discount parameter, there is no way to provide fee discounts for privileged addresses (for example, partners, early adopters, or certain strategic users). This prevents the protocol from offering any custom fee structure.
As the fee discount flag is hardcoded, the protocol cannot adjust fees on a per-address basis without redeploying or upgrading the contract. This limitation reduces flexibility and may be inconsistent with the intended business model.

This behaviour is analogous wherever VaultRouterBranch::getIndexTokenSwapRate is called in any function in the zaros contracts in scope.

## Impact

Inflexible Fee Policy:  The inability to dynamically apply fee discounts means that all users are subject to the same fee structure, even if the protocol’s economic model intends to incentivize certain users with discounted fees.

Operational Limitations: The lack of whitelisting increases administrative overhead since adjusting fee discount policies would require contract redeployment or an upgrade, rather than a simple update of a whitelist mapping.

## Tools Used

Manual Review

## Recommendations

Introduce a Whitelist Storage Variable: Create a storage variable (e.g., a mapping of addresses to a boolean) to maintain a whitelist of addresses that are eligible for discounted redeem fees.

```solidity
mapping(address => bool) public feeDiscountWhitelist;
```

Pass Fee Discount as a Parameter in the Redeem Function: Modify the redeem function to accept a bool shouldDiscountRedeemFee parameter. This parameter should not be hardcoded but instead determined based on whether the caller is whitelisted.

Implement a Whitelist Check: In all relevant functions, check if msg.sender (or the relevant address) is in the whitelist:

```solidity
bool applyDiscount = feeDiscountWhitelist[msg.sender];
ctx.expectedAssetsX18 = getIndexTokenSwapRate(vaultId, ctx.shares, applyDiscount);
```

Provide Administrative Functions: Add functions that allow protocol administrators to add or remove addresses from the whitelist, enabling dynamic adjustments to fee discount eligibility.



# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect Naming Convention and Description for Market::wethRewardPerVaultShare            



## Summary

There is a misalignment between the variable’s name Market::wethRewardPerVaultShare and its actual usage and meaning . The name suggests that the variable holds a per-share (i.e., a ratio) amount of WETH rewards relative to vault credit. This is also inferred in the docs description which say:

```solidity
/// @param wethRewardPerVaultShare The amount of weth reward accumulated by the market per vault delegated credit
```

In practice, it simply stores the total wETH reward accumulated for the entire market.

## Vulnerability Details

See market::receiveWethReward below:

```solidity

    function receiveWethReward(
        Data storage self,
        address asset,
        UD60x18 receivedProtocolWethRewardX18,
        UD60x18 receivedVaultsWethRewardX18
    )
        internal
    {
        // if a market credit deposit asset has been used to acquire the received weth, we need to reset its balance
        if (asset != address(0)) {
            // removes the given asset from the received market fees enumerable map as we assume it's been fully
            // swapped to weth
            self.receivedFees.remove(asset);
        }

        // increment the amount of pending weth reward to be distributed to fee recipients
        self.availableProtocolWethReward =
            ud60x18(self.availableProtocolWethReward).add(receivedProtocolWethRewardX18).intoUint128();

        // increment the all time weth reward storage
        self.wethRewardPerVaultShare =
            ud60x18(self.wethRewardPerVaultShare).add(receivedVaultsWethRewardX18).intoUint128();
           
    }
```

In this function, self.wethRewardPerVaultShare is incremented by receivedVaultsWethRewardX18 which is the total vaults weth reward for the market which is different to the description in the natspec about the variable. There is no division or per-share calculation involved. This naming mismatch can create confusion for anyone reading or maintaining the code.

## Impact

Misleading naming can cause future maintainers to incorrectly use the variable or build features on top of wrong assumptions, possibly introducing errors in reward distribution logic or reporting down the line. It can lead to confusion or misimplementation if developers rely on the inaccurate name or documentation to make assumptions about how rewards are calculated.

## Tools Used

Manual Review

## Recommendations

Rename the Variable: Change market::wethRewardPerVaultShare to a more descriptive name, such as totalWethRewardsForVaults, accumulatedVaultWethRewards, or similar.

Update Documentation: Ensure that the natspec comments and any external references reflect the true nature of the variable (i.e., it is a total accumulated reward rather than a per-share metric).

Confirm Future Calculation Requirements: If a per-share metric is needed, add additional logic or a separate variable that divides the accumulated WETH amount by the total delegated credit to correctly represent a per-share figure.



