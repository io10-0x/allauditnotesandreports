# Ignite - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Absence of Slashing Logic for QI Stakers which leads to incentive misalignment and protocol imbalance](#H-01)
  - ### [H-02. Admin can delete initial price feed configurations for avax and qi tokens breaking protocol invariant](#H-02)
- ## Medium Risk Findings
  - ### [M-01. DOS can occur in `Ignite::registerWithStake` which restricts users from registering stake](#M-01)
  - ### [M-02. Hardcoded Values lead to inability to update fee percentage and minimum avax requirement for QI deposits](#M-02)
  - ### [M-03. Denial of Service (DoS) Due to Handling of Chainlink Price Feeds with N/A Decimals](#M-03)
- ## Low Risk Findings
  - ### [L-01. Redundant Code in Ignite Contract](#L-01)
  - ### [L-02. Inconsistency in QI Deposit Calculation for registerWithStake](#L-02)
  - ### [L-03. Handling of Tokens with Decimals Greater Than 18 in registerWithErc20Fee causes DOS](#L-03)
  - ### [L-04. Incorrect Qi Reward Emission in Ignite::releaseLockedTokens ](#L-04)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Benqi

### Dates: Jan 13th, 2025 - Jan 27th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-01-benqi)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 2
- Medium: 3
- Low: 4

# High Risk Findings

## <a id='H-01'></a>H-01. Absence of Slashing Logic for QI Stakers which leads to incentive misalignment and protocol imbalance

## Summary

Allows QI stakers to redeem their stake and rewards after the validation period ends, even if they have misbehaved during the validation period. This critical oversight undermines the protocol's integrity by enabling malicious actors to exploit the staking system without facing penalties. The absence of a slashing mechanism creates significant incentive misalignment, as QI stakers are rewarded regardless of their behavior, encouraging exploitation and jeopardizing the protocol's long-term security.

## Vulnerability Details

The `staking::registerwithstake` and `staking::registerwithprevalidatedqistake` functions are as follows:

```solidity
 /**
     * @notice Register a new node for validation and lock up QI and AVAX
     * @param  nodeId Node ID of the validator
     * @param  blsProofOfPossession BLS proof of possession (public key + signature)
     * @param  validationDuration Duration of the validation in seconds
     */
    function registerWithStake(
        string calldata nodeId,
        bytes calldata blsProofOfPossession,
        uint validationDuration
    ) external payable nonReentrant {
        require(
            msg.value >= minimumAvaxDeposit &&
            msg.value <= maximumAvaxDeposit &&
            msg.value % 1e9 == 0,
            "Invalid value"
        );

        // Verify that the sender can receive AVAX
        (bool success, ) = msg.sender.call("");
        require(success);

        (, int256 avaxPrice, , uint avaxPriceUpdatedAt, ) = priceFeeds[AVAX].latestRoundData();
        (, int256 qiPrice, , uint qiPriceUpdatedAt, ) = priceFeeds[address(qi)].latestRoundData();

        require(qiPrice > 0 && avaxPrice > qiPrice);
        require(block.timestamp - avaxPriceUpdatedAt <= maxPriceAges[AVAX]);
        require(block.timestamp - qiPriceUpdatedAt <= maxPriceAges[address(qi)]);

        // QI deposit amount is 10 % (thus, note the divider) of the AVAX value
        // that BENQI subsidises for the validator.
        uint qiAmount = uint(avaxPrice) * (2000e18 - msg.value) / uint(qiPrice) / 10;

        require(qiAmount > 0);

        qi.safeTransferFrom(msg.sender, address(this), qiAmount);

        _registerWithChecks(
            msg.sender,
            nodeId,
            blsProofOfPossession,
            validationDuration,
            false,
            address(qi),
            qiAmount,
            false
        );
    }

 /**
     * @notice Register a new node with a prevalidated QI deposit amount
     * @param  beneficiary User on whose behalf the registration is made
     * @param  nodeId Node ID of the validator
     * @param  blsProofOfPossession BLS proof of possession (public key + signature)
     * @param  validationDuration Duration of the validation in seconds
     * @param  qiAmount The amount of QI that was staked
     */
    function registerWithPrevalidatedQiStake(
        address beneficiary,
        string calldata nodeId,
        bytes calldata blsProofOfPossession,
        uint validationDuration,
        uint qiAmount
    ) external {
        require(
            hasRole(ROLE_REGISTER_WITH_FLEXIBLE_PRICE_CHECK, msg.sender),
            "ROLE_REGISTER_WITH_FLEXIBLE_PRICE_CHECK"
        );

        (, int256 avaxPrice, , uint avaxPriceUpdatedAt, ) = priceFeeds[AVAX].latestRoundData();
        (, int256 qiPrice, , uint qiPriceUpdatedAt, ) = priceFeeds[address(qi)].latestRoundData();

        require(avaxPrice > 0 && qiPrice > 0);
        require(block.timestamp - avaxPriceUpdatedAt <= maxPriceAges[AVAX]);
        require(block.timestamp - qiPriceUpdatedAt <= maxPriceAges[address(qi)]);

        // 200 AVAX + 1 AVAX fee
        uint expectedQiAmount = uint(avaxPrice) * 201e18 / uint(qiPrice);

        require(qiAmount >= expectedQiAmount * 9 / 10);

        qi.safeTransferFrom(msg.sender, address(this), qiAmount);

        _registerWithChecks(
            beneficiary,
            nodeId,
            blsProofOfPossession,
            validationDuration,
            false,
            address(qi),
            qiAmount,
            true
        );
    }


```

Both of these functions have similar behaviour as they take a staked amount of assets from the user to start a validator. At the end of the validation period, rewards are paid to both of these stakers. For users who stake with `Ignite::registerwithstake`, their rewards are sent in avax to `Ignite::releaseLockedTokens`. See relevant part of function that assigns rewards below:

```solidity
} else if (msg.value > registration.tokenDeposits.avaxAmount) {
            uint rewards = msg.value - registration.tokenDeposits.avaxAmount;

            registration.rewardAmount = rewards;
            minimumContractBalance += msg.value;
```

Rewards are then paid out in `Ignite::redeemAfterExpiry` with the following lines:

```solidity
  else {
            avaxRedemptionAmount = avaxDepositAmount + registration.rewardAmount;
            qiRedemptionAmount = qiDepositAmount;

            if (qiRewardEligibilityByNodeId[nodeId]) {
                qiRedemptionAmount += validatorRewarder.claimRewards(
                    registration.validationDuration,
                    qiDepositAmount
                );
            }

            minimumContractBalance -= avaxRedemptionAmount;
        }

        _deleteRegistration(nodeId);

        qi.safeTransfer(msg.sender, qiRedemptionAmount);

        if (avaxRedemptionAmount > 0) {
            (bool success, ) = msg.sender.call{ value: avaxRedemptionAmount}("");
            require(success);
        }
```

In the above lines you can also see how users who register with `Ignite::registerwithprevalidatedqistake` are rewarded which is via `ValidatorRewarder::claimRewards`. These are where the similarities between users who call `Ignite::registerwithstake` and users who call `Ignite::registerwithprevalidatedqistake` ends.

The discrepancy lies in the slashing mechanism of these 2 registrations methods. Bad actors who register with `Ignite::registerwithstake` are slashed in `Ignite::releaseLockedTokens` with the following lines:

```solidity
else {
            require(msg.value == registration.tokenDeposits.avaxAmount);

            registration.slashed = true;

            uint qiSlashAmount;
            uint avaxSlashAmount;

            if (registration.qiSlashPercentage > 0) {
                // tokenDeposits.tokenAmount is always denominated in QI for stake model registrations
                qiSlashAmount = registration.tokenDeposits.tokenAmount * registration.qiSlashPercentage / 10_000;

                qi.safeTransfer(SLASHED_TOKEN_RECIPIENT, qiSlashAmount);
            }

            if (registration.avaxSlashPercentage > 0) {
                avaxSlashAmount = registration.tokenDeposits.avaxAmount * registration.avaxSlashPercentage / 10_000;

                minimumContractBalance += msg.value - avaxSlashAmount;

                (bool success, ) = SLASHED_TOKEN_RECIPIENT.call{ value: avaxSlashAmount }("");
                require(success);
            } else {
                minimumContractBalance += msg.value;
            }

            emit ValidatorSlashed(nodeId, qiSlashAmount, avaxSlashAmount);
        }
```

There is no corresponding logic to slash bad actors who register with `Ignite::registerwithprevalidatedqistake`. `Ignite::releaseLockedTokens` only sets `Ignite::qiRewardEligibilityByNodeId[nodeId]` to false if the validator setup failed. The only other applicable logic is below:

```solidity

        if (qiRewardEligibilityByNodeId[nodeId]) {
            require(msg.value == 0);

            uint fee = registration.tokenDeposits.tokenAmount / 201;
            registration.tokenDeposits.tokenAmount -= fee;

            qi.safeTransfer(FEE_RECIPIENT, fee);

            emit ValidatorRewarded(nodeId, 0);}
```

which takes a fee from the user which happens regardless of whether the actor is bad or good. There is no other logic to handle bad actors who register with `Ignite::registerwithprevalidatedqistake`. As a result, there is no incentive for actors who register via this function to act honestly.

This is different to users who register with fees via `Ignite::registerwithAVAXfee` or `Ignite::registerWithErc20Fee` as users pay the entire registration fee in AVAX at the time of calling `Ignite::registerwithAVAXfee` or `Ignite::registerWithErc20Fee`.
Once the fee is paid, the user has no remaining stake or deposit in the protocol that could be slashed. Unlike users who register with QI or AVAX stakes, these users do not have any staked collateral tied to their validator registration. Slashing is typically implemented as a penalty on staked collateral, which is absent in this case.

See the following example for an end to end application of this exploit. A malicious staker participates in the validation process with no intention of honest behavior by calling `Ignite::registerwithprevalidatedqistake`. During the validation period, the staker misbehaves (e.g., fails to validate correctly or acts maliciously). After the validation period and `Ignite::releaseLockedTokens` being called by the address with the correct role, the staker calls `Ignite::redeemAfterExpiry`. Despite misbehavior, the staker redeems both their initial deposit and earned rewards after the validation period ends.The protocol cannot enforce penalties, incentivizing further malicious activity.

## Impact

Unfair Rewards for Misbehavior: Misbehaving QI stakers receive full rewards and deposit refunds, leading to economic inefficiency and unfair treatment of honest participants.

Protocol Exploitation: Bad actors can dominate staking, undermine the validation process, and destabilize the system without facing any consequences.

Incentive Misalignment: Users who register via `Ignite::registerwithstake` are penalised for misbehavior by slashing, while QI stakers face no equivalent penalty, creating an imbalance in the system. Honest validators lose motivation to participate if misbehaving validators are rewarded equally.

## Tools Used

Manual Review

## Recommendations

There are a number of ways to mitigate this. One possible way would be to include an extra parameter in `Ignite::releaseLockedTokens` which would specify whether or not the qi staker misbehaved and then implement a slashing mechanism for users who fit this criteria. This would remove the incentive misalignment and ensure all actors are incentived to act in an honest manner.

## <a id='H-02'></a>H-02. Admin can delete initial price feed configurations for avax and qi tokens breaking protocol invariant

## Summary

The protocol documentation states that "AVAX and QI price feed configurations must not be deleted", implying that these configurations are immutable. However, an admin with the DEFAULT_ADMIN_ROLE can bypass this guarantee by removing AVAX or QI as payment tokens using `Ignite::removePaymentToken` and re-adding AVAX or QI with a new price feed address or configuration using `Ignite::addPaymentToken`. This loophole enables the admin to reset the price feed configurations for AVAX and QI, potentially introducing misconfigurations, and contradicts the protocol’s stated guarantees.

## Vulnerability Details

See both relevant functions below:

```solidity

    /**
     * @notice Add a new fee payment token option
     * @param  token ERC-20 token address
     * @param  priceFeedAddress Chainlink price feed address for the token
     * @param  maxPriceAge Maximum price feed response age
     */
    function addPaymentToken(
        address token,
        address priceFeedAddress,
        uint maxPriceAge
    ) external {
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));
        require(token != address(0));
        require(!paymentTokens.contains(token));

        IPriceFeed priceFeed = IPriceFeed(priceFeedAddress);

        paymentTokens.add(token);
        priceFeeds[token] = priceFeed;
        maxPriceAges[token] = maxPriceAge;

        (, int256 price, , uint updatedAt, ) = priceFeed.latestRoundData();
        require(price > 0);
        require(block.timestamp - updatedAt <= maxPriceAge);

        emit PaymentTokenAdded(token);
    }

    /**
     * @notice Remove an existing fee payment token option
     * @param  token Token to remove
     */
    function removePaymentToken(address token) external {
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));
        require(paymentTokens.contains(token));

        // AVAX and QI price feed configuration must not be deleted
        if (token != AVAX && token != address(qi)) {
            delete priceFeeds[token];
            delete maxPriceAges[token];
        }

        paymentTokens.remove(token);

        emit PaymentTokenRemoved(token);
    }

```

In the above code snippet, it states that AVAX and QI price feed configuration must not be deleted. This can easily be bypassed by the admin via the poc below which displays how this can be done:

```javascript
it("breakavax and qi cannot be deleted invariant", async function () {
        //remove qi address as payment token
        await ignite.removePaymentToken(qi.address);
        //check that qi address is still in the payment tokens
        expect(await ignite.getTotalErc20PaymentMethods()).to.equal(0);
        //the qi pricefeedaddress should still be the same as per the invariant
        expect(await ignite.priceFeeds(qi.address)).to.equal(
          qiPriceFeed.address
        );
        //admin will now re add qi as payment token but change the pricefeed address which breaks the invariant
        newqiPriceFeed = await deployPriceFeed(2_000_000);
        await ignite.addPaymentToken(qi.address, newqiPriceFeed.address, 120);
        //check that the pricefeed address has changed
        expect(await ignite.priceFeeds(qi.address)).to.equal(
          newqiPriceFeed.address
        );
      });
    });
```

Running this test in ignite.test.js file will pass and shows that the original qi price feed address set by the protocol can indeed be deleted by the admin which breaks the invariant. An analogous test case involving the avax price feed address will also pass.

## Impact

Misconfiguration Risk: The ability to reset the AVAX or QI price feed configurations breaks the invariant set in the documentation and poses a systemic threat to the protocol as most user facing functions depend on the correctness and efficiency of the avax and qi price feed addresses. These functions include `Ignite::registerwithStake`, `Ignite::registerWithErc20Fee` and `Ignite::registerWithPrevalidatedQiStake`

A negligent admin could point these price feeds to incorrect addresses that do not represent valid price feeds, causing complete breakdowns in token valuation mechanisms or malicious addresses that could feed manipulated or exploitable data into the protocol, leading to financial loss or economic attacks against users and validators.

## Tools Used

Hardhat, Manual Review

## Recommendations

To align with the protocol's intended functionality, AVAX and QI should not be removable as payment tokens under any circumstances, even by an admin. `Ignite::removePaymentToken` function should explicitly prevent the removal of AVAX and QI to ensure that their configurations remain intact and immutable.

The updated function should look like this:

```solidity
/**
 * @notice Remove an existing fee payment token option
 * @param  token Token to remove
 */
function removePaymentToken(address token) external {
    require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender), "Not authorized");
    require(paymentTokens.contains(token), "Token not found");

    // Prevent removal of AVAX and QI
    require(token != AVAX && token != address(qi), "Cannot remove AVAX or QI");

    delete priceFeeds[token];
    delete maxPriceAges[token];
    paymentTokens.remove(token);

    emit PaymentTokenRemoved(token);
}
```

By making AVAX and QI immutable as payment tokens, the protocol eliminates the risk of accidental or malicious removal by an admin, ensuring system stability.

Potential Drawback

Chainlink may decide to migrate or change the price feed contract addresses for AVAX or QI. If this happens, the current implementation would leave the protocol unable to update these price feeds. This could lead to stale or invalid price data, which would disrupt staking, fee calculations, and validator operations.

To address the drawback above, remove the invariant entirely by allowing AVAX and QI to be removed or re-added as payment tokens. This approach provides flexibility for updating price feeds but requires robust safeguards to prevent misuse. See below:

```solidity

/**
 * @notice Remove an existing fee payment token option
 * @param  token Token to remove
 */
function removePaymentToken(address token) external {
    require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender), "Not authorized");
    require(paymentTokens.contains(token), "Token not found");

    delete priceFeeds[token];
    delete maxPriceAges[token];
    paymentTokens.remove(token);

    emit PaymentTokenRemoved(token);
}
```

Ensure that any removal of AVAX or QI is immediately followed by re-adding them with updated price feed configurations or invoke the pause function to pause all operations in the contract and make the configuration to avoid DOS to any actors trying to call functions during re-addition.

# Medium Risk Findings

## <a id='M-01'></a>M-01. DOS can occur in `Ignite::registerWithStake` which restricts users from registering stake

## Summary

The `Ignite::registerWithStake` function in the Ignite contract performs a call operation to msg.sender with a 0 AVAX value to validate that the sender can receive AVAX. However, if the msg.sender is a contract with a receive function that reverts when it receives 0 AVAX, this behavior causes a Denial of Service (DoS), preventing such contracts from interacting with `Ignite::registerWithStake`.

## Vulnerability Details

The vulnerability lies in the following lines of the `Ignite::registerWithStake` function:

```solidity
(bool success, ) = msg.sender.call("");
require(success);
```

The call sends a 0 AVAX value to msg.sender to ensure that the recipient address can receive funds.
If msg.sender is a contract that reverts upon receiving 0 AVAX, the registerWithStake function will fail, making it unusable for users interacting through such contracts.

## POC

An example contract would have the following code:

```solidity
//SDPX-License-Identifier: MIT

pragma solidity ^0.8.0;

import {Ignite} from "./Ignite.sol";

contract Contractwithweirdreceive {
    bytes public constant blsKey =
        "8d609cdd38ffc9ad01c91d1ae4fccb8cd6c75a6ad33a401da42283b0c3b59bbaf5abc172335ea4d9c31baa936818f0ab";
    bytes public constant blsSignature =
        "8c12c805e7dfe4bfe38be44685ee852d931d73b3c0820a1343d731909120cee4895f9b60990520a90d06a031a42e0f8616d415b543408c24be0da90d5e7fa8242f4fd32dadf34c790996ca474dbdbcd763f82c53880db19fd3b30d13cee278b4";
    bytes public constant blsPoP = abi.encodePacked(blsKey, blsSignature);
    Ignite public ignite;

    constructor(address igniteaddress) {
        ignite = Ignite(igniteaddress);
    }

    function callregisterwithstake() external payable {
        ignite.registerWithStake{value: 500 ether}(
            "NodeID-1",
            blsPoP,
            86400 * 14
        );
    }

    receive() external payable {
        if (msg.value == 0) {
            revert("Don't send 0 AVAX");
        }
    }
}

```

Add the following test to the Ignite.test.js file in the repo:

```javascript
it("registerwithstakerevertswrongly with contract with conditional receive", async function () {
  //c test that contract with conditional receive reverts when calling registerwithstake
  await setup();

  const conditionalreceivecontract = await hre.ethers.getContractFactory(
    "Contractwithweirdreceive"
  );
  const receivecontract = await conditionalreceivecontract.deploy(
    ignite.address
  );
  await receivecontract.deployed();

  // Set AVAX price to $20 and QI price to $0.01
  await avaxPriceFeed.setPrice(2_000_000_000);
  await qiPriceFeed.setPrice(1_000_000);

  // Mint 10M QI and allow Ignite to spend it
  await qi.mint(hre.ethers.utils.parseEther("10000000"));
  await qi.approve(ignite.address, hre.ethers.utils.parseEther("10000000"));

  expect(receivecontract.callregisterwithstake()).to.be.reverted;
});
```

## Impact

Denial of Service: Users interacting with the Ignite contract via proxy contracts that reject 0 AVAX will be unable to call `Ignite::registerWithStake`, effectively locking them out of the protocol. The requirement for all contracts interacting with `Ignite::registerWithStake` to accept 0 AVAX creates unnecessary restrictions, potentially alienating users.

## Tools Used

Hardhat

## Recommendations

- Remove the 0 AVAX Call:

If validating the recipient's ability to accept funds is not critical, consider removing the call operation entirely.

```diff
- (bool success, ) = msg.sender.call("");
- require(success);
```

Check Code Length:

Instead of sending 0 AVAX, check whether msg.sender is a contract by verifying its code length:

```solidity
require(msg.sender.code.length == 0, "Contracts not allowed");
```

- Graceful Failure:

Allow the function to proceed even if the call fails. For example:

```solidity
(bool success, ) = msg.sender.call("");
if (!success) {
    emit Warning("Recipient cannot receive funds, but continuing");
}
```

## <a id='M-02'></a>M-02. Hardcoded Values lead to inability to update fee percentage and minimum avax requirement for QI deposits

## Summary

&#x20;` Ignite::releaseLockedTokens` function calculates a fee as 0.5% of the QI deposit using the formula:

```solidity
uint fee = registration.tokenDeposits.tokenAmount / 201;
```

This fee calculation is hardcoded into the contract, meaning the percentage (0.5%) cannot be adjusted without deploying an upgraded contract. This lack of flexibility poses a significant risk to the protocol's ability to adapt to future changes in tokenomics, economic conditions, or governance decisions, potentially leading to financial inefficiencies or suboptimal outcomes.

Analogous reasoning for the minimum avax requirement in `Ignite::registerWithPrevalidatedQiStake` function which also contains a hardcoded minimum AVAX requirement of 201 AVAX, which is used to calculate the expected QI deposit:

```solidity
uint expectedQiAmount = (uint(avaxPrice) * 201e18) / uint(qiPrice);
```

This value is fixed in the contract, meaning that any adjustments to the minimum AVAX requirement would require deploying a new contract.

## Vulnerability Details

The critical portions of the code are:

```solidity
uint fee = registration.tokenDeposits.tokenAmount / 201; //c dividing by 201 to get 0.5% of the token amount
```

```solidity
uint expectedQiAmount = (uint(avaxPrice) * 201e18) / uint(qiPrice);
```

Hardcoded Value 1: The divisor 201 hardcodes the fee percentage to 0.5% (1/201).
Hardcoded Value 2: The multiplier 201e18 represents the minimum AVAX amount (200 AVAX stake + 1 AVAX hosting fee) required for `Ignite::registerWithPrevalidatedQiStake`.

The fee percentage and minimum AVAX amounts cannot be modified without redeploying the entire contract, which may be impractical or infeasible in certain scenarios. This is inconsistent with the `Ignite::maximumSubsidisationAmount`, `Ignite::avaxSlashPercentage` , `Ignite:: qiPriceMultiplier` and `Ignite::qiSlashPercentage` variables in the ignite contract which all have setter functions that allow the user with DEFAULT_ADMIN_ROLE to change these variables at any time which allows for better adaptability to user requests and economic demand. See below:

```solidity
 /**
     * @notice Change the QI slash percentage
     * @param  newPercentage The new percentage in bps
     */
    function setQiSlashPercentage(uint newPercentage) external {
        //c makes sense
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));
        require(newPercentage <= 10_000);

        uint oldPercentage = qiSlashPercentage;
        qiSlashPercentage = newPercentage;

        emit QiSlashPercentageChanged(oldPercentage, newPercentage);
    }

    /**
     * @notice Change the AVAX slash percentage
     * @param  newPercentage The new percentage in bps
     */
    function setAvaxSlashPercentage(uint newPercentage) external {
        //c makes sense
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));
        require(newPercentage <= 10_000);

        uint oldPercentage = avaxSlashPercentage;
        avaxSlashPercentage = newPercentage;

        emit AvaxSlashPercentageChanged(oldPercentage, newPercentage);
    }

    /**
     * @notice Change the protocol maximum AVAX subsidisation amount
     * @param  newMaximumSubsidisationAmount New maximum subsidisation amount in AVAX (18 decimals)
     */
    function setMaximumSubsidisationAmount(
        uint newMaximumSubsidisationAmount
    ) external {
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));

        uint oldMaximumSubsidisationAmount = maximumSubsidisationAmount;
        maximumSubsidisationAmount = newMaximumSubsidisationAmount;

        emit MaximumSubsidisationAmountChanged(
            oldMaximumSubsidisationAmount,
            newMaximumSubsidisationAmount
        );
    }

    /**
     * @notice Update the QI price multiplier used for fee payments in QI
     * @param newQiPriceMultiplier New price multiplier in bps
     */
    function setQiPriceMultiplier(uint newQiPriceMultiplier) external {
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));
        require(newQiPriceMultiplier <= 10_000);

        uint oldQiPriceMultiplier = qiPriceMultiplier;
        qiPriceMultiplier = newQiPriceMultiplier;

        emit QiPriceMultiplierUpdated(
            oldQiPriceMultiplier,
            newQiPriceMultiplier
        );
    }

```

## Impact

Economic Risks:

The current 0.5% fee may become unsuitable due to changes in the economic landscape or tokenomics (e.g., market volatility or changes in QI valuation). Fluctuations in AVAX or QI prices, may necessitate adjustments to the minimum AVAX requirement.

An inability to adjust the fee and minimum AVAX requirement could result in overcharging or undercharging users, affecting their willingness to participate.

Increased Maintenance Overhead:

Redeploying and upgrading the contract solely to change the fee or minimum avax amount increases operational complexity and introduces migration risks for existing users and their deposits.

Impact
Severity: Medium
Likelihood: High (changing economic conditions will likely necessitate fee adjustments in the future).

## Tools Used

Manual Code Review: Identified the hardcoded fee formula.
Economic Analysis: Assessed the potential need for dynamic fee adjustment based on tokenomics and governance.

## Recommendations

Introduce a Configurable Fee and Minimum AVAX Mechanism
Replace the hardcoded divisor with a storage variable that can be updated via a role restricted function:

```solidity
uint public feeDivisor = 201; // Default value for 0.5%

function updateFeeDivisor(uint newDivisor) external  {
    require(newDivisor > 0, "Divisor must be greater than 0");
require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender));
    feeDivisor = newDivisor;
}
```

Analogous reasoning for the minimum avax amount mitigation.

## <a id='M-03'></a>M-03. Denial of Service (DoS) Due to Handling of Chainlink Price Feeds with N/A Decimals

## Summary

`staking::_getPriceInUSD` function assumes that all Chainlink price feeds provide a valid decimals value, which is used in the calculation to scale prices to 18 decimals. However, certain Chainlink price feeds may return N/A for the decimals field (as documented at <https://docs.chain.link/data-feeds/price-feeds/addresses?network=avalanche&page=1>). This can cause the function to revert when interacting with these feeds.

Since `staking::_getPriceInUSD` is a critical component of multiple functions (`staking::convertTokenToQi`, `staking::convertAvaxToToken`, and `staking::stakeWithERC20`), this bug creates a potential Denial of Service (DoS), rendering the contract unusable for tokens linked to these price feeds.

## Vulnerability Details

The following lines in `staking::_getPriceInUSD` rely on the decimals field from Chainlink price feeds with `staking::feedDecimalData` and the decimals field from the metadata of the token linked to the price feed with `staking::tokenDecimalData` :

```solidity
uint256 feedDecimalDelta = uint256(18) - uint256(priceFeeds[token].decimals());
if (feedDecimalDelta > 0) {
    scaledTokenPrice = scaledTokenPrice * (10 ** feedDecimalDelta);
}
uint256 tokenDecimalDelta = (token == AVAX)
            ? 18
            : IERC20Metadata(token).decimals();
        uint256 decimalDelta = uint256(18) - tokenDecimalDelta;
        if (decimalDelta > 0) {
            scaledTokenPrice = scaledTokenPrice * (10 ** decimalDelta);
```

Both variables do the same job which is to scale the token price to the valid amount of decimals. This means that both of these checks are not necessary. One of these checks is sufficient to scale the token price appropriately. The inclusion of the `staking::feedDecimalData` introduces an extra unnecessary assumption on the price feed contracts which is that priceFeeds\[token].decimals() always returns a valid number.

In reality, certain Chainlink feeds (e.g., Aave network emergency count token) return N/A for decimals. This implies that chainlink could deploy price feeds for tokens that return N/A for decimals which leads to unexpected behavior:

If decimals() is N/A, this line will revert due to invalid arithmetic operations.

The following functions will be affected if the price feed address decimals is N/A:

`staking::convertTokenToQi`:

Relies on `staking::_getPriceInUSD` to fetch the prices of both the token and QI.
Failure in `staking::_getPriceInUSD` will cause this function to revert.

`staking::convertAvaxToToken`:

Calls `staking::_getPriceInUSD` for both AVAX and the specified token.
Any failure in `staking::_getPriceInUSD` renders this function inoperable.

`staking::stakeWithERC20`:

Calls `staking::convertAvaxToToken` to calculate `staking::totalRequiredToken` and hosting fees.
Any failure in `staking::convertAvaxToToken`: blocks this function entirely, preventing users from staking with the affected tokens.

## Impact

Denial of Service (DoS):

Tokens linked to price feeds with N/A decimals cannot interact with the contract. This issue affects all staking operations, token conversions, and other dependent functions, effectively rendering them unusable for these tokens.

Protocol Disruption:

Users interacting with the protocol using affected tokens will face failed transactions, leading to poor user experience

## Tools Used

Manual Review

## Recommendations

Remove the following lines of code from `staking::_getPriceInUSD`:

```diff
/**
     * @dev Gets the price of a given token in USD.
     * @param token The address of the token.
     * @return The price of the token amount in USD.
     */
    function _getPriceInUSD(address token) internal view returns (uint256) {
        (, int256 tokenPrice, , uint256 tokenUpdatedAt, ) = priceFeeds[token]
            .latestRoundData();
        require(
            tokenPrice > 0 &&
                block.timestamp - tokenUpdatedAt <= maxPriceAges[token],
            "Invalid or stale token price"
        );
        uint256 scaledTokenPrice = uint256(tokenPrice);
      -  uint256 feedDecimalDelta = uint256(18) -
      -     uint256(priceFeeds[token].decimals());
      - if (feedDecimalDelta > 0) {
      -    scaledTokenPrice = scaledTokenPrice * (10 ** feedDecimalDelta);
      - }
        uint256 tokenDecimalDelta = (token == AVAX)
            ? 18
            : IERC20Metadata(token).decimals();
        uint256 decimalDelta = uint256(18) - tokenDecimalDelta;
        if (decimalDelta > 0) {
            scaledTokenPrice = scaledTokenPrice * (10 ** decimalDelta);
        }
        return scaledTokenPrice;
    }

```

Removing the lines relating to `staking::feedDecimalData` will allow the function to continue working as intended but remove one less assumption of all chainlink price feed contracts including decimals which can potentially cause a DOS if such a price feed contract that returns N/A is linked to an accepted token address. It is safer to stay with the assumption that the token will return a value for decimals() as most ERC20's have this functionality and chainlink price feed contracts do not follow the standard ERC20 methodology

# Low Risk Findings

## <a id='L-01'></a>L-01. Redundant Code in Ignite Contract

## Summary

The `initialize` function of the `Ignite` contract includes a line that sets the `sAVAX` variable:

`sAVAX = IStakedAvax(_sAVAX);`

However, this variable is never used within the rest of the contract. The presence of this unused variable adds unnecessary complexity to the code and can mislead developers or auditors into assuming its relevance.

## Vulnerability Details

- **File**: `Ignite.sol`
- **Code**:

  `sAVAX = IStakedAvax(_sAVAX);`

- **Description**: The `sAVAX` variable is initialized but remains unused throughout the contract. It does not influence any functionality or calculations and serves no practical purpose in the current implementation.

## Impact

- **Potential Maintenance Risks**: Future developers might incorrectly assume the `sAVAX` variable has a functional role, leading to unnecessary updates or modifications.

## Tools Used

**Manual Code Review**: The issue was identified during a manual inspection of the `Ignite` contract.

## Recommendations

**Remove the Unused Variable**:

- Delete the line `sAVAX = IStakedAvax(_sAVAX);` from the `initialize` function if the variable is not required. Also remove the \*\*`import "./IStakedAvax.sol" ` \*\* line as the interface is also never used.

- **Verify Functional Relevance**:

- If the `sAVAX` variable is intended to serve a purpose in future updates, consider adding comments or documentation explaining its planned role.

## <a id='L-02'></a>L-02. Inconsistency in QI Deposit Calculation for registerWithStake

## Summary

The `Ignite::registerWithStake` function enforces that users deposit 10% of the AVAX value subsidized by BENQI in QI tokens. However, this calculation is inconsistent with the examples provided in the Benqi Ignite Whitepaper, which specify fixed QI deposit amounts regardless of the subsidized AVAX amount.

This discrepancy may confuse users and lead to incorrect assumptions about how much QI they need to deposit when registering.

## Vulnerability Details

The calculation for the required QI deposit in `Ignite::registerWithStake` is as follows:

```solidity

uint qiAmount = (uint(avaxPrice) * (2000e18 - msg.value)) /
    uint(qiPrice) /
    10;
```

According to the formula, the user must deposit 10% of the AVAX value subsidized by BENQI (i.e., 2000 AVAX - msg.value).

Whitepaper Examples:
Stake 500 AVAX + 150 AVAX worth of QI.
Stake 1,000 AVAX + 150 AVAX worth of QI.
The whitepaper indicates a fixed QI deposit of 150 AVAX worth of QI, while the implemented formula dynamically adjusts the required QI deposit based on the subsidized AVAX amount. For example:

If a user stakes 500 AVAX, the required QI deposit according to the code would be (1500 \* avaxPrice) / (10 \* qiPrice) (not always equivalent to 150 AVAX worth of QI). See whitepaper example at: <https://docs.benqi.fi/ignite/payg-and-stake#stake>

## Impact

User Confusion:

Users expecting the fixed deposit amount described in the whitepaper may fail to meet the requirements due to the dynamic calculation in the contract.
This could lead to unnecessary transaction reverts or frustration.

## Tools Used

Manual Code Review: Analyzed the implementation of the `Ignite::registerWithStake` function.
Whitepaper Review: Verified expected behavior based on examples in the whitepaper.

## Recommendations

Align Code with Whitepaper:

Update the QI deposit calculation to reflect the fixed deposit amount as described in the whitepaper:

```solidity
uint qiAmount = (150e18 * uint(avaxPrice)) / uint(qiPrice);
```

Update Documentation:

If the dynamic 10% calculation is intentional, update the whitepaper to clarify the formula and provide examples that align with the implementation.

## <a id='L-03'></a>L-03. Handling of Tokens with Decimals Greater Than 18 in registerWithErc20Fee causes DOS

## Summary

`Ignite::registerWithErc20Fee` function calculates the required tokenAmount for the registration fee using a formula that assumes the token's decimals are less than or equal to 18. However, if a token with more than 18 decimals is used, the formula will fail due to an underflow in the division step, breaking the functionality for such tokens. This issue arises because the formula does not account for tokens with decimals greater than 18.

The compatibilities section of the repo states the following:
ERC20
Rebasing, and fee-on-transfer tokens are excluded.
USDC explicitly supported (blacklisted addresses will result in failed transfers)

This suggests that only rebasing and fee-on-transfer tokens are excluded but tokens with 19 decimals are not excluded

## Vulnerability Details

The problematic calculation is:

```solidity
uint tokenAmount = (uint(avaxPrice) * registrationFee) /
    uint(tokenPrice) /
    10 ** (18 - token.decimals());
```

Key Assumptions:

The calculation assumes token.decimals() is always less than or equal to 18.
If token.decimals() is greater than 18, the expression 10 \*\* (18 - token.decimals()) will result in an underflow in Solidity's integer arithmetic.

## Impact

The protocol is incompatible with tokens that have more than 18 decimals, which, while rare, are valid in the ERC-20 standard.

## POC

```javascript
describe("TOKENWITH19DECIMALS", async function () {
  let invalidtoken;
  let invalidtokenpricefeed;

  before(async function () {
    invalidtoken = await deployErc20Token(19);
    invalidtokenpricefeed = await deployPriceFeed(5_00_000_000);

    await ignite.addPaymentToken(
      invalidtoken.address,
      invalidtokenpricefeed.address,
      120
    );
  });

  it("allowsregistering by paying a fee in invalidtoken", async function () {
    const feeInAvax = await ignite.getRegistrationFee(86400 * 14);

    // Mint invalidtoken and allow Ignite to spend it
    await invalidtoken.mint(hre.ethers.utils.parseEther("10000000"));
    await invalidtoken.approve(
      ignite.address,
      hre.ethers.utils.parseEther("10000000")
    );

    const initialUserinvalidtokenBalance = await invalidtoken.balanceOf(
      admin.address
    );
    const initialIgniteinvalidtokenBalance = await invalidtoken.balanceOf(
      ignite.address
    );

    expect(initialUserinvalidtokenBalance).to.equal(
      hre.ethers.utils.parseEther("10000000")
    );
    expect(initialIgniteinvalidtokenBalance).to.equal(0);

    expect(
      await ignite.registerWithErc20Fee(
        invalidtoken.address,
        "NodeID-2",
        blsPoP,
        86400 * 14
      )
    ).to.be.reverted;
  });
});
```

The above test will revert with an arithmetic underflow

## Tools Used

Hardhat

## Recommendations

Exclude tokens with 19 decimals from the list of allowed tokens in the documentation

## <a id='L-04'></a>L-04. Incorrect Qi Reward Emission in Ignite::releaseLockedTokens

## Summary

The `Ignite::releaseLockedTokens` function emits an `Ignite::ValidatorRewarded` event with incorrect reward information, as the function does not actually calculate or distribute any rewards. Instead, rewards for validators are distributed via `ValidatorRewarder::claimRewards` during the execution of `Ignite::redeemAfterExpiry`. This creates confusion and inconsistency, as the event is emitted in the wrong function and with inaccurate data (reward amount 0).

## Vulnerability Details

In `Ignite::releaseLockedTokens` :

```solidity
if (qiRewardEligibilityByNodeId[nodeId]) {
    require(msg.value == 0);

    uint fee = registration.tokenDeposits.tokenAmount / 201;
    registration.tokenDeposits.tokenAmount -= fee;

    qi.safeTransfer(FEE_RECIPIENT, fee);

    emit ValidatorRewarded(nodeId, 0); // Incorrect reward emission
}
```

Incorrect Reward Event:

The `Ignite::ValidatorRewarded` event is emitted in `Ignite::releaseLockedTokens`, but no actual rewards are calculated or distributed here. The reward amount (0) is hardcoded and does not reflect the actual reward distributed to the validator.

Correct Reward Logic in `Ignite::redeemAfterExpiry`:

The reward distribution is performed in `Ignite::redeemAfterExpiry` via `ValidatorRewarder::claimRewards`:

```solidity
qiRedemptionAmount += validatorRewarder.claimRewards(
    registration.validationDuration,
    qiDepositAmount
);
```

This is where the actual reward is calculated and returned, making it the appropriate place to emit the`Ignite::ValidatorRewarded`

## Impact

Critical Misrepresentation of Protocol Actions:

Emitting the `Ignite::ValidatorRewarded` event in the wrong function with incorrect reward amounts (reward amount = 0) creates a critical misrepresentation of protocol actions. Validators may believe they are not being rewarded when they actually are, leading to distrust and potential disputes within the network.

Operational Confusion:

Incorrect event emission may lead to difficulties in debugging or analyzing the reward distribution process, impacting protocol operations.

## Tools Used

Manual Code Review: Identified the misplaced `Ignite::ValidatorRewarded` event and verified the actual reward distribution logic in `Ignite::redeemAfterExpiry`.

## Recommendations

Emit the `Ignite::ValidatorRewarded` event in the `Ignite::redeemAfterExpiry` function after rewards are calculated and distributed by the `ValidatorRewarder::claimRewards` function:

```solidity


if (qiRewardEligibilityByNodeId[nodeId]) {
    uint rewardamount = validatorRewarder.claimRewards(
        registration.validationDuration,
        qiDepositAmount)
emit ValidatorRewarded(nodeId, rewardamount);
qiRedemptionAmount += rewardamount
    );

}
```

Since no rewards are distributed in `Ignite::releaseLockedTokens`, remove the misleading `Ignite::ValidatorRewarded` event from this function.
