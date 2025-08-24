# 1 ERC-5095 OVERVIEW, WHAT TO CONSIDER WHEN READING EIP'S

Spectra's whole idea is to allow users to do one of 2 things and both paths can be seen in:

![spectra_contracts_architecture](./spectra_contracts_architecture.png) .

TRhe first user path allows any user to deploy a principal token and a curve pool via a factory contract. The second path allows a user to directly deposit an ERC4626 (vault token) or the underlying token of that ERC4626 directly into the principal token contract.

We are going to focus on the principal token contract because the guidelines say that is is ERC5095 complaint and I have never heard of ERC5095 before so we are going to briefly go through this to get an understanding. Although the principal token is not in scope, it is important to know this for you to get a general understanding of what this code is doing.

You can read more about ERC-5095 at: https://eips.ethereum.org/EIPS/eip-5095

The idea is that I will red the EIP and explain any lines that may seem confusing.
 In the motivation section, it describes examples where principal tokens are implemented and what kind of protocols use them. It says:
 "The primary examples include yield tokenization platforms which strip future yield leaving a principal token behind, as well as fixed-rate money-markets which utilize principal tokens as a medium to lend/borrow."

 Lets go over what 'stripping future yield' means:

When you deposit a yield-bearing asset (like stETH, cUSDC, or yDAI), it naturally earns yield over time.
Yield tokenization splits that asset into two separate tokens:

 Principal Token – Represents the initial value of your deposit
 Yield Token – Represents the right to the future yield that will accumulate

This separation allows both parts to be traded or used independently.

 What does “stripping future yield” mean?
It means taking a yield-bearing asset (e.g., stETH or cUSDC) and splitting it into two parts:

The Principal — the amount that will be returned at maturity (or just the non-yielding part)
The Yield — the stream of interest or yield the asset will earn over time

This process is called “stripping the yield”, just like in traditional finance where a bond's coupon payments are stripped from the principal.

Example in DeFi
Let’s say you deposit 1000 USDC into a fixed-rate lending protocol for 1 year at 5% APY.

Instead of holding the yield-bearing asset as a single token, the protocol mints:

1000 pUSDC → Principal Token (redeemable for 1000 USDC in 1 year)
yUSDC → Yield Token (entitles the holder to the 5% yield over the year)

Now:
You can sell the yield to someone else right now (e.g., yUSDC)
Or sell the principal and hold the yield
Or use either in DeFi — e.g., to borrow/lend separately



So in the methods description, there is a previewredeem function that is also shown in the contracts architecture diagram I linked above. This function is located in the principal token contract and is explained in erc5095. In the explanation, it says that redemption fees must be considered when previewing redeem. This is a key check that i need to make sure that the previewRedeem function takes fees into account. As a security researcher, when you read an EIP, these nuances are what you are to look out for. Little things like this are where bugs lie because if the fees arent included, then you need to think about how you can exploit that.

Another nuance is located in the explanation of the redeem function where it says:
"Interfaces and other contracts MUST NOT expect fund custody to be present. While custodial redemption of Principal Tokens through the Principal Token contract is extremely useful for integrators, some protocols may find giving the Principal Token itself custody breaks their backwards compatibility."

What this means is that when designing a principal token contract, whoever is interacting with one, should not expect the underlying amounts to be stored in the principal token contracts. Although it could be, that is up to the discretion of the protocol as to where the funds are kept. The reason this is important is like for aave for example, which isnt a principal token but when aTokens are gotten by users from a certain pool, the underlying tokens are stored in the aToken contract and not the pool and a vulnerability was found when a protocol tried to query the pool contract to get the underlying token balance which wasnt stored in the pool contracts. So stating where the underlying tokens should be is important.

You will notice that EIP-5095 has methods for redeem and withdraw and you might be wondering what the difference between the 2 are?  Lets compare the descriptions. For redeem:
"At or after maturity, burns exactly principalAmount of Principal Tokens from from and sends underlyingAmount of underlying tokens to to."

For withdraw:

"Burns principalAmount from holder and sends exactly underlyingAmount of underlying tokens to receiver."

 Why Do We Need Both?
 redeem is is focused on the amount of principal to burn
You know how many Principal Tokens you want to redeem. After maturity, the exchange rate is locked in, so the protocol knows exactly how much underlying to give you per token.

This is the classic "I’m done, give me my money back" function.

withdraw is focused on the amount of underlying to get back
You say: “I want exactly X amount of the underlying back — burn whatever amount of principal is necessary to give me that.” So the nuance from the description in the EIP-5095 is the use of 'exactly' where redeem says burn exactly principalAmount and withdraw says burn principalAmount which can be any principal amount and this is how much detail you must go into when reading things. One word can make a lot of difference.

This is useful:
Before maturity, where yield hasn’t fully accrued
For integrations where the exact amount of underlying is important (e.g. fixed income products, swaps, or batching redemptions)

 Real-World Analogy
redeem → You cash in your $100 bond after it matures and get $100 back (plus interest).
withdraw → You say, “I want $50 back right now, even if it means forfeiting some interest,” and the system burns enough of your bond to give you $50 of principal.

Another key point is in the security considerations section which says "As is common across many standards, it is strongly recommended to mirror the underlying token’s decimals if at all possible, to eliminate possible sources of confusion and simplify integration across front-ends and for other off-chain users." This is another key point to keep in mind.




# 2 TALK ABOUT HOW A BEACON IS ANOTHER NAME FOR THE ADDRESS OF AN IMPLEMENTATION CONTRACT. SEE BEACONPROXY.SOL

# 3 GETTING PRICE DERIVATES SOLELY FROM A SINGLE SOURCE IS ALWAYS A BAD IDEA

You already know about oracle manipulation and how bad an idea that is as so many hacks come from this but oracle manipulation is a much deeper topic and has many layers and the more audits you do, you will see many codebases that attempt to mitigate this but it is always good to know what to look out for to spot oracle manipulation exploits. Oracle manipulation doesnt always come in the form of manipulating prices. It can also come ffrom any price derivatives. So any variable that is dependent on prices for its calculations. If you see any codebase where prices or price derivates are gotten from a single source, alarm bells should always be ringing in your head. I will show you an example where spectra did it correctly to avoid oracle mainpulation.

```solidity
//**
     * @dev This function returns the TWAP rate PT/IBT on a Curve StableSwap NG pool
     * This accounts for special cases where underlying asset becomes insolvent and has decreasing exchangeRate
     * @param pool Address of the Curve Pool to get rate from
     * @return PT/IBT exchange rate
     */
    function getPTToIBTRateSNG(address pool) public view returns (uint256) {
        IPrincipalToken pt = IPrincipalToken(ICurveNGPool(pool).coins(1));
        uint256 maturity = pt.maturity();
        if (maturity <= block.timestamp) {
            return pt.previewRedeemForIBT(pt.getIBTUnit()); //c PrincipalToken::previewRedeemForIBT is a function that returns what one unit of the IBT token is which is calculated by 10**ibtdecimals so if the interest bearing token has 18 decimals, then one unit of the IBT token is 1e18. the previewRedeemForIBT function is used to convert amount of PT shares to amount of IBT with current rates. you might be wondering why the ibt unit is used here instead of the the ibt unit. This is not a bug because the way the principal token is designed, it has the same decimals as the ibt token. see principaltoken::decimals() which overrides all the ERC20 metadata functions.
        } else {
            uint256[] memory storedRates = IStableSwapNG(pool).stored_rates();
            return
                pt.getIBTUnit().mulDiv(storedRates[1], storedRates[0]).mulDiv(
                    IStableSwapNG(pool).price_oracle(0),
                    CURVE_UNIT //c 1e18
                );
        } /*c lets break down this formula.  pt.getIBTUnit().mulDiv(storedRates[1], storedRates[0]) gets 1 unit of the pt token by doing pt.getIBTUnit() and then multplies that by the stored rate of the pt token which is storedRates[1]. remember that the stored rate of the principal token is simply the present value of 1 principal token which takes into account the initial price of the pool and the current ibt rate because remember that stored[1] comes from the PrincipalToken::getPTRate which starts off by getting the ibt rate and that ibt ratye is gotten by doing IERC4626(_ibt).previewRedeem(ibtUnit).toRay(underlyingDecimals) which gets the ibt rate in terms of its underlying asset. See RateAdjustmentOracle::value() for more details and PrincipalToken:: _getCurrentPTandIBTRates. So stored[0] gives us the principal token value in terms of the underlying asset of the ibt token.
        
        we then divide that by the stored rate of the ibt token which is storedRates[0]. stored rate of the ibt token is how much 1 ibt token is worth in terms of its underlying asset. Dividing storedrates[1] by storedrates[0]gives us how much 1 ibt token is worth in terms of the pt token in real terms.

        So what you might be wondering is that since we have the real pttoibt rate, why are we including the price oracle?  The reason is that the curve price oracle only bases its calculations on the pools internal state. so the stableswap ng pool has the following function:
        
        def price_oracle(i: uint256) -> uint256:
    return self._calc_moving_average(
        self.last_prices_packed[i],
        self.ma_exp_time,
        self.ma_last_time & (2**128 - 1)
    )
        This is how the price oracle is calculated and last_prices_packed is a variable that calculates prices based on the current state of the pool. so this is purely based on the baalnces of the pool at different timestamps which calculate a moving average. the stored_rates function is not used when calculating the price returned by the price oracle. To answer the question, the pool itself is a market maker and so is the principal token as in both situations, tokens can be exchanged for each other. so when trying to calculate the pttoibt rate, we need to take into account the rates from every market maker and combine them. Think about chainlink price feeds. They combine prices from different price feeds to get a more accurate prices. This is exactly what is happening here but for the rates. It makes sure the impact of the rate from a single source is reduced. So if someone decided to manipulate the pool, then the rate would be normalized using the real rate gotten from the stored rates. If the underlying asset of the interest bearing token became insolvent, the rate would be normalized using the real rate gotten from the stored rates which would reflect the insolvent asset as the stored rate of the interest bearing token converts it to its value in assets. This is why the natspec says: This accounts for special cases where underlying asset becomes insolvent and has decreasing exchangeRate. Lets see an example:

                Let's say:
        PT = fixed claim to 1 USDC → storedRates[1] = 1e18

        IBT = now worth 1.05 USDC → storedRates[0] = 1.05e18

        So:
        storedRates[1] / storedRates[0] = 0.952
        Meaning: PT is worth 0.952 IBT

        But suppose Curve pool has imbalance due to trading activity and reports:

        price_oracle() = 1.03e18
        So pool thinks: 1 PT = 1.03 IBT

        Now the full value is:

        ptUnit × 0.952 × 1.03 = ≈ 0.981
        So the actual market value of PT = 0.981 IBT — above its intrinsic value.

        */
    }
```

Read all the comments I made in this function. The idea is that spectra aimed to get the rate of principal token to interest bearing token. The rate is obviously gotten from dividing the 2 prices. So the rate is a derivative of the price. Instead of using just the prices from the pt/ibt pool or just the stored rates representing the real pt/ibt rate, they combined the 2 and I explain in the comments how beneficial this is.


# 4 HOW PRINCIPAL AND YIELD TOKENS ACTUALLY WORK

I will be posting some code blocks from the PrincipalToken contract where i go in depth about how principal tokens work and their relationship to yield tokens. If you are ever wondering how these tokens are implemented, have a read below and also go into the PrincipalToken contract where I have left a lot more comments on  functions that will give you good insight on how principal tokens work so make sure to have a read. 

```solidity
function deposit( /*c there are some things that need to become clear about principal tokens and yield tokens. so when a user deposits an ibt into this principal token contract, the ibt gets split into 2 tokens. The first token is the principal token and the second is the interest bearing token. The principal token represents the initial deposit of the user. The way this works is via snapshots. We know that a maturity is set on all principal tokens and this is the key. So the idea is that a snapshot is taken capturing the rate of principal tokens to ibt tokens at the time when a token is deposited. Another snapshot is taken that shows the rate of ibt tokens to underlying asset tokens at the time when a token is deposited. The reason for this to make sure that the current value of the ibt token is recorded at the time when a token is deposited. This value of the ibt at the snapshot time is the value of the the initial deposit of the user. This is the value that the principal tokens will be able to redeem for once the token matures.

        The other side of the coin is the yield tokens. The yield tokens represent the yield that the initial deposit will generate from when it was deposited until the maturity of the principal token contract. again, this is recorded via snapshots.A snapshot will use the interest rate of the ibt token to predict how much yield will have been generated between the time when the deposit was made and the maturity of the principal token contract. This interest rate is always going to change though due to market conditions. So from deposit to maturity, the value of the yield tokens will change. the way the value of the yield tokens is calculated is via CurveOracleLib::getYTToIBTRateSNG which i have already covered in detail. 

        The yield token VALUE always converges to 0 as the current time converges towards maturity. The logic behind this is because the actual yield that the yield token generates fluctuates based on the interest rate of the ibt token which is determined by the market and the protocol that controls the ibt token. For example, aave has aTokens which are interest bearing tokens which gain interest based on whatever factors aave use to determine the exchange rate of the aToken in relation to the underlying asset. So the yield of the yield token is not fixed and fluctuates based on market conditions.aTokens are actually rebase tokens in reality but for this example, just keep the assumption that they operate with an exchange rate. So the value of the yield token is based on what the market thinks the yield will be at maturity. If people think that aave will be very profitable by maturity time and pay out more yield to aToken holders, they will buy more yield tokens to anticipate this yield. Lets see an example of this: 

        Bob buys 1 YT-stETH for ~0.13 ETH at an implied 15% APY, with an expiry one year from now.

        Consider these scenarios after 1 year:

        If realized APY is 10% -> Bob accrued 0.10 eth in yield, which results in a net loss of 0.03 eth [0.13-0.10]

        If realized APY is 15% -> Bob accrued 0.15 eth in yield, which results in a fixed-rate alike return of 0.02 eth [0.15-0.13] 

        If realized APY is 20% -> Bob accrued 0.20 eth in yield, which results in a net gain of 0.07 eth [0.2-0,13]

        so bob buys 1 YT-stETH for ~0.13 ETH where right now stETH has a 15% APY. Here is something else I need to explain. When principal tokns are minted, the shares minted of the principal token and the yield token are the same BUT they have different underlying amounts. 1 PT represents 1 unit of the initial deposit. 1 YT represents the yield that the initial deposit will generate from when it was deposited until the maturity of the principal token contract. So if Bob buys 1YT which had 15% apy, at maturity, the amount of wth Bob will get is 0.15 eth.This apy is not fixed and in a year, when the yt expires, the apy could be anything. So Bob is gambling on the fact that in a year, the apy will be higher than 15% so that he can redeem more eth than the yield token was orginally supposed to give.

        So the value of the yt comes from the perceived value of the yield token. The more people think that the yield token will be worth more, the more they will buy the yield token and vice versa. This keeps happening until the maturity of the principal token contract where the yield token will be worth 0 because at that point, everyone knows how much yield the YT has produced so there is no more uncertainty to gamble on. The code will prove how this works when you go through it but this is a high level explanation of how the yield token works.
        */
        uint256 assets,
        address ptReceiver,
        address ytReceiver
    ) public override nonReentrant returns (uint256 shares) {
        address _ibt = ibt;
        address _underlying = underlying_;
        IERC20(_underlying).safeTransferFrom(msg.sender, address(this), assets);
        IERC20(_underlying).safeIncreaseAllowance(_ibt, assets);
        uint256 ibts = IERC4626(_ibt).deposit(assets, address(this));
        shares = _depositIBT(ibts, ptReceiver, ytReceiver);
    }

```



# 5  ALWAYS USE NUMERICAL EXAMPLES WITH MATH FORMULAE. NUMERATOR/DENOMINATOR REPRESENTATION 

With the code snippet below, i want to introduce 2 concepts. The first is pretty straightforward and it has to do with interpreting division. What i mean is that if you have A/B , the result is always what 1 B is worth in terms of A. The example can be seen in the code block below where in the comments, i note that  _oldPTTRate/_oldIBTRate gives PTs in terms of IBT tokens. So based on our calculations, 1 IBT is worth 0.95238 PT tokens. So it is not how much 1 PT is worth in terms of IBT. It is always how much 1 denominator is worth in terms of numerator. This is an easy concept to get confused so i thought it would be easier if I cleared it up here once and for all.

The second concept is arguably the more important one and it has to do with whenever you see a formula in any codebase you are looking at. As usual, assumption analysis is used to zoom in before we zoom out. The zoom out concept was introduced in the velvet v4 audit notes as a step further toward unique bugs.

Anyway, when performing assumption analysis on any formula or equation, the best way to do it is to use numerical examples and then explain the formula using those numerical examples. This will actually make the formula look more real to you and easier to understand when you are explaining it. You must do this with every formula no matter how "easy" it looks on first glance. Write out the numerical example and then explain it out like you see in the code block below. You will be surprised as to how much stuff can get uncovered by doing this. Once you fully understand the formula, then you can zoom out and see what comes from that


```solidity
function computeYield(
        address _user,
        uint256 _currentYieldInIBT,
        uint256 _oldIBTRate,
        uint256 _ibtRate,
        uint256 _oldPTRate,
        uint256 _ptRate,
        address _yt
    ) external view returns (uint256 updatedYieldInIBT) {
        if (_oldPTRate == _ptRate && _ibtRate == _oldIBTRate) {
            return _currentYieldInIBT;
        } //c so this if block is checking if the pt rate and the ibt rate have not changed since the last yield update. If they have not changed, then the yield is the same as the current yield in IBT. This is because the ibt rate gives use the current exchange rate of ibt tokens to underlying assets so if the exchange rate has not changed, then the yield is the same as the current yield in IBT. if the exchange rate has increased, then it means that some yield has acrued and need to be accounted for but we will look at this below.
        uint8 ytDecimals = IERC20Metadata(_yt).decimals();
        uint256 newYieldInIBTRay;
        uint256 userYTBalanceInRay = IYieldToken(_yt).actualBalanceOf(_user).toRay(ytDecimals);
        // ibtOfPT is the yield generated by each PT corresponding to the YTs that the user holds
        uint256 ibtOfPTInRay = userYTBalanceInRay.mulDiv(_oldPTRate, _oldIBTRate); /*c as usual, lets visualize this calculation with real math 

        Let’s say:

        _oldPTRate = 1e27 (which is equivalent to 1.0 in Ray)

        _oldIBTRate = 1.05e27 (meaning 1 IBT is worth 1.05 in Ray terms)

        userYTBalanceInRay = 0.1e27 (user holds 0.1 units of YT, in Ray)

        Then:

        ibtOfPTInRay=0.1e27× 1e27/1.05e27 =0.1e27×0.95238 ≈0.095238e27

        so _oldPTTRate/_oldIBTRate gives PTs in terms of IBT tokens. So based on our calculations, 1 IBT is worth 0.95238 PT tokens. This is a very important concept to understand. If you ever have a/b , the result is always what 1 unit of b is worth in terms of a and not the other way around. This is a very important concept to understand as you have gotten this wrong in the past. 

        So we have how much 1 IBT is worth in terms of PT tokens. Multiplying this by the user's YT balance in Ray gives us how much the user's YT balance is worth in terms of PT tokens and this makes sense because as I stated above, when a user deposits, they are minted the same amount of shares in PT tokens as they are in YT tokens. So if 1 IBT is worth 0.95238 PT tokens, then 1 YT token should be worth exactly the same in PT. 

        */

```

It is extremely important to note here how in this example, i used all the decimals associated to make the calculations. This is important when doing numerical examples. So in the example, i didnt have _oldPTRate = 1. I included the e27 to represent the decimals. You must always do this. If you are writing an example with x = 100 * 2 = 200 for example, this is wrong because solidity sees that 100 as 100e18*2e18 = 200e36 so you see these values are not the same and your examples MUST reflect this. If it doesnt, it can lead you to a lot of false positives



# 6 VARIABLES DEFINED WITH CALLDATA LOCATION 

TALK ABOUT HOW CALLDATA WORKS AND HOW IT CANNOT BE DECLARED INSIDE A FUNCTION BODY. IT CAN ONLY BE USED FOR FUNCTION PARAMETERS AND RETURN DATA

In Dispatcher::_dispatch, there is the following code block:

```solidity
else if (command == Commands.FLASH_LOAN) {
            (address lender, address token, uint256 amount, bytes memory data) = abi.decode(
                _inputs,
                (address, address, uint256, bytes)
            );
            if (!IRegistry(registry).isRegisteredPT(lender)) {
                revert InvalidFlashloanLender(lender);
            }
            flashloanLender = lender;
            IERC3156FlashLender(lender).flashLoan(
                IERC3156FlashBorrower(address(this)),
                token,
                amount,
                data
            );
            flashloanLender = address(0);

             /**
```

The key line to pay attention to is how when the inputs are decoded with abi.decode, the data variable is decoded to bytes memory. This data variable is then passed to the flashLoan function. Now this raised a question because in IERC3156FlashLender, this is how the function is defined:

```solidity
     * @dev Initiate a flash loan.
     * @param receiver The receiver of the tokens in the loan, and the receiver of the callback.
     * @param token The loan currency.
     * @param amount The amount of tokens lent.
     * @param data Arbitrary data structure, intended to contain user-defined parameters.
     */
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool);
```

Notice how the data parameter here is declared as bytes calldata but in the _dispatch function, a bytes memory is passed. This had me questioning if this would work as it seems strange. So i went into remix and had a sample contract with the following function:

```solidity

  function testcalldata() public returns (bytes memory) {
        bytes memory data = abi.encode(1);
        bytes calldata data1 = abi.decode(data,(bytes));
    }
```

This ended up giving me the following compiler error: TypeError: Type bytes memory is not implicitly convertible to expected type bytes calldata. So i was wondering why does this error not show up in Dispatcher::_dispatch ? After research, this is what i found and think is worth noting.

Direct Assignment vs Function Parameter:
In your test case, you're trying to do a direct assignment: bytes calldata data1 = ...

In the flash loan case, we're passing a parameter to a function
These are different scenarios with different rules
Why Your Test Case Fails:
calldata is a special location that can only be used for:
Function parameters
Return data from external calls
You cannot declare a calldata variable in a function body
You cannot assign to a calldata variable
This is why you get the error: you're trying to do something that's fundamentally not allowed in Solidity.

So the reason the data variable in Dispatcher::_dispatch is decoded to bytes memory is because solidity doesnt allowing declaring bytes calldata inside a function body. You can research why this is later and add to these notes but this is the reason it is decoded to bytes memory. When it is passed to the flashloan function, solidity handles the conversion of the data variable from bytes memory to bytes calldata. 

Calldata is a location in the EVM seperate from memory, storage and the stack. It is mainly used to store read-only data like parameters of functions or incoming data to a contract. So in our example, where we had:

```solidity
function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool);
```

The receiver, token, amount and data are all initially stored in the calldata region of the EVM. This data cannot be changed as all data in calldata is read-only. So if we changed any of the parameters like say we had:

```solidity
function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool){

        amount = 1e18;
    }
```

In the calldata, amount is still going to be whatever was passed when the function was called but another variable called amount will then be created which will store 1e18 in the stack. So solidity accesses the calldata region using the CALLDATALOAD opocde which we have looked at extensively in the assemblyandfv course.

To see this, take the following sample contract to remix:

```solidity
pragma solidity 0.8.26;

contract Tests {
        
    function testcalldata(uint256 x) public  {
       x = 1e18;
    }
        }

```

Compile and copy the bytecode. Take the bytecode to the evm playground, take out the contract creation code and run only the runtime code with the appropraite selector in the calldata and you will see 1e18(0xde0b6b3a7640000) pushed on the stack. It doesnt go into memory.


# 7 ONLYINITIALISING BUG THAT A LOT OF PROTOCOLS IGNORE

When a contract is meant to be upgradeable, as you know, the contract usually uses an init function with the initializer modifier but there are issues that can come with that which i cover beow so have a look.

```solidity

    function __Dispatcher_init(address _routerUtil, address _kyberRouter) internal initializer { /* //bug if multiple contracts inherit from the dispatcher, since onlyinitializing is not used, it means that the dispatcher can only be initialized once. This is a problem because if a child contract wants to initialize the dispatcher along with other initializations, in the same function, this wont work. let me explain why. This is how the initializer is defined in openzeppelin-contracts-upgradeable:

        modifier initializer() {
        // solhint-disable-next-line var-name-mixedcase
        InitializableStorage storage $ = _getInitializableStorage();

        // Cache values to avoid duplicated sloads
        bool isTopLevelCall = !$._initializing;
        uint64 initialized = $._initialized;

        // Allowed calls:
        // - initialSetup: the contract is not in the initializing state and no previous version was
        //                 initialized
        // - construction: the contract is initialized at version 1 (no reininitialization) and the
        //                 current contract is just being deployed
        bool initialSetup = initialized == 0 && isTopLevelCall;
        bool construction = initialized == 1 && address(this).code.length == 0;

        if (!initialSetup && !construction) {
            revert InvalidInitialization();
        }
        $._initialized = 1;
        if (isTopLevelCall) {
            $._initializing = true;
        }
        _;
        if (isTopLevelCall) {
            $._initializing = false;
            emit Initialized(1);
        }
    }

    Notice how the function sets _initialized to 1. Once this is done, the contract cannot be initialized again. So if you had:

    contract ChildA is Dispatcher, Initializable {
    uint256 public childAValue;

    // ChildA calls the parent's initialization as part of its own initializer.
    function initialize(uint256 _d, uint256 _a) public initializer {
        __Dispatcher_init(_d); 
        __someothercontract__init(); 
        childAValue = _a;
    } 
    
    since the dispatcher has already set _initialized to 1, __someothercontract__init() will revert which is why we need to use the onlyInitializing modifier which will allow all the initializations to be done in the same function.

        */
```


So if you had the following contract:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {Dispatcher} from "src/router/Dispatcher.sol";
import {AccessManagedUpgradeable} from "lib/openzeppelin-contracts-upgradeable/contracts/access/manager/AccessManagedUpgradeable.sol";

contract TestContract is AccessManagedUpgradeable, Dispatcher {
    constructor(address _registry) Dispatcher(_registry) {}
    
    function initialize(address _routerUtil, address _kyberRouter) external initializer {
        __AccessManaged_init(msg.sender);
        __Dispatcher_init(_routerUtil, _kyberRouter);
       
    }
}
```
This would always revert with InvalidInitialization(). See the vulnerability below as a perfect example.

 Vulnerability Details
Dispatcher::__Dispatcher_init is a base contract with the intended aim of being called in any inheriting contract that is to be deployed.

```solidity
 function __Dispatcher_init(address _routerUtil, address _kyberRouter) internal initializer { 
        if (_routerUtil == address(0)) {
            revert AddressError();
        }
        routerUtil = _routerUtil;
        kyberRouter = _kyberRouter;
    }

```

The issue lies in the use of the initializer modifier. If the contract inheriting Dispatcher.sol is an upgradeable contract, then it will also contain its own init function typically nesting init functions of other contracts it inherits from. In a situation like this, the typical behaviour is to use an onlyInitializing modifier for all nested init functions. This is because if a child contract wants to initialize the dispatcher along with other initializations, in the same function, this wont work. This is how the initializer is defined in openzeppelin's Initializable.sol:

```solidity
        modifier initializer() {
        // solhint-disable-next-line var-name-mixedcase
        InitializableStorage storage $ = _getInitializableStorage();

        // Cache values to avoid duplicated sloads
        bool isTopLevelCall = !$._initializing;
        uint64 initialized = $._initialized;

        // Allowed calls:
        // - initialSetup: the contract is not in the initializing state and no previous version was
        //                 initialized
        // - construction: the contract is initialized at version 1 (no reininitialization) and the
        //                 current contract is just being deployed
        bool initialSetup = initialized == 0 && isTopLevelCall;
        bool construction = initialized == 1 && address(this).code.length == 0;

        if (!initialSetup && !construction) {
            revert InvalidInitialization();
        }
        $._initialized = 1;
        if (isTopLevelCall) {
            $._initializing = true;
        }
        _;
        if (isTopLevelCall) {
            $._initializing = false;
            emit Initialized(1);
        }
    }

```

Notice how the function sets _initialized to 1. Once this is done, the contract cannot be initialized again. So if you had:

```solidity
    contract ChildA is Dispatcher, Initializable {
    uint256 public childAValue;

    // ChildA calls the parent's initialization as part of its own initializer.
    function initialize(uint256 _d, uint256 _a) public initializer {
        __Dispatcher_init(_d); 
        __someothercontract__init(); 
        childAValue = _a;
    } }
```
    
Since the dispatcher has already set _initialized to 1, __someothercontract__init() will revert with an InvalidInitialisation custom error. This is why we need to use the onlyInitializing modifier in Dispatcher::__Dispatcher_init which will allow all the initializations from the child contract to run in the same function allowing for proper composability.

Although there are workarounds for this like calling the init function inside the constructor of the deployed contract, using different methods could lead to errors when other contracts need to be initialised which contain different logic which could lead to mistakes down the line so the most preferred method is to use the onlyInitializing modifier for parent contracts.



# 8 BINARY SEARCH ALGORITHM IMPLEMENTATION

I thought this part of the spectra was code was extremely interesting as I did a course on algorithm and specialisation where binary search algorithm was covered and this is the first time I have seen it in actual code so i thought I would assess it and show how it works making references to the notes I made on that algorithm so make sure to check this out and the comments I made below as it is really insightful.

```solidity
 /**
     * @param _curvePool : PT/IBT curve pool
     * @param _i token index
     * @param _j token index
     * @param _targetDy amount out desired
     * @return dx The amount of token to provide in order to obtain _targetDy after swap
     */
    function getDx(/*c so this function aims to find out how much of token x is needed to get _targetDy amount of token y. in our case, since coins[0] is ibt and coins[1] is pt, this function aims to find out how much of ibt is needed to get _targetDy amount of pt. so ibt is dx and pt is dy. This isnt a fixed value but just for understanding sake, we will keep these assumptions The way i know this is because get_dy in the curve pool implementation looks like this:
        
        def get_dy(i: int128, j: int128, dx: uint256) -> uint256:
    """
    @notice Calculate the current output dy given input dx
    @dev Index values can be found via the `coins` public getter method
    @param i Index value for the coin to send
    @param j Index value of the coin to receive
    @param dx Amount of `i` being exchanged
    @return Amount of `j` predicted
    """
    return StableSwapViews(factory.views_implementation()).get_dy(i, j, dx, self)

    so as you can see from the parameters, i can be the index of any token we want to send and j can be the index of any token we want to receive. so in our case, i is 0 and j is 1. so for this example, we assumethis function is trying to find out how much of ibt is needed to get _targetDy amount of pt. 

        
        The idea is based around performing a binary search which we are going to go over now. If you know what a binary search is, then you will understand why we start from _targetDy as our initial guess and why we need an initial guess. Luckily, we have done an algorithm course so if you go into your files , in the algo and specialization pt 2 docs, all you need to know about binary search is there so go and have a read of that before you come back here as that is crucial for you to understand what is going on here. */
        address _curvePool,
        uint256 _i,
        uint256 _j,
        uint256 _targetDy
    ) external view returns (uint256 dx) {
        // Initial guesses
        uint256 _minGuess = type(uint256).max; 
        uint256 _maxGuess = type(uint256).max;
        uint256 _factor100; //c this is defined in _runLoop as search interval scaling factor
        uint256 _guess = ICurvePool(_curvePool).get_dy(_i, _j, _targetDy); //c So what is happeneing here is that we are assuming that _targetDy(amount of pt we want out) is the amount of ibt we want to swap to get pt. The function returns the amount of pt we will get out for a _targetDy amount of ibt.

        if (_guess > _targetDy) {
            _maxGuess = _targetDy; //c so this is saying that if the amount of pt we get out for a _targetDy amount of ibt (_guess) is greater than the amount of pt we want out, then we can pretty much rule out the initial guess as dx because we know that the amount of pt we get out is greater than the amount of pt we want out. As a result, we know that the maximum value of ibt (dx) we need to get _targetDy amount of pt cannot be more than _targetDy which is why we set _maxGuess to _targetDy. so if this if block hits, then our array for our binary search will be [type(uint256).max, _targetDy] which doesnt make much sense for now but these arrays are reset in the _runLoop function so dont worry too much about it right now
            _factor100 = 10; /*c the reason why the factor is 10 when _maxGuess is set to _targetDy and is set to 1000 when _minGuess is set to _targetDy is because if we already know that the amount of pt we get out for a _targetDy amount of ibt (_guess) is greater than the amount of pt we want out, then we know that targetDy is too high to be dx so in the _runLoop function, there is the following line:

             if (_minGuess == type(uint256).max || _maxGuess == type(uint256).max) {
            _guess = (_guess * _factor100) / 100; 

            so if _maxGuess is set to _targetDy, then the factor will be 10 because we want our new guess to be less than _targetDy. So we get 10% of _targetDy and use that as our new guess and it makes sense because we know we have to scale down the guess to get the correct dx. Not sure if 10% is the ideal scaling amount but whatever

            when _minGuess is set to _targetDy, then the factor will be 1000 because we want our new guess to be greater than _targetDy. So we get 1000% of _targetDy and use that as our new guess and it makes sense because we know we have to scale up the guess to get the correct dx. Not sure if 1000% is the ideal scaling amount but whatever

            */
        } else {
            _minGuess = _targetDy; //c analogous reasoning as above. since the amount of pt we get out for a _targetDy amount of ibt (_guess) is less than the amount of pt we want out, then we know that the minimum value of ibt (dx) we need to get _targetDy amount of pt cannot be less than _targetDy which is why we set _minGuess to _targetDy. so if this if block hits, then our array for our binary search will be [_targetDy, type(uint256).max]
            _factor100 = 1000; //q a possible question is what if for example, _maxGuess is _targetDy so we know we have to scale down the guess but scaling doen by 10% is not enough and dx is say 5% of _targetDy. In reality that is probably not likely because a user creating a pool will probably not want the discrepancy between the pt and ibt to be that high i.e. they probably dont want to create a pool where 1 ibt is worth 0.05 pt. i mean it can be reported as an issue but it is just unlikely to happen
        } 
        uint256 loops;
        _guess = _targetDy; //c so the initial guess or middle value for the binary value gets reset to _targetDy
        while (!_dxSolved(_curvePool, _i, _j, _guess, _targetDy, _minGuess, _maxGuess)) {
            loops++;

            (_minGuess, _maxGuess, _guess) = _runLoop( //c so this is where the binary search is happening. we are passing in the array for our binary search and the function is returning the lower bound, upper bound and the guess for the next iteration. so with reference to our binary search, _minGuess is the lower bound, _maxGuess is the upper bound and _guess is the middle value. again go look at the notes in the algo and specialization pt 2 docs for more info on binary search to get what this means.
                _minGuess,
                _maxGuess,
                _factor100,
                _guess,
                _targetDy,
                _curvePool,
                _i,
                _j
            );

            if (loops >= MAX_ITERATIONS_BINSEARCH) {
                revert SolutionNotFound();
            } //c so these iterations are happening until the guess converges to the correct dx. Using the binary search algorithm, eventually, it will converge to the correct dx or revert if the target doesnt get found after MAX_ITERATIONS_BINSEARCH(255) iterations
        }
        dx = _guess;
    }

    /**
     * @dev Runs bisection search
     * @param _minGuess lower bound on searched value
     * @param _maxGuess upper bound on searched value
     * @param _factor100 search interval scaling factor
     * @param _guess The previous guess for the `dx` value that is being refined through the search process
     * @param _targetDy The target output of the `get_dy` function, which the search aims to achieve by adjusting `dx`.
     * @param _curvePool PT/IBT curve pool
     * @param _i token index, either 0 or 1
     * @param _j token index, either 0 or 1, must be different than _i
     * @return The lower bound on _guess, upper bound on _guess and next _guess
     */
    function _runLoop(
        uint256 _minGuess,
        uint256 _maxGuess,
        uint256 _factor100,
        uint256 _guess,
        uint256 _targetDy,
        address _curvePool,
        uint256 _i,
        uint256 _j
    ) internal view returns (uint256, uint256, uint256) {
        if (_minGuess == type(uint256).max || _maxGuess == type(uint256).max) {
            _guess = (_guess * _factor100) / 100; //c so this if block is checking if _minGuess or _maxGuess is type(uint256).max. which it is on the first run so we know that the initial guess is incorrect because the while loop condition which calls _dxSolved returns false if _minGuess or _maxGuess is type(uint256).max. so we need to scale the guess by a factor of 100 which is defined as 10 if guess > targetDy and 1000 if guess < targetDy in the getDx function above. I still dont know why the factors are different but I will figure that out 
        } else {
            _guess = (_maxGuess + _minGuess) >> 1; //c so if _minGuess and _maxGuess arent type(uint256).max, then we can set the guess tto the correct middle value required for a binary search. Again, go look at the notes in the algo and specialization pt 2 docs for more info on binary search to get what this means.
        }
        uint256 dy = ICurvePool(_curvePool).get_dy(_i, _j, _guess); //c so this then checks if our new guess is the correct dx and it does that by calling get_dy with the guess which will give us the amount of pt we get out for a _guess amount of ibt. The if blocks below then check if the guess is too high or too low and if too low, then we know that the new lower bound is the guess and if too high, then we know that the new upper bound is the guess.
        if (dy < _targetDy) {
            _minGuess = _guess;
        } else if (dy > _targetDy) {
            _maxGuess = _guess;
        }
        return (_minGuess, _maxGuess, _guess);
    }

    /**
     * @dev Returns true if algorithm converged
     * @param _curvePool PT/IBT curve pool
     * @param _i token index, either 0 or 1
     * @param _j token index, either 0 or 1, must be different than _i
     * @param _dx The current guess for the `dx` value that is being refined through the search process.
     * @param _targetDy The target output of the `get_dy` function, which the search aims to achieve by adjusting `dx`.
     * @param _minGuess lower bound on searched value
     * @param _maxGuess upper bound on searched value
     * @return true if the solution to the search problem was found, false otherwise
     */
    function _dxSolved(
        address _curvePool,
        uint256 _i,
        uint256 _j,
        uint256 _dx,
        uint256 _targetDy,
        uint256 _minGuess,
        uint256 _maxGuess
    ) internal view returns (bool) {
        if (_minGuess == type(uint256).max || _maxGuess == type(uint256).max) {
            return false; //c so when this is called in getdx in a while loop for the first run, this false will always hit as either _minGuess or _maxGuess will be type(uint256).max on the first run
        }
        uint256 dy = ICurvePool(_curvePool).get_dy(_i, _j, _dx);  //c if the first if block doesnt hit, then it is signalling that we need to check if the guess generated is correct and it does that by calling get_dy with the guess and checking if it is equal to _targetDy. if it is, then we have found the correct dx and we return true.
        if (dy == _targetDy) {
            return true;
        }
        uint256 dy1 = ICurvePool(_curvePool).get_dy(_i, _j, _dx + 1); //c so this is giving a bit of delta to the guess and what i mean is that if the guess (dx) doesnt hit the target dy, then we can check if the guess + 1 does. then if the targetDy is between the two, then we assume that the guess is correct and we return true. So it gives us a bit of wiggle room to work with. I would argue that this wiggle room is too small. It is currently set to 1 wei which is too small. I would suggest increasing this amount slightly more even if means the user loses an insignificant amount of wei as it will make the binary search run with less iterations.
        if (dy < _targetDy && _targetDy < dy1) {
            return true;
        }
        return false;
    }

    /**
     * @notice Returns the balances of the two tokens in provided curve pool
     * @param _curvePool address of the curve pool
     * @return The IBT and PT balances of the curve pool
     */
    function _getCurvePoolBalances(address _curvePool) internal view returns (uint256, uint256) {
        return (ICurvePool(_curvePool).balances(0), ICurvePool(_curvePool).balances(1)); //c looking at the balances function in the curve pools which you can view at https://etherscan.io/address/0xDCc91f930b42619377C200BA05b7513f2958b202#code
    }
```

# 9 FLASH LOAN







