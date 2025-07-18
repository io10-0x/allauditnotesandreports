# Audit Analysis Overview

# STEP 1 - FIRST PASS AND ASSUMPTION ANALYSIS (FROM IGNITEBYBENQI AUDIT NOTES)
In your security notes, I detailed a tincho process and this process, I spoke about going through each line of code to find bugs. The tincho strategy is good but not the most efficient. The method I am about to propose is the best way to find bugs in any protocol. In any protocol you are looking at, keep in mind that the main thing you are supposed to find are high/medium bugs. Finding lows are cool but arent going to get you paid like you want to. 

When you read previous reports, every high/medium finding has one thing in common. They have to do with funds being either mishandled, lost, or being unable to receive. They are all centered around user/protocol funds. This means that your aim should be to focus on all the functions where funds are handled in the contract. For example, deposit, withdraw, liquidate, swap or any functions like this that directly interact with user funds. These are the most important functions in any protocol. These should be your main targets in any protocol. Your job is to find a way to make any of these functions do something that they arent supposed to do. How do you do this?

Well every line of code in a function is based on an assumption. Assumptions are what are used to write any codebase. Every function assumes some things for them to work. These assumptions are where the money is in every contest. Your job is to figure out what is assumed in each line you are looking at and then make sure that assumption is correct. This is how you will find the important bugs. By testing these assumptions, it will lead you to check more functions in the contract and each line in these functions have their own assumptions which you will need to test.

So you start from a function and test the assumptions for a particular line and to test the assumption is correct, you will have to look at another function that the line is calling and that function will have its own assumptions you will also have to confirm and this way, you expand from one functions to other important functions in the contract or any other external calls made.

I will show you a short example of how this works but if you look at the registernode function in the staking.sol, you will see how I did this fully.

```solidity
 function registerNode(
        address user,
        string calldata nodeId,
        bytes calldata blsProofOfPossession,
        uint256 index
    ) external onlyRole(ZEEVE_ADMIN_ROLE) whenNotPaused {
        //c a1: the msg.sender is the zeeve admin who apparently is the only one allowed to call this function
        //ca1 comment from dev: The user facing functions are stakeWithAVAX and stakeWithERC20. registerNode is called by Zeeve when the node is provisioned and Ignite can be informed of it.

        require(
            bytes(nodeId).length > 0 && blsProofOfPossession.length == 144, //c outside of scope. we can assume that the nodeId and the BLS key are valid based on this check
            "Invalid node or BLS key"
        );
        require(
            igniteContract.registrationIndicesByNodeId(nodeId) == 0, //c a2: this registrationIndicesByNodeId function returning 0 means that the node is not registered yet
            //qa2: is there a way to make this function return 0 even if the node is already registered ?
            //ca2 at the moment, since _register is called after every registration and the function is updated, this doesnt seem possible
            "Node ID already registered"
        );
```

So this is a snippet from the registernode function in the staking contract which was one of the most important functions in the contract. I started my assumptions with the first comment which says a1 which means assumption 1. so thinking about this function, what is the first thing i see about line of the function that has been assumed. I would normally start with all assumptions about the address user line. One such assumption would be that the user is always going to be a valid address (non zero). I didnt do this in the above function but this is normally would I would do. a1 wuld check that first line and I would go line by line thinking of as many assumptions as possible that were made in that line.

Take a1 for example, the assumption was that the zeeve admin role was the only one allowed to call registernode. This assumption needs to be tested to make sure it is correct because in reality, why is no other user supposed to call registernode if they want to register a node. To validate this assumption, I had to ask the devs if this was intended and when i got the validation, i added the comment from the dev about it as you see. Another assumption on that line is that the onlyRole modifier will actually restrict anyone else from calling this function.
This will lead me to look at that modifier and see what is assumed there and see if i can break that. This is what i mean when i say that assumption analysis will amlost never restrict you to that line, you will eventually have to look at other lines in other functions to validate the assumption made on that line.

Lets look at another assumption . a2 looks at the igniteContract.registrationIndicesByNodeId(nodeId) == 0 line and an assumption made on that line is that if registrationIndicesByNodeId function returns 0, it means that the node is not registered yet. This is a HUGE ASSUMPTION to validate becuase it means you have to go into the ignite contract and look at everywhere registrationIndicesByNodeId is used to make sure that there is no point it is set to 0. You also need to check that there is no possible way to set it to 0 for a registered node. I did this and added a comment line under to say the reason why the assumption was valid after my research. This is what you have to do to be thorough about finding high/medium bugs. If you dont do it like this, you will miss things. You can have as many assumptions a line makes as possible. If you look in the staking contract, you will see some lines where I deduced multiple assumptions. Assumption analysis is a very good strategy for you because you are amazing at formulating questions so for each line, you will be able to find deep assumptions quickly an then you can go about validating/invalidating them.

This is how you will become untouchable. Assumption analysis. Every contract you look at must be FULL of lines with assumptions. If not, you are not doing a good job. You need to be able to come up with as many critical assumptions as possible as quickly as possible but the more you do this, the better you will be at it.

Using this assumption analysis not only helps you think about what the protocol's code is doing, but also what they are not doing. Let me explain what this means from the ignite contract. see
the function below:

```solidity
function registerWithPrevalidatedQiStake(
        address beneficiary,
        string calldata nodeId,
        bytes calldata blsProofOfPossession,
        uint validationDuration,
        uint qiAmount
    )
        external
    //c a1: assumes that the node id and bls proof of possession are valid and unique. this checks out because only the staking contract can call this function
    //and in the registernode function that calls this function, there are checks that make these assumptions valid. same goes for validation duration
    //c a2: assumes that qi amount is non zero or dust value. this checks out as long as avaxstakedamount and hosting fee are large enough so we can safely assume this checks out
    {
        //bug no nonreentrant modifier here so i can reenter this function but need to figure out how to exploit this. it is a known issue but if i can escalate it, it will be a high severity issue
        //c I thought this was an issue but the internal _register function contains the whennotpaused modifier which means no one can register when the contract is paused
        require(
            hasRole(ROLE_REGISTER_WITH_FLEXIBLE_PRICE_CHECK, msg.sender),
            "ROLE_REGISTER_WITH_FLEXIBLE_PRICE_CHECK"
        ); //c a3: assumes hasRole function works as expected and that the role is correctly spelt. this checks out. I have glanced through this in the access contract and
        //the contract is from openzeppelin so it is safe to assume that it works properly

        (, int256 avaxPrice, , uint avaxPriceUpdatedAt, ) = priceFeeds[AVAX]
            .latestRoundData(); //c a4: assumes that the chainlink price feeds dont return any other useful information by only selecting 2 variables from it.
        //this checks out because the other values are roundid, startedat and answeredinround which are not useful in this context
        (, int256 qiPrice, , uint qiPriceUpdatedAt, ) = priceFeeds[address(qi)]
            .latestRoundData();

        require(avaxPrice > 0 && qiPrice > 0);
        require(block.timestamp - avaxPriceUpdatedAt <= maxPriceAges[AVAX]);
        require(
            block.timestamp - qiPriceUpdatedAt <= maxPriceAges[address(qi)]
        );
        //c a5: assumes that the chainklink oracles will always work.
        //bugKNOWN a5: if the chainlink oracles are compromised, the contract will not work as intended. this is a known issue so no need to report

        // 200 AVAX + 1 AVAX fee
        uint expectedQiAmount = (uint(avaxPrice) * 201e18) / uint(qiPrice); //q what is the significance of 201e18 i am guessing users to have a qi stake that is worth at least 201 avax (or 90% of it based on the next line)
        //c a6: assumes that this formula has no overflow or precision loss and returns the qi amount with correct token decimals. this checks out.
        //c a7: assumes that the 201 avax minimum requirement is never going to change
        //bugREPORTED a7 this isnt a correct assumption. if the protocol decide that the 201 avax minimum requirement is too large or too small , they have to upgrade the contract to change it.
        require(qiAmount >= (expectedQiAmount * 9) / 10); //c this is to make sure that the user has a qi stake that is worth at least 90% of 201 avax. the reason for 90% is because the
        //"ROLE_REGISTER_WITH_FLEXIBLE_PRICE_CHECK" will be assigned to the staking contract and in the staking.sol contract, this function is called from the registernode function but the prices are checked in the function
        //and then again in this function so prices could change between when it is calculated in the zeeve contract and when it is calculated in this function so because of this, the protocol dont require qiamount == expectedqiamount.
        //To account for price changes, the protocol requires that the qi amount is at least 90% of the expected qi amount

        qi.safeTransferFrom(msg.sender, address(this), qiAmount);
        //c a8: assumes that the safetransferfrom function correctly transfers the qi amount from the staking contract to this contract. this checks out as we can safely assume that the safetransferfrom function that comes from openzeppelin works as expected
        //c a9: assumes that the qi token is safe in this contract and cannot be drained. NEED TO CHECK FOR ANY BALANCE OF CHECK EXPLOITS I CAN DO

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

I used assumption analysis to look at this function and i got almost to the end of the function at a9 and i was trying to validate a9 and to do this, I searched up qi to see everywhere that qi was mentioned in the contract and when i looked at the releaselockedtokens function, i saw something interesting which was that for users who registerwithstake got slashed but all users who registerwithprevalidatedqistake do not get slashed. There was no logic to slash bad actors who registerwithprevalidatedqistake. So although this had nothing to do with validating a9, it was a high vulnerability as you can imagine. You can read about this finding in codehawks under your submissions as a high submission. This is just another example of the power of assumption analysis.

For assumption analysis to be effective and for you to be able to formulate the best assumptions for each line you look at, you MUST have at least completed a first pass. What I mean by this is that when you first look at a codebase, your plan should be to pick an entry point. An entry point is simply an interesting function in the codebase. Since we are trying to find bugs that directly affect funds, you will most likely look for functions like deposit/withdraw or something that has to do with funds moving back and forth.

Then you will go through the whole function and make comments on everything you understand about what is happening. Go into every external or internal call that function makes and do the same. Make as many comments as you can. This will help you understand the full flow of a path. You can even uncover bugs during this first pass. It is exactly like assumption analysis but you arent going into what the exact assumptions are and the reason is simply, if you dont fully know everything the function does, how can you know what the assumptions are? Once you have gone through the full flow end to end, then you go through the whole flow again but this time as ususal, in each line perform assumption analysis. You will be unstoppable like this.

# STEP 2 - ZOOM OUT ANALYSIS (FROM VELVET V4 AUDIT NOTES)
This section will introduce a step further than assumption analysis. So far, we have a first pass and assumption analysis. Now I will be introducing the zoom out methodology which must come immediately after assumption analysis. The fact that you didnt do this, led to missing this bug which was very easily spotted. I will give some background on the bug first before explaining the zoom out methodology:

The vault computes a global unusedCollateralPercentage for each controller (such as Aave) based on the total collateral and total debt across all tokens. This percentage is calculated using:

TokenBalanceLibrary::getControllersData
```solidity
      uint256 unusedCollateralPercentage;
      if (accountData.totalCollateral == 0) {
        unusedCollateralPercentage = 1e18; // 100% unused if no collateral
      } else {
        unusedCollateralPercentage =
          ((accountData.totalCollateral - accountData.totalDebt) * 1e18) /
          accountData.totalCollateral;
      }
```
This computed percentage is then uniformly applied to the balance of every token marked as borrowable, irrespective of whether each individual token has been borrowed against or how much it has been borrowed.

TokenBalanceLibrary::_getAdjustedTokenBalance
```solidity
  function _getAdjustedTokenBalance(
    address _token,
    address _vault,
    IProtocolConfig _protocolConfig,
    ControllerData[] memory controllersData
  ) public view returns (uint256 tokenBalance) {
    if (_token == address(0) || _vault == address(0))
      revert ErrorLibrary.InvalidAddress(); // Ensures neither the token nor the vault address is zero.
 _protocolConfig.isBorrowableToken(_token));
    if (_protocolConfig.isBorrowableToken(_token)) {
      address controller = _protocolConfig.marketControllers(_token);
controller);
      ControllerData memory controllerData = findControllerData(
        controllersData,
        controller
      );

      uint256 rawBalance = _getTokenBalanceOf(_token, _vault);
      tokenBalance =
        (rawBalance * controllerData.unusedCollateralPercentage) /
        1e18;
    } else {
      tokenBalance = _getTokenBalanceOf(_token, _vault);
    }
  }
  ```
Thus, even if a specific token has not been borrowed or has been borrowed at a different proportion, it still incorrectly receives the same global collateral usage percentage.

You can view the full finding at https://cantina.xyz/code/8cf9c7a0-a7a6-446a-8577-1e2c254eb5a8/findings?severity=medium,high&q=unused+collateral&finding=465

This bug seems very easy to spot so why did you not spot it? The reason is because you didnt apply the zoom out methodology. A lot of times, you tend to zoom into different functions as that is the point of a first pass and assumption analysis. You are going through every function in a user path, assumption analyzing each line and going as deep as you can which is great and what you are supposed to do. 

The problem is that the job isnt done there though and this is where the zoom out methodology becomes important. After doing a first pass and assumption analysis, you need to take the function or the path that you have just analysed and see where else this function is used in the codebase. Just because a path/function seems happy and everything works fine, when you see how that path is applied to other parts of the codebase, it may introduce bugs and this is the highest level of bugs you can find. In this zoom out phase is where you will find a lot of bugs.

Lets look at how zooming out would've helped us spot this bug. I initially looked at the whole flow of TokenBalanceLibrary::getControllersData and assessed the unusedCollateralPercentage calculation with numeric examples and all but the problem was that after doing the first pass, when i saw the unusedCollateralPercentage used in TokenBalanceLibrary::_getAdjustedTokenBalance, i assumed that since the calculation was correct for the percentage, every where it would be used, would also be correct which obviously was the same assumption the devs had and as you can see from this finding, that is definitely not the case.

If you used zoom out analysis, after assessing the unusedCollateralPercentage calculation, you would have them zoomed out and asked, if TokenBalanceLibrary::_getAdjustedTokenBalance happens, where unusedCollateralPercentage is used, will it still work as intended? This is the idea of zooming out. So you look at a function with a first pass or/and assumption analysis, , you then use the brain trigger "if x happens, does this function/variable still work as intended?". Ypou know your brain works very well with brain triggers. so this brain trigger is what fires the zoom out analysis. This is what zooming out means. With this brain trigger, thinking about what x could be will help you discover bugs and x can literally be anything. Dont limit your thinking to what is in the codebase. Also think about x being anything that could happen with any external integrations that the codebase uses. x can even be anything that can happen on the chain being deployed on. This is how far you can zoom out. This is the core of the zoom out methodology. Use this brain trigger to figure out all the things that x could possibly be and see how many more bugs you will find.

To break this down, what I mean is that if i used zoom out analysis, I would have zoomed out of the unusedCollateralPercentage calculation to search for where it was used in the codebase and I wouldve found TokenBalanceLibrary::_getAdjustedTokenBalance. Then i wouldve asked if unusedCollateralPercentage was used correctly here. On asking this question, I wouldve realised that the answer was no because as the finding details, This computed percentage is then uniformly applied to the balance of every token marked as borrowable, irrespective of whether each individual token has been borrowed against or how much it has been borrowed. This is how I wouldve found this bug. 

This zoom out methodology would also have been extremely useful for another finding I missed which will probably turn out to be a rare issue. Lets have a look at it:

When a Portfolio undergoes full liquidation of one of its assets, that asset cannot be removed through the removePortfolioToken() function, because it checks for the token balance:

```solidity
function removePortfolioToken(
    address _token
  ) external onlyAssetManager nonReentrant protocolNotPaused {
    address[] memory currentTokens = _getCurrentTokens();
    //@audit it is a portfolio token but the full liquidated
    if (!_isPortfolioToken(_token, currentTokens))
      revert ErrorLibrary.NotPortfolioToken();
		...
		
    uint256 tokenBalance = _getTokenBalanceOf(_token, _vault);
    //@audit After going through a full liquidation the balance is 0 
    _tokenRemoval(_token, tokenBalance);
  }
  
  function _tokenRemoval(address _token, uint256 _tokenBalance) internal {
		//@audit will revert here
    if (_tokenBalance == 0) revert ErrorLibrary.BalanceOfVaultIsZero();
	  ...
	}
  ```
Having a token with zero balance as a portfolio token prevents any deposits from occurring, temporarily DOSing deposits. The only recovery method is to donate tokens to restore the balance before removing the token.

Users can still withdraw from the portfolio through emergency withdrawal.

With this finding, zoom out methodology would have easily shown you that when a vault is liquidated, the balance becomes 0, you looked at the _tokenRemoval function which in isolation, when you zoomed in with the first pass and the assumption analysis, looked good but if you zoomed out and asked where is this function used ? You would have seen that it can be used to remove tokens that have been liquidated and if the tokens are liquidated, the balance is 0 and _tokenRemoval doesnt allow 0 token removals as the code above shows and that would have been another easy vulnverabilty. So to summarise, just because a function works as intended, doesnt mean it cannot cause a vulnerability. You need to zoom out and question where the function/path/variable is used in the codebase and what the implications of this are.

Once zoom out analysis is used for a function, the next thing is function state change analysis which is a concept I introduced in RAACV2AuditNotes so check that out.

# STEP 3 - FUNCTION STATE CHANGE ANALYSIS (FROM RAACV2AUDITNOTES)

We have covered 2 very important auditing methods that I use in previous audit notes namely assumption analysis and zoom out methodology. Now I will introduce a new analysis method that builds on both of the previous ones that will help you always spot bugs and it is bold to say but with a mix of these 2 analysis methods and enough time, there shouldnt be a single bug out there thay you cannot spot.

Function state change analysis can be broken down into 2 main types. Single function state change analysis and cross function state change analysis.They both work in 3 steps. The first step has to do with identifying an entry point function which we covered in x audit notes  (say function A) and looking at all state variable changes in that function and seeing state variable x for example that gets read or changed in function A. 

Next step is thinking about what time would state variable x need to change to cause unexpected behaviour? By this I mean would someone have to front run another user and change the state to cause unexpected behaviour ? Is there a certain time like during protocol pause or emergency pause or any other time related functionality that the codebase uses where changing this state variable will cause unexpected behaviour? Do I have to change the state after the user has called the function so when they call it again , it would cause unexpected behaviour ? These are all things to consider in this phase 

The final step is asking what value do I need this state variable to be to cause unexpected behaviour ? Does it need to be higher than a certain value ? Lower ? The same ? What value change will cause the most impact?


The main cause of catastrophic bugs in any system are unexpected state variable changes so by following this 3 step analysis on every function once you have context on the codebase and covered assumption analysis and zooming out, there is no bug you shouldn’t be able to find. This is the winning formula


Lets look at a finding to see how function state change analysis would have helped you easily spot this bug.

The RWAVault contract lacks slippage protection mechanisms in redeemNFT.


NFT redemption without maximum share limit: In redeemNFT, users cannot specify the maximum number of shares they're willing to burn for a randomly selected NFT. The function calculates sharesBurned = convertToShares(nftPrice) without allowing users to set an upper bound, potentially forcing them to burn more shares than expected if the NFT's value or vault share price changes unfavorably.

```solidity
    function redeemNFT() external override nonReentrant notBlacklisted(msg.sender) {
        if (address(vrfConsumer) == address(0)) revert InvalidVRFAddress();

        (address adapter, uint256 tokenId) = getNextRandomNFT();

        bytes memory data = abi.encode(tokenId);
        uint256 nftPrice = IVaultAssetAdapter(adapter).getAssetValue(data);
        if (nftPrice == 0) revert InvalidNFT();
        uint256 sharesBurned = convertToShares(nftPrice);
        if (IVaultToken(vaultToken).balanceOf(msg.sender) < sharesBurned) revert InsufficientBalance();

        IVaultToken(vaultToken).burn(msg.sender, sharesBurned);
        IVaultAssetAdapter(adapter).withdraw(data, msg.sender);
        // send the request for the next token to redeem
        vrfConsumer.requestRandomWords();

        emit RedeemNFT(msg.sender, IVaultAssetAdapter(adapter).getAssetToken(), tokenId, sharesBurned);
    }

```

Now there is no parameter that even gives any hint that slippage protection is needed here so assumption analysis during a first pass wont show you this bug as there is no line that handles slippage to analyse so that analysis method wont have helped you spot the bug. Zoom out methodology wont work here either because redeemNFT isnt called anywhere else in the codebase and this is literally an entry point so zooming out will probably take yout back to thinking about where the NFT's come from or logic that actually has to do with getting the NFT's to the adapter where they are redeemed which is great but going down that route wont have helped you find this slippage bug.

Lets see how function state change analysis would have worked following our 3 step process. The first step is identifying our entry point which we already have with the redeemNFT function and then we look into the state variables in this function and pick one of interest and as you can see, there is a convertToShares function which does the following:

```solidity
function convertToShares(uint256 assetAmount) public view override returns (uint256) {
        uint256 supply = IVaultToken(vaultToken).totalSupply();
        uint256 assets = totalAssets();
        if (supply == 0 || assets == 0) return assetAmount;
        // Round up division: (a * b + c - 1) / c
        return (assetAmount * supply + assets - 1) / assets;
    }

```

Immediately you can already see 2 state variables here which are the totalSupply of the vault token returned by the totalSupply which is a state variable and the totalAssets function which does the following:

```solidity
function totalAssets() public view override returns (uint256 totalValue) {
        for (uint256 i = 0; i < adapters.length; i++) {
            totalValue += IVaultAssetAdapter(adapters[i]).totalValue();
        }
        return totalValue;
    }
   
 /// @inheritdoc IVaultAssetAdapter
    function totalValue() public view virtual returns (uint256 _totalValue) {
        uint256 len = _tokenIds.length;
        for (uint256 i = 0; i < len; ++i) {
            _totalValue += priceOracle.getLatestPrice(_tokenIds[i]);
        }
        return _totalValue;
    }

    function getLatestPrice(uint256 _tokenId) external view returns (uint256 _crvUSDPrice, uint256 _lastUpdateTimestamp) {
        uint256 usdPrice = tokenToHousePrice[_tokenId]; 
        if (address(dataFeed) == address(0)) {
            // Fallback: assuming 1:1 price
            _crvUSDPrice = usdPrice;
        } else {
            uint8 decimals = dataFeed.decimals();
            (, int256 answer, , , ) = dataFeed.latestRoundData();

            if (answer <= 0) return _fallBackHousePrice(_tokenId);

            if (circuitBreakerEnabled) {
                uint256 adjustedMinThreshold = _adjustThresholdToDecimals(minPriceThreshold, decimals); 
                uint256 adjustedMaxThreshold = _adjustThresholdToDecimals(maxPriceThreshold, decimals);
                
                if (uint256(answer) < adjustedMinThreshold || uint256(answer) > adjustedMaxThreshold) {
                    return _fallBackHousePrice(_tokenId); 
                }
            }
            _crvUSDPrice = (usdPrice * 10 ** decimals) / uint256(answer);
        }
        _lastUpdateTimestamp = tokenToLastUpdateTimestamp[_tokenId];
    }

```

So we can see that the totalAssets function uses the tokenToHousePrice mapping which is another state variable. So to complete step 1, lets just focus on the tokenToHousePrice state variable.

Next step is thinking about what time would the tokenToHousePrice mapping need to change to cause unexpected behaviour? Well for our context on the redeemNFT function, this state would need to change prior to any user calling redeemNFT because what could happen is that a user has an nft that they want to redeem that costs 1000 and they have 2000 shares in their balance but they only intend to spend 1000 on the nft. if the tokenToHousePrice mapping changed prior to the user calling the redeemNFT function, then the getLatestPrice will return a different value to the totalAssets function which changes its value and can end up with the user spending more/less than what they expected to pay for the NFT. So with this step, we have identified when the state variable will need to change to cause unexpected behaviour. So we already know wwe have a problem.

The final step is asking what value do I need this state variable to be to cause unexpected behaviour ? Does it need to be higher than a certain value ? Lower ? The same ? What value change will cause the most impact? For our example, the biggest impact would be if the tokenToHousePrice mapping increased the token price prior to the user calling redeemNFT causing the user to pay more than they expected. So the resolution to this would be to add a slippage check to make sure that the user is paying what they expect for each NFT and this is where the slippage check would come in.

This is the power of function state change analysis. There is no bug that cannot be caught using one of the 3 analysis methods detailed here. This can only be done after the first 2 analysis becuase without them, you dont have enough context to answer each question that each step requires and the prior analysis steps can help you spot bugs along the way. You cant go wrong with this.
