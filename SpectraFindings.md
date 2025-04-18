# 1 Dispatcher.sol using initializer modifier as a base contract can lead to initialization issues


Description
Brief/Intro
The Dispatcher contract in the Spectra protocol incorrectly uses the initializer modifier in its __Dispatcher_init function. This design prevents proper multi-contract initialization flows in upgradeable contracts that inherit from Dispatcher. As a result, any contract that tries to initialize Dispatcher alongside other upgradeable modules will fail or revert. This breaks composability, makes inheritance fragile, and increases the risk of uninitialized variables — potentially leading to misconfigured contracts or disabled protocol functionality.

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

Impact Details
If left unaddressed, this bug can result in:

Inoperable or uninitialized contracts: Contracts inheriting Dispatcher will revert on deployment if they attempt to compose __Dispatcher_init() with other initializers. This blocks correct protocol configuration and renders contracts unusable.

Protocol fragmentation: To bypass this bug, teams may initialize Dispatcher in constructors or in separate functions, leading to inconsistent patterns across contracts. This increases the risk of human error, especially in multi-contract deployments.

Loss of upgrade safety and maintainability: Using initializer in shared base contracts goes against established upgradeable contract patterns (e.g., OpenZeppelin’s). It breaks composability and prevents safe reuse in upgradeable contract trees.

If a protocol mistakenly inherits and composes this base without modifying the initializer to onlyInitializing, their upgrade paths could be broken, or initialization could be skipped entirely, leading to loss of control over critical contract parameters.

Proof of Concept
Proof of Concept
To show the initialization issue, write the following contract:
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

Include the following test in RouterTest.t.sol:

```solidity
 function testDispatcherInit() public {
        testContract = new TestContract(address(registry));
        address kyberRouter = router.getKyberRouter();
       vm.expectRevert(Initializable.InvalidInitialization.selector);
        testContract.initialize(address(routerUtil), kyberRouter);
        
    }
```

# 2 Dispatcher incorrect validation causes principal tokens to be stuck in inheriting contract allowing attacker to steal user funds

Brief/Intro
The Dispatcher::_dispatch function in Spectra is responsible for routing user commands to various protocol operations, such as transfers, redemptions, and pool interactions. A bug in the REDEEM_PT_FOR_ASSET command causes the dispatcher to incorrectly prevent PT redemptions if the contract lacks a balance of YT — even after maturity, when YT is no longer required. This causes user funds to become stuck or misrouted, leading to potential loss or theft.

Vulnerability Details
Dispatcher::_dispatch is a command that allows any user to execute certain commands that relate to token transfers, pool creation and interactions with pt, ibt and yt tokens. Contracts can inherit the functionality of the dispatcher to allow function routing to allowed contracts.

The vulnerability in Dispatcher::_dispatch is in the following code block:
```solidity
else if (command == Commands.REDEEM_PT_FOR_ASSET) {
            (address pt, uint256 shares, address recipient, uint256 minAssets) = abi.decode(
                _inputs,
                (address, uint256, address, uint256)
            ); 
            shares = _resolveTokenValue(pt, shares);
            shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this))); 
            recipient = _resolveAddress(recipient);
            IPrincipalToken(pt).redeem(shares, recipient, address(this), minAssets);
PrincipalToken::redeem works as follows:

 /** @dev See {IPrincipalToken-redeem}. */
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public override nonReentrant returns (uint256 assets) {
        _beforeRedeem(shares, owner);
        emit Redeem(owner, receiver, shares);
        assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));
    }

    /**
     * @dev Internal function for preparing redeem and burning PT:YT shares.
     * @param _shares The amount of shares being redeemed
     * @param _owner The address of shares' owner
     */
    function _beforeRedeem(uint256 _shares, address _owner) internal {
        if (_owner != msg.sender) {
            _spendAllowance(_owner, msg.sender, _shares);
        }
        if (_shares > _maxBurnable(_owner)) {
            revert InsufficientBalance();
        }
        if (block.timestamp >= expiry) {
            if (ratesAtExpiryStored == RAE_NOT_STORED) {
                storeRatesAtExpiry();
            }
        } else {
            updateYield(_owner);
            IYieldToken(yt).burnWithoutYieldUpdate(_owner, msg.sender, _shares);
        }
        _burn(_owner, _shares);
    }
```
_beforeRedeem performs some checks before allowing a user redeem assets for pt. These checks include checking if the principal token is at or past maturity, in this case, it only updates the rates at expiry. If maturity hasnt been reached, then the user is required to burn an equivalent amount of yt tokens to redeem their assets. This is consistent with the natspec at the start of the principal token which says:

" The shares of the vaults are composed by PT/YT pairs. These are always minted at same times and amounts upon deposits.Until expiry burning shares necessitates to burn both tokens. At expiry, burning PTs is sufficient. This check is making sure that the contract has enough yt to redeem the shares which is not necessary for pt's redemption at expiry" .

Dispatcher::_dispatch in the code block above, does not check if the principal token is at maturity or not and simply allocates shares based on how much yt the user has sent to the contract. As a result, users who have sold/transferred their yt's will not be able to redeem pt via the dispatcher as it will always calculate the amount of shares to redeem as 0. Any user who attempts to redeem PT for asset via the dispatcher will have their principal tokens stuck in the contract which allows an attacker to steal the user's principal tokens by calling the transfer command via the dispatcher.

Impact Details
This vulnerability creates a situation where any user attempting to redeem PT for assets via the dispatcher will be blocked from redeeming if they no longer hold the corresponding YT tokens — even if the PT has reached maturity. This occurs because the dispatcher limits the redeemable share amount to the contract's balance of YT, which is only necessary pre-maturity. After maturity, the PT can be redeemed independently.

As a result, users can mistakenly transfer their PTs to the dispatcher and trigger a redeem operation, expecting to receive underlying assets. Instead, the dispatcher will attempt to redeem 0 shares (based on the YT balance check), and the user will receive nothing, while their PTs remain stuck in the dispatcher.

Consequences: Funds Stuck (Denial of Redemption): Users' PT tokens become irretrievable if they mistakenly redeem via the dispatcher post-maturity, leading to permanent loss of access to their principal assets.

Theft via Malicious Redemption: An attacker can monitor for such stuck PTs and, by using a subsequent dispatcher TRANSFER_ERC20 command, redirect those PTs to themselves or front-run users to steal trapped PTs.

Loss of User Funds: This could result in significant monetary loss, especially for high-value deposits at maturity.


Proof of Concept
This test was run in Router.t.sol using the router as the child contract for the dispatcher to display the vulnerability
```solidity
function testRedeemPTForAsset() public { //c for testing purposes
        uint256 amount = 3e18;
       

        //c approve principaltoken contract to take ibt tokens
        ibt.approve(address(principalToken), amount);

        //c deposit ibt into pt contract
        principalToken.depositIBT(amount, testUser);

        //c get maturity 
        uint256 maturity = principalToken.maturity();
        //c wait for maturity
        vm.warp(maturity + 100);
        vm.roll(block.number + 1);

        //c give allowance to router to take pt tokens
        principalToken.approve(address(router), amount);
        
        //c attempt to use router to redeem pt for asset
        bytes memory commands = abi.encodePacked(
            bytes1(uint8(Commands.TRANSFER_FROM)),
            bytes1(uint8(Commands.REDEEM_PT_FOR_ASSET))
        );
        bytes[] memory inputs = new bytes[](2);
        inputs[0] = abi.encode(address(principalToken), amount);
        inputs[1] = abi.encode(address(principalToken), amount, testUser, 0);
        

        uint256 underlyingBalanceOfUserBefore = underlying.balanceOf(testUser);
        uint256 underlyingBalanceOfRouterBefore = underlying.balanceOf(address(router));
        uint256 underlyingBalanceOfOtherBefore = underlying.balanceOf(other);
        uint256 ptBalanceOfUserBefore = principalToken.balanceOf(testUser);
        uint256 ptBalanceOfRouterBefore = principalToken.balanceOf(address(router));
        uint256 ptBalanceOfOtherBefore = principalToken.balanceOf(other);

        router.execute(commands, inputs);

        /*assertEq(
            underlying.balanceOf(testUser),
            underlyingBalanceOfUserBefore,
            "User's underlying balance after router execution is wrong"
        );
        assertEq(
            underlying.balanceOf(address(router)),
            underlyingBalanceOfRouterBefore,
            "Router's underlying balance after router execution is wrong"
        );
        assertEq(
            principalToken.balanceOf(testUser),
            ptBalanceOfUserBefore,
            "User's PT balance after router execution is wrong"
        );
        assertEq(
            principalToken.balanceOf(address(router)),
            ptBalanceOfRouterBefore,
            "Router's PT balance after router execution is wrong"
        ); */

        //c with the pt's now stuck in the router, anyone can call transfer and get the pt's
        address attacker = vm.addr(100);
        vm.startPrank(attacker);
        bytes[] memory inputs1 = new bytes[](1);
        bytes memory commands1 = abi.encodePacked(
            bytes1(uint8(Commands.TRANSFER))
        );
        inputs1[0] = abi.encode(address(principalToken), attacker, amount);
        router.execute(commands1, inputs1);
        vm.stopPrank();

        assertEq(principalToken.balanceOf(attacker), amount);
            
    }
```
To see, that 0 underlying asset were transferred to testUser and the user's principal tokens are still in the router contract, uncomment the assert statements

# 3 Harcoded implementation id can lead to pool deployment DOS

Brief/Intro 

FactorySNG in the Spectra protocol uses a hardcoded implementation ID of 0 to deploy Curve StableSwap NG pools. Curve allows implementation contracts to be upgraded or replaced at any index. If the implementation at index 0 is updated by Curve (as is already standard practice), the Spectra deployment logic will unknowingly use a new, possibly incompatible contract — introducing the risk of misconfigured pools, broken oracle integration, or unexpected logic execution. This could cause pool misbehavior or denial of service (DoS) for all future Spectra deployments using Curve StableSwap NG.

Vulnerability Details 

FactorySNG has the following code:

uint256 constant IMPLEMENTATION_ID = 0; This is used to deploy the plain curve stableswap NG pool as follows:

curvePoolAddr = IStableSwapNGFactory(curveFactory).deploy_plain_pool( "Spectra-PT/IBT", "SPT-PT/IBT", _coins, _p.A, _p.fee, _p.fee_mul, _p.ma_exp_time, IMPLEMENTATION_ID, assetTypes, oracleMethodSigs, oracleAddresses ); } The issue lies in how the implementation id's work in the curve protocol. The curve documentation says " Implementation contracts are upgradable. They can be either replaced or additional implementation contracts can be set. Therefore, please always make sure to check the most recent ones" which can be seen at https://docs.curve.fi/deployments/amm/#stableswap-ng. The fact that 0 is hardcoded here is an issue because it means that the implementation contract at index 0 of the mapping can be replaced and if that happens, the wrong implementation contract will be used which can have adverse effects on the curve pool deployment for spectra.

An example of this can be seen where a bug was found in the stableswap ng pool and the implementation was updated to fix it. the details said "This bug only affects the use of the oracle and does not impact token exchanges or any liquidity actions at all. The AMM still functions as intended. Pools deployed after Dec-12-2023 09:39:35 AM +UTC do not include the bug, as the fixed AMM implementation of the StableSwapNG Factory was set to the updated version." If a situation occured where curve decide to upgrade the implementation contract at index 0 again, they could decide to adapt some variables used in the pool deployment or adapt the way the oracles work to produce the stored rates. Anything could change which is why curve advice users to confirm the implementation they are using.

As a result, spectra needs to make sure that the implementation address at index 0 always matches the address they expect and should allow for the implementation id to be changed via an access controlled function rather than being constant because if the address was moved to index 1 or completely removed and replaced with a contract with different functionality, it could lead to a DOS for curve stableswap NG pool creation.

Impact Details 
Incorrect Pool Deployment If Curve updates implementation 0 with breaking changes or deprecates its behavior, Spectra may deploy new pools with:

Broken oracle integration Unexpected math assumptions Incompatible asset types Malfunctioning or unstable liquidity behavior

Denial of Service (DoS) If implementation 0 is removed, disabled, or made incompatible, future pool creation will fail, resulting in a permanent DoS for any attempt to onboard new assets through Curve pools.

Security Regression Even if the replacement is functional, new logic at index 0 may bypass important validations or safety checks Spectra assumes to be present (e.g., precision normalization, stored rate format), potentially exposing the protocol to slippage manipulation, or price oracle errors.

Loss of Predictability and Upgrade Safety By not enforcing a dynamic or verifiable implementation ID, Spectra loses any ability to adapt to new Curve versions — and could unknowingly expose users to changes in underlying AMM mechanics without warning.

Proof of Concept
Spectra’s FactorySNG contract hardcodes IMPLEMENTATION_ID = 0 when deploying Curve StableSwap NG pools. However, Curve’s factory explicitly allows upgrading or replacing implementations at any ID — including 0. In fact, Curve already replaced the contract at index 0 in December 2023 to patch a known bug.

If Curve updates index 0 again (e.g., changes how oracle rates are calculated, modifies internal precision logic, or disables it entirely), Spectra would unknowingly deploy pools with new, potentially incompatible logic. This introduces risk of:

Incorrect pool deployment Oracle desync or mispricing Permanent DoS to future pool creation Unexpected behavior with no way to recover

Because the ID is not configurable, and Spectra doesn’t verify the implementation used, this is a fragile, externally-controlled single point of failure.

Fix: Make the implementation ID configurable and validate its address before deployment.


# 4 Low enough initialPrice variable leads to 0 pt balance and _addInitialLiquidity DOS


Vulnerability Details
FactorySNG::_addInitialLiquidity allows users who have created a pool to add the initial liquidity into the pool. The issue occurs in the following code block in the function

```solidity
        // using fictive pool balances, the user is adding liquidity in a ratio that (closely) matches the empty pool's initial price
        // with ptBalance = IBT_UNIT for having a fictive PT balance reference, ibtBalance = IBT_UNIT x initialPrice
        uint256 ptBalance = 10 ** IERC20Metadata(ibt).decimals();
        uint256 ibtBalance = ptBalance.mulDiv(_initialPrice, CurvePoolUtil.CURVE_UNIT);
        // compute the worth of the fictive IBT balance in the pool in PT
        uint256 ibtBalanceInPT = IPrincipalToken(pt).previewDepositIBT(ibtBalance);
        // compute the portion of IBT to deposit in PT
        uint256 ibtsToTokenize = _initialLiquidityInIBT.mulDiv(
            ptBalance,
            ibtBalanceInPT + ptBalance
        );
```

Imagine a scenario where the ibt has 8 decimals and a pool owner wants to set the initialPrice of the pool at 1e7. This would mean:

ptBalance = 1e8 ibtBalance = (1e8 * 1e7)/1e18 == 0 as solidity doesn't accept decimals

As a result of this, the IStableSwapNG(_curvePool).add_liquidity(amounts, 0, msg.sender);
call at the end of the function will not work as the curve stableswap NG pool does not accept depositing 0 amounts as initial deposits which you can see in the add_liquidity function on the curve stableswap ng pool implementation



Proof of Concept
To demonstrate this, i created the following contract:

//SPDX-License-Identifier: MIT
pragma solidity 0.8.20;
```solidity
import {BaseFeedCurvePTIBTSNG} from "src/spectra-oracles/chainlinkFeeds/stableswap-ng/BaseFeedCurvePTIBT.sol";
import {CurveOracleLib} from "src/libraries/CurveOracleLib.sol";
contract TestContract is BaseFeedCurvePTIBTSNG  {

    string public constant description = "IBT/PT Curve Pool Oracle: PT price in asset";
    constructor(address _pt, address _pool) BaseFeedCurvePTIBTSNG(_pt, _pool) {}
}
```

I also added the following function to MockERC20.sol and MockIBT.sol:

```solidity
function decimals() public view virtual override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {
        return 8;
    }   
```

In MockIBT.sol, i also made pricePerFullShare = 1e8 to reflect an ibt with 8 decimals. The following test was run in RateAdjustmentOracle.t.sol:

```solidity
 function testDecimals() public {
        uint256 initialLiquidityIBT = 2e18;

        uint256 initialPrice = 1e7;

        IFactorySNG.CurvePoolParams memory curvePoolParams = IFactorySNG.CurvePoolParams({
            A: 1500,
            fee: 1000000,
            fee_mul: 20000000000,
            ma_exp_time: 600,
            initial_price: initialPrice,
            rate_adjustment_oracle: address(0)
        });

        (address _pt, , address _curvePool) = __deployPoolWithInitialLiquidity(
            initialLiquidityIBT,
            curvePoolParams
        );

        testContract = new TestContract(_pt, _curvePool);
        (, int256 answer, , , ) = testContract.latestRoundData();
        uint256 castAnswer = uint256(answer);
        console.log(castAnswer);
        assertApproxEqAbs(castAnswer, 10 ** 15, 1, "initial pt to ibt rate is wrong");
       
        uint256[] memory stored_rates = IStableSwapNG(_curvePool).stored_rates();
        uint256 decimals = IPrincipalToken(_pt).decimals();
        assertEq(decimals, 8, "decimals are wrong");
    } 
```


# 5 Quote Prices do not reflect initialPrice incorrectly when ibt's with < 18 decimals are used

Brief/Intro
A precision-handling bug in CurveOracleLib::getPTToIBTRateSNG causes incorrect PT-to-IBT exchange rate calculations when the underlying IBT token has a decimal precision other than 18. This discrepancy emerges from how Curve's _stored_rates() and scale_factor are applied during the rate derivation process. If left unpatched, this issue can result in the protocol returning an incorrect principal token (PT) value, which introduces oracle price inconsistencies, potentially leading to mispriced assets

Vulnerability Details
CurveOracleLib::getPTToIBTRateSNG contains the following code:

```solidity
  function getPTToIBTRateSNG(address pool) public view returns (uint256) {
        IPrincipalToken pt = IPrincipalToken(ICurveNGPool(pool).coins(1));
        uint256 maturity = pt.maturity();
        if (maturity <= block.timestamp) {
            return pt.previewRedeemForIBT(pt.getIBTUnit());
        } else {
            uint256[] memory storedRates = IStableSwapNG(pool).stored_rates();
            return
                pt.getIBTUnit().mulDiv(storedRates[1], storedRates[0]).mulDiv(
                    IStableSwapNG(pool).price_oracle(0),
                    CURVE_UNIT
                );
        }
    }
```

The idea is to get how much 1 PT is worth in IBT tokens factoring the PT prices returned from the rate adjustment oracle and the curve internal prices recorded in the pool.

Curve returns the rates from the rate adjustment oracle using the following function:

```solidity
def _stored_rates() -> DynArray[uint256, MAX_COINS]:
    """
    @notice Gets rate multipliers for each coin.
    @dev If the coin has a rate oracle that has been properly initialised,
         this method queries that rate by static-calling an external
         contract.
    """
    rates: DynArray[uint256, MAX_COINS] = rate_multipliers

    for i in range(N_COINS_128, bound=MAX_COINS_128):

        if asset_types[i] == 1 and not rate_oracles[i] == 0:

            # NOTE: fetched_rate is assumed to be 10**18 precision
            oracle_response: Bytes[32] = raw_call(
                convert(rate_oracles[i] % 2**160, address),
                _abi_encode(rate_oracles[i] & ORACLE_BIT_MASK),
                max_outsize=32,
                is_static_call=True,
            )
            assert len(oracle_response) == 32
            fetched_rate: uint256 = convert(oracle_response, uint256)

            rates[i] = unsafe_div(rates[i] * fetched_rate, PRECISION)

        elif asset_types[i] == 3:  # ERC4626

            # fetched_rate: uint256 = ERC4626(coins[i]).convertToAssets(call_amount[i]) * scale_factor[i]
            # here: call_amount has ERC4626 precision, but the returned value is scaled up to 18
            # using scale_factor which is (18 - n) if underlying asset has n decimals.
            rates[i] = unsafe_div(
                rates[i] * ERC4626(coins[i]).convertToAssets(call_amount[i]) * scale_factor[i],
                PRECISION
            )  # 1e18 precision

    return rates
```

To walkthrough an example of a happy path, assume the following values for the ibt tokens at the initialisation of the pool where no time has passed:

decimals = 18 rates[i] = 1e18 (this is gotten from the rate multipliers which are internally calculated by the curve factory contract and is calculated by _rate_multipliers.append(10 ** (36 - decimals[i]) and can be seen in the deploy_plain_pool function) call_amount[i] = 1e18 scale_factor[i] = 1 (calculated via _scale_factor.append(10**(18 - convert(ERC20Detailed(_underlying_asset).decimals(), uint256))) which is initialised in the curve stableswap ng pool implementation) IStableSwapNG(pool).price_oracle(0) = 1e18 initialPrice = 1e15 PT price returned by rate adjustment oracle = 1e15

With these values , the rate returned for the ibt by the curve pool can be calculated as follows:

rates[i] = (1e181e181)/1e18 = 1e18

As a result, getPTToIBTRateSNG at the initialisation of the pool where no time has passed can be calculated as:

((1e18* 1e15)/1e18) * (1e18/1e18) = 1e15 This correctly represents the pt the ibt rate at the pool initialisation as it matches the initial price set by the pool creator.

Now consider a scenario where:

decimals = 8 rates[i] = 1e28 (this is gotten from the rate multipliers which are internally calculated by the curve factory contract and is calculated by _rate_multipliers.append(10 ** (36 - decimals[i]) and can be seen in the deploy_plain_pool function) call_amount[i] = 1e8 (converting 1e8 of ibt to underlying asset which is also set up to have 8 decimals) scale_factor[i] = 1e10 (calculated via _scale_factor.append(10**(18 - convert(ERC20Detailed(_underlying_asset).decimals(), uint256))) which is initialised in the curve stableswap ng pool implementation) IStableSwapNG(pool).price_oracle(0) = 1e18 initialPrice = 1e15 PT price returned by rate adjustment oracle = 1e25 (this is how curve's _stored_rates will calculate the rate based on an 8 decimal token)

With these values , the rate returned for the ibt by the curve pool can be calculated as follows:

rates[i] = (1e281e81e10)/1e18 = 1e28

As a result, getPTToIBTRateSNG at the initialisation of the pool where no time has passed can be calculated as:

((1e8* 1e25)/1e28) * (1e18/1e18) = 1e5 This incorrectly represents the pt the ibt rate at the pool initialisation as it does not match the initial price set by the pool creator which is where the bug is.

As a result, when the pool price is queried, via BaseOracle::latestRoundData : it returns an inaccurate price .

Impact Details
This bug significantly impacts protocols and users who interact with Curve NG pools that integrate interest-bearing tokens (IBTs) with non-18 decimal precision, especially those with 8 decimals. These tokens are common in the Ethereum DeFi ecosystem and include: cWBTC (Compound) – 8 decimals yvWBTC (Yearn) – vaults built on WBTC, which has 8 decimals Beefy vaults or other ERC4626 wrappers on WBTC ERC4626 vaults built on 8-decimal tokens (e.g., staked Bitcoin derivatives or bridged BTC assets)

Because Curve NG’s internal pricing logic and oracle feeds are designed around 18-decimal assumptions, using these 8-decimal IBTs without proper normalization via rate_multipliers and scale_factor causes:

Incorrect rate reporting for PT-to-IBT conversions Inaccurate oracle prices returned via price_oracle() and getPTToIBTRateSNG()

This impacts downstream integrations in protocols relying on Curve oracle data, including: Lending/borrowing protocols (mispriced collateral) Structured products (yield accounting) Automated portfolio managers or vaults

A mispriced PT/IBT rate leads to economic imbalances and opens the door for: Arbitrage opportunities exploiting PTs sold below their intrinsic value Vault accounting errors Broken TWAP-based logic for time-dependent strategies

As Curve NG expands ERC4626 support, many vaults and tokens do not use 18 decimals. This makes the bug highly likely to be encountered, especially for pools using cWBTC, yvWBTC, and similar wrappers, which are commonly deployed on Ethereum mainnet.


Proof of Concept
To demonstrate this, i created the following contract:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {BaseFeedCurvePTIBTSNG} from "src/spectra-oracles/chainlinkFeeds/stableswap-ng/BaseFeedCurvePTIBT.sol";
import {CurveOracleLib} from "src/libraries/CurveOracleLib.sol";
contract TestContract is BaseFeedCurvePTIBTSNG  {

    string public constant description = "IBT/PT Curve Pool Oracle: PT price in asset";
    constructor(address _pt, address _pool) BaseFeedCurvePTIBTSNG(_pt, _pool) {}
}
```

I also added the following function to MockERC20.sol and MockIBT.sol:

```solidity
function decimals() public view virtual override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {
        return 8;
    }   
```

In MockIBT.sol, i also made pricePerFullShare = 1e8 to reflect an ibt with 8 decimals. The following test was run in RateAdjustmentOracle.t.sol:

```solidity
 function testDecimals() public {
        uint256 initialLiquidityIBT = 2e18;

        uint256 initialPrice = 1e8;

        IFactorySNG.CurvePoolParams memory curvePoolParams = IFactorySNG.CurvePoolParams({
            A: 1500,
            fee: 1000000,
            fee_mul: 20000000000,
            ma_exp_time: 600,
            initial_price: initialPrice,
            rate_adjustment_oracle: address(0)
        });

        (address _pt, , address _curvePool) = __deployPoolWithInitialLiquidity(
            initialLiquidityIBT,
            curvePoolParams
        );

        testContract = new TestContract(_pt, _curvePool);
        (, int256 answer, , , ) = testContract.latestRoundData();
        uint256 castAnswer = uint256(answer);
        console.log(castAnswer);
        assertApproxEqAbs(castAnswer, 10 ** 15, 1, "initial pt to ibt rate is wrong");
       
        uint256[] memory stored_rates = IStableSwapNG(_curvePool).stored_rates();
        uint256 decimals = IPrincipalToken(_pt).decimals();
        assertEq(decimals, 8, "decimals are wrong");
    } 
```