# 1 Non reentrant modifier bricks Bonding::launch

Summary 

The Bonding::launch function in the Virtual Protocol is permanently unusable due to an incorrectly configured nonReentrant modifier. This modifier is intended to prevent reentrant calls to sensitive functions but is misapplied here in a way that always triggers a revert, even during valid, non-reentrant calls. This prevents any user from launching new bonded tokens through the normal interface, effectively disabling the feature entirely.

Vulnerability Details

Bonding::launch allows users to launch prototype tokens that are initially bonded with $VIRTUAL and are allowed to graduate after the token reaches a pre determined graduation threshold. The key sections of the function to consider are below:

```solidity

         function launch(
        string memory _name,
        string memory _ticker,
        uint8[] memory cores,
        string memory desc,
        string memory img,
        string[4] memory urls,
        uint256 purchaseAmount
    ) public nonReentrant returns (address, address, uint) {
        return launchFor(_name, _ticker, cores, desc, img, urls, purchaseAmount, msg.sender);
    }

    function launchFor(
        string memory _name,
        string memory _ticker,
        uint8[] memory cores,
        string memory desc,
        string memory img,
        string[4] memory urls,
        uint256 purchaseAmount,
        address creator
    ) public nonReentrant returns (address, address, uint) {
  
```

Bonding::launch has a nonRentrant modifier inherited from ReentrancyGuardUpgradeable that does the following:

```solidity
 modifier nonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }

    function _nonReentrantBefore() private {
        ReentrancyGuardStorage storage $ = _getReentrancyGuardStorage();
        // On the first call to nonReentrant, _status will be NOT_ENTERED
        if ($._status == ENTERED) {
            revert ReentrancyGuardReentrantCall();
        }

        // Any calls to nonReentrant after this point will fail
        $._status = ENTERED;
    }
```

As a result, when Bonding::launch is called, it always reverts with the ReentrancyGuardReentrantCall custom error.

Impact
The impact of this bug is a complete Denial of Service (DoS) of the bonding mechanism:

No users can launch new bonded tokens via the primary launch function.

Developers relying on this pathway for token creation are blocked.

This disables a core feature of the protocol — onboarding and experimentation through prototype tokens.

Proof Of Concept

The following test was run in test/bonding.js:

```javascript
 it("non reentrant launchbug", async function () { 
    const { virtualToken, bonding, fRouter, fFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury, deployer } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await expect(bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat", //c nice
        "",
        ["", "", "", ""],
        parseEther("200")
      )).to.be.revertedWithCustomError(bonding, "ReentrancyGuardReentrantCall()");
    // Treasury should have 100 $V
    // We bought $CAT with 1$V   
  });

```

RECOMMENDATION

Remove the nonRentrant modifier from Bonding::launch.




# 2 INVALID FINDING BECAUSE THE ONLY WAY SPAM SELLS STOP OTHER USERS FROM SELLING IS IF ATTACKER KEEPS SUBMITTING SELLS TO FILL UP EACH BLOCK BY PAYING HIGHER GAS PRICES SO THE SELLER CANT GET THEIR TX INTO THE BLOCK BUT THAT WILL BE EXTRMELY EXPENSIVE TO DO AND THE SELLER COULS JUST UP THEIR GAS PRICES AND ESSENTIALLY START A GAS WAR

Token creator can block withdrawals with spam sells

Summary
The Bonding::sell function enables users to sell tokens back into the bonding curve. However, due to the absence of a minimum sell threshold, an attacker (especially the token creator) can perform griefing attacks by spamming the sell function with minimal (1 wei) sell amounts. This can block other users from accessing the function leading to a denial-of-service scenario for legitimate sellers.

Vulnerability Details

Bonding::sell allows users to sell tokens they have purchased from the bonding curve. The issue stems from the fact that there is no minimum sell amount. The only minimum checks occur in the Frouter.sell call which calls:

```solidity
 token.safeTransferFrom(to, pairAddress, amountIn);
```

FERC20::transferFrom contains the following check:

```solidity
function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
        _transfer(sender, recipient, amount);

        _approve(sender, _msgSender(), _allowances[sender][_msgSender()] - amount);

        return true;
    }

      function _transfer(address from, address to, uint256 amount) private {
        require(from != address(0), "ERC20: transfer from the zero address");
        require(to != address(0), "ERC20: transfer to the zero address");
        require(amount > 0, "Transfer amount must be greater than zero");

        if (!isExcludedFromMaxTx[from]) {
            require(amount <= _maxTxAmount, "Exceeds MaxTx");
        }

        _balances[from] = _balances[from] - amount;
        _balances[to] = _balances[to] + amount;

        emit Transfer(from, to, amount);
    }
```
The only check is that the amount is greater than 0. As a result, if a token creator sees that reserveA is getting close to the grad threshold, they could brick the Bonding::sell by spamming the function with 1 wei sells which will block any participant from selling for relatively cheap considering virtual operates on a L2. As a result, no one will be able to sell tokens until the token reaches the grad threshold. This is especially concerning for tokens nearing their graduation threshold. The creator may intentionally block access to selling (to trap user value) just before graduation.

Impact
Denial of Service (DoS): An attacker (especially the token creator) can prevent other users from selling tokens by flooding the Bonding::sell function with tiny transactions.

Economic Griefing: By exploiting cheap L2 gas, an attacker can make it prohibitively expensive for users to exit their positions.

Value Locking: Users may be forced to wait for graduation to claim any value, undermining the intended flexibility of selling tokens prior to graduation.


Proof Of Concept

This test was run in test/bonding.js

```javascript
it("spam sell preventing users from selling", async function () { 
    const { virtualToken, bonding, fRouter, fFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury, deployer } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat", //c nice
        "",
        ["", "", "", ""],
        parseEther("200")
      );
    // Treasury should have 100 $V
    // We bought $CAT with 1$V

    
      //c random person trades 30000 $VIRTUAL for $CAT
      const tokenAddy = await bonding.tokenInfos(0);
      const tokenContract = await ethers.getContractAt("ERC20", tokenAddy);
      const preBalance = await tokenContract.balanceOf(trader.address);
      console.log("preBalance", preBalance);
      
      await virtualToken.mint(trader.address, parseEther("30000"));
      await virtualToken.connect(trader).approve(fRouter.target, parseEther("30000"));
      await bonding.connect(trader).buy(parseEther("30000"), tokenAddy);  

      //c get how much $CAT the trader got
      
      const traderBalance = await tokenContract.balanceOf(trader.address);
      console.log("traderBalance", traderBalance);

      //c get reserves
      const pair = await fFactory.getPair(tokenAddy, virtualToken.target);
      const pairContract = await ethers.getContractAt("FPair", pair);
      const [r1, r2] = await pairContract.getReserves();
      console.log("Reserves", r1, r2);

      //c check if token is still trading
      const info = await bonding.tokenInfo(tokenAddy);
      const trading = info.trading;
      console.log("trading", trading);

      //c check if token is trading on uniswap
      const tradingOnUniswap = info.tradingOnUniswap;
      console.log("tradingOnUniswap", tradingOnUniswap);


      //c at this point, there are only 5 million tokens left until the token can graduate and founder sees that a buy is happening that will take the token over the gradthreshold but trader wants to sell some tokens so founder front runs the sell by spamming 1 wei sells

      //c spam 0 sells

      //c approve tokenContract
      await tokenContract.connect(founder).approve(fRouter.target, parseEther("1"));
      for (let i = 0; i < 50; i++) {
        await bonding.connect(founder).sell(1, tokenAddy);
      }

      //c this will be relatively cheap for the founder as 50 runs will only cost roughly 203811 gwei. 
      
  });

  

```

See the result from the gas report:

```
|  Bonding         ·  sell                    ·     167111  ·     203811  ·     167845  ·           50  ·          -  │
```

Recommendation

Use a require(amount >= MIN_SELL_THRESHOLD) guard clause, where MIN_SELL_THRESHOLD is a reasonable value (e.g., 1e9 wei or higher) to avoid dust-based attacks.


# 3 SUBMITTED Launches stuck in bonding phase due to lack of initial core check

Summary
The vulnerability in the Virtuals platform stems from inconsistent validation of the cores array across different functions in the bonding and agent deployment process. While AgentFactoryV3::initFromBondingCurve enforces a cores.length > 0 requirement, Bonding::launchFor does not, allowing launches without cores. This creates a mismatch in expectations: tokens that reach the gradThreshold in the bonding curve will attempt to graduate to a Uniswap pool but will revert due to the missing cores check, leaving them stuck in the bonding phase.

Additionally, while AgentFactoryV3::proposeAgent checks for non-empty cores, it does not validate whether the provided core IDs are registered in the CoreRegistry. This allows agents to be tokenized with unsupported/invalid core types, violating the platform’s intended design.

Vulnerability Details

Bonding::launchFor allows any type of coin to be launched via the virtuals platform which includes memecoins as can be seen in the "should be able to launch memecoin" test in test/bonding.js. The key aspect of this vulnerability is the cores argument taken in the Bonding::launchFor token. See below:

```solidity
 function launchFor(
        string memory _name,
        string memory _ticker,
        uint8[] memory cores,
        string memory desc,
        string memory img,
        string[4] memory urls,
        uint256 purchaseAmount,
        address creator
    ) public nonReentrant returns (address, address, uint) {
```

Any user can launch a token in the bonding curve and there is no requirement for the cores array to have any inputs as the cores functionality was created mainly for ai agents that are being tokenized to ai agent tokens. As a result, a user who is creating a memecoin can pass an empty array as the cores argument and will be able to launch their coin in the bonding curve without any issues which is expected as cores are not necessary for memecoins.

The issue lies when reserveA reaches the gradThreshold in Bonding::buy which is meant to automatically graduate the token to a uniswap pool.

```solidity
  if (newReserveA <= gradThreshold && tokenInfo[tokenAddress].trading) {
            _openTradingOnUniswap(tokenAddress);
        }
```

Bonding::_openTradingOnUniswap has the following code block:

```solidity
uint256 id = IAgentFactoryV3(agentFactory).initFromBondingCurve(
            string.concat(_token.data._name, " by Virtuals"),
            _token.data.ticker,
            _token.cores,
            _deployParams.tbaSalt,
            _deployParams.tbaImplementation,
            _deployParams.daoVotingPeriod,
            _deployParams.daoThreshold,
            assetBalance,
            _token.creator
        );
```

AgentFactoryV3::initfromBondingCurve has the following check:

```solidity
require(cores.length > 0, "Cores must be provided");
```

This check is not enforced in Bonding::launchFor which allows users to launch tokens freely without core requirements which make sense as not every token launched on virtuals are agent tokens. The enforcement of this check in AgentFactoryV3::initfromBondingCurve means that tokens that should graduate will always revert which is not intended behaviour and will lead to tokens without cores not being able to graduate to uniswap pools and stay stuck in the bonding phase. 

On the other hand, for ai agents that launch using AgentFactoryV3::proposeAgent to register their ai agent on virtuals are also required to enter the respective cores their agent will be implementing. The checks for these agents are the same. See below:

```solidity
function proposeAgent(
        string memory name,
        string memory symbol,
        string memory tokenURI,
        uint8[] memory cores,
        bytes32 tbaSalt,
        address tbaImplementation,
        uint32 daoVotingPeriod,
        uint256 daoThreshold
    ) public whenNotPaused returns (uint256) {
        address sender = _msgSender();
        require(IERC20(assetToken).balanceOf(sender) >= applicationThreshold, "Insufficient asset token");
        require(
            IERC20(assetToken).allowance(sender, address(this)) >= applicationThreshold,
            "Insufficient asset token allowance"
        );
        require(cores.length > 0, "Cores must be provided");

        IERC20(assetToken).safeTransferFrom(sender, address(this), applicationThreshold);

```
The requirement makes sure that the cores array is not empty. However, there is no validation of the contents of the array. Virtuals contains a CoreRegistry contract which keeps tracks of all cores supported by Virtuals as well as their descriptions.

```solidity
abstract contract CoreRegistry is Initializable {
    mapping(uint8 => string) public coreTypes;
    uint8 _nextCoreType;

    event NewCoreType(uint8 coreType, string label);

    function __CoreRegistry_init() internal onlyInitializing {
        coreTypes[_nextCoreType++] = "Cognitive Core";
        coreTypes[_nextCoreType++] = "Voice and Sound Core";
        coreTypes[_nextCoreType++] = "Visual Core";
        coreTypes[_nextCoreType++] = "Domain Expertise Core";
    }

    function _addCoreType(string memory label) internal {
        uint8 coreType = _nextCoreType++;
        coreTypes[coreType] = label;
        emit NewCoreType(coreType, label);
    }
}
```
These core types are never validated against the user's input from AgentFactory::proposeAgent or AgentFactoryV3::initfromBondingCurve. As a result, a user is able to launch and tokenize their ai agent with core values that are unsupported by the virtuals platform which is not intended behaviour. 

Impact

Token Soft-Lock: Any token launched without cores can never graduate to a Uniswap pool once it reaches the gradThreshold. This soft-locks liquidity and prevents further lifecycle progression.

Broken UX: From a user’s perspective, tokens will appear to meet the graduation criteria but will fail silently or revert without clear indication of why they remain in bonding.

Economic Censorship Vector: Malicious actors or even innocent creators launching coreless tokens may create projects that cannot mature, which undermines the legitimacy and intended flow of the protocol.

Protocol Integrity: Graduation logic becomes inconsistent — some tokens graduate successfully while others get permanently stuck based on a hidden internal assumption (presence of cores[]), introducing silent fragility.

Potential Exploitation: AI agents can be tokenized with nonexistent or unsupported core types, undermining the platform’s governance and utility guarantees.

Proof Of Concept

The following test was run in test/bonding.js:

```javascript
it("memecoin launch gets stuck in bonding phase", async function () { //c for testing purposes
    const { virtualToken, bonding, fRouter, fFactory, agentFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury, deployer, poorMan } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [],
        "it is a cat", //c nice
        "",
        ["", "", "", ""],
        parseEther("200")
      );
    // Treasury should have 100 $V
    // We bought $CAT with 1$V

    
      //c random person trades 30000 $VIRTUAL for $CAT
      const tokenAddy = await bonding.tokenInfos(0);
      const tokenContract = await ethers.getContractAt("ERC20", tokenAddy);
      const preBalance = await tokenContract.balanceOf(trader.address);
      console.log("preBalance", preBalance);

      //c poorMan buys 20000 $CAT
      await virtualToken.mint(poorMan.address, parseEther("20000"));
      await virtualToken.connect(poorMan).approve(fRouter.target, parseEther("20000"));
      await bonding.connect(poorMan).buy(parseEther("20000"), tokenAddy);  

      //c trader buys 30000 $CAT
      
      await virtualToken.mint(trader.address, parseEther("30000"));
      await virtualToken.connect(trader).approve(fRouter.target, parseEther("30000"));
      await expect(bonding.connect(trader).buy(parseEther("30000"), tokenAddy)).to.be.reverted;   
  });
```

Recommendation 

Soft-Fail Option: In Bonding::_openTradingOnUniswap, check if the cores.length > 0 before calling AgentFactoryV3::initFromBondingCurve. If not, fallback to a simple Uniswap listing without agent logic (or allow manual override).

Explicit Type Declaration: During Bonding::launchFor, allow token creators to explicitly declare if the token is an agent token. If not, bypass the AgentFactoryV3 logic entirely.

Alternatively, if all tokens are expected to eventually graduate via the agent factory, then Bonding::launchFor should validate that cores.length > 0 at launch time to avoid mid-life failures.


# 4 DO NOT ADD THIS. I THINK THEY ALREADY KNOW ABOUT THIS. SEE COMMENTS IN AGENTTOKEN CONTRACT

BUY/SELL TAXES ARE UNFAIRLY APPLIED TO LIQUIDITY ADDITION/REMOVAL

Summary
The AgentToken contract implements fee-on-transfer logic intended to apply only during buy and sell actions on a Uniswap V2-like pool. However, due to reliance on the Uniswap pair address alone to detect buys and sells, the contract inadvertently applies taxes on liquidity provisioning and removal actions. This occurs because liquidity interactions also involve the pair contract, causing the _taxProcessing function to misclassify them as taxable trades. As a result, users are charged taxes during LP-related operations, which is unintended and may lead to a negative user experience or loss of funds.

Vulnerability Details

AgentToken.sol is a fee-on-transfer token but this fee is only applicable to buys/sells via the uniswap pair. This is evident in AgentToken::_taxProcessing:

```solidity
 function _taxProcessing(
        bool applyTax_,
        address to_,
        address from_,
        uint256 sentAmount_
    ) internal returns (uint256 amountLessTax_) {
        amountLessTax_ = sentAmount_;
        unchecked {
            if (_tokenHasTax && applyTax_ && !_autoSwapInProgress) {
                uint256 tax;

                // on sell
                if (isLiquidityPool(to_) && totalSellTaxBasisPoints() > 0) {
                    if (projectSellTaxBasisPoints > 0) {
                        uint256 projectTax = ((sentAmount_ * projectSellTaxBasisPoints) / BP_DENOM);
                        projectTaxPendingSwap += uint128(projectTax);
                        tax += projectTax;
                    }
                }
                // on buy
                else if (isLiquidityPool(from_) && totalBuyTaxBasisPoints() > 0) {
                    if (projectBuyTaxBasisPoints > 0) {
                        uint256 projectTax = ((sentAmount_ * projectBuyTaxBasisPoints) / BP_DENOM);
                        projectTaxPendingSwap += uint128(projectTax);
                        tax += projectTax;
                    }
                }

                if (tax > 0) {
                    _balances[address(this)] += tax;
                    emit Transfer(from_, address(this), tax);
                    amountLessTax_ -= tax;
                }
            }
        }
        return (amountLessTax_);
    }

```

This function is applied on any transfers of agentToken. The issue stems from the isLiquidityPool function.

```solidity
function isLiquidityPool(address queryAddress_) public view returns (bool) {
        /** @dev We check the uniswapV2Pair address first as this is an immutable variable and therefore does not need
         * to be fetched from storage, saving gas if this address IS the uniswapV2Pool. We also add this address
         * to the enumerated set for ease of reference (for example it is returned in the getter), and it does
         * not add gas to any other calls, that still complete in 0(1) time.
         */
        return (queryAddress_ == uniswapV2Pair || _liquidityPools.contains(queryAddress_));
    }
```

This function checks if the address is a liquidity pool i.e. the uniswap pool contract that contains the agentToken/assetToken pair. If this function returns true, taxes are applied to the transaction that involves the liquidity pool contract. AgentToken::_taxProcessing applies the buy tax when the pool contract is the from address and the sell tax is applied when the pool contract is the to address.

The issue is that the uniswap router also interacts with the pool contract when a user adds/removes liquidity from the pool which is not a buy/sell action which means that users are unfairly charged taxes on actions that are not buys/sells. This is not intended behaviour and breaks the tax logic.

Impact

Unintended taxation of liquidity providers: Users who add or remove liquidity are charged tax even though they are not performing trades.


Proof Of Concept

This test was run in test/bonding.js:

```javascript
it("taxes are unfairly applied to liquidity addition/removal", async function () {
    const { virtualToken, bonding, fRouter, agentFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await virtualToken.mint(trader.address, parseEther("120000"));
    await virtualToken
      .connect(trader)
      .approve(fRouter.target, parseEther("120000"));

    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat",
        "",
        ["", "", "", ""],
        parseEther("200")
      );

    const tokenInfo = await bonding.tokenInfo(await bonding.tokenInfos(0));
    expect(tokenInfo.agentToken).to.equal(
      "0x0000000000000000000000000000000000000000"
    );
    await expect(
      bonding.connect(trader).buy(parseEther("50000"), tokenInfo.token)
    ).to.emit(bonding, "Graduated");

    //c get token address from newpersona event
    const factoryFilter = agentFactory.filters.NewPersona;
    const factoryEvents = await agentFactory.queryFilter(factoryFilter, -1);
    const factoryEvent = factoryEvents[0];
    const { token: tokenAddress } = factoryEvent.args;
    console.log("tokenAddress", tokenAddress);

    const tokenContract = await ethers.getContractAt("ERC20", tokenAddress);

    //c buy some of the new token from the uni pool
    const router =await ethers.getContractAt("IUniswapV2Router02", process.env.UNISWAP_ROUTER);
    //c get timestamp
    const block= await ethers.provider.getBlock("latest");
    const timestamp = block.timestamp;
    console.log("timestamp", timestamp);
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).swapExactTokensForTokensSupportingFeeOnTransferTokens(parseEther("10"), 0, [virtualToken.target, tokenAddress], trader.address, timestamp + 1000);

    const pretokenBalance = await tokenContract.balanceOf(tokenAddress);
    console.log("pretokenBalance", pretokenBalance);

    //c add liquidity
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await tokenContract.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).addLiquidity(tokenAddress, virtualToken.target, parseEther("10"), parseEther("10"), 0, 0, trader.address, timestamp + 1000);

    //c proof that token balance increased which means that taxes were increased post liquidity add which is not the intended behaviour of the tax token

    
    const posttokenBalance = await tokenContract.balanceOf(tokenAddress);
    console.log("posttokenBalance", posttokenBalance);

    assert(posttokenBalance > pretokenBalance);
    
  });
```

Recommendation

To resolve the misclassification of transactions that interact with the liquidity pool, introduce a custom router that acts as the exclusive gateway to all trading operations (buy/sell) involving the AgentToken's Uniswap pair. Here's how the solution would work:

Use a Custom Router to Distinguish Trades from LP Actions
Deploy a custom router contract that wraps around Uniswap’s router and explicitly distinguishes between:

Swaps (buys/sells): When the router executes a swap, it sets a state flag (e.g., isSwapInProgress) on the AgentToken contract to signal that taxes should be applied.

Liquidity provisioning (add/remove): These actions would bypass the swap flag, ensuring no taxes are applied.



# 5 FERC20::UPDATEMAXTX and EXCLUDEFROMMAXTX CAN NEVER BE RESET DUE TO LACK OF LOGIC IN BONDING CONTRACT

Summary
The vulnerability stems from an inflexibility in the FERC20 token contract's maxTx (maximum transaction) settings for tokens launched via the Bonding::launchFor function. While the FERC20 contract includes functions (updateMaxTx and excludeFromMaxTx) to modify transaction limits, they are restricted by an onlyOwner modifier—where the owner is the Bonding contract itself. However, the Bonding contract does not expose any functionality to call these FERC20 functions, rendering them permanently inaccessible. Consequently:

The maxTx value remains fixed at deployment for all tokens in the bonding curve.

No adjustments can be made to transaction limits, even if market conditions or user needs change.

This creates a rigidity in the system that may hinder usability and adaptability during the bonding phase.

Vulnerability Details

Bonding::launchFor enables users to launch tokens by creating an FERC20 token that can be freely traded with an assetToken which is usually $VIRTUAL. 

```solidity
   function launchFor(
        string memory _name,
        string memory _ticker,
        uint8[] memory cores,
        string memory desc,
        string memory img,
        string[4] memory urls,
        uint256 purchaseAmount,
        address creator
    ) public nonReentrant returns (address, address, uint) {
        require(purchaseAmount > fee, "Purchase amount must be greater than fee");
        address assetToken = router.assetToken();
        require(IERC20(assetToken).balanceOf(msg.sender) >= purchaseAmount, "Insufficient amount");
        uint256 initialPurchase = (purchaseAmount - fee);
        IERC20(assetToken).safeTransferFrom(msg.sender, _feeTo, fee);
        IERC20(assetToken).safeTransferFrom(msg.sender, address(this), initialPurchase);

        FERC20 token = new FERC20(string.concat("fun ", _name), _ticker, initialSupply, maxTx);
        uint256 supply = token.totalSupply();
```

Since the token is launched by the Bonding contract, it means the bonding contract has ownership of the deployed token. FERC20::updatemaxTx and FERC20::excludefommaxtx are both functions that can be used to adjust the max tx for any tokens currently in the bonding curve. These functions currently have the onlyOwner modifier which means they are only callable by the bonding contract.

```solidity

    function updateMaxTx(uint256 _maxTx) public onlyOwner {
        _updateMaxTx(_maxTx);
    }

    function excludeFromMaxTx(address user) public onlyOwner {
        require(user != address(0), "ERC20: Exclude Max Tx from the zero address");

        isExcludedFromMaxTx[user] = true;
    }
```

However, there is no logic in the bonding contract that allows these functions to be callable. As a result, these functions can never be called and the maxTx value remains fixed for any token launhced remains the same from deployment until graduation which can lead to issues regarding flexibility for users trading the tokens on the bonding curve.

Impact

Inflexible Trading Conditions: Tokens stuck with a fixed maxTx limit may become impractical for trading if the limit is too low (restricting liquidity) or too high (risking market manipulation).

Reduced Utility for Legitimate Users: Large investors or market makers may be unable to execute meaningful trades if maxTx is too restrictive. Projects cannot adjust limits in response to token volatility or adoption phases.

Centralization Risk: Since the Bonding contract owns the tokens but cannot modify maxTx, even the protocol itself cannot respond to emergent needs, creating a single point of failure.

Potential Bottlenecks at Graduation: If tokens graduate to Uniswap with poorly configured maxTx values, it could lead to fragmented liquidity or inefficient price discovery.


Proof Of Code

This test was tun in test/bonding.js:

```javascript
 it("maxTx has no way of being implemented", async function () {//c for testing purposes
    const { virtualToken, bonding, fRouter, agentFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await virtualToken.mint(trader.address, parseEther("120000"));
    await virtualToken
      .connect(trader)
      .approve(fRouter.target, parseEther("120000"));

    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat",
        "",
        ["", "", "", ""],
        parseEther("200")
      );

    const tokenInfo = await bonding.tokenInfo(await bonding.tokenInfos(0));
    expect(tokenInfo.agentToken).to.equal(
      "0x0000000000000000000000000000000000000000"
    );
    const tokenAddy = await bonding.tokenInfos(0);
    const tokenContract = await ethers.getContractAt("FERC20", tokenAddy);
   
    //c attempt to updatemaxtx 
    await tokenContract.updateMaxTx(parseEther("100"));
  });
```

Recommendations

Add functions in the Bonding contract to call FERC20::updateMaxTx and excludeFromMaxTx, such as:

```solidity
function setTokenMaxTx(address token, uint256 newMax) external onlyOwner {  
    FERC20(token).updateMaxTx(newMax);  
}  
```
Restrict access to trusted roles (e.g., governance, multisig) to prevent abuse.


# 6  SUBMITTED LACK OF ACCESS CONTROL IN AGENTNFTV2::ADDVALIDATOR ALLOWS ATTACKER TO MANIPULATE VALIDATOR SCORES AND REWARDS

Summary
The AgentNftV2::addValidator function can be called by anyone, allowing arbitrary addresses to be added as validators for a given virtualId. When this happens, the _initValidatorScore function gives each added validator a base score equal to the current number of proposals (totalProposals). This score represents the number of proposals the validator has not participated in, and it is used to calculate rewards via AgentRewardV2::_distributeValidatorRewards.

Since this can be triggered for any address without their consent and without requiring any staked tokens or veTokens, malicious users can mass-register addresses (even random or dormant ones) to frontload them with zero participation scores, long before those users ever decide to interact with the system.

Vulnerability Details

AgentNftV2::addValidator is a function used to add validators to an agent via its virtual id. 

```solidity
 function addValidator(uint256 virtualId, address validator) public {
        if (isValidator(virtualId, validator)) {
            return;
        }
        _addValidator(virtualId, validator);
        _initValidatorScore(virtualId, validator);
    }

```

Validators are added when an address has been delegated voting power i.e. when a user stakes lp tokens via AgentVeToken::stake.

```solidity
function stake(uint256 amount, address receiver, address delegatee) public {
        require(canStake || totalSupply() == 0, "Staking is disabled for private agent"); // Either public or first staker

        address sender = _msgSender();
        require(amount > 0, "Cannot stake 0");
        require(IERC20(assetToken).balanceOf(sender) >= amount, "Insufficient asset token balance");
        require(IERC20(assetToken).allowance(sender, address(this)) >= amount, "Insufficient asset token allowance");

        IAgentNft registry = IAgentNft(agentNft);
        uint256 virtualId = registry.stakingTokenToVirtualId(address(this));

        require(!registry.isBlacklisted(virtualId), "Agent Blacklisted");

        if (totalSupply() == 0) {
            initialLock = amount;
        }

        registry.addValidator(virtualId, delegatee);

        IERC20(assetToken).safeTransferFrom(sender, address(this), amount);
        _mint(receiver, amount);
        _delegate(receiver, delegatee);
        _balanceCheckpoints[receiver].push(clock(), SafeCast.toUint208(balanceOf(receiver)));
    }
```

For an address to be counted as a validator, the address needs to have delegated voting power. In AgentNftV2::addValidator, it calls _initVadlidatorScore:

```solidity
 function _initValidatorScore(uint256 virtualId, address validator) internal {
        _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId); 
    }
```

This gives every new validator a base score which is equal to the total number of proposals that the agentDAO has. In AgentNftV2, _getMaxScore is set to the totalProposals function:

```solidity
function totalProposals(uint256 virtualId) public view returns (uint256) {
        VirtualInfo memory info = virtualInfos[virtualId];
        IAgentDAO dao = IAgentDAO(info.dao);
        return dao.proposalCount(); 
    }

```

The reason for this can be seen in AgentRewardV2::_distributeValidatorRewards:

```solidity
function _distributeValidatorRewards(
        uint256 amount,
        uint256 virtualId,
        uint48 rewardId,
        uint256 totalStaked
    ) private {
        IAgentNft nft = IAgentNft(agentNft);
        // Calculate weighted validator shares
        uint256 validatorCount = nft.validatorCount(virtualId);
        uint256 totalProposals = nft.totalProposals(virtualId);

        for (uint256 i = 0; i < validatorCount; i++) {
            address validator = nft.validatorAt(virtualId, i);

            // Get validator revenue by votes weightage
            address stakingAddress = getVirtualTokenAddress(nft, virtualId);
            uint256 votes = IERC5805(stakingAddress).getVotes(validator);
            uint256 validatorRewards = (amount * votes) / totalStaked;

            // Calc validator reward based on participation rate
            uint256 participationReward = totalProposals == 0
                ? 0
                : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
            _validatorRewards[validator][rewardId] = participationReward;

            validatorPoolRewards += validatorRewards - participationReward;
        }
    }

```

The idea is that a validator is only rewarded based on their participation in governance. If a validator does not participate by voting in proposals, their score decreases. So if they have voted on 0 proposals, their participation reward will be 0 and so on.

AgentNftV2::addvalidator is callable by anyone so an attacker could add almost all existing addresses on base as validators which would give all the addresses a base score of whatever the number of proposals are at the time they call the function. Since these addresses will be unaware that they are registered validators,  their score will continue to decrease as the number of proposals in the agentDAO increase. As a result, when the user in control of the address decides to stake lp tokens and participate in an agent token, the rewards they receive from AgentRewardV2 will be reduced by the number of proposals that have taken place up to that point. The issue is further compounded by the fact that AgentNftV2::addvalidator does not require the caller to have any amount of agentveTokens which makes it by callable by anyone.


Impact

Reduced Validator Rewards: Users who later decide to legitimately stake LP tokens and participate in governance will start with a penalized score, reducing the rewards they receive from AgentRewardV2, regardless of future participation.

Involuntary Validator Assignment: Addresses can be added as validators without their knowledge or consent, which effectively punishes future participation through no fault of the address holder.

Griefing Attack Vector: An attacker can grief an entire user base or pre-funded wallets by calling addValidator across a wide list of addresses, permanently diluting their reward potential and discouraging future engagement.

No Cost to Exploit: The attack is cheap and scalable. Since addValidator does not require staking or ownership of veTokens, it can be performed by any external address with no capital at risk.

Proof Of Code

```javascript
 it("validator scores can be manipulated", async function () {//c for testing purposes
    const { virtualToken, bonding, fRouter, agentFactory, agentNft} = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury, poorMan } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await virtualToken.mint(trader.address, parseEther("120000"));
    await virtualToken
      .connect(trader)
      .approve(fRouter.target, parseEther("120000"));

    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat",
        "",
        ["", "", "", ""],
        parseEther("200")
      );

    const tokenInfo = await bonding.tokenInfo(await bonding.tokenInfos(0));
    expect(tokenInfo.agentToken).to.equal(
      "0x0000000000000000000000000000000000000000"
    );
    await expect(
      bonding.connect(trader).buy(parseEther("50000"), tokenInfo.token)
    ).to.emit(bonding, "Graduated");

    //c get token address from newpersona event
    const factoryFilter = agentFactory.filters.NewPersona;
    const factoryEvents = await agentFactory.queryFilter(factoryFilter, -1);
    const factoryEvent = factoryEvents[0];
    const { dao: daoAddress } = factoryEvent.args;
    console.log("daoAddress", daoAddress);
    const { token: tokenAddress } = factoryEvent.args;
    console.log("tokenAddress", tokenAddress);
    const { veToken: veTokenAddress } = factoryEvent.args;
    console.log("veTokenAddress", veTokenAddress);
    const { lp: lpAddress } = factoryEvent.args;
    console.log("lpAddress", lpAddress);

    const block= await ethers.provider.getBlock("latest");
    const timestamp = block.timestamp;
    console.log("timestamp", timestamp);

    const daoContract = await ethers.getContractAt("AgentDAO", daoAddress);
    const veTokenContract = await ethers.getContractAt("AgentVeToken", veTokenAddress);
    const router =await ethers.getContractAt("IUniswapV2Router02", process.env.UNISWAP_ROUTER);
    const tokenContract = await ethers.getContractAt("AgentToken", tokenAddress);
    const lpContract = await ethers.getContractAt("ERC20", lpAddress);
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).swapExactTokensForTokensSupportingFeeOnTransferTokens(parseEther("10"), 0, [virtualToken.target, tokenAddress], trader.address, timestamp + 1000);

    
    //c add liquidity
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await tokenContract.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).addLiquidity(tokenAddress, virtualToken.target, parseEther("10"), parseEther("10"), 0, 0, trader.address, timestamp + 1000);


//c random person adds trader as validator
await agentNft.connect(founder).addValidator(1, trader.address);

 //c get score of trader prior to proposal voting
 const pretraderScore = await agentNft.validatorScore(1, trader.address);
 console.log("pretraderScore", pretraderScore);
    //c founder makes 10 proposals
    const founderBal = await veTokenContract.balanceOf(founder.address);
    console.log("founderBal", founderBal);

    await veTokenContract.connect(founder).delegate(founder.address);

  for (let i = 0; i < 10; i++) {
    const targets = [virtualToken.target];
    const values = [0];
    const calldatas = ["0x3351733f0000000000000000000000000b3e328455c4059eeb9e3f84b5543f74e24e7e1b0000000000000000000000000b3e328455c4059eeb9e3f84b5543f74e24e7e1b0000000000000000000000000000000000000000000000000de0b6b3a76400000000000000000000000000000000000000000000000000001bc16d674ec8000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"]
    const description = `Difference${i}`
    await daoContract.connect(founder).propose(targets, values, calldatas, description); 
  }
    let proposalId;
    let proposals;

    //c trader stakes lp tokens
 const traderBal = await lpContract.balanceOf(trader.address);
 console.log("traderBal", traderBal);
 await lpContract.connect(trader).approve(veTokenAddress, traderBal);
 await veTokenContract.connect(trader).stake(traderBal, trader.address, trader.address);


 //c if voting ends after this point and trader doesnt know he is a validator, he never votes on these proposals. Lets see the difference in their score if they did vote

  //c trader votes on all proposals
  for (let i = 0; i < 10; i++) {
  proposals = await daoContract.connect(trader).proposalDetailsAt(i);
  proposalId = proposals[0];
    await daoContract.connect(trader).castVote(proposalId, 1);
  }

 //c get score of trader after proposal voting
 const posttraderScore = await agentNft.validatorScore(1, trader.address);
 console.log("posttraderScore", posttraderScore);

 assert(posttraderScore > pretraderScore);

 //c so the attacker has managed to reduce the score of the founder by 10 and this will affect the validators rewards as we will see in AgentRewardV2::distributeValidatorRewards
 
  });
  
```

Recommendation

Restrict addValidator access: Only allow calls from trusted internal contracts, such as the staking mechanism that properly checks for voting delegation and LP staking (AgentVeToken::stake).

Alternatively, require the caller to hold or be staking veTokens for the virtualId.

Prevent validator registration without intent: Introduce explicit user consent before assigning validator status. For instance, a user might have to opt in to being a validator via signature or transaction.

# 7  NOT WORTH SUBMITTING AS EVERYONE WILL REPORT THIS
AGENTFACTORY::EXECUTEAPPLICATION IS CALLABLE BY ANYONE WHICH BYPASSES THE VIRTUALGENESISDAO PROCESS COMPLETELY. 

Summary 
The AgentFactory::executeApplication function is missing an access control check, allowing any external address to call it directly. According to the intended virtual genesis process (as documented in the Code4rena audit of Virtuals Protocol), executeApplication should only be invoked as part of an approved governance proposal via VirtualGenesisDAO. However, the current implementation in both AgentFactoryV2 and AgentFactoryV3 does not enforce this, enabling unauthorized agent creation.

This directly bypasses the proposal-voting-execution flow, undermining the governance gate that is meant to approve new agent creation and minting.

Vulnerability Details

According to the audit information at https://code4rena.com/audits/2025-04-virtuals-protocol, this is the virtual genesis process:

Submit a new application at AgentFactory
a. It will transfer VIRTUAL to AgentFactory
Propose at VirtualGenesisDAO (action = VirtualFactory.executeApplication )
Start voting at VirtualGenesisDAO
Execute proposal at VirtualGenesisDAO , it will do following:
a. Clone AgentToken
b. Clone AgentDAO
c. Mint AgentNft
d. Stake VIRTUAL -> $`PERSONA (depending on the symbol sent to application)
e. Create TBA with AgentNft

The current implementation in all agent factories (AgentFactoryV2, AgentFactoryV3), the executeApplication function has no restrictions on the caller which means anyone can create an agentToken via AgentFactory::executeApplication without having to go through the proposal route in the VirtualGenesisDAO contract. This directly contradicts the intended implementation from the documentation and is a high risk issue that should be addressed immediately.


Impact

Unauthorized agent creation: Malicious actors can create arbitrary agents without community approval.

Supply inflation risk: AgentTokens and AgentNFTs can be cloned and minted without constraint.

Proof Of Code

```javascript
 it("agentFactory::executeApplication is callable by anyone", async function () {
    const { virtualToken, bonding, fRouter, agentFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury } = await getAccounts();

    //c mint some virtual token to founder
    await virtualToken.mint(founder.address, parseEther("1000000"));

    //c approval to factory
    await virtualToken.connect(founder).approve(agentFactory.target, parseEther("1000000"));

    //c propose an agent
     
     await agentFactory.connect(founder).proposeAgent("test", "TEST", "test", [0, 1, 2], "0xa7647ac9429fdce477ebd9a95510385b756c757c26149e740abbab0ad1be2f16", process.env.TBA_IMPLEMENTATION, 600, 1000000000000000000000n);

     const filter = agentFactory.filters.NewApplication; //c this is an event filter. when the proposeAgent function is called, the NewApplication event is emitted. so this filter is used to get the event. its an ethers thing.
     const events = await agentFactory.queryFilter(filter, -1);
     const event = events[0];
     const { id } = event.args;


    //c execute the application
    await agentFactory.connect(founder).executeApplication(id, true);

    
  });

```

Recommendation 

Apply proper access controls to AgentFactory::executeApplication


# 8 SUBMITTED 
INCORRECT ELO CALCULATOR INPUTS LEAD TO INCORRECT MATURITY CALCULATIONS

Summary

The vulnerability exists in the ELO rating calculation system used to determine agent maturity. The issue stems from incorrect parameter ordering in the EloCalculator::battleElo function when calling Elo::ratingChange. The function passes eloB as ratingA and eloA as ratingB, but then incorrectly applies the rating changes based on the returned negative flag. This causes the ELO rating changes to be applied in the wrong direction, leading to incorrect maturity calculations that increase more than intended for winning players.

Vulnerability Details

AgentDAO::_castVote is overriden from Openzeppellin GovernorUpgradeable contract as follows:

```solidity
 function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal override returns (uint256) {
        bool votedPreviously = hasVoted(proposalId, account);

        uint256 weight = super._castVote(proposalId, account, support, reason, params);

        if (!votedPreviously && hasVoted(proposalId, account)) {
            ++_totalScore;
            _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
            if (params.length > 0 && support == 1) {
                _updateMaturity(account, proposalId, weight, params);
            }
        }

        if (support == 1) {
            _tryAutoExecute(proposalId);
        }

        return weight;
    }
```

If a user decides to vote using GovernorUpgradeable::castVoteWithReasonAndParams, they are allowed to pass in a bytes array containing votes that will be used in the _updateMaturity call. The _updateMaturity function calls AgentDAO::_calMaturity which is the function responsible for calculating the proposal maturity. The function makes a call to EloCalculator::battleElo which does the following:

```solidity
   // Get winner elo rating
    function battleElo(uint256 currentRating, uint8[] memory battles) public view returns (uint256) {
        uint256 eloA = 1000;
        uint256 eloB = 1000;
        for (uint256 i = 0; i < battles.length; i++) {
            uint256 result = mapBattleResultToGameResult(battles[i]);
            (uint256 change, bool negative) = Elo.ratingChange(eloB, eloA, result, k);
            change = _roundUp(change, 100);
            if (negative) {
                eloA -= change;
                eloB += change;
            } else {
                eloA += change;
                eloB -= change;
            }
        }
        return currentRating + eloA - 1000;
    }

```

The params sent by the user calling GovernorUpgradeable::castVoteWithReasonAndParams is decoded into a votes array and that array is passed into EloCalculator::battleElo as the battles array. EloCalculator::mapBattleResultToGameResult is used to map the value sent into the battles array to the expected values for the battle that occurs in Elo::ratingChange. See below:

```solidity
 function mapBattleResultToGameResult(uint8 result) internal pure returns (uint256) {
        if (result == 1) {
            return 100;
        } else if (result == 2) {
            return 50;
        } else if (result == 3) {
            return 50;
        }
        return 0;
    }
```

The idea is that if the user passes 1 as an element in the array, 1 maps to 100 which represents a win against the opponent. 50 means a draw and 0 means a loss as we will see in Elo::ratingChange. 

```solidity

    /// @notice Calculates the change in ELO rating, after a given outcome.
    /// @param ratingA the ELO rating of the player A
    /// @param ratingB the ELO rating of the player B
    /// @param score the score of the player A, scaled by 100. 100 = win, 50 = draw, 0 = loss
    /// @param kFactor the k-factor or development multiplier used to calculate the change in ELO rating. 20 is the typical value
    /// @return change the change in ELO rating of player A, with 2 decimals of precision. 1501 = 15.01 ELO change
    /// @return negative the directional change of player A's ELO. Opposite sign for player B
 function ratingChange(
        uint256 ratingA,
        uint256 ratingB,
        uint256 score,
        uint256 kFactor
    ) internal pure returns (uint256 change, bool negative) {
        uint256 _kFactor; // scaled up `kFactor` by 100
        bool _negative = ratingB < ratingA;
        uint256 ratingDiff; // absolute value difference between `ratingA` and `ratingB`

        unchecked {
            // scale up the inputs by a factor of 100
            // since our elo math is scaled up by 100 (to avoid low precision integer division)
            _kFactor = kFactor * 10_000;
            ratingDiff = _negative ? ratingA - ratingB : ratingB - ratingA;
        }

        // checks against overflow/underflow, discovered via fuzzing
        // large rating diffs leads to 10^ratingDiff being too large to fit in a uint256
        require(ratingDiff < 1126, "Rating difference too large");
        // large rating diffs when applying the scale factor leads to underflow (800 - ratingDiff)
        if (_negative) require(ratingDiff < 800, "Rating difference too large");

        // ----------------------------------------------------------------------
        // Below, we'll be running simplified versions of the following formulas:
        // expected score = 1 / (1 + 10 ^ (ratingDiff / 400))
        // elo change = kFactor * (score - expectedScore)

        uint256 n; // numerator of the power, with scaling, (numerator of `ratingDiff / 400`)
        uint256 _powered; // the value of 10 ^ numerator
        uint256 powered; // the value of 16th root of 10 ^ numerator (fully resolved 10 ^ (ratingDiff / 400))
        uint256 kExpectedScore; // the expected score with K factor distributed
        uint256 kScore; // the actual score with K factor distributed

        unchecked {
            // apply offset of 800 to scale the result by 100
            n = _negative ? 800 - ratingDiff : 800 + ratingDiff;

            // (x / 400) is the same as ((x / 25) / 16))
            _powered = fp.rpow(10, n / 25, 1); // divide by 25 to avoid reach uint256 max
            powered = sixteenthRoot(_powered); // x ^ (1 / 16) is the same as 16th root of x

            // given `change = kFactor * (score - expectedScore)` we can distribute kFactor to both terms
            kExpectedScore = _kFactor / (100 + powered); // both numerator and denominator scaled up by 100
            kScore = kFactor * score; // input score is already scaled up by 100

            // determines the sign of the ELO change
            negative = kScore < kExpectedScore;
            change = negative ? kExpectedScore - kScore : kScore - kExpectedScore;
        }

```

From a high level, lets discuss what this function is doing. The idea is that we have 2 players. player A and player B. They both start with the same base score of 1000 from EloCalculator::battleElo. These scores serve as a base rating for the players. This function is meant to calculate the change in ratings between the 2 players based on the score of the players of a game. The function will always return the change in rating for player A as the natspec says. So the function returns a negative boolean and a change value. The negative boolean being true indicates that player A's rating should reduce and player B's rating should increase. The change value is the amount of rating change. So the formula that calculates the change is:

    ```
    expected score = 1 / (1 + 10 ^ (ratingDiff / 400))
    elo change = kFactor * (score - expectedScore)
    ```

    So the expected score is supposed to calculate an expected score for player A based on the rating difference between the 2 players. The expected score is then compared to the actual score that player A got in the game based on the score passed in the arguments of the ratingChange function. So there is the following line in the function:

    ```solidity
     negative = kScore < kExpectedScore;
    ```

     So if the actual score player A got is less than the expected score, then the negative boolean is true meaning that player A's rating should reduce and the rating change is to be applied to player A. Now the vulnerability is that in EloCalculator::battleElo, when this rating change function is called, this is what is called:
     
     ```solidity
     (uint256 change, bool negative) = Elo.ratingChange(eloB, eloA, result, k);
     ```

     So the ratingA which is supposed to be player A's rating, eloB is passed instead, which means that the rating change is calculated based on player B's rating This isnt really a problem though until you look at the rest of how EloCalculator::battleElo. See below:

      ```solidity
      if (negative) {
                eloA -= change;
                eloB += change;
            } else {
                eloA += change;
                eloB -= change;
            }
        }
        return currentRating + eloA - 1000;

      ```

        It is saying that if the negative boolean is true, then player A's rating should reduce and player B's rating should increase. This is wrong because as I said above, the rating change is based on player B's rating. So if the negative boolean is true, then player B's rating should reduce and player A's rating should increase but that is not what happens which is where the vulnerability is. 

      The effect of this vulnerability is that is incorrect maturity calculations based on what the user passes in as the battles array. The example I will display in my POC will show the following. Say a user passes all 1's in the battles array, this signifies that player A wins everytime. If player A wins against player B in every round, the way the ELO is designed to work is that the gain in rating with every win should diminish. Every time you call battleElo against the same opponent(s), the ratingDiff grows, the powered = 10^(ratingDiff/400) grows, so expectedScore goes up, and change = K×(score−expectedScore) goes down. That’s why consecutive wins yield smaller and smaller increases. With this incorrect input validation, the opposite situation occurs where the elo change increases with each win for the same player which leads to an incorrect elo change value and as a result, an incorrect maturity value returned to the AgentDAO::_updateMaturity function.

      The issue is further compounded as the incorrect maturity value is aggregated across every user who votes with params and AgentDAO::getMaturity returns this skewed maturity value and is used for key calculations like in ServiceNft::mint and ServiceNft::updateImpact which are key functions used to register improvements on ai agents.

      Impact

      Incorrect Maturity Calculations: The vulnerability leads to incorrect maturity calculations for agents, as the ELO rating changes are applied in the wrong direction. This affects the core functionality of the protocol's maturity scoring system.

      Unfair Advantage: The bug causes ELO ratings to increase more than intended for winning players, as the rating changes are applied incorrectly. This creates an unfair advantage for certain agents in the system.

      Protocol Integrity: The maturity calculation is a critical component of the protocol's governance and agent evaluation system. Incorrect calculations undermine the protocol's ability to accurately assess agent performance and maturity.

      Economic Impact: Since maturity scores affect rewards and governance power, incorrect calculations could lead to misallocation of rewards and voting power in the system.


      Proof Of Code

      ```javascript
      
  it("maturity is calculated incorrectly", async function () {
    //c deploy eloCalculator
    const { founder, trader, treasury } = await getAccounts();
    const eloCalc = await upgrades.deployProxy(
      await ethers.getContractFactory("EloCalculator"),
      [founder.address]
    );

    let battles = [1];
    let diffBug;
    let diffBugArray = [];
    let diff;
    let diffArray = [];
    let oldRating = 0n;
    let oldChange = 0n;
    const currentRating = 100n; //c matches default maturity in AgentDAO::_updateMaturity
    for (let i = 0; i < 10; i++) {
      const newRating = await eloCalc.battleElo(currentRating, battles);
      console.log("newRating", newRating.toString());
      
      if (oldRating !== 0n) {
        diffBug = newRating - oldRating;
        diffBugArray.push(diffBug.toString());
      }
   
      //c test ratingChange
      const change = await eloCalc.ratingChangeTest(currentRating, battles);
      console.log("change", change.toString());

      if (oldChange !== 0n) {
        diff = change - oldChange;
        diffArray.push(diff.toString());
      }
      
      oldRating = newRating;
      oldChange = change;
      battles.push(1);

      console.log("diffBugArray", diffBugArray);
      console.log("diffArray", diffArray);
    }
  });

      ```


For this test, i created another function in EloCalculator to show the discrepancy between the expected elo rating change per win and the actual rating change per win. Notice how in diffBugArray, in each iteration, the difference in rating changes keep on increasing which is incorrect as the details have explained and the diffArray show the correct calculations where the rating change accurately reduces per win for the same player against the same opponent.

Recommendation

Fix the parameter ordering in EloCalculator::battleElo


# 9 SUBMITTED
 MATURITY WILL INCORRECTLY ALWAYS BE 100 FOR ALL SERVICENFT::MINT PROPOSALS

Summary

The maturity calculation for ServiceNFT mint proposals is fundamentally broken due to a timing issue in the AgentDAO contract. When users vote on ServiceNFT mint proposals, the maturity calculation always returns 100 because it attempts to read the core service before the proposal has been executed. This occurs because the core service mapping is only updated when ServiceNFT::mint is executed, but the maturity calculation happens during voting, before execution. As a result, all ServiceNFT mint proposals will have a maturity of 100 regardless of the voting parameters provided.

Vulnerability Details

From the docs at https://code4rena.com/audits/2025-04-virtuals-protocol, this is the flow for submitting a contribution proposal:

Create proposal at AgentDAO (action = ServiceNft.mint)
Mint ContributionNft , it will authenticate by checking whether sender is the proposal's proposer.

This makes sense because ServiceNft::mint cannot be called by any other contract than the agentDAO contract related to that virtual id. It does this by calling the following lines:

```solidity
IAgentNft.VirtualInfo memory info = IAgentNft(personaNft).virtualInfo(virtualId)
require(_msgSender() == info.dao, "Caller is not VIRTUAL DAO");
```

This goes to the AgentNftV2 instance and gets the virtualInfo for the virtual id which includes the address of the agentDAO contract related to that virtual id.

The issue is that in AgentDAO::_updateMaturity, which is called whenever a user votes in favour of a proposal with a reason and params, calls _calcMaturity and that does the following:


```solidity
        function _calcMaturity(uint256 proposalId, uint8[] memory votes) internal view returns (uint256) {
      address contributionNft = IAgentNft(_agentNft).getContributionNft();
      address serviceNft = IAgentNft(_agentNft).getServiceNft();
      uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId); /
      uint8 core = IContributionNft(contributionNft).getCore(proposalId); 
      uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
      // All services start with 100 maturity
      uint256 maturity = 100;
      if (coreService > 0) {
          maturity = IServiceNft(serviceNft).getMaturity(coreService);
          maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
      }

      return maturity;
  }
```
The bit to focus on is:

```solidity
  int8 core = IContributionNft(contributionNft).getCore(proposalId); 
  uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
```

The first step is that the core is retrieved from the contributionnft. The core number is an argument which is passed in by the proposer when ContributionNft::mint is called after have  the servicenft::mint proposal has been created according to the main activities details.
After this, the core and virtualId are passed into the ServiceNft::getCoreService.

The issue happens here because the core service is only ever updated in serviceNft::mint which the proposal is meant to execute but this function call happens in AgentDAO::_castVote before the proposal has been executed. ServiceNft::getCoreService will always return 0 as a result of this oversight and the maturity will always be 100 as the if block will always be false as the coreService will never be > 0.

Impact

Broken Maturity System: The maturity system, which is a core component of the protocol's governance and agent evaluation, is rendered ineffective for ServiceNFT mint proposals.

Loss of Governance Value: Since maturity affects rewards and governance power, the inability to properly calculate maturity means that voting on ServiceNFT mint proposals provides no meaningful impact on the protocol's governance system.

Reduced Protocol Utility: The protocol's ability to accurately assess and reward agent improvements through the maturity system is compromised, as all ServiceNFT mint proposals will have the same base maturity value.

Economic Impact: The incorrect maturity calculations could lead to misallocation of rewards and voting power in the system, as the maturity value is used in various protocol calculations.

Proof Of Code 

```javascript
it("maturity is always 100", async function () {
    //c deploy eloCalculator
    const { virtualToken, bonding, fRouter, agentFactory, service, contribution} = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury, poorMan } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await virtualToken.mint(trader.address, parseEther("120000"));
    await virtualToken
      .connect(trader)
      .approve(fRouter.target, parseEther("120000"));

    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat",
        "",
        ["", "", "", ""],
        parseEther("200")
      );

    const tokenInfo = await bonding.tokenInfo(await bonding.tokenInfos(0));
    expect(tokenInfo.agentToken).to.equal(
      "0x0000000000000000000000000000000000000000"
    );
    await expect(
      bonding.connect(trader).buy(parseEther("50000"), tokenInfo.token)
    ).to.emit(bonding, "Graduated");

    //c get token address from newpersona event
    const factoryFilter = agentFactory.filters.NewPersona;
    const factoryEvents = await agentFactory.queryFilter(factoryFilter, -1);
    const factoryEvent = factoryEvents[0];
    const { dao: daoAddress } = factoryEvent.args;
    console.log("daoAddress", daoAddress);
    const { token: tokenAddress } = factoryEvent.args;
    console.log("tokenAddress", tokenAddress);
    const { veToken: veTokenAddress } = factoryEvent.args;
    console.log("veTokenAddress", veTokenAddress);
    const { lp: lpAddress } = factoryEvent.args;
    console.log("lpAddress", lpAddress);

    const block= await ethers.provider.getBlock("latest");
    const timestamp = block.timestamp;
    console.log("timestamp", timestamp);

    const daoContract = await ethers.getContractAt("AgentDAO", daoAddress);
    const veTokenContract = await ethers.getContractAt("AgentVeToken", veTokenAddress);
    const router =await ethers.getContractAt("IUniswapV2Router02", process.env.UNISWAP_ROUTER);
    const tokenContract = await ethers.getContractAt("AgentToken", tokenAddress);
    const lpContract = await ethers.getContractAt("ERC20", lpAddress);
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).swapExactTokensForTokensSupportingFeeOnTransferTokens(parseEther("10"), 0, [virtualToken.target, tokenAddress], trader.address, timestamp + 1000);

    
    //c add liquidity
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await tokenContract.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).addLiquidity(tokenAddress, virtualToken.target, parseEther("10"), parseEther("10"), 0, 0, trader.address, timestamp + 1000);


    
    //c trader stakes lp tokens
 const traderBal = await lpContract.balanceOf(trader.address);
 console.log("traderBal", traderBal);
 await lpContract.connect(trader).approve(veTokenAddress, traderBal);
 await veTokenContract.connect(trader).stake(traderBal, trader.address, trader.address);


    //c founder makes serviceNft::mint proposal
    const founderBal = await veTokenContract.balanceOf(founder.address);
    console.log("founderBal", founderBal);

    await veTokenContract.connect(founder).delegate(founder.address);


    const targets = [service.target];
    const values = [0];
    const calldatas = ["0x590e60e900000000000000000000000000000000000000000000000000000000000000017f10eddc75153e5c78d3b132087db7a02de3651c1bafbeaef5f3b164a37f8585"] //c calldata for serviceNft::mint with virtualId 1 and descHash 0x7f10eddc75153e5c78d3b132087db7a02de3651c1bafbeaef5f3b164a37f8585 which is keccak256 for "Entertainment".
    const description = `Entertainment`
    await daoContract.connect(founder).propose(targets, values, calldatas, description); 


    
    let proposalId;
    let proposals;

  //c trader votes on servicenft::mint proposal with reason and params that should update maturity
  proposals = await daoContract.connect(trader).proposalDetailsAt(0);
  proposalId = proposals[0];
  const descriptionHash= proposals[4];
  console.log("descriptionHash", descriptionHash);

    //c founder mints contributuion nft
    await contribution.connect(founder).mint(founder.address, 1, 1, "test", proposalId, 1, true, 2);
  
  const reason = "I feel like it";
  const battles = [1,1,1,1];            
  const params = defaultAbiCoder.encode(["uint8[]"], [battles]);
  await daoContract.connect(trader).castVoteWithReasonAndParams(proposalId, 1, reason, params);


  //c get maturity from agentDAO
  const maturity = await daoContract.getMaturity(proposalId);
  console.log("maturity", maturity);

  //C expected maturity
  const eloCalc = await upgrades.deployProxy(
    await ethers.getContractFactory("EloCalculator"),
    [founder.address]
  );

  const expectedMaturity = await eloCalc.battleElo(100, battles);
  console.log("expectedMaturity", expectedMaturity);

  assert(maturity !== expectedMaturity, "maturity is same as expected maturity");
  });

```

Recommendation

Restructure the maturity calculation flow to occur after proposal execution:
Move the maturity calculation to happen after ServiceNFT::mint is executed
Store the voting parameters during the voting phase
Calculate and update maturity when the proposal is executed

Alternative approach: Implement a two-phase maturity system:
Phase 1: Store the voting parameters and votes during the voting period
Phase 2: Calculate and apply the maturity changes after the proposal is executed and the core service is created


# 10 SUBMITTED
 CRITICAL FUNCTIONS CANNOT BE PAUSED

Summary
The AgentFactoryV3 and AgentMigrator contracts inherit from OpenZeppelin's Pausable contract but fail to properly implement pausability across all critical functions. While proposeAgent in AgentFactoryV3 correctly implements the whenNotPaused modifier, executeApplication lacks this protection. Similarly, migrateAgent in AgentMigrator can be executed even when the contract is paused. This creates an inconsistent state where certain critical operations remain accessible during pause periods.

Vulnerability Details

There are a few critical functions that are not restricted by the pausable functionality to be applied in the AgentFactoryV3 and AgentMigrator contracts. Both of these contracts inherit from openzeppelin's Pausable contract which is used to pause critical functionality in a contract in certain scenarios. The pausability functionality stops users from calling key functions in a situation where changes need to be implemented for various reasons. 

AgentFactoryV3::executeApplication allows any proposed agent vis AgentFactoryV3::proposeAgent to be tokenized via actions like creating the relevant agentToken, AgentDAO and AgentVeToken contracts that allow action in the virtual protocol. Although AgentFactoryV3::proposeAgent has the whenNotPaused modifier, which prevents new agents from being proposed when the factory is paused, AgentFactoryV3::executeApplication does not have the same modifier which means that any proposed agents can be executed when the contract is paused which produces incostencies in the contract.

```solidity
 function executeApplication(uint256 id, bool canStake) public noReentrant {
        // This will bootstrap an Agent with following components:
        // C1: Agent Token
        // C2: LP Pool + Initial liquidity
        // C3: Agent veToken
        // C4: Agent DAO
        // C5: Agent NFT
        // C6: TBA
        // C7: Stake liquidity token to get veToken

        Application storage application = _applications[id];

        require(msg.sender == application.proposer || hasRole(WITHDRAW_ROLE, msg.sender), "Not proposer");

        _executeApplication(id, canStake, _tokenSupplyParams);
    }
```

The same behavior is evident in AgentMigrator::migrateAgent which is only callable by the founder of the agent. When the contract is paused, founders are still capable of migrating agents which should not be allowed.

```solidity
function migrateAgent(uint256 id, string memory name, string memory symbol, bool canStake) external noReentrant {
        require(!migratedAgents[id], "Agent already migrated");

        IAgentNft.VirtualInfo memory virtualInfo = _nft.virtualInfo(id);
        address founder = virtualInfo.founder;
        require(founder == _msgSender(), "Not founder");

        // Deploy Agent token & LP
        address token = _createNewAgentToken(name, symbol);
        address lp = IAgentToken(token).liquidityPools()[0];
        IERC20(_assetToken).transferFrom(founder, token, initialAmount);
        IAgentToken(token).addInitialLiquidity(address(this));

        // Deploy AgentVeToken
        address veToken = _createNewAgentVeToken(
            string.concat("Staked ", name),
            string.concat("s", symbol),
            lp,
            founder,
            canStake
        );

        // Deploy DAO
        IGovernor oldDAO = IGovernor(virtualInfo.dao);
        address payable dao = payable(
            _createNewDAO(oldDAO.name(), IVotes(veToken), uint32(oldDAO.votingPeriod()), oldDAO.proposalThreshold())
        );
        // Update AgentNft
        _nft.migrateVirtual(id, dao, token, lp, veToken);

        IERC20(lp).approve(veToken, type(uint256).max);
        IAgentVeToken(veToken).stake(IERC20(lp).balanceOf(address(this)), founder, founder);

        migratedAgents[id] = true;

        emit AgentMigrated(id, dao, token, lp, veToken);
    }

```


Impact

The lack of proper pausability implementation has several significant impacts:
Protocol Inconsistency: While new agent proposals are blocked during pause, existing proposals can still be executed, creating an inconsistent protocol state.
Emergency Response Compromise: The pause functionality's primary purpose is to allow protocol administrators to halt critical operations during emergencies or issues. The current implementation partially defeats this purpose.


Proof Of Code

```javascript
it("critical functions can be called when paused", async function () {
    const { virtualToken, bonding, fRouter, agentFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury } = await getAccounts();

    //c mint some virtual token to founder
    await virtualToken.mint(founder.address, parseEther("1000000"));

    //c approval to factory
    await virtualToken.connect(founder).approve(agentFactory.target, parseEther("1000000"));

    //c propose an agent
     
     await agentFactory.connect(founder).proposeAgent("test", "TEST", "test", [0, 1, 2], "0xa7647ac9429fdce477ebd9a95510385b756c757c26149e740abbab0ad1be2f16", process.env.TBA_IMPLEMENTATION, 600, 1000000000000000000000n);

     const filter = agentFactory.filters.NewApplication; //c this is an event filter. when the proposeAgent function is called, the NewApplication event is emitted. so this filter is used to get the event. its an ethers thing.
     const events = await agentFactory.queryFilter(filter, -1);
     const event = events[0];
     const { id } = event.args;

     //c pause the agentFactory
     await agentFactory.pause();


    //c execute the application and it will work even when contract is paused
    await expect(agentFactory.connect(founder).executeApplication(id, true)).to.emit(agentFactory, "NewPersona");
    
  });

```

Recommendation

Add the whenNotPaused modifier to both executeApplication and migrateAgent functions



# 11 SUBMITTED
TAXES ON BUYS/SELLS CAN BE PREVENTED FROM DISTRIBUTION TO AGENT CREATORS

Summary
The AgentToken::distributeTaxTokens function is publicly callable and allows anyone to send tax tokens directly to the AgentTax contract. When this happens, the tokens bypass the normal tax distribution flow and can be withdrawn directly to the treasury via AgentTax::withdraw without being split between the agent creator (70%) and ACP incentives (30%) as intended. This creates a situation where agent creators lose their rightful share of tax revenue, and the issue is compounded by the ability to migrate agents before the treasury can rectify the situation.

Vulnerability Details

AgentToken::distributeTaxTokens is a function that can be used to send agent tokens directly to the projectTaxRecipient. When agent tokens are created via AgentFactoryV3, there are tax parameters set and these taxes are applied to the agent token on transfers and the recipient of the tokens is the project tax recipient which is set via AgentFactoryV3::setTokenTaxParams. Taxes on these tokens are sent to whoever the protocol set to receive the taxes. The taxes are processed in AgentToken::_taxProcessing.

All currently live ai agents that have been created on base all have the same project tax recipient of 0x7E26173192D72fd6D75A759F888d61c2cdbB64B1. This address is a contract which is the AgentTax.sol implementation. This is the address that has been set in AgentFactoryV3::setTokenTaxParams as the projectTaxRecipient. You can see this at [AgentToken live contract](https://basescan.org/address/0x129966d7d25775b57e3c5b13b2e1c2045fbc4926#readContract).

```solidity

 function _createNewAgentToken(
        string memory name,
        string memory symbol,
        bytes memory tokenSupplyParams_
    ) internal returns (address instance) {
        instance = Clones.clone(tokenImplementation); 
        IAgentToken(instance).initialize(
            [_tokenAdmin, _uniswapRouter, assetToken],
            abi.encode(name, symbol),
            tokenSupplyParams_,
            _tokenTaxParams 
        );

        allTradingTokens.push(instance);
        return instance;
    }
 function setTokenTaxParams(
        uint256 projectBuyTaxBasisPoints,
        uint256 projectSellTaxBasisPoints,
        uint256 taxSwapThresholdBasisPoints,
        address projectTaxRecipient
    ) public onlyRole(DEFAULT_ADMIN_ROLE) {
        _tokenTaxParams = abi.encode(
            projectBuyTaxBasisPoints,
            projectSellTaxBasisPoints,
            taxSwapThresholdBasisPoints,
            projectTaxRecipient
        );
    }

```

The vulnerability stems from AgentToken::distributeTaxTokens as it is a function that is currently callable by anyone with no access control. The function distributes the taxes from buys/sells to the projectTaxRecipient. The standard flow that is used by each AgentToken deployed is that on transfers, if the taxes received from each transfer meets a certain swap threshold, the tokens are autoswapped for the pair token which is $VIRTUAL. The virtual token is then transfered to the projectTaxRecipient which has a handleAgentTaxes function which is used to convert the virtual tokens sent by the agentToken to a pre determined assetToken via AgentTax::_swapForAsset. This function then splits the tokens after the swap as per [the virtual documentation](https://basescan.org/address/0x129966d7d25775b57e3c5b13b2e1c2045fbc4926#readContract) which says that for sentient agents, the 1% fee is split between the agent creator (70%) and ACP incentives (30%).

```solidity
function _swapForAsset(uint256 agentId, uint256 minOutput, uint256 maxOverride) internal returns (bool, uint256) {
        TaxAmounts storage agentAmounts = agentTaxAmounts[agentId];
        uint256 amountToSwap = agentAmounts.amountCollected - agentAmounts.amountSwapped;

        uint256 balance = IERC20(taxToken).balanceOf(address(this));

        require(balance >= amountToSwap, "Insufficient balance");

        TaxRecipient memory taxRecipient = _getTaxRecipient(agentId);
        require(taxRecipient.tba != address(0), "Agent does not have TBA");

        if (amountToSwap < minSwapThreshold) {
            return (false, 0);
        }

        if (amountToSwap > maxOverride) {
            amountToSwap = maxOverride;
        }

        address[] memory path = new address[](2);
        path[0] = taxToken;
        path[1] = assetToken;

        uint256[] memory amountsOut = router.getAmountsOut(amountToSwap, path);
        require(amountsOut.length > 1, "Failed to fetch token price");

        try
            router.swapExactTokensForTokens(amountToSwap, minOutput, path, address(this), block.timestamp + 300) 
        returns (uint256[] memory amounts) {
            uint256 assetReceived = amounts[1];
            emit SwapExecuted(agentId, amountToSwap, assetReceived);

            uint256 feeAmount = (assetReceived * feeRate) / DENOM;
            uint256 creatorFee = assetReceived - feeAmount;

            if (creatorFee > 0) {
                IERC20(assetToken).safeTransfer(taxRecipient.creator, creatorFee); 
                if (address(tbaBonus) != address(0)) {
                    tbaBonus.distributeBonus(agentId, taxRecipient.creator, creatorFee);
                }
            }

            if (feeAmount > 0) {
                IERC20(assetToken).safeTransfer(treasury, feeAmount);
            }

            agentAmounts.amountSwapped += amountToSwap;

            return (true, amounts[1]);
        } catch {
            emit SwapFailed(agentId, amountToSwap);
            return (false, 0);
        }
    }

```

As stated earlier, when AgentToken::distributeTaxTokens is called, the agentTokens are transfered directly to the AgentTax contract. When this happens, the flow changes as the only way these tokens can leave the contract is via AgentTax::withdraw which is a function that sends all the tokens directly to the treasury. The issue with this is that these tokens sent are tax tokens which means they also have to follow the normal flow stated in the documentation which is that for sentient agents, the 1% fee is split between the agent creator (70%) and ACP incentives (30%). When  AgentTax::withdraw is called, all tokens are sent to the treasury which eliminates the 70% that should be sent to the agent creator.

A malicious user could spam AgentToken::distributeTaxTokens to send all tokens to the AgentTax contract before the agentToken contract reaches the swap threshold and doing this on repeat will prevent the agent creators from getting any of their allocation of the taxes as all of the tokens will go to the treasury via AgentTax::withdraw.

A potential counter argument to this could be that once the tokens are withdrawn, the splits could be recalculated and distributed accordingly. The counter to this would be to consider a scenario where a founder calls AgentMigrator::migrateAgent before the treasury could rectify this issue. This would mean that the creator AND virtuals could end up with worthless tokens because the agentToken will now be deployed at a brand new address. As a result, the old agentToken/VIRTUAL lp will rapidly depreciate as people will be unbonding their virtual to be able to bond with the new agentToken. This means that virtuals and the agent creator end up gaining no value via the taxes without realizing. 

This situation can be scaled even further given that there are currently 17000+ ai agents [17000+ ai agents](https://www.virtuals.io/about)  and this can be done on every single one of them which will in the best case scenario, will cost virtuals a lot of time and money to individually recalculate the splits and make the transfers without considering the lost value for the depreciation of a lot of the ai agents over time which is a huge factor to consider. With fluctuating prices and a volatile tokens like ai agentTokens, the value of the damage will only scale up the more agentTokens get deployed and the more agentTokens can be accessed by the malicious user.

Impact

Bypass of Intended Tax Distribution: The vulnerability allows attackers to force tax token transfers to the AgentTax contract before reaching the normal swap threshold, circumventing the intended 70/30 split mechanism.

Permanent Loss of Creator Funds: When tokens are withdrawn via AgentTax::withdraw, 100% of funds go to the treasury instead of the intended 70% to creators and 30% to ACP incentives.

Compounding Effect During Migration: The impact is exacerbated during agent migrations, as old tokens become worthless before proper tax distribution can occur, making recovery impossible.

Systemic Risk: With over 17,000 live AI agents, this vulnerability could affect the entire ecosystem simultaneously, not just individual tokens.

Proof Of Code

```javascript
it("tax is not split between agent creator and ACP incentives", async function () {
    const { virtualToken, bonding, fRouter, agentFactory } = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury } = await getAccounts(); 

     //c mint some virtual token to founder
     await virtualToken.mint(founder.address, parseEther("1000000"));

     //c approval to factory
     await virtualToken.connect(founder).approve(agentFactory.target, parseEther("1000000"));


     //c reset token tax params with proper agent tax contract

    const agentTax = await ethers.getContractAt("AgentTax", process.env.AGENT_TAX);
     await agentFactory.setTokenTaxParams(
      process.env.TAX,
      process.env.TAX,
      process.env.SWAP_THRESHOLD,
      process.env.AGENT_TAX);
 
     //c propose an agent
      
      await agentFactory.connect(founder).proposeAgent("test", "TEST", "test", [0, 1, 2], "0xa7647ac9429fdce477ebd9a95510385b756c757c26149e740abbab0ad1be2f16", process.env.TBA_IMPLEMENTATION, 600, 1000000000000000000000n);
 
      const filter = agentFactory.filters.NewApplication; //c this is an event filter. when the proposeAgent function is called, the NewApplication event is emitted. so this filter is used to get the event. its an ethers thing.
      const events = await agentFactory.queryFilter(filter, -1);
      const event = events[0];
      const { id } = event.args;
 
     
     await expect(agentFactory.connect(founder).executeApplication(id, true)).to.emit(agentFactory, "NewPersona");

      //c get token address from newpersona event
    const factoryFilter = agentFactory.filters.NewPersona;
    const factoryEvents = await agentFactory.queryFilter(factoryFilter, -1);
    const factoryEvent = factoryEvents[0];
    const { token: tokenAddress } = factoryEvent.args;
    console.log("tokenAddress", tokenAddress);

    const tokenContract = await ethers.getContractAt("AgentToken", tokenAddress);
    //c trader buys agent token and gets charged tax
    await virtualToken.mint(trader.address, parseEther("120000"));
    //c buy some of the new token from the uni pool
    const router =await ethers.getContractAt("IUniswapV2Router02", process.env.UNISWAP_ROUTER);
    //c get timestamp
    const block= await ethers.provider.getBlock("latest");
    const timestamp = block.timestamp;
    console.log("timestamp", timestamp);
    const pretokenBalance = await tokenContract.balanceOf(tokenAddress);
    console.log("pretokenBalance", pretokenBalance);
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).swapExactTokensForTokensSupportingFeeOnTransferTokens(parseEther("10"), 0, [virtualToken.target, tokenAddress], trader.address, timestamp + 1000);

    const posttokenBalance = await tokenContract.balanceOf(tokenAddress);
    console.log("posttokenBalance", posttokenBalance);

    //c random person calls distributeTax
    await tokenContract.connect(trader).distributeTaxTokens();

    //c get balance from transfer to projectTaxRecipient
    const projectTaxRecipientBalance = await tokenContract.balanceOf(process.env.AGENT_TAX);
    console.log("projectTaxRecipientBalance", projectTaxRecipientBalance);
    
    //c withdraw to treasury
    let admin="0xE220329659D41B2a9F26E83816B424bDAcF62567";

    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [admin],
    });
    const adminSigner = await ethers.getSigner(admin);
    await agentTax.connect(adminSigner).withdraw(tokenContract.target);

  });

  
```


Recommendation 

Add access control to AgentToken::distributeTaxTokens to restrict who can call this function. Consider making it only callable by the contract owner or a designated admin role.
Modify AgentTax::withdraw to implement the proper tax distribution flow:
When tokens are withdrawn, immediately swap them for the asset token
Split the proceeds according to the 70/30 ratio between agent creator and treasury
Add proper error handling and slippage protection for the swap
Consider adding a minimum threshold to prevent dust amounts from being processed




# 12 SUBMITTED

PRECISION LOSS IN AGENTDAO::GETMATURITY

Summary
The AgentDAO::getMaturity function suffers from precision loss in its calculation due to a mismatch between how maturity is accumulated and how votes are counted. When users vote on proposals, some may use castVoteWithReasonAndParams (which updates maturity) while others use castVote (which doesn't). This discrepancy, combined with Solidity's integer division, can cause the maturity calculation to incorrectly return 0, even when there are valid maturity contributions from other voters.

Vulnerability Details

AgentDAO::getmaturity is used to get the maturity value for a proposal. The maturity is determined during a proposal's active period where a user can vote on a proposal using AgentDAO::castvotewithreasonandparams. The maturity value is used to rate the performance of an ai agent. 


```solidity
  function _updateMaturity(address account, uint256 proposalId, uint256 weight, bytes memory params) internal {
        // Check is this a contribution proposal
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address owner = IERC721(contributionNft).ownerOf(proposalId);
        if (owner == address(0)) {
            return;
        }

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (!isModel) {
            return;
        }

        uint8[] memory votes = abi.decode(params, (uint8[]));
        uint256 maturity = _calcMaturity(proposalId, votes);

        _proposalMaturities[proposalId] += (maturity * weight);

        emit ValidatorEloRating(proposalId, account, maturity, votes);
    }

     function getMaturity(uint256 proposalId) public view returns (uint256) {
        (, uint256 forVotes, ) = proposalVotes(proposalId);
        return Math.min(10000, _proposalMaturities[proposalId] / forVotes);
    }
```

The issue is that a user can cast a vote using the normal castVote function without reason and params and decide not to update the maturity and their votes will count toward the getMaturity calculation. This can lead to the getMaturity calculation incorrectly returning 0 due to precision loss in solidity division.

Looking at the _updateMaturity function,  the maturity is multiplied by the weight of the user voting in support of the proposal. Not every user who voted for the proposal will have voted with params and triggered the _updateMaturity function. so the number of forVotes doesnt reflect the weights initially added to each maturity. It could be significantly more. In fact, it could be so much more that this returns 0. Lets see an example:

User A votes for the proposal with params. _updateMaturity is called and user A's weight is 1000 votes.  Assume maturity is 500, then the _proposalMaturities[proposalId] is 500 * 1000 = 500000.

User B votes for the proposal with no params. Just a normal castVote call. Assume User B's weight is 5,000,000 votes. 

Now the getMaturity function is called.

forVotes = 5,000,000 + 1000 = 5,001,000

_proposalMaturities[proposalId] = 500000

maturity = 500000 / 5001000 = 0 because solidity doesnt do decimals so it just returns 0 which is the wrong answer. 

ServiceNft::mint contains the following line:

```solidity
_maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId);
```

This shows that the getMaturity is directly used in minting service nft's. As a result, the precision loss could lead to incorrect values being stored as the maturity value for the proposal id.

Impact

Incorrect maturity values (particularly 0) are stored in ServiceNFTs via ServiceNft::mint
Agent performance ratings are skewed due to inaccurate maturity calculations
The issue affects all proposals where there's a mix of votes with and without params
The impact is more severe when there's a large discrepancy between voting weights
The vulnerability can be exploited by users voting without params to dilute maturity calculations


Proof Of Code

```javascript
it("maturity precision loss", async function () {
    //c deploy eloCalculator
    const { virtualToken, bonding, fRouter, agentFactory, service, contribution, agentNft} = await loadFixture(
      deployBaseContracts
    );
    const { founder, trader, treasury, poorMan } = await getAccounts();

    await virtualToken.mint(founder.address, parseEther("200"));
    await virtualToken
      .connect(founder)
      .approve(bonding.target, parseEther("1000"));
    await virtualToken.mint(trader.address, parseEther("120000"));
    await virtualToken
      .connect(trader)
      .approve(fRouter.target, parseEther("120000"));

    await bonding
      .connect(founder)
      .launch(
        "Cat",
        "$CAT",
        [0, 1, 2],
        "it is a cat",
        "",
        ["", "", "", ""],
        parseEther("200")
      );

    const tokenInfo = await bonding.tokenInfo(await bonding.tokenInfos(0));
    expect(tokenInfo.agentToken).to.equal(
      "0x0000000000000000000000000000000000000000"
    );
    await expect(
      bonding.connect(trader).buy(parseEther("50000"), tokenInfo.token)
    ).to.emit(bonding, "Graduated");

    //c get token address from newpersona event
    const factoryFilter = agentFactory.filters.NewPersona;
    const factoryEvents = await agentFactory.queryFilter(factoryFilter, -1);
    const factoryEvent = factoryEvents[0];
    const { dao: daoAddress } = factoryEvent.args;
    console.log("daoAddress", daoAddress);
    const { token: tokenAddress } = factoryEvent.args;
    console.log("tokenAddress", tokenAddress);
    const { veToken: veTokenAddress } = factoryEvent.args;
    console.log("veTokenAddress", veTokenAddress);
    const { lp: lpAddress } = factoryEvent.args;
    console.log("lpAddress", lpAddress);

    const block= await ethers.provider.getBlock("latest");
    const timestamp = block.timestamp;
    console.log("timestamp", timestamp);

    const daoContract = await ethers.getContractAt("AgentDAO", daoAddress);
    const veTokenContract = await ethers.getContractAt("AgentVeToken", veTokenAddress);
    const router =await ethers.getContractAt("IUniswapV2Router02", process.env.UNISWAP_ROUTER);
    const tokenContract = await ethers.getContractAt("AgentToken", tokenAddress);
    const lpContract = await ethers.getContractAt("ERC20", lpAddress);
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).swapExactTokensForTokensSupportingFeeOnTransferTokens(parseEther("10"), 0, [virtualToken.target, tokenAddress], trader.address, timestamp + 1000);

    
    //c add liquidity
    await virtualToken.connect(trader).approve(router.target, parseEther("10"));
    await tokenContract.connect(trader).approve(router.target, parseEther("10"));
    await router.connect(trader).addLiquidity(tokenAddress, virtualToken.target, parseEther("10"), parseEther("10"), 0, 0, trader.address, timestamp + 1000);


    
    //c trader stakes lp tokens
 const traderBal = await lpContract.balanceOf(trader.address);
 console.log("traderBal", traderBal);
 await lpContract.connect(trader).approve(veTokenAddress, traderBal);
 await veTokenContract.connect(trader).stake(traderBal, trader.address, trader.address);

  //c poorMan gets some lp tokens
  await virtualToken.mint(poorMan.address, parseEther("100"));
  await virtualToken.connect(poorMan).approve(router.target, parseEther("100"));
  await router.connect(poorMan).swapExactTokensForTokensSupportingFeeOnTransferTokens(parseEther("50"), 0, [virtualToken.target, tokenAddress], poorMan.address, timestamp + 1000);
  await virtualToken.connect(trader).approve(router.target, parseEther("50"));
    await tokenContract.connect(poorMan).approve(router.target, parseEther("50"));
    await router.connect(poorMan).addLiquidity(tokenAddress, virtualToken.target, parseEther("50"), parseEther("50"), 0, 0, poorMan.address, timestamp + 1000);


    //c poorMan stakes lp tokens
    const poorManBal = await lpContract.balanceOf(poorMan.address);
    console.log("poorManBal", poorManBal);
    await lpContract.connect(poorMan).approve(veTokenAddress, poorManBal);
    await veTokenContract.connect(poorMan).stake(poorManBal, poorMan.address, poorMan.address);

    //c founder makes serviceNft::mint proposal
    const founderBal = await veTokenContract.balanceOf(founder.address);
    console.log("founderBal", founderBal);

    await veTokenContract.connect(founder).delegate(founder.address);


    const targets = [service.target];
    const values = [0];
    const calldatas = ["0x590e60e900000000000000000000000000000000000000000000000000000000000000017f10eddc75153e5c78d3b132087db7a02de3651c1bafbeaef5f3b164a37f8585"] //c calldata for serviceNft::mint with virtualId 1 and descHash 0x7f10eddc75153e5c78d3b132087db7a02de3651c1bafbeaef5f3b164a37f8585 which is keccak256 for "Entertainment".
    const description = `Entertainment`
    await daoContract.connect(founder).propose(targets, values, calldatas, description); 


    
    let proposalId;
    let proposals;

  //c trader votes on servicenft::mint proposal with reason and params that should update maturity
  proposals = await daoContract.connect(trader).proposalDetailsAt(0);
  proposalId = proposals[0];
  const descriptionHash= proposals[4];
  console.log("descriptionHash", descriptionHash);

    //c founder mints contribution nft
    await contribution.connect(founder).mint(founder.address, 1, 1, "test", proposalId, 1, true, 2);

    //c set elo calculator
    const eloCalc = await upgrades.deployProxy(
      await ethers.getContractFactory("EloCalculator"),
      [founder.address]
    );

    await agentNft.setEloCalculator(eloCalc.target);
  
    //c user votes on proposal
  const reason = "I feel like it";
  const battles = [1,1,1,1];            
  const params = defaultAbiCoder.encode(["uint8[]"], [battles]);
  await daoContract.connect(trader).castVoteWithReasonAndParams(proposalId, 1, reason, params);


  //c founder votes on proposal with no params
  await daoContract.connect(founder).castVote(proposalId, 1);

  //c get maturity from agentDAO
  const maturity = await daoContract.getMaturity(proposalId);
  console.log("maturity", maturity);
  });

```

Recommendation

Consider tracking maturity-contributing votes separately from total votes to ensure accurate calculations. Either require all votes to include params or modify castVote to use a default maturity value when params aren't provided


# 13 SUBMITTED

 USERS WITHOUT VIRGEN POINTS CAN PARTICIPATE IN GENESIS LAUNCHES

Summary

The Genesis::participate function allows any user to participate in genesis launches by simply providing an arbitrary point amount, without validating if they actually own those points. This directly contradicts the protocol's documentation which states that VIRGEN points are required for participation and must be earned through measurable, net-positive actions. The current implementation only checks that the point amount is greater than zero, enabling users to bypass the intended point-based access control mechanism.

Vulnerability Details

Genesis::participate is a function that allows users to pledge their points and an amount of virtual tokens towards a new ai agent launch.

```solidity
 function participate(uint256 pointAmt, uint256 virtualsAmt) external nonReentrant whenActive {
        require(pointAmt > 0, "Point amount must be greater than 0"); 
        require(virtualsAmt > 0, "Virtuals must be greater than 0");

        // Check single submission upper limit
        require(virtualsAmt <= maxContributionVirtualAmount, "Exceeds maximum virtuals per contribution");

        // Add balance check
        require(IERC20(virtualTokenAddress).balanceOf(msg.sender) >= virtualsAmt, "Insufficient Virtual Token balance");
        // Add allowance check
        require(
            IERC20(virtualTokenAddress).allowance(msg.sender, address(this)) >= virtualsAmt,
            "Insufficient Virtual Token allowance"
        );

        // Update participant list
        if (mapAddrToVirtuals[msg.sender] == 0) {
            participants.push(msg.sender);
        }

        // Update state
        mapAddrToVirtuals[msg.sender] += virtualsAmt;

        IERC20(virtualTokenAddress).safeTransferFrom(msg.sender, address(this), virtualsAmt);

        emit Participated(genesisId, msg.sender, pointAmt, virtualsAmt);
    }
```

In the [documentation](https://whitepaper.virtuals.io/about-virtuals/tokenization-platform/genesis-launch/genesis-points)., users must have a certain amount of points to be able to participate in genesis launches to ensure fair participation. The documentation clarifies that points are not awarded arbitrarily. They represent measurable, net-positive actions that align with the Virtuals protocol’s goals. The ability for any user to award themselves any amount of points directly violates this statement. In this function, a user who has no points can simply enter an arbitrary amount of points and as long as they have the corresponding amount of virtual tokens, they will be able to unfairly participate in the launch.

Impact

- **Fair Participation Compromised**: The core mechanism for ensuring fair participation in genesis launches is broken, as users can participate without earning points through legitimate means.

- **Protocol Integrity Undermined**: The ability to arbitrarily claim points undermines the protocol's goal of rewarding users for positive contributions.

- **Economic Impact**: Users who have legitimately earned points through protocol participation may face unfair competition from users who bypass the point requirement.

Proof Of Code

In the genesis.js file, it calls Genesis::participate multiple times with dummy accounts who have no points but simply pass an arbitrary points value. This proves that any user without virgens pointsn can freely participate in token launches without restrictions.

Recommendation

1. Integrate with a VIRGEN points contract to properly track and validate point ownership.

2. Add a points transfer mechanism if points should be spent during participation.

3. Implement proper access control for point distribution to ensure points can only be awarded through legitimate means.

