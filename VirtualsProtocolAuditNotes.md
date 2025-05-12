# 1 STORING FUNCTIONS IN STORAGE, FUNCTION TYPE VARIABLE VS FUNCTION DEFINITION

In ValidatorRegistry.sol, there is the following code block:

```solidity
abstract contract ValidatorRegistry is IValidatorRegistry, Initializable {
    mapping(uint256 virtualId => mapping(address account => bool isValidator)) private _validatorsMap;
    mapping(address account => mapping(uint256 virtualId => uint256 score)) private _baseValidatorScore;
    mapping(uint256 virtualId => address[] validators) private _validators;

    function(uint256, address) view returns (uint256) private _getScoreOf;
    function(uint256) view returns (uint256) private _getMaxScore;
    function(uint256, address, uint256) view returns (uint256) private _getPastScore;

    function __ValidatorRegistry_init(
        function(uint256, address) view returns (uint256) getScoreOf_,
        function(uint256) view returns (uint256) getMaxScore_,
        function(uint256, address, uint256) view returns (uint256) getPastScore_
    ) internal onlyInitializing {
        _getScoreOf = getScoreOf_;
        _getMaxScore = getMaxScore_;
        _getPastScore = getPastScore_; //c these functions that are initialised in the AgentNftV2.sol contract are getter functions and when you look in the functions of this contract, you can see where they are used.
    }
```

Notice how before the init function, there are 3 functions declared and these functions dont look like the standard functions we are used to seeing. These functions are declared at the start of the contract. At this point, I had never seen this before so i was wondering how this was possible. 

This is where we will learn a thing or 2 about function type declarations vs function definitions. Function definitions as the name suggests is where you have a defined function. i.e. a function that does something. So the normal functions you see in contracts you look at are function definitions where you have the function and then inside the curly braces, you can see what the function does.

However, there are function type declarations which again, as the name suggests, is where we define the function type and this is basically a pointer to a function. So function type declarations simply point to where a function definition is. That is all they do. 

Notice how function type declarations are written as follows:

```solidity
function(uint256, address) view returns (uint256) private _getScoreOf;
```

It starts with function followed by the data types the function will take , whether it is a view function and whether it returns anything. Notice that this isnt a function definition so there is no visisbility defined. The private visibility you see after the function type declaration is the visibility of the storage slot. So we are putting _getScoreOf at slot 0 for example, we are setting the visibility of the slot to private. 

The slot visibility has nothing to do with the function definition that will be passed to that slot. For example, if in the init function, i passed the following function definition:

```solidity
function transferFrom(address to, uint256 value) public view returns(uint256){
    //whatever I want function to do
}
```

as the getScoreOf argument, this would be fine because even though the function i defined is public, _getScoreOf does not have any visibility restrictions. As long as it is a function definition, it will work, The visibility used for function type declarations only refer to the storage slot.


Now lets see what actually happens in the EVM when we store a function definition in a function type declaration in storage. To do this, we will be utilizing a minimal contract below:

```solidity
pragma solidity 0.8.26;

abstract contract Tests {

    function(uint256, address) view returns (uint256) private _getScoreOf; 

    function init(function(uint256, address) view returns (uint256) getScoreOf_) internal {
        _getScoreOf = getScoreOf_;
    }

    function useScore() public view returns(uint256){
    return _getScoreOf(4, address(this));
}
}

contract Check is Tests{

    function setInit() public{ 
        init(makeSure);
    }

    function makeSure(uint256 num, address addr) public view returns (uint256) {
        uint256 num1 = 1;
        uint256 num2 = 3;

        return num1 + num2;
}
}
```
As you can see, what this contract aims to achieve is rather simple, we have the _getScoreof function type declaration, the idea is that we want to store the makeSure function definition in that type declaration in storage and then call the _getScoreOf function in storage using the useScore function to see what happens. Pretty simple stuff.

To do this, take this contract exactly as it is, compile it in remix, copy the bytecode, take out the contract creation bit(first fe). Take this to the evm.codes playground which you already know a lot about. Then take the selector for setInit() which is 0xd94b9b37 and put that in the calldata input region and run the bytecode. This will be our first run.

What will happen is that since setinit simply puts the makeSure function definition at slot 0 (_getScoreOf), what you will see in the EVM is that 9B is stored at slot 0. I will just go ahead and say it now that 9B is the program counter of where the makeSure function will be in this contract. You already know what program counters are from the assembly and fv course notes.

So 9B is going to be the program counter for the makeSure function. So when we store any function definition in a function type declaration in storage, what is actually stored in storage is the program counter of that function definition. 

You might be asking how I know this. To see this in the EVM, once you have run setInit() in the playground, you will see 9B in storage, then get the selector for useScore() and put that in the calldata and run that and what you will see happen is that the EVM will go to slot 0, get the 9B program counter and perform a jump to there which shows that the EVM goes to 9B to perform the _getScoreOf type declaration which is just pointing to makeSure() function at program counter 9B.

You can also see what happens from the 9b program counter below:

```
[9b]	JUMPDEST	
[9c]	PUSH0	
[9d]	DUP1	
[9e]	PUSH1	01
[a0]	SWAP1	
[a1]	POP	
[a2]	PUSH0	
[a3]	PUSH1	03
```

Does this look familiar to what is happening in the makeSure function? It does so we know that our deduction is correct. So this is what is happening behing the scenes.

So what are the dangers of this? Well if your contract is immutable then the dangers are 0. But the issue comes if this is an implementation contract for a proxy and then the proxy is upgraded to a new implementation contract. Say the implementation contract adds 2 new functions. What will happen is that the bytecode for the new contract will change and the pointer at slot 0 that currently had the 9B program counter, 9B will more than likely no longer be the pc for the makeSure function anymore. This is because the program counters are determined from the runtime bytecode and adding more functions will change the bytecode which reassigns all the program counters. You can research this more in your spare time. So if you try to call _getScoreOf on the proxy after upgrading, it will most likely do something else because 9B will probably be doing something else. The solution would be to call setInit on the proxy immediately after upgrading to reset the pc at slot 0 to the new pc for the makeSure function. 

This is a key vulnerability to keep in mind and you can even test it yourself. Add a new function to our minimal contract example and go and remix and compile it and call setInit and you will see that the program counter stored in slot 0 wont be 9B anymore.





# 2 HARDHAT VERSION WORKAROUND

 ADD NOTE ABOUT HARDHAT VERSION 2.19.4 WHEN TRYING TO FORK BASE NETWORK , IT WAS SHOWING THE ERROR BELOW, I HAD TO UPGRADE THE HARDHAT VERSION FOR 2.22.17 FOR IT TO WORK. THIS IS A NOTE IF YOU RUN INTO A SIMILAR ERROR LATER ON

So this is a quick note about making sure to use a working hardhat version when trying to install hardhat. The error I was encountering was that I ws trying to fork the base network in the hardhat config file and everytime i tried to run my tests, I ran into the problem below

```bash
 Invalid value undefined supplied to : RpcBlockWithTransactions | null/totalDifficulty: QUANTITY
      at decodeJsonRpcResponse (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/core/jsonrpc/types/output/decodeJsonRpcResponse.ts:15:11)
      at JsonRpcClient._perform (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/jsonrpc/client.ts:286:48)
      at processTicksAndRejections (node:internal/process/task_queues:95:5)
      at ForkBlockchain._getBlockByNumber (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/provider/fork/ForkBlockchain.ts:245:22)
      at ForkBlockchain.getBlock (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/provider/fork/ForkBlockchain.ts:68:13)
      at Function.create (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:220:31)
      at HardhatNetworkProvider._init (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:260:28)
      at HardhatNetworkProvider._send (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:198:5)
      at HardhatNetworkProvider.request (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:124:18)
      at getSigners (/home/io10-0x/securityresearchcyfrin/2025-04-virtuals-protocol/node_modules/@nomicfoundation/hardhat-ethers/src/internal/helpers.ts:46:16)
```

The way I resolved it was to install a different hardhat version which was 2.22.17 and it worked perfectly after this change. This may be outdated in the future when you are reading this but trying to update the version when running into issues is always a good debugging step.



# 3 EIP-1167 AND OPENZEPPELIN CLONES FUNCTIONALITY 

This section will cover what EIP-1167 does and how openzeppelin implements in. All you have to do is read the comments I left on the function below. You can also go and read the actual EIP online if you want some more scope.

```solidity
 function clone(address implementation, uint256 value) internal returns (address instance) {
        if (address(this).balance < value) {
            revert Errors.InsufficientBalance(address(this).balance, value);
        }
        assembly ("memory-safe") { //c so this function is a lightweight way of deploying a proxy that is pointing to an implementation contract without having to deploy a brand new proxy contract. I will explain what is happening
            // Cleans the upper 96 bits of the `implementation` word, then packs the first 3 bytes
            // of the `implementation` address with the bytecode before the address.
            mstore(0x00, or(shr(0xe8, shl(0x60, implementation)), 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000))
            /* c the shl(0x60, implementation) is removing the first 12 bytes (96 bits) from the implementation because as you know, everything in the evm is left padded to 32 bytes and an address is 20 bytes so we need to remove the first 12 bytes (96 bits) from the implementation to remove the extra 12 bytes the evm padded the address with so that implementation now starts with the first 20 bytes of the address and padded with 12 bytes worth of 0's to make 32 bytes because just cuz we shifted left, the length still has to remain at 32 bytes as this is the evm and it only interprets things in 32 bytes form.

            For example, say we had the address 0xcDb32eaa051698E11B84c3f7037CCbc86E8784Ac. This is 20 bytes. In the evm, it would be padded to 32 bytes by adding 12 bytes of 0's to the left of the address. so it would be 0x000000000000000000000000cDb32eaa051698E11B84c3f7037CCbc86E8784Ac.

            Now, say we shift the address to the left by 96 bits. 96 bits is 12 bytes. So move the pointer from the left up by 12 bytes which means 0x000000000000000000000000cDb32eaa051698E11B84c3f7037CCbc86E8784Ac becomes 0xcDb32eaa051698E11B84c3f7037CCbc86E8784Ac00000000000000000000000000000000. Notice it is still 32 bytes but the pointer is now at the first 20 bytes of the address. 

            Then we shift right now by 232 bits which is 29 bytes. so starting from the the right, we move the pointer 29 bytes to the left which gives us 0x(29 bytes of 0's)cDb32e. As we learnt in the assembly and fv course, left padding with 0's is insignificant but right padding with 0's could mean something significant. 

            Also notice how with the or function, the way i explained it was the shl first and then the shr. This is because in yul, the or, and, xor functions all read right to left instead of the normal left to right which i also go more into in point 7 of these notes.
            
            From here, we perform a bitwise or with 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000. This is the first part of the runtime bytecode code of a proxy contract. we know that a proxy contract is a very minimal contract. It contains usually only logic that forwards all calls to the implementation contract via a delegatecall which is stuff we are very familiar with by now. 

            Now the whole idea of this is that we are trying to use the normal runtime bytecode of a minimal proxy contract. so when you compile a simple proxy contract that only contains logic that forwards all calls to the implementation contract via a delegatecall, you get a runtime bytecode that looks like this:

            3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3

            You can even take this to the playground on evm.codes and see what it can do or look at EIP-1167 which shows what happens in the stack with this bytecode. Where it says bebebebe is where we are supposed to input the token implementation contract and that is all this function is trying to do. Unlike the regular proxy pattern where one proxy points to one implementation contract, several clones can point to the same implementation contract. Clones cannot be upgraded as these proxy contracts are shipped out with no upgrade functionality so this is something important to note.
            
            
            Notice how the last 3 bytes of this code are 0's because when the bitwise or happens, it essentially adds the last 3 bytes of 0x(29 bytes of 0's)cDb32e to 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000. This is because with bitwise or, it compares 2 bytes and if any of the bytes is non zero, it returns the non zero byte as the result. So this forms the first 32 bytes part of our runtime code. The next 32 bytes needs to then contain the last 17 bytes of our implementation contract address followed by the rest of the runtime bytecode which is 0x5af43d82803e903d91602b57fd5bf3. That is what the mstore(0x20, or(shl(0x78, implementation), 0x5af43d82803e903d91602b57fd5bf3)) is doing. Finally, we call create(value, 0x09, 0x37) which is the opcode for creating a contract and the create opcode attaches the contract creation code we need to the runtime code and deploys a contract which is all stuff we know about. It is important to know how all this works and to know more, have a look at EIP-1167 or take this bytecode to the playground and see what it can do.

            */
            // Packs the remaining 17 bytes of `implementation` with the bytecode after the address.
            mstore(0x20, or(shl(0x78, implementation), 0x5af43d82803e903d91602b57fd5bf3))
            instance := create(value, 0x09, 0x37)
        }
        if (instance == address(0)) {
            revert Errors.FailedDeployment();
        }
    }
```

# 4 MSG.DATA HELPS GIVE CALLDATA CONTEXT . TALK ABOUT MSG.DATA AND ITS RELEVANCE

This section is aimed to cover msg.data which is a global solidity variable and all it does is get the calldata for the original call. Similar to msg.value, this value is persistent throughout the function call. It doesnt change. Lets see an example to explain this.


```solidity
function example(uint256 a, address b) external {
    IERC20(b).transfer(msg.sender, a); // Nested call to transfer()
}
```
When you call example(123, 0x123...):

msg.data (visible inside example or transfer):

0x3bfa8c8d... // Selector for `example(uint256,address)`
000...07b    // 123 (uint256)
000...123... // address (0x123...)
The transfer() call does not appear in msg.data—it’s a separate internal transaction.

If example() Calls a transfer() Function: Which Selector Appears in msg.data?
When you call example(uint256,address), which then internally calls another function (like transfer), the msg.data remains unchanged—it still reflects the original calldata of the external call (example). Here’s why:

1. msg.data is Immutable for the Entire Transaction
msg.data always contains the original calldata sent to the contract.

Even if example() calls transfer() internally, msg.data does not update to show the nested call’s selector.


# 5 DIFFERENCE BETWEEN BINARY, DECIMALS AND HEXADECIMALS, HOW SOLIDITY HANDLES BINARY VALUES WITH CONVERSION TO HEX

This section is going to mainly cover the difference between binary numbers and hexadecimals and the reason for this is because for the longest time, i thought when i saw bytes data = bytes(whatever value), when you view this in solidity, the value we get back from data is in hexadecimal so i just thought bytes represented hexadecimals but that is not true. Bytes are binary numbers of base 2 and hexadecimals have base 16. I will cover why solidity always returns hexadecimals in the comments of the code below so have a read as you will also see how openzeppelin's Governor::_encodeStateBitmap works so youre in for a 2 in 1 here lfg.

```solidity
/**
     * @dev Encodes a `ProposalState` into a `bytes32` representation where each bit enabled corresponds to
     * the underlying position in the `ProposalState` enum. For example:
     *
     * 0x000...10000
     *   ^^^^^^------ ...
     *         ^----- Succeeded
     *          ^---- Defeated
     *           ^--- Canceled
     *            ^-- Active
     *             ^- Pending
     */
    function _encodeStateBitmap(ProposalState proposalState) internal pure returns (bytes32) {
        return bytes32(1 << uint8(proposalState));
    } /*c we need to go over what this function is doing. From previous audits, we know that 1 << any number means that we are left shifting 1 by any number of positions. For example, if we have 1 << 2, we are left shifting 1 by 2 positions which will give us 0b100. Notice how i used 0b instead of 0x which we are used to. This is a great introduction to the fact that binary and hexadecimals are completely different things. Lets go over this idea:

    Decimal (10)
            This is the standard number system we use every day — base 10.

            Digits: 0 to 9
            So when you see:
            10 (decimal) = 1 × 10¹ + 0 × 10⁰ = 10. Each place is worth 10 times more than the one to its right. So with the number 10 , the 1 is worth 10 times more than the 0 to the right of it which is why normal numbers we use, we call them base 10

            2. Binary (0b10)
            This is base 2, used by computers.
            Digits: 0 and 1 only.
            Each place is worth 2 times more than the one to its right.

            So:
            0b10 = 1 × 2¹ + 0 × 2⁰ = 2
            That’s why 0b10 in binary = 2 in decimal

            3. Hexadecimal (0x10)
            This is base 16, often used in programming and smart contracts.

            Digits: 0–9, then A–F (where A = 10, B = 11, ..., F = 15)

            Each place is worth 16 times more than the one to its right.

            So:
            0x10 = 1 × 16¹ + 0 × 16⁰ = 16
            That’s why 0x10 in hex = 16 in decimal

            Now you understand this , lets go back to the function:

            return bytes32(1 << uint8(proposalState));

            We are dealing with bytes32 here which means we are dealing in binary and not hexadecimals. So we first convert the proposal state from the proposal state enum to a uint8. which is just the number it represents. we know that with enums, they are represented by numbers and if you go back over past solidity notes, you should remember this.

            So the enum in question here is:

            enum ProposalState {
        Pending,
        Active,
        Canceled,
        Defeated,
        Succeeded,
        Queued,
        Expired,
        Executed
    }

    So if we have ProposalState.Active, this will be converted to a uint8 which is 1.

    Now we have 1 << uint8(1) which means we are left shifting 1 by 1 position.

    So 1 << 1 = 0b10 = 2 in decimal. You might be asking why 1 << 1 gives 0b10 and not 0b1. This is because remember the command is 1 << 1 so we are left shifting 1 which is 0b1 in binary by 1 which gives us 0b10 which is 2 in decimal. 

    So this binary value is then wrapped as a bytes32. What this does is to convert our binary value to a 32 byte hexadecimal value. So 0b10 gets converted to 0x2 in 32 bytes form. This is because as we know, solidity and the EVM work with hexadecimal values and uses them for almost everything. So every data type you see is logged as a hexadecimal value which you already know. So when the proposal is set to active, the bytes32 value is 0x0000000000000000000000000000000000000000000000000000000000000002 and above is how the conversion is done.

    It is also worth noting that using cast can also convert to/from binary using cast --to-base 2 bin for example which will give 0b10. This is a cool tip i bet you didnt know.
    */
```


# 6 COOL ELO CALCULATOR EXPLANATION

Virtuals implemented an elo calculator function which compares the ratings of 2 players who both start with the same score and then it predicts their rating based on the outcome of a game. It is a really cool formula and idea and I think it is worth noting how it is implemented in case you come across something like this in the future. Follow the comments in the code below. You can also go into EloCalculator contract in the repo to see more functions involved but the main 2 functions that explain what is going on are below.

```solidity
 // Get winner elo rating
    function battleElo(uint256 currentRating, uint8[] memory battles) public view returns (uint256) {
        //c the battles array is the votes array that is passed in from the AgentDAO::_updateMaturity function.
        uint256 eloA = 1000;
        uint256 eloB = 1000;
        for (uint256 i = 0; i < battles.length; i++) {
            uint256 result = mapBattleResultToGameResult(battles[i]);
            (uint256 change, bool negative) = Elo.ratingChange(eloB, eloA, result, k); //c changed eloB and eloA around for testing purposes
            change = _roundUp(change, 100); //c this is the reason an overflow doesnt happen. for example, if change is 1500, this function does 1500+100-1/100 = 15.99 which is 15. So the change is rounded down to 15.
            if (negative) { //bug see Elo::ratingChange for more details
                eloA -= change;
                eloB += change;
            } else {
                eloA += change;
                eloB -= change;
            }
        }
        return currentRating + eloA - 1000;
    }
    /*c so we will use an example to illustrate what is going on in this loop. So a user has casted a vote with reason and passed a params argument which is a uint8 array of [1,2]. So AgentDAO::_updateMaturity will call this function with currentRating being 100 and the battles array being [1,2]. In the loop, the first battle value is 1 so result is 100. Then Elo.ratingChange is called with eloA being 1000 and eloB being 1000 and result being 100 and k being 30 as you can see from the initialize function. Go to the Elo.sol contract to see how the rating change is calculated and my notes on it. 

    After reviewing it, the function should make sense . What i explained there will then set eloA to a new value and same with eloB. Then the loop will run again but this time the battles element will be 2 so result will be 50. Then the ratingchange function will be called again with the updated values of eloA and eloB. You can use my notes on that function to see what the value will be on the second run. 

    


    */  
```

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
        uint256 score, //c score is the value from the battles array from EloCalculator::battleElo
        uint256 kFactor
    ) internal pure returns (uint256 change, bool negative) {
        uint256 _kFactor; // scaled up `kFactor` by 100
        //c notice that _kFactor and kFactor are not the same. kFactor is passed in from whatever contract is calling this function. _kFactor is a new variable initialized to 0
        bool _negative = ratingB < ratingA; //c following our example from EloCalculator::battleElo, ratingA is 1000 and ratingB is 1000 so _negative is false. this line is saying that if your opponent has a lower rating than you, then the ratingdiff should not be in your favour by 'your' i mean player A. By 'in your favour' you will see what that means when the rating diff is scaled by 800
        uint256 ratingDiff; // absolute value difference between `ratingA` and `ratingB`

        unchecked {
            // scale up the inputs by a factor of 100
            // since our elo math is scaled up by 100 (to avoid low precision integer division)
            _kFactor = kFactor * 10_000; //q following our example from EloCalculator::battleElo, kFactor is 30 so _kFactor is 300000 but the comment on _kFactor says that it is scaled up by 100 so why do they scale by 10000 here? this is a valid question but irrelevant for the explanation
            ratingDiff = _negative ? ratingA - ratingB : ratingB - ratingA; //c following our example from EloCalculator::battleElo, ratingA is 1000 and ratingB is 1000 so ratingDiff is 0.
        }

        // checks against overflow/underflow, discovered via fuzzing
        // large rating diffs leads to 10^ratingDiff being too large to fit in a uint256
        require(ratingDiff < 1126, "Rating difference too large");
        // large rating diffs when applying the scale factor leads to underflow (800 - ratingDiff)
        if (_negative) require(ratingDiff < 800, "Rating difference too large"); //c these 2 require statements just stop the math done below from potentially overflowing as you will see
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
            n = _negative ? 800 - ratingDiff : 800 + ratingDiff; //c normally i would check here for underflow but following our example from EloCalculator::battleElo, ratingDiff is 0 so n is 800 and also the require statements prevent over/underflow.

            // (x / 400) is the same as ((x / 25) / 16))
            _powered = fp.rpow(10, n / 25, 1); // divide by 25 to avoid reach uint256 max
            powered = sixteenthRoot(_powered); // x ^ (1 / 16) is the same as 16th root of x
           //c let me explain what the above 2 lines are doing ratingDiff/400 can be rewritten as (ratingDiff/25)/16
            //This means 10^(ratingDiff/400) is the same as 10^((ratingDiff/25)/16)
            //Which is the same as (10^(ratingDiff/25))^(1/16)
            //c following our example from EloCalculator::battleElo, ratingDiff is 0 so n is 800 so _powered is 10^800/25 = 10^32 and powered is 10^32/16= 10^2 which is 100 which is what this function means when it keeps talking about scaling by 100 because ratingDiff is offset by 800, this is what is makes all of this work. So if ratingDiff wasnt offset by 800, then expectedscore = 1/(1+10^0)= 1/(1+1) but with the 800 offset, expectedscore = 1/(1+100) which is where the 100 scaling comes from

            // given `change = kFactor * (score - expectedScore)` we can distribute kFactor to both terms
            kExpectedScore = _kFactor / (100 + powered); // both numerator and denominator scaled up by 100 . so far we have expectedscore = 1/(1+100) since powered is already scaled by 100, we need to scale the 1 by 100 which is why you see 100 + powered instead of 1 + powered


            //c with this kexpectedscore is just multiplying _kFactor (scaled k factor) by the expected score which we see in the formula above as 1 / (1 + 10 ^ (ratingDiff / 400))
            
            
            //q the numerator and denominator are not scaled up by 100. In actual fact, _kfactor is scaled up by 10000 by multiplying it by 10000. dont know if this is planned or not


            kScore = kFactor * score; // input score is already scaled up by 100
            //c dont be deceived by the comment saying the input score is scaled by 100. what this is describing is what happens in the EloCalculator::mapBattleResultToGameResult function where the votes are scaled by a value but that value isnt 100. Not sure what it is scaled by but it is scaled by something

            //c following on from my example, elo change = kFactor * (score - expectedScore) so kScore is 30(remember difference between kFactor and _kFactor) * 100 = 3000 and kExpectedScore is 300,000/200 = 1500 which means the elo change is 3000 - 1500 = 1500


            // determines the sign of the ELO change
            negative = kScore < kExpectedScore;
            change = negative ? kExpectedScore - kScore : kScore - kExpectedScore;
        }
    }

    /*c now i have covered the intricacies of this function, let me explain from a high level what this function is doing. the idea is that we have 2 players. player A and player B. They both start with the same base score of 1000 from EloCalculator::battleElo. These scores serve as a base rating for the players. This function is meant to calculate the change in ratings between the 2 players based on the score of the players of a game. The function will always return the change in rating for player A  as the natspec says. So the function returns a negative boolean and a change value. The negative boolean being trueindicates that player A's rating should reduce and player B's rating should increase. The change value is the amount of rating change. So the formula that calculates the change is:

    expected score = 1 / (1 + 10 ^ (ratingDiff / 400))
    elo change = kFactor * (score - expectedScore)

    So the expected score is supposed to calculate an expected score for player A based on the rating difference between the 2 players. The expected score is then compared to the actual score that player A got in the game based on the score passed in the arguments of this function. So there is the following line in the function:

     negative = kScore < kExpectedScore;

     //bug So if the actual score player A got is less than the expected score, then the negative boolean is true meaning that player A's rating should reduce and the rating change is to be applied to player A. Now the bug is that in EloCalculator::battleElo, when this rating change function is called, this is what is called:

     (uint256 change, bool negative) = Elo.ratingChange(eloB, eloA, result, k);

     so the ratingA which is supposed to be player A's rating, eloB is passed there, which means that the rating change is calculated based on player B's rating This isnt really a problem though until you look at the rest of how EloCalculator::battleElo. See below:

      if (negative) {
                eloA -= change;
                eloB += change;
            } else {
                eloA += change;
                eloB -= change;
            }
        }
        return currentRating + eloA - 1000;

        It is saying that if the negative boolean is true, then player A's rating should reduce and player B's rating should increase. This is wrong because as I said above, the rating change is based on player B's rating. So if the negative boolean is true, then player B's rating should reduce and player A's rating should increase but that is not what happens which is where the bug is.
    */

```



# 7 NOTES ON YUL AND/OR RIGHT TO LEFT READ FUNCTIONS AND WHERE FUNCTION PARAMETERS ARE STORED IN THE EVM, KEY NOTE ON STATIC VS DYNAMIC ARRAY TYPES, SOLIDITY HEAD AND TAIL ENCODING FORMAT

```solidity
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Airdrop {
    /**
     *
     * @param _token ERC20 token to airdrop
     * @param _recipients list of recipients
     * @param _amounts list of amounts to send each recipient
     * @param _total total amount to transfer from caller
     */
    function airdrop(
        IERC20 _token,
        address[] calldata _recipients,//c note that when you have function parameters, if the calldata storage location is attached as you see here, the data is accessed as read-only from the calldata storage location but for types like the _total argument below which is declared without a location specified will simply be added on the stack and I will show you what this looks like in a minimal contract below.
        uint256[] calldata _amounts,
        uint256 _total 
    ) external {
        // bytes selector for transferFrom(address,address,uint256)
        bytes4 transferFrom = 0x23b872dd;
        // bytes selector for transfer(address,uint256)
        bytes4 transfer = 0xa9059cbb;

        assembly {
            // store transferFrom selector
            let transferFromData := add(0x20, mload(0x40)) //c at this point, 0x80 is at the free memory pointer which is standard solidity free memory slot so we are adding 32 bytes to that for some reason to store the transferFrom selector there as seen below
            mstore(transferFromData, transferFrom)
            // store caller address
            mstore(add(transferFromData, 0x04), caller()) //c standard forming the calldata for the transferFrom function
            // store address
            mstore(add(transferFromData, 0x24), address()) //c address is the address of the executing account which in this case is this airdrop contract
            // store _total
            mstore(add(transferFromData, 0x44), _total) //c this is the amount of tokens to be transferred from the caller to the airdrop contract
            // call transferFrom for _total

            if iszero(
                and(
                    // The arguments of `and` are evaluated from right to left.
                    or(eq(mload(0x00), 1), iszero(returndatasize())), // Returned 1 or nothing.
                    call(gas(), _token, 0, transferFromData, 0x64, 0x00, 0x20)
                ) /*c very interesting, i didnt know this but in yul, the functions AND , OR  are evaluated from right to left. So what happens here is that we have an overall AND function. This function is evaluated right to left which means the first thing that will run is the call function and this function will call transferFrom which is what we set up in memory above to transfer the total amount of tokens to the airdrop contract. The call function returns 1 if the call is successful and 0 if it is not. 
                
                Then we go to the OR function and this is also evaluated right to left so iszero(returndatasize()) is evaluated first. RETURNDATASIZE() gets whatever data is returned from the previous call which is the transferFrom function. As we know, the transferFrom function returns a boolean value. i.e. 1 for true and 0 for false. So if the transferFrom fails, it returns 0 and and then iszero returns 1 and we know this from evm codes website where we can see all the opcodes and their descriptions.

                Then eq(mload(0x00), 1) is evaluated. In solidity, at slot 0 in memory is usually empty by default so this will be 0. so eq will return 0 as mload(0x00) is != 1. So we now have or(0,1). we know that this is a bitwise OR which returns 1 if either of the two values is 1. so we now have and(1,1) if the function is successful which gives 1 as that is how bitwise AND works.

                I feel like this is a longwinded way to do this. They couldve simply checked the return value of call function and if it is 0, they could have reverted as it means the function failed but they did this for some reason.

                */

            ) {
                revert(0, 0)
            }

            // store transfer selector
            let transferData := add(0x20, mload(0x40))
            mstore(transferData, transfer)

            // store length of _recipients
            let sz := _amounts.length //c note that in yul, let variables are stored on the stack. so sz is stored on the stack.

            // loop through _recipients
            for {
                let i := 0
            } lt(i, sz) {
                // increment i
                i := add(i, 1) //q what happens in every iteration of this loop to the i variable? i is initiated in teh stack as 0 and is incremented in the stack after every iteration. I will give a sample contract and calldata you can use to see this.
            } {
                // store offset for _amounts[i]
                let offset := mul(i, 0x20) //c this makes sense because if i is 0, the offset is 0. if i is 1, the offset is 0x20. if i is 2, the offset is 0x40. and so on. It always goes up by 32 bytes
                // store _amounts[i]
                let amt := calldataload(add(_amounts.offset, offset)) //c calldataload gets the current input data of this airdrop function and as we know, the calldata has the selector and all the arguments concatenated BUT as you know, with arrays, when you abi.encode an array, the format is as follows. The pointer to the start of the length of the array, the length of the array and then each element in the array in that order. so _amounts.offset in yul does something slightly different though. .offset goes to teh start of the elements in the array instead of the start of the length of the array. More on this below
                // store _recipients[i]
                let recp := calldataload(add(_recipients.offset, offset)) //c same idea as above point
                // store _recipients[i] in transferData
                mstore(add(transferData, 0x04), recp)
                // store _amounts[i] in transferData
                mstore(add(transferData, 0x24), amt)
                // call transfer for _amounts[i] to _recipients[i]
                // Perform the transfer, reverting upon failure.
                if iszero(
                    and(
                        // The arguments of `and` are evaluated from right to left.
                        or(eq(mload(0x00), 1), iszero(returndatasize())), // Returned 1 or nothing.
                        call(gas(), _token, 0, transferData, 0x44, 0x00, 0x20)
                    )
                ) {
                    revert(0, 0)
                }
            }
        }
    }
}


```

Static types (IERC20, uint256, etc.) are passed via the EVM stack.
They're encoded directly in calldata as 32-byte values.
But once decoded, they live on the stack in Solidity.
That’s why you don’t need a calldata label — it's irrelevant for static types.

Dynamic types (like arrays, bytes, string) must specify a data location:
memory, calldata, or storage

In your case, calldata means they stay in the calldata region and are read directly from it
That’s why _recipients.offset and _amounts.offset make sense — they're offsets in calldata

So are _token and _total stored in calldata?
 No, they’re not accessed from calldata anymore after decoding.

They are:
Encoded into calldata when the function is called
But copied onto the stack when the function starts
From that point, Solidity accesses them via the stack (not via a calldata pointer)


As I stated in the above comments, I said i would show you how a minimal contract works and how calldata parameters differ from normal ones. So our minimal contract is as follows:

```solidity
contract Owner {

   function testNum(uint256 a, uint256 b, uint256[] calldata num) public pure returns (uint256) {
   return num[0] + a + b;

} 
}
```

This minimal contract contains a testNum function that takes 2 uint256 arguments and one uint256 array calldata argument and does a simple addition. To see how this works, take this contrac tto remix, compile it, copy bytecode, take to evm.codes playground and remove contract creation code.

Now we need to call the testNum function to see how our parameters are processed in the EVM. So to form the calldata, i decided to use chisel and i did the following. I first typed:

```solidity
uint256[1] public num1;
num1[0]= 2;
abi.encodeWithSignature("testNum(uint256,uint256,uint256[])", 1,5,num1);
```

Looks perfectly ok right? Wrong but you will see why below. The result of this abi encoding gives:

```
0x17d0076800000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000005000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000
```

Now whats wrong with this? Notice that we configured an array and as you know, when encoding a dynamic array, we expect to see the pointer (offset) to where the length of the elements in the array start, the length of the array and then the elements in the array but we dont see that at all here. We just see the raw value of 2 encoded. The reason is because if you look at what we entered into chisel, we had uint256[1] public num1. So num1 is a storage variable which is an array BUT this array is one you defined with a fixed length of 1. This means that the array is no longer dynamic. It has a fixed length that is known at compile time.

Previously, you thought all arrays were dynamic but that isnt true. An array is only dynamic when it doesnt have a fixed length. So if you had uint256[] public num1, this is a dynamic array that can have any length. So when the compiler tries to compile this, it will see that no length was defined which makes it a dynamic array but once you add a value in the square brackets, the compileer knows that this will always be the length of the array so it will treat it like a static type like a normal uint256 or any other static type which is why when you encode it, you see it just gives the elements in the array which was just 2 in this case.

This is in contrast to when i did the following:

```solidity
uint256[] memory numbers = new uint256[](1);
numbers[0] = 100;
abi.encodeWithSignature("testNum(uint256,uint256,uint256[])", 1,5,numbers);
```

This was the output:

```
0x17d007680000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000500000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000
```

Now what you can see here is more like what we expect to see. We see the selector, the first argument which is 1(uint256 static), the second argument, 5 (uint256 static), and then we have the offset to the length of the array which is 0x60, the length of the array and then the elements in the array (0x64) which is 100 in decimals.

The reason this is different to what we had earlier is because new uint256[](1); creates an instance of a dynamic array (uint256[]) with length 1. Although we have stated that the length is 1, it is an instance of a dynamic array so the compiler sees this as an instance of a dynamic array so it assumes the length could be anything although we have specified what we want it to be, that speciifcation is only noticed at runtime which is why when we encode this, we get it in the form of a dynamic array.

Another important part to cover is that you see that the offset is 0x06 which is 96 bytes so you might be wondering why that is. We know that the offset is a pointer to the length of the dynamic type. That pointer begins from the start of the data. So in the calldata you see above, what the offset is saying is that immediately after the 0x, skip 96 bytes to find the length of the array. In this scenario, since we use encodeWithSignature, solidity knows that there is a selector involved so the selector is skipped. So what it is saying is that 96 bytes AFTER the selector is where the length of the array is. 

The reason for the offset pointing to the length of the array is because how dynamic types are stored in solidity is with the length of the dynamic type followed by the actual data. This is the same with bytes and strings as well. This is very important to note. 

Another thing to keep in mind is the head + tail layout that the abi uses in decoding. What this means is that when we encode anything using the abi, it is structured using a head + tail layout. See below:

┌ 4-byte selector (if building calldata) ┐
├─ head ─────────────────────────────────┤
│ arg1 (static)                         │
│ arg2 (static)                         │
│ arg3 OFFSET →─────────────────────────┼───┐
│ arg4 (static)                         │   │
│ arg5 OFFSET →─────────────────────────┼───┤  ⇠ offsets jump into the tail
└────────────────────────────────────────┘   │
                ▲   byte 0 of head           │  (selector is not counted)
                │                            ▼
┌─ tail ─────────────────────────────────────────────────┐
│ arg3 length  (N)                                       │  ← offset for arg3 lands **here**
│ arg3 data (padded)                                     │
│ arg5 length  (M)                                       │  ← offset for arg5 lands **here**
│ arg5 data (padded)                                     │
└────────────────────────────────────────────────────────┘

What this means is that when we encode, what happens is that if a selector is involved, that is the first thing that gets added which is part of the head. The head then contains the arguments BUT in a certain format. The head only contains raw values for static types. So if we have a uint256, address, bool, etc, the raw value is what you will see in the head. If we have a dynamic type like dynamic arrays, strings, bytes, etc, the OFFSET to the length of the type is what is stored in the head. The tail then contains the contents of all dynamic types. So the length of the type and the actual data related to that type will be in the tail

So in our example, where we encoded:

```solidity
abi.encodeWithSignature("testNum(uint256,uint256,uint256[])", 1,5,numbers);
```

The head will contain the selector of testNum(uint256,uint256,uint256[]), followed by 1 which is a static type, then 5 which is also a static type. Then we have the offset to the numbers dynamic array. That is the head section. Then the tail will have the length of the numbers array followed by the 0x64 we stored in it.

Another example would be if we had:
```solidity
abi.encodeWithSignature("testNum(uint256,uint256[],uint256)", 1,numbers,5);
```

We will get the following result:

```
0xe6954d320000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000
```

As you can see, the head contains the selector, followed by our first argument, 1, then the OFFSET to the length of the numbers array which is still the only element in the tail so it is still 0x60. Then our third argument 5. This is the end of the head. In the tail, then we have the length of the numbers array followed by 0x64 which we stored. This is an important concept to grasp and you need to understand how this works. It will help you a lot when decoding data.

Also note that all abi.encodes read from memory so if you had the following:

```solidity
uint256[] public numbers;
numbers.push(1);
bytes memory data = abi.encodeWithSignature("testoffsetVal(uint256[], uint256)", numbers, 2)
```

You would get the following result:

```
0x94ada46a00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Notice how the head contains on offset of 0x40. So if we skip 64 bytes from the start of the calldata, it has just 0's and there is no 1 that we pushed to the array. This is because abi.encode only reads from memory. If you try to get it to read from storage which we know is computationally more expensive, it will fallback to returning an empty array so make sure all your abi.encode elements are in memory.

On another note, I spoke in the comments of the contract above how i gets incremented in the stack after each iteration of the loop, I wanted to see how this gets done so i took the following minimal contract:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

/**
 * @title Owner
 * @dev Set & change owner
 */
contract Owner {

   function testoffsetVal(uint256[] calldata value, uint256 newVal) public pure returns(uint256){
    assembly{
        let sz := value.length
        for{let i :=0} lt(i,sz){
              i := add(i, 1)
        }{
            newVal := add(newVal,1)
        }
    }
   }

}
```

Got the bytecode, removed the contract creation bit, took it to evm.codes playground as usual and added the following calldata:

```
0xb893c5770000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000014000000000000000000000000000000000000000000000000000000000000000500000000000000000000000000000000000000000000000000000000
````


I then looked at what was happening in each iteration and indeed, the loop starts with 0 being put into the stack ,then newVal is incremented by 1, then i is incremented by 1 in the stack and then the same happens again. You can go into evm.codes and see how this works for yourself.


# 8 AUTOMATED CASTING SOLIDITY ERROR, HOW SOLIDITY HANDLES TIMES AS UINT

This is mainly a side note to an issue I saw on X that I feel is definitely worth noting. Look at the following image to see the problematic code block:

![AutomatedCasting1](./AutomatedCasting1.png)

The above function overflow when _multiplier >=28 which doesnt make much sense because 7 days is casted as a uint256 and multiplied by _multiplier which is a uint8 and the result is casted to uint256 which seems alright so where could the overflow come from?

We could guess "well multiplier * 7 days calculation does not fit into uint8 casting so it overflows before outer uint256 casting happens". 
But then, why does it fail at 28 and not 27, or even 1? This calculation would not fit into uint8 casting either.

This overflow happens since Solidity will automatically assign 7 days as a uint24.
Since the smallest uint that the 7 days fits in is uint24, solidity will cast the number in uint24, we can think of it as uint24(604800). The calculation of uint8(_multiplier) * uint24(604800) will be done before the inner uint256 cast of 7 days. As the bigger casting here is uint24, compiler will attempt this multiplication in the context of uint24. So the multiplication will be uint256(uint256(uint24(604800 * _multiplier)))
If _multiplier is >= 28 the result of this multiplication will no longer fit in the uint24 cast. So the function will revert with overflow before the inner uint256 cast is done.
In order to be safe with your code, do not always rely on automated castings done by the compiler and manually cast your values.  The following version of this function would work correctly.

![AutomatedCasting2](./AutomatedCasting2.png)
