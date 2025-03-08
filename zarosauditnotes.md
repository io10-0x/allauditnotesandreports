
# 1 ERC-7201 NAMESPACES STORAGE VARIABLES TO SOLVE STORAGE COLLISIONS IN UPGRADEABLE CONTRACTS

The main reason for learning this is because the zaros docs begin by introducing a tree proxy pattern which they claim to be ERC-7201 compatible. For me to be able to explore what that pattern meant, i needed to know what ERC-7201 did which is why we are learning this. Their docs at: https://docs.zaros.fi/overview/getting-started/tree-proxy-pattern also talk about EIP-1822, EIP-2535, and EIP-7504 which I will also have to grasp before moving forward to understand what this tree proxy pattern is trying to do. This is the core of their whole system so if I dont understand all of this inside out, i cannot perform an adequate audit.

ERC-7201 focuses on solving the storage collision problem in upgradeable implementation contracts. We have already learnt about upgradeable contracts and the storage collision issue in the foundry notes so go have a look at that for a recap .
This rare skills article below perfectly describes erc-7201 as a whole so before you continue reading ths, go have a full read of this article because most of what I will be explaining will stem
from this article. https://www.rareskills.io/post/erc-7201

The first point I want to explain is where the article says "The optimal solution would involve assigning each implementation contract in the inheritance chain its own dedicated storage location.Unfortunately, Solidity currently lacks a native mechanism to do this (a namespace for variables in the contract). Therefore, constructions of this nature must be implemented within the bounds of Solidity and YUL. This can be achieved using structs. "

At this moment, I dont know too much about storage locations as I am yet to take the assembly course but looking into this will give me the background I need before taking the course. What this means though is that as you know, solidity assigns storage slots to variables sequentially for scalar data types like uint256, address and bool. The article covers this and your notes also cover this but doesnt cover it like we are about to cover it. This is the formula that solidity uses to determine storage slots for all data types.

```
Lroot :=root|Lroot + n|keccak256(Lroot)|keccak256(h(k) ⊕ Lroot)
```

The storage layout formula is as follows:

Lroot := root

Lroot is the base storage slot for a contract or variable.
For simple variables (e.g., uint256 x), Lroot represents the storage slot index where the value is stored. Solidity starts assigning slots sequentially from 0.
Lroot + n

This represents sequential storage for scalar types like uint256, address, or bool.
For example:
uint256 a → Slot 0
uint256 b → Slot 1
address c → Slot 2

keccak256(Lroot)

Used for dynamic data types, such as mappings and arrays.
For dynamic types, Solidity calculates the base storage slot using keccak256 of Lroot. This ensures that the dynamic data doesn’t overlap with sequential scalar variables.
Dynamic data types can grow or shrink in size, unlike fixed-size variables such as uint256 or address. To handle this flexibility, Solidity assigns storage slots dynamically rather than sequentially. So keep in mind that sequential storage slots are only used for fixed size variables. For dynamic data types like arrays or mappings, the storage slot is assigned for that data type to a specific loacation and that location is the hash of the base slot. So say we have 2 variables occupying slots 0 and 1 and we have an array next to add to storage. That array length will be stored in slot 2 but the first element of the array will be stored in slot keccak256(2), the second will be at keccak256(2)+1 and so on until all elements of the array are stored.

The reason we use keccak256 for dynamic data types is because even if another mapping is stored after the first mapping, keccak256(3) will be a completely different storage slot to keccak256(2) so its not like they will be remotely close to each other. They are not sequential when using keccak256 to hash them so there is very little risk of overlap.

keccak256(h(k) ⊕ Lroot)
Where:

- **`slot`**: The calculated storage slot for the key-value pair in the mapping.
- **`h(k)`**: The hash of the key (`k`) in the mapping, usually calculated with `keccak256`.
- **`⊕` (XOR)**: The bitwise XOR operation.
- **`Lroot`**: The base storage slot of the mapping, assigned during contract compilation.

Mappings have a slightly different formula as they use the bitwise XOR operator. The circle plus (⊕) symbol in the formula represents the bitwise XOR operation, used to uniquely combine the key (h(k)) and the base slot (Lroot) for mappings in Solidity. So as usual with dynamic arrays, say we have 2 variables in slots 0 and 1, slot 2 will contain the length of the mapping just like arrays do but determining the storage slot for each key-value pair in the mapping is slightly more challenging because you need to make sure that there are no storage collisions and what I mean is that we need to make sure 2 different mapping keys are not stored in the same storage position. So we cannot do what we did with arrays by adding each extra element in the array to keccak256(base slot) + 1 storage slot because unlike arrays, mappings have no order or indexing. So to assign each mapping key to a storage slot. this is what the bitwise XOR operation helps us do.

XOR (Exclusive OR) Operation

XOR Truth Table (Per Bit)

| Input A | Input B | Output (A ⊕ B) |
| ------- | ------- | -------------- |
| 0       | 0       | 0              |
| 0       | 1       | 1              |
| 1       | 0       | 1              |
| 1       | 1       | 0              |

How XOR Works for Larger Numbers

The XOR operation is applied **bitwise** to binary representations of the operands. Each bit is XORed independently using the truth table above.

Example: XOR Two Numbers
Let’s XOR `5` (binary `0101`) and `3` (binary `0011`):

| Bit Position | A (5 in binary) | B (3 in binary) | A ⊕ B |
| ------------ | --------------- | --------------- | ----- |
| 3            | 0               | 0               | 0     |
| 2            | 1               | 0               | 1     |
| 1            | 0               | 1               | 1     |
| 0            | 1               | 1               | 0     |

Result in binary: `0110` (which is `6` in decimal).

---

Solidity Storage Layout for Mappings

Formula Breakdown

The formula for calculating storage slots ensures that each key-value pair in a mapping is stored at a unique location. See example below:

1. **Base Slot (`Lroot`)**:

   - This is the fixed storage slot for the mapping itself, assigned during compilation.
   - It does not change regardless of how many keys are added to the mapping.

2. **Key Hash (`h(k)`)**:

   - The key is hashed using `keccak256` to generate a deterministic value.

3. **XOR Operation (`⊕`)**:

   - Combines the key hash (`h(k)`) with the base slot (`Lroot`) to ensure uniqueness.

4. **Final Slot Calculation (`keccak256`)**:
   - The result of the XOR operation is hashed using `keccak256` to compute the final storage slot for the key-value pair.

---

Example in Solidity

Contract Example:

```solidity
contract Example {
    mapping(address => uint256) public balances;
    mapping(uint256 => string) public data;
}
```

Base Slots (Lroot):

balances is assigned Lroot = 0.
data is assigned Lroot = 1.

Key-Value Pair Slots:

For balances[0x123...], the storage slot is calculated as:
keccak256(h(0x123...) ⊕ 0) which means both the key and the base slot are converted to binary and assessed using the truth table to generate a new binary from the combination of the key hash binary and the binary of the base storage slot. This binary is then hashed using keccak256 to give a unique storage slot for every key and base slot that cannot be overlapped to cause storage collision.

For data[42], the storage slot is calculated as:
keccak256(h(42) ⊕ 1)

Dynamic Slot Calculation:
Each key in the mapping has its own unique storage slot calculated using the formula. This isnt always the case with mappings though. There is a slight difference when mappings are inside structs which we will look at shortly.

Now we understand how storage slots are assigned in solidity, lets get back to the namespaces idea. To avoid storage collisions, what we need is a way to give a namespace to different variables in a particular contract. What i mean is that we need a way to say that uint256 public number variable in a particular contract must go in storage slot 0 unless we overwrite this. This is how namespaces work. You assign a slot to a particular variable that you name. This cannot happen in solidity as mentioned earlier but there are ways around this using structs which is what we are about to have a look at.

So this is essentially what we are trying to achieve by giving each contract its own namespace. We want the root to not always start from 0 for every contract. see image below. In solidity, this isnt possible though as I highlighted earlier and the article also highlights.
![Contract Namespace Image](./contractnamespace.png "Namespace example from rareskills article")

So the proposed way to do this in ERC-7201 was using structs and putting all the vairables you need inside the struct and then changing the slot where the struct is stored using assembly language. See example below:

```solidity
contract StructOnStorage {

        // NO STATE VARIABLES

    struct MyStruct{
        uint256 fieldA;
        mapping(uint => uint) fieldB;
    }

    function setMyStruct() public {
        MyStruct storage myStruct; // Grab a struct

         assembly {
            myStruct.slot := 0x02 // Change its base slot
         }

         myStruct.fieldA = 100; // FieldA will be in the first slot from the base at 0x02, which is 0x02 itself
         myStruct.fieldB[10] = 101; // The storage address of this mapping item will be calculated below
         myStruct.fieldB[12] = 120;
    }

    function getMyStruct() public view returns (uint256 fieldA, uint256 fielbBSingleValue) {

        // keccak256(abi.encode(key, struct base + location inside the struct)
        // The mapping is located in the second slot inside the struct, so struct base + 1
        bytes32 locationSingleValue = keccak256(abi.encode(0x0a, 0x02 + 1));

        assembly {
            fieldA := sload(0x02) // Read storage at 0x02
            fielbBSingleValue := sload(locationSingleValue)
        }
    }
}
```

This is very straightforward and the article goes through what is happening here. The article says that "Hypothetically, if we declared this struct as the first storage variable (which ERC-7201 does not do), fieldA will be in slot 0, the base of the fieldB mapping will be in storage 2, and so forth."

What this means is that say in the contract, we typed MyStruct storage myStruct where we would normally have state variables. If we did this, the fieldA would be assigned storage slot 0 as solidity normally does. In setMyStruct, that is where we declare the struct and this is different to declaring a state variable because whatever we declare in a function only lasts for the duration of that function. In this case though, what we are going to do will persist. What I mean is that in the assembly command, what we did was to assign myStruct to storageslot 2 (0x02).
This will persist in the contract so at storage slot 2 of this contract, that slot will always be taken up by that struct. so fieldA will be in slot 2, the base of the fieldB mapping will be in storage 2, and so forth. Since all our variables in the contract are in that struct, we have essentially set the contracts root slot to 2 which we couldnt do naturally with solidity.

Another thing to note is that in the getMyStruct function, we want to get the storage slot of fieldB[10]. We learnt that the base slot of the mapping will be storage slot 3 as we said above. Normally with state variables, as we discussed earlier, each key of the mapping would be stored at slot keccak256(h(k) ⊕ Lroot). Notice how when getting the storage slot for the mapping key fieldB[10] at field B, we dont use the xor operator to get the storage slot when the mapping is inside a struct. The reason is because since the mapping is in a struct, the formula used to declare each keys storage slot is slightly different. It is:keccak256(abi.encode(k,base slot+offset)).

So the takeaway point is :

Mappings directly declared in a contract (not inside structs) use:
keccak256(h(k)⊕L)
Here, L is the base slot of the mapping and k is the mapping key.

Mappings nested within structs calculate the storage slot as:
keccak256(abi.encode(k,base slot+offset))
Here, K is the hexadecimal of the mapping key, the base slot is the slot we set for the struct in setMyStruct and the offset is the index of the data type in the struct.
This avoids ambiguity caused by struct offsets and simplifies layout management.

This is the same behaviour regardless of if the mapping is declared as a state variable or not. This is what you see in the example contract above. Since the mapping is in a struct, the storage slot of each key in the mapping is calculated with keccak256(abi.encode(k,base slot+offset)) which is what locationsinglevalue in the contract above gets. All of this makes sense.

In the above example, we randomly assigned a new base slot to this struct but in reality, we wouldnt just do that. There is a proposed formula in ERC-7201 used to assign a slot for a namespaced contract (the struct that contains all the proposed variables for the contract). This formula is :

keccak256(keccak256(namespace) - 1) & ~0xff

Breaking Down the Formula
keccak256(namespace):

The namespace is a unique identifier (usually a string like "my.namespace").
Taking the keccak256 hash of this namespace generates a unique base storage slot for the contract's data.
Example:

```solidity
bytes32 namespace = keccak256("my.namespace");
```

keccak256(namespace) - 1:

Subtracting 1 from the hash ensures that the resulting slot cannot easily be linked back to its namespace preimage. A preimage is simply the input that produces the hash. So in the above example, my.namespace is the preimage. This provides an additional layer of security because someone knowing only the namespace cannot easily guess the root storage slot. If they know the formula subtracts 1 and they also know the namespace, this step makes no difference and since this formula is public knowledge, there is no real benefit of adding this step.

keccak256(keccak256(namespace) - 1):

The second keccak256 hash ensures conflict avoidance:
Storage slots for dynamic variables (e.g., mappings and arrays) are determined using a keccak256 hash in Solidity as we have seen earlier.
By taking an additional hash here, we significantly reduce the likelihood of accidentally colliding with those slots.

& ~0xff:

The bitwise AND NOT operation (& ~0xff) clears the last byte of the resulting hash, effectively setting the rightmost 8 bits to 0. You dont really have to know what the bitwise and not operator does in depth but just keep this mind.This ensures that the resulting storage location is aligned to 256-slot boundaries. This has two advantages:
Future Verkle Trees Compatibility: In a future Ethereum upgrade, storage access might be optimized for 256-slot "chunks". Aligning to these chunks ensures efficient access.
Namespace Isolation: Each namespaced contract gets a contiguous block of storage slots, with no risk of overlap.

So any contracts using ERC-7201 will determine their new root storage slot using this formula so bear that in mind whenever you see it.

With this, now you understand most of what there is to know about ERC-7201 so when you look at the zaros tree proxy pattern which claims to be ERC-7201 compatible, you will understand fully what that means and how it is implemented. This is a very important ERC and will help you loads going forward.

# 2 EIP-1822 UUPS Universally Upgradeable Smart Contracts

We have covered universally upgradeable smart contracts in your foundry web3 dev notes so have a look there at how it works but if you want to read the official EIP, this is the link https://eips.ethereum.org/EIPS/eip-1822 . We never looked at the official EIP in the course so i didnt know it was 1822 but for your own reading, you can check this out in your own time. For now, we already know what this does so no need to go into it again.

# 3 EIP-2535 Diamond Standard Multi Faceted Proxy

This standard was briefly spoken about in the foundry web3 course but we are going to study the EIP now to understand exactly what a diamond proxy is and how to use it. I will be using 2 links to discuss how diamond standard proxies work. See below:
https://eips.ethereum.org/EIPS/eip-2535
https://www.quicknode.com/guides/ethereum-development/smart-contracts/the-diamond-standard-eip-2535-explained-part-1
https://www.quicknode.com/guides/ethereum-development/smart-contracts/the-diamond-standard-eip-2535-explained-part-2

Essentially, a diamond proxy is a single proxy that points to multiple implementation contracts. these implementation contracts are known as facets. Dont worry about the naming convention. it isnt that relevant in this case. These facets are connected to libraries that define the namespace for that contract in the proxy contract as you will see in the example below.

The Diamond Standard is an improvement over EIP-1538. It has the same idea: To map single functions for a delegatecall to addresses, instead of proxying a whole contract through.The important part of the Diamond Standard is the way storage works. Unlike the unstructured storage pattern that OpenZeppelin uses, the Diamond Storage is putting a single struct to a specific storage slot.Function wise it looks like this, given from the EIP Page:

```solidity
// A contract that implements diamond storage.
library LibA {

  // This struct contains state variables we care about.
  struct DiamondStorage {
    address owner;
    bytes32 dataA;
  }

  // Returns the struct from a specified position in contract storage
  // ds is short for DiamondStorage
  function diamondStorage() internal pure returns(DiamondStorage storage ds) {
    // Specifies a random position from a hash of a string
    bytes32 storagePosition = keccak256("diamond.storage.LibA")
    // Set the position of our struct in contract storage
    assembly {ds.slot := storagePosition}
  }
}

// Our facet uses the diamond storage defined above.
contract FacetA {

  function setDataA(bytes32 _dataA) external {
    LibA.DiamondStorage storage ds = LibA.diamondStorage();
    require(ds.owner == msg.sender, "Must be owner.");
    ds.dataA = _dataA
  }

  function getDataA() external view returns (bytes32) {
    return LibDiamond.diamondStorage().dataA
  }
}
```

Having this, you can have as many LibXYZ and FacetXYZ as you want, they are always in a separate storage slot as a whole, because of the whole struct. To completely understand it, this is stored in the Proxy contract that does the delegatecall, not in the Faucet itself. That's why you can share storage across other faucets. Every storage slot is defined manually (keccak256("diamond.storage.LibXYZ")).As you can see, everything we learnt from ERC-7201 becomes very useful here. The same way the namespace was used in the struct is the same way it is used in the library to create the namespace for facetA. so when the diamond contract sets facet A as one of its implementation addresses, all variables that are relevant to facet A will have their specific storage slots in the diamond proxy contract which prevents the storage collision problem. This allows us to have as many implementation contracts (facets) as we want and we can upgrade any of them at any time we want. The best way to visualize this would be to look at some code that implements the diamond proxy and there is no better way to do this than from the person who made the standard. The main code is at https://github.com/mudgen/diamond-1-hardhat/tree/main . I have cloned this repo in the securityresearchcyfrin directory under diamond-1-hardhat and performed assumption analysis on every line of code to really drill down how this process works and also how to deploy, etc. This will be the complete guide on how to use diamond proxies but in terms of notes, there are no new concepts here that we dont know about so all you need to do is go over to that file and understand how it all meshes together.

The explantion i gave here and in the above links are quite straightforward but in the directory as you will see, it is a lot more complicated than I have explained it here. The idea is still the same but it just takes a lot of steps to implement which is why a lot of people dont like the diamond standard and have come up with other solutions like the one below and the tree proxy format used by zaros.

# 4 EIP-7504 Improvement Proposal for Diamond Proxies

Again, there isnt too much new stuff covered here. This EIP simply suggests a better way to work with diamond proxies so it takes the diamond proxy concept and attempts to make improvements on it. You can read about it and implement it yourself at https://github.com/ethereum/EIPs/blob/e1f6b9e5b8d1d56f77bb61e605ddf04f231998e2/EIPS/eip-7504.md . This is pretty straightforward once you know how a diamond proxy works. I do strongly suggest you have a read through this at least and make sure you understand fully what is going on. In terms of zaros, fully knowing this EIP isnt necessary as it uses it for their tree proxy pattern so all i need to understand is what EIP-7504 actually does which i do and I should be able to understand the tree proxy pattern. The only concept that was absolutely crucial to understand was EIP-7201 which their proxy pattern is designed to be compatible with and EIP-2535 which is the diamond proxy which their tree proxy is trying to improve on just like EIP-7504 aims to do. Anything extra is additional learning which I will do but isnt entirely useful right now.



# 5 CHAINLINK AUTOMATION DOS

The motivation for this exploit came from a tweet from giovanni di sienna from cyfrin team. The exploit is based on how protocols can DOS themselves when using chainlink automation. If you see this out in the wild, you know to point it out. This can be important.

So, say you have some piece of critical functionality that acts as a trigger for off-chain computation via Functions, such as Chainlink Automation. In the scenarios where you don't want a new request to be executed until the most recent request has been fulfilled, you might have something that looks like this:
![Chainlink Automation DOS Image](./chainlinkautomationdos.png "Chainlink Automation DOS Example")

The above image shows a typical example of a protocol using some logic to make sure that one request to the chainlink automation must finish before another one starts. this is done by checking in performUpkeep function if (lastRequestId == bytes32(0)) and then in the fulfilRequest function at the end of the request, setting last request equal to bytes32(0). This means at the end of every request fulfillment, lastrequestid is reset to make sure that when performupkeep is called by chainlink automation, the if statement will be true if the last request has been fully completed.

We have covered chainlink automation in the foundry course so you should already be familiar with performupkeep and how all of this works. The assumption is that the triggerrequest function gets the request id and resets the last request id and then calls fulfillrequest. 

It is VERY IMPORTANT to note that:

1️⃣ Only ever one of response or err will be assigned a non-empty value in the fulfillrequest function.

2️⃣ The executing DON will not retry a fulfilment if it fails. Your subscription will be billed and that will be that.

This means that you really don't want any of the fulfilment logic to revert for valid request ids. This is especially important if your core business logic depends on some logic being executed at the end of fulfilment, such as lastRequestId as shown above.

⚠️ HOWEVER!!! ⚠️ There are of course a number of ways that this could happen:

- If err contains non-empty bytes then it might be tempting to bubble-up the revert in the fulfillment logic – don't!

- If err contains non-empty bytes then you absolutely must also remember that any attempt at decoding response will revert because it is empty.

- If the actual fulfilment business logic reverts, execution will revert. This includes external calls, but also don't forget about any internal/private functions!

🔧 So here are the steps to avoid it:

- If err contains non-empty bytes then you should not simply revert but rather handle the error gracefully, falling through to any cleanup logic that needs to be executed at the end.

- Validate response against its expected length to ensure that the decoding does not revert.

- Handle reverts from all external calls using try/catch blocks.

- Unless you are absolutely certain you have covered all bases, consider refactoring the internal fulfilment logic into public function(s). This way, you can call it externally with this.doFulfillment() and handle it the same as any other external call.

These are very important tips you should look out for when auditing any protocol that uses chainlink keepers (automation).




# 6 ISSUE COMPILING LARGE PROJECTS IN WSL

with zaros, i was having a lot of issues compiling the project as they had over 400 files to compile and everytime i tried to run forge build, after waiting about 20 minutes, the process would terminate with the following error: solc exited with signal: 9 (SIGKILL) <empty output>

The reason was because the allocated cpu and memory in wsl when compiling is being overloaded and it eventually runs out of memory when compiling and kills the process automatically. To see this, when its compiling, open a new terminal in wsl and run htop. you will see the cpu and memory usage that the process is using and after a while you will see it take up all the allocated wsl memory. Usually, when all the memory is being taken up by a process, some of the load is sent to a swap file and in htop, you will see how much load is being sent to the swap file. The reason the compile was failing was because even the swap file memory was being fully eaten up by the compilation. To fix this, i had to allocate more memory to wsl as a whole and to the swap file. To do this, I had to create a .wslconfig file in the C://Users/<YourUsername> directory. I ran into an issue while doing this though. I tried to create the file directly using the UI and this created the .wslconfig as a text file so when i configured wsl, and restarted it, the memory allocations and all the settings i put didnt change in wsl which had me confused. The problem was that the config file cannot be a text file as that wont be recognised by wsl so i had to create the file another way to make it have the correct format. See instructions below:

Open your WSL terminal:
```bash
wsl
```
Navigate to your Windows home directory:
```bash
cd /mnt/c/Users/<YourUsername>
```
Use nano to create or edit the .wslconfig file:
```bash
nano .wslconfig
```
Add the following configuration and save:

```
[wsl2]
#Limits VM memory to use no more than 4 GB, this can be set as whole numbers using GB or MB
memory=8GB
#Sets the VM to use two virtual processors
processors=8
swap=4GB
```
Press CTRL + O, then Enter to save.
Press CTRL + X to exit.

Doing this and restarting wsl should allocate the settings in the config and if you run htop in wsl, you should see the correct allocations and after about 25 mins, the project should compile.

# 7 TALK ABOUT THE WAY ZAROS UPGRADE THEIR BRANCHES AND LEAVES AS THERE IS NO AUTHORISE UPGRADE IN ANY OF THE BRANCH CONTRACT BUT THEY HAVE AN UPGRADEBRANCH IN SRC/TREEPROXY/BRANCHES SO ONCE YOU HAVE LEARNT ABOUT IT, COME HERE AND TALK ABOUT IT. ALSO TALK ABOUT THE ADVANTAGES OF THE TREE PROXY PATTERN OVER DIAMOND STANDARD. IMPORTANCE OF UNDERSTANDING THE DIAMOND STANDARD

I will go on to talk about the differences between diamond standard and the tree proxy standard although there arent many but I want to quickly touch on the importance of really understanding the diamond proxy pattern as it not only helped you here but can help you solve a lot of problems going forward as a lot of protocols will be using this standard. How it helped me was that I found a potential high in the codebase and atttempted to write a POC for it which was in the deposit.t.sol file and looked like this :

```solidity
function test_creditdelegationweight(
        uint128 vaultId,
        uint128 assetsToDeposit,
        bool depositFeeZero
    )
        external
        whenDepositedAssetsAreNotZero
        whenWhitelistIsDisabledOrUserIsAllowed
        whenVaultDoesExist
        whenTheDepositFeeIsZeroOrNot
        whenTheDepositCapIsNotReached
        whenSharesMintedAreMoreThanMinAmount
    {
        // ensure valid vault and load vault config
        vaultId = uint128(bound(vaultId, INITIAL_VAULT_ID, FINAL_VAULT_ID));
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);

        uint128 depositFee = depositFeeZero ? uint128(0) : uint128(vaultsConfig[fuzzVaultConfig.vaultId].depositFee);

        _setDepositFee(depositFee, fuzzVaultConfig.vaultId);

        // ensure valid deposit amount
        address user = users.naruto.account;
        assetsToDeposit =
            uint128(bound(assetsToDeposit, calculateMinOfSharesToStake(vaultId), fuzzVaultConfig.depositCap));
            console.log(fuzzVaultConfig.depositCap);
        deal(fuzzVaultConfig.asset, user, 100e18);

      

        marketMakingEngine.workaround_Vault_setTotalCreditDelegationWeight(vaultId, 1e10);

        // perform the deposit
        vm.startPrank(user);
        marketMakingEngine.deposit(vaultId, 1e18, 0, "", false);
        marketMakingEngine.deposit(vaultId, 1e18, 0, "", false);
        vm.stopPrank();



    //confirm the marketids that are connected to the vault
       uint128[] memory latestconnectedmarkets = marketMakingEngine.workaround_Vault_getConnectedMarkets(vaultId);

       uint128 sumofindividualcreditdelegatedweight;

    //get the creditdelegation from each storage slot for each market
    /*for (uint i = 0; i < latestconnectedmarkets.length; i++) {
        uint128 marketId = latestconnectedmarkets[i];
        CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId);
        console.log(creditDelegation.weight);
        actualtotalcreditdelegatedweight += creditDelegation.weight;
    } */

   //c In the commented out loop, this was the first loop i tried to run to get the creditdelegation weight from the credit delegation leaf data struct. On first glance, this looks good but when I ran it, all the logs for the weight of each market kept returning 0 and I was wondering why. The logs of the test showed that when the deposit function was called and it called updateVaultAndCreditDelegationWeight, I added some logs in that function to make sure that it was updating the weight and indeed it was. So why when i tried to get the weight from the creditdelegation leaf, in the above loop, it was returning 0?= for each connectedmarket ? I then looked at other parts of this test file to see how storage slots were being loaded as I thought maybe I was loading it wrong. This was where I noticed the issue which was that in the loop, I called CreditDelegation.Data storage creditDelegation = CreditDelegation.load(vaultId, marketId); randomly and although this points to the correct storage slot, it doesnt point to the correct storage slot in trhe context of the proxy contract and as you know about ugradeable contracts, all the storage isnt in any of the implementations, it is in the proxy. The proxy contracts always called different workaround functions from harnesses and these harnesses like vaultharness.sol for example, simply does what I did above but the difference is that it puts it in a function and the proxy engine contract calls this function so when the proxy calls this, it loads the storage slot in the proxy contract which is where all the action happens. So I went to the vaultharness.sol contract (coudlve made my own harness but chose not to), added a function called workaround_CreditDelegation_getweight which simply did what I wanted to do in the above loop but now i can call it in the context of the proxy and retrieve the correct data. So i refactored the loop below:

   for (uint i = 0; i < latestconnectedmarkets.length; i++) {
        uint128 marketId = latestconnectedmarkets[i];
        uint128 weight = marketMakingEngine.workaround_CreditDelegation_getweight(vaultId, marketId);
        sumofindividualcreditdelegatedweight += weight;
    }
//c when i first ran the above loop, i got the following custom error UnsupportedFunction(0x96a85592). So I ran Cast Sig and saw that it was this new function in the loop which I added to the VaultHarness, and the reason I added it was obviously to be able to access that storage slot in the proxy contract and get the weights I needed. Since it was a custom error, I knew it would be defined somewhere in this repo so ran Ctrl + Shift + F to search all files for that error, and I saw it was used in a RootProxy contract, which was inherited by MarketMakingEngine.sol. So I knew it had something to do with the proxy. After searching for what this root proxy contract does via gpt, I saw that all the selectors of every function called by MarketMakingEngine.sol are registered, which is standard practice for diamond proxy contracts. So I knew that the workaround_CreditDelegation_getweight function I added in vaultharness.sol wasn’t registered in the proxy contract. Looking in the Base.t.sol, I saw a getMarketMakerBranchesSelectors function, which was used to do the registering. This function interacted with a TreeProxyUtils.sol file, where all function selectors of all functions that MarketMakingEngine.sol calls are registered. So, I simply went to the vault harness registered functions in that contract, increased the size of the array that stored the selectors by 1,  added the function selector of the workaround_CreditDelegation_getWeight I created in the VaultHarness, and ran the test again. It worked.

    console.log(sumofindividualcreditdelegatedweight);

    uint128 postdeposittotalcreditdelegationweight= marketMakingEngine.workaround_Vault_getTotalCreditDelegationWeight(vaultId);

   assertGt(sumofindividualcreditdelegatedweight,  postdeposittotalcreditdelegationweight); //c this assert proves the bug that the sum of each weight given to each connected market is more than the total credit delegated weight of the vault. this is a bug because it means that one market with extremely high activity can take up all the credit delegated weight of the vault and other markets wont be able to get any credit delegated weight. this is a bug because the total credit delegated weight of the vault should be the sum of all the credit delegated weight of each market connected to the vault. well it doesnt fully prove it, i still need to write another test to prove this.



   assert( postdeposittotalcreditdelegationweight != 2e18); //c this assert checks the state as is so with the new bug i found that when a user deposits and recalculatevaultscreditcapacity is called, it doesnt recalculate the vaults capacity based on the deposit the user is about to make. it doesnt actually recalculate anything. it just calculates the credit based on the last deposit and this test is proof of this because after 2 deposits above are made, the totalcreditdelegatedweight should be 2e18 but it isnt. it is 1e18 but the vault has 2e18 worth of assets in it. 
     
        
    }

```
If you read all the comments on this POC, you will understand the importance of understanding the diamond standard and all its variations.

# 8 CUSTOM DATA TYPES IN SOLIDITY

Auditing zaros was the first time I came across custome data types ans how they are configured. When looking at the vaultrouterbranch contract, i saw a DepositContext struct containing a UD60x18 data type I had never seen before so I decided to look into it. I made all relevant notes in the valuetype.sol file related to ud60x18 data type to explain in detail how custom types work and what ud60x18 intends to acheive. See the contents of the file below. All the comments I made were to explain what was going on and you will learn loads from this. I do intend to go deeper in the files to explore more if i get some time but from the comments, you should get a good understanding of whats going on:
 ```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.19;

import "./Casting.sol" as Casting;
import "./Helpers.sol" as Helpers;
import "./Math.sol" as Math;

/// @notice The unsigned 60.18-decimal fixed-point number representation, which can have up to 60 digits and up to 18
/// decimals. The values of this are bound by the minimum and the maximum values permitted by the Solidity type uint256.
/// @dev The value type is defined here so it can be imported in all other files.
type UD60x18 is uint256;
//c the new type UD60x18 is defined and wraps a uint256 data type. Once this custom type is defined on an existing data type , it automatically comes with a wrap and unwrap function which can be used to wrap and unwrap the data type to and from the custom type. Once you create a new custom type though, it inherits the behaviour of the data type it wraps. What Is Inherited:
//Storage Representation: The custom type UD60x18 is stored in memory and storage as a uint256. There is no difference in how the data is stored compared to a normal uint256.
//Value Range:Since the underlying type is int256, the range of values for SD59x18 is the same as int256, i.e., from -2^255 to 2^255 - 1.
//Default Behavior:Without additional libraries or custom functions, the type behaves exactly like a standard uint256. This means it can use arithmetic operations like addition, subtraction, multiplication, and division in the same way.

//c what actually makes this type different from the uint256 data type is the fact that it comes with a set of functions that can be used to perform mathematical operations on the data type. This is done with the using for global directive you can see below. This directive makes the functions in the library callable on the UD60x18 type. These are the only things that make this type different from the uint256 data type.

//c lets talk about what this custom type actually does. from its name, we can infer what the intended behaviour is. it is called UD60x18 and the description above says The unsigned 60.18-decimal fixed-point number representation, which can have up to 60 digits and up to 18 decimals. The values of this are bound by the minimum and the maximum values permitted by the Solidity type uint256. What this is trying to tell you is that whatever value you enter as a UD60x18 is treated as if it has 18 decimal places. So lets use an example to drill this point home. Say we have UD60x18 fixedPointValue = UD60x18.wrap(2e18); In solidity terms, these are really the same value of 2e18 but what makes it different is the way it is treated based on all the functions added to UD60x18 below. As i said, the aim of UD60x18 is to treat the value as if it has 18 decimal places. The best way to understand this is when you multiply 2e18 by 2e18 as a uint, you know you will get 4e36 as that is the natural multiplcation but with UD60x18, we will get 4e18 because the 18 0's are treated as decimal places so essentially we are doing 2 *2 which is 4 and then we add 18 zeros to the right of the 4 to make it 4e18. This is the essence of UD60x18. To see this, go into the math.sol file and look at the mul function. You will see that it multiplies the two values and then divides by 1e18 to get the correct value. it looks like this:

/* /// @notice Multiplies two UD60x18 numbers together, returning a new UD60x18 number.
///
/// @dev Uses {Common.mulDiv} to enable overflow-safe multiplication and division.
///
/// Notes:
/// - Refer to the notes in {Common.mulDiv}.
///
/// Requirements:
/// - Refer to the requirements in {Common.mulDiv}.
///
/// @dev See the documentation in {Common.mulDiv18}.
/// @param x The multiplicand as a UD60x18 number.
/// @param y The multiplier as a UD60x18 number.
/// @return result The product as a UD60x18 number.
/// @custom:smtchecker abstract-function-nondet
function mul(UD60x18 x, UD60x18 y) pure returns (UD60x18 result) {
    result = wrap(Common.mulDiv18(x.unwrap(), y.unwrap()));
} */

//c notice how it calls a mulDiv18 function from the common library and in that function, all that happens is that the 2 values are multiplied and divided by 1e18 to keep with the UD60x18 format. It does a lot of assembly stuff to do that but you can read more into how the assembly works but on a high level, this is what happens. This is a new concept but it is very important to understand how it works. It is a very important concept in the zaros codebase. Once contest is over, i do encourage you to have a play around with other functions imported into this custom type and see if you can break them.

//c the description also says that The values of this are bound by the minimum and the maximum values permitted by the Solidity type uint256. this is done automatically as it inherits from a uint256 as we discussed earlier. You can see it used to limit some inherites functions if you go into the constants.sol file and you will see a constant MAX_UD60x18 set to the maximum value of a uint256. You can also see where it is used in the math.sol file to enforce it on some functions.



/*//////////////////////////////////////////////////////////////////////////
                                    CASTING
//////////////////////////////////////////////////////////////////////////*/

using {
    Casting.intoSD1x18,
    Casting.intoSD21x18,
    Casting.intoSD59x18,
    Casting.intoUD2x18,
    Casting.intoUD21x18,
    Casting.intoUint128,
    Casting.intoUint256,
    Casting.intoUint40,
    Casting.unwrap
} for UD60x18 global;

/*//////////////////////////////////////////////////////////////////////////
                            MATHEMATICAL FUNCTIONS
//////////////////////////////////////////////////////////////////////////*/

// The global "using for" directive makes the functions in this library callable on the UD60x18 type.
using {
    Math.avg,
    Math.ceil,
    Math.div,
    Math.exp,
    Math.exp2,
    Math.floor,
    Math.frac,
    Math.gm,
    Math.inv,
    Math.ln,
    Math.log10,
    Math.log2,
    Math.mul,
    Math.pow,
    Math.powu,
    Math.sqrt
} for UD60x18 global;

/*//////////////////////////////////////////////////////////////////////////
                                HELPER FUNCTIONS
//////////////////////////////////////////////////////////////////////////*/

// The global "using for" directive makes the functions in this library callable on the UD60x18 type.
using {
    Helpers.add,
    Helpers.and,
    Helpers.eq,
    Helpers.gt,
    Helpers.gte,
    Helpers.isZero,
    Helpers.lshift,
    Helpers.lt,
    Helpers.lte,
    Helpers.mod,
    Helpers.neq,
    Helpers.not,
    Helpers.or,
    Helpers.rshift,
    Helpers.sub,
    Helpers.uncheckedAdd,
    Helpers.uncheckedSub,
    Helpers.xor
} for UD60x18 global;

/*//////////////////////////////////////////////////////////////////////////
                                    OPERATORS
//////////////////////////////////////////////////////////////////////////*/

// The global "using for" directive makes it possible to use these operators on the UD60x18 type.
using {
    Helpers.add as +,
    Helpers.and2 as &,
    Math.div as /,
    Helpers.eq as ==,
    Helpers.gt as >,
    Helpers.gte as >=,
    Helpers.lt as <,
    Helpers.lte as <=,
    Helpers.or as |,
    Helpers.mod as %,
    Math.mul as *,
    Helpers.neq as !=,
    Helpers.not as ~,
    Helpers.sub as -,
    Helpers.xor as ^
} for UD60x18 global;

 ```

# 9 INTERNAL VS EXTERNAL LIBRARIES 

The insiparation from this came when looking at base.t.sol , I will show an example. in base.t.sol, there is tis function which is called:
```solidity
function connectMarketsAndVaults() internal {
        uint256[] memory perpMarketsCreditConfigIds =
            new uint256[](FINAL_PERP_MARKET_CREDIT_CONFIG_ID - INITIAL_PERP_MARKET_CREDIT_CONFIG_ID + 1);

        uint256[] memory vaultIds = new uint256[](FINAL_VAULT_ID - INITIAL_VAULT_ID + 1);

        uint256 arrayIndex = 0;

        for (uint256 i = INITIAL_PERP_MARKET_CREDIT_CONFIG_ID; i <= FINAL_PERP_MARKET_CREDIT_CONFIG_ID; i++) {
            marketMakingEngine.workaround_updateMarketTotalDelegatedCreditUsd(uint128(i), 1e10);
            perpMarketsCreditConfigIds[arrayIndex++] = i; 
        }

        arrayIndex = 0;

        for (uint256 i = INITIAL_VAULT_ID; i <= FINAL_VAULT_ID; i++) {
            vaultIds[arrayIndex++] = i;
        }

        marketMakingEngine.connectVaultsAndMarkets(perpMarketsCreditConfigIds, vaultIds);
    }
```
 This function calls connectvaultsandmarkets which is in a marketmakingengineconfigbranch contract. see function below:
```solidity

    /// @notice Configure connected vaults on market
    /// @dev Only owner can call this function
    /// @param marketIds The market ids.
    /// @param vaultIds The vault ids.
    function connectVaultsAndMarkets(uint256[] calldata marketIds, uint256[] calldata vaultIds) external onlyOwner {
        // revert if no marketIds are provided
        if (marketIds.length == 0) {
            revert Errors.ZeroInput("connectedMarketIds");
        }

        // revert if no vaultIds are provided
        if (vaultIds.length == 0) {
            revert Errors.ZeroInput("connectedVaultIds");
        }

        // define array that will contain a single vault id to recalculate credit for
        uint256[] memory vaultIdToRecalculate = new uint256[](1);

        // iterate over vault ids
        for (uint128 i; i < vaultIds.length; i++) {
            // if vault has connected markets
            if (Vault.load(vaultIds[i].toUint128()).connectedMarkets.length > 0) {
                // cache vault id
                vaultIdToRecalculate[0] = vaultIds[i];

                // recalculate the credit capacity of the vault
                Vault.recalculateVaultsCreditCapacity(vaultIdToRecalculate);
            }
        }

        for (uint256 i; i < marketIds.length; i++) {
            _configureMarketConnectedVaults(marketIds[i].toUint128(), vaultIds);
        }

        // perform state updates for markets connected to each market id
        for (uint256 i; i < vaultIds.length; i++) {
            _configureVaultConnectedMarkets(vaultIds[i].toUint128(), marketIds);
        }
    }

```

for now, dont worry about what any of these functions do. What you should focus on is the fact that this function calls vault.load and we know that the load function points the data struct of the vault leaf to a particular base slot. This made sense to me but my next idea was to look for where the vault leaf was deployed because the assumption was that for the vault leaf to be able to assign storage slots, it must be deployed to the blockchain just like any other contract would. I didnt find anywhere in base.t.sol where the vault leaf library was deployed. Tp take it further, i didnt find where any leaf library wwas deployed. The question now became how can we assign storage slots from a file that was never deployed. This is because all the leaves are not contracts. They are internal libraries and internal libraries dont have to be deployed. Once the contract to be deployed imports the leaf library and calls the function, the function is executed in the context of the contract that imported it. So whichever contract (branch according to zaros tree proxy pattern) inherits a leaf internal library is what is assigning that storage slot and not the library itself so there is no need to ever deploy the library.This is a very important concept to understand. Lets see what the difference between internal and external libraries are.

 Solidity Libraries: Internal vs External

Solidity libraries do not always need to be deployed. Whether a library is deployed depends on how the library is used:

**1. Internal Libraries (Not Deployed)**

- If all the functions in a library are marked as `internal`, the library code is **inlined** into the contracts that use it during compilation.
- This means that the library is not deployed as a separate contract, and its code is included directly in the bytecode of the contract that uses it.

**Example:**
```solidity
library Math {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        return a + b;
    }
}

contract Example {
    using Math for uint256;

    function sum(uint256 a, uint256 b) external pure returns (uint256) {
        return a.add(b);
    }
}
```
In this case, the Math library is not deployed separately; its code is embedded in the Example contract.

External Libraries (Deployed)
If a library contains public or external functions, it must be deployed as a separate contract. Contracts that use the library will call its functions via delegatecall.
The library is deployed once, and its address is linked to the contract during deployment.
This makes it possible to share the library's code across multiple contracts, reducing the overall deployment cost for the ecosystem.
Example:
```solidity

library ExternalMath {
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
}

contract Example {
    using ExternalMath for uint256;

    function sum(uint256 a, uint256 b) external pure returns (uint256) {
        return a.add(b);
    }
}
```
In this case:

ExternalMath is deployed as a separate contract.
The Example contract uses the address of ExternalMath to delegate calls to its functions.


# 10 STORAGE POINTERS IN SOLIDITY

The inspiration for this came from looking at the recalculateVaultsCreditCapacity of vault.sol internal leaf library which contained the following line:
```solidity
 EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1] 
 ```

 The line also contained this natsepc "loads the connected markets storage pointer by taking the last configured market ids uint set". This had me confused because I was like how does assigning a new variable called connectedmarkets point to the storage slot of the self.connectedMarkets[connectedMarketsConfigLength - 1] array. Keep in mind that self.connectedMarkets is an array of array. Read the comments in the vaultrouterbranch deposit function and the recalculateVaultsCreditCapacity of vault.sol internal leaf library to know more about the inner workings of this. So i was like how does assigning a variabe here point to the storage of self.connectedMarkets[connectedMarketsConfigLength - 1] array. I thought the only way to do this was using assembly like we did earlier in the ERC-7201 example to assign the struct to a storage slot. This is where I learnt about the storage keyword in solidity and how it works. See below:

In Solidity, when you declare a variable using the storage keyword and assign it a value from another storage-based variable (like self.connectedMarkets), it acts as a pointer to the same storage location. This happens automatically without requiring explicit use of assembly or manual manipulation of slots.

How Storage Pointers Work
When you write:

```solidity

EnumerableSet.UintSet storage connectedMarkets = self.connectedMarkets[connectedMarketsConfigLength - 1];
```
What happens internally is:

self.connectedMarkets[connectedMarketsConfigLength - 1] references a specific UintSet (enumerableset.sol, read comments in vault.sol and vaultrouterbranch.sol to recap enumerableset if you want to know more) stored in self.connectedMarkets at the given index. By assigning it to connectedMarkets with the storage keyword, Solidity does not create a new copy. Instead, connectedMarkets becomes a pointer to the same storage location as self.connectedMarkets[connectedMarketsConfigLength - 1].
Any modifications to connectedMarkets directly affect the underlying data in self.connectedMarkets.

Solidity’s storage keyword handles the pointer logic for you automatically. When accessing an array or a mapping, Solidity internally calculates the storage slot for the data. For example:

For self.connectedMarkets[connectedMarketsConfigLength - 1], Solidity calculates:
keccak256(abi.encode(index, baseSlot))
Where:
index is connectedMarketsConfigLength - 1. baseSlot is the storage slot of self.connectedMarkets which we know because we know that self.connectedMarkets is an array which contains arrays so self.connectedMarkets[connectedMarketsConfigLength - 1] is also an array. In solidity, the array length is derived by keccak256(baseslot). so since self.connectedMarkets[connectedMarketsConfigLength - 1] is located inside another array, its storage slot is determined using keccak256(abi.encode(index, baseSlot)) where abi.encode simply concatenates the index and the base slot as we have learnt in hardhat courses. Once the slot is determined, assigning the data to a storage variable creates a reference to the same slot.


If you had used the memory keyword instead of storage:

```solidity
EnumerableSet.UintSet memory connectedMarkets = self.connectedMarkets
[connectedMarketsConfigLength - 1];
```
This would create a copy of the data in memory. Any changes to connectedMarkets would not affect the original self.connectedMarkets. However, with storage, Solidity ensures that changes to connectedMarkets are reflected in self.connectedMarkets because both point to the same underlying storage.

# 11 ENUMERABLESET.SOL, ENUMERABLEMAP.SOL AND ITS IMPORTANCE IN GAS SAVING AND POTENTIAL EXPLOITS

If you see any protocols that deals with arrays and mappings, it is best to advise them to use enumerableset.sol as it offers benefits to using a normal array for reasons I highlighted in one of the comments in VaultRouterBranch::deposit and Market::getCreditDepositsValueUsd. Go through and read the comments as there is a lot of insightful stuff on enumerable sets there and the reasons why it is preferred to vanilla maps and arrays. Any protocols that use vanilla maps or arrays may be vulnerable to exploits as they have to peform different actions with mappings and arrays in each function which can lead to bugs. Also, enum sets from openzeppelin just makes working with arrays a lot easier so keep that in mind. Make sure to read the comments left in the files I referenced as that is where most of the infromation is.


# 12  INTRODUCTION TO SEQUENCERS AND PRICE ADAPTERS

I have also learnt a little about sequencers which I did when writing another POC and ran into an error and had to debug. I have put all the learning in the commments under the POC and I will paste it below.
```solidity
 function testFuzz_realizeddebtisdoublecounted(
        uint256 marketId,
        uint256 amount,
        uint256 amounttodepositcredit,
        uint128 vaultId
    )
        external
    {

        amounttodepositcredit = bound({ x: amount, min: 1, max: type(uint128).max });

        PerpMarketCreditConfig memory fuzzMarketConfig = getFuzzPerpMarketCreditConfig(marketId);
       vaultId = uint128(bound(vaultId, INITIAL_VAULT_ID, FINAL_VAULT_ID));
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);

        vm.assume(fuzzVaultConfig.asset != address(usdc));

        deal({ token: address(fuzzVaultConfig.asset), to: address(fuzzMarketConfig.engine), give: amount });
        changePrank({ msgSender: address(fuzzMarketConfig.engine) });

        marketMakingEngine.depositCreditForMarket(fuzzMarketConfig.marketId, fuzzVaultConfig.asset, amounttodepositcredit);

        uint256 mmBalance = IERC20(fuzzVaultConfig.asset).balanceOf(address(marketMakingEngine));


        uint128 depositFee = uint128(vaultsConfig[fuzzVaultConfig.vaultId].depositFee);

        _setDepositFee(depositFee, fuzzVaultConfig.vaultId);

       //c all code below makes the deposit into the vault that checks realized debt
        address user = users.naruto.account;
        deal(fuzzVaultConfig.asset, user, 100e18);

      

        marketMakingEngine.workaround_Vault_setTotalCreditDelegationWeight(vaultId, 1e10);

        // perform the deposit
        vm.startPrank(user);
        marketMakingEngine.deposit(vaultId, 1e18, 0, "", false);
        marketMakingEngine.deposit(vaultId, 1e18, 0, "", false);
        vm.stopPrank();


        // it should deposit credit for market
        assertEq(mmBalance, amount);
    }

}

/* I was getting a SafeCastOverflowedIntDowncast(128, 54498090952761805331968824837253488018400000000000) custom error and i wanted to debug this error. i have looked at the logs of the result and i can see it has to do with sometime after the getrealiseddebt function in the market leaf is trying to run. Lets just focus on this getrealizeddebt function for a moment. Before that, I will paste the relevant sections of the log below:


├─ [2371] Perps Engine::getUnrealizedDebt(1) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [30560] ERC1967Proxy::fallback() [staticcall]
    │   │   │   ├─ [25688] PriceAdapter::getPrice() [delegatecall]
    │   │   │   │   ├─ [2287] MockPriceFeed::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   ├─ [2358] MockSequencerUptimeFeed::latestRoundData() [staticcall]
    │   │   │   │   │   └─ ← [Return] 0, 0, 1, 1680220800 [1.68e9], 0
    │   │   │   │   ├─ [2414] MockPriceFeed::latestRoundData() [staticcall]
    │   │   │   │   │   └─ ← [Return] 0, 2000000000000000000000 [2e21], 0, 1680220800 [1.68e9], 0
    │   │   │   │   ├─ [2300] MockPriceFeed::aggregator() [staticcall]
    │   │   │   │   │   └─ ← [Return] MockAggregator: [0xc8A5BFbae3B568fb51e9ad4D1287f051091dcc93]
    │   │   │   │   ├─ [160] MockAggregator::minAnswer() [staticcall]
    │   │   │   │   │   └─ ← [Return] 1999999999999999999999 [1.999e21]
    │   │   │   │   ├─ [193] MockAggregator::maxAnswer() [staticcall]
    │   │   │   │   │   └─ ← [Return] 2000000000000000000001 [2e21]
    │   │   │   │   └─ ← [Return] 2000000000000000000000 [2e21]
    │   │   │   └─ ← [Return] 2000000000000000000000 [2e21]
    │   │   ├─ [0] console::log(0) [staticcall]
    │   │   │   └─ ← [Stop] 
    │   │   ├─ [0] console::log(544980909527618053319688248372534880184000 [5.449e41]) [staticcall]
    │   │   │   └─ ← [Stop] 
    │   │   ├─ [0] console::log("continue") [staticcall]
    │   │   │   └─ ← [Stop] 
    │   │   ├─ [0] console::log(10000000000 [1e10]) [staticcall]
    │   │   │   └─ ← [Stop] 
    │   │   └─ ← [Revert] SafeCastOverflowedIntDowncast(128, 54498090952761805331968824837253488018400000000000 [5.449e49])

 this getrealiseddebtfunction calls another function called getcreditdepositsvalueusd in the same leaf and in that function another function called getadjustedprice is called from the collateral leaf. this function calls a getprice function from a price adapter address which is why you see a price adapter::getprice in the logs above. So I had a look at what a price adapter actually does.

 in base.t.sol , i have seen that it calls marketMakingEngine.configureCollateral a few times to set allowed collateral up for the market maker engine. this function takes a price adapter address as an argument from a margin collateral mapping. This margin collateral mapping comes from a margincollateral.sol file in the scripts folder. this file is similar to what we have seen with vaults and markets where there is a library with functions that set up all the relevant vaults and markets, etc. This margincollateral.sol sets up config for collateral assets. in that file, there is a setupMarginCollaterals function and it sets up each collateral address with a defined struct and that struct has a price adapter variable which takes a price adapter address. in the config setup, the address is called by deploying a price adapter contract using a deploypriceadapter function. exploring that function showed me that the price adapter was deployed using a proxy. it also led me to the priceadapter.sol contract which showed me what a price adapter was.

priceadapter.sol is an upgradeable contract that contains one main function which is the getprice function. This function calls another getprice function from a chainlinkutil library. the chainlinkutil getprice function takes a getpriceparams struct which takes priceFeed, priceFeedHeartbeatSeconds and sequencerUptimeFeed as arguments. I was a bit confused as to what the difference between a price feed and a sequencer was. I looked in the base.t.sol file and it had the following line mockSequencerUptimeFeed = address(new MockSequencerUptimeFeed(0)); so i looked for that contract and all it contained was a latestrounddata which I thought was what got price data like a price feed contract would do but that is not the case as you will see below.


lets look at what the difference is between a price feed and a sequencer. A price feed is what we are used to seeing from chainlink and how we have always gotten prices of assets from chainlink oracles but remember that zaros intends to work on arbitrum and it is a L2. The core difference between L1's and L2 is the idea of a sequencer.


A **sequencer** is typically **off-chain infrastructure** or software that is responsible for ordering and batching transactions in certain blockchain systems, especially in **Layer 2 (L2) scaling solutions** like Optimism, Arbitrum, and others. However, **smart contracts** often interact with the sequencer to validate or manage its behavior.

---
  - The sequencer is a node or server operated by the Layer 2 (L2) provider or a centralized entity.
  - Its primary responsibilities include:
    - Collecting user transactions.
    - Ordering them deterministically.
    - Batching and submitting them to the Layer 1 (L1) blockchain.
    - Providing rapid transaction confirmations on L2.

We arent going to dive too deep into the technicalities of how a sequencer works as that isnt too relevant to the zaros codebase but we will look at the role of smart contracts in the sequencer.


 **Role of Smart Contracts**
While the sequencer itself is off-chain, certain **on-chain smart contracts** interact with it to ensure transparency, reliability, and security.

 1. **Sequencer Uptime Feeds**
   - Smart contracts like the **Sequencer Uptime Feed** monitor the status of the sequencer.
   - These contracts record:
     - Whether the sequencer is currently up or down.
     - The last timestamp when the sequencer's status changed.

   **Example**: Chainlink's Sequencer Uptime Feeds provide a decentralized mechanism to check if an L2 sequencer is operational. This is what the link to the chainlink docs I sent above does. They have a sequencer uptime feed that is used to query the sequencer to see whether it is currently running or not as well as other data about the seuqncer. This is what the latetstrounddata in the MockSequencerUptimeFeed contract does. It is used to check the status of the sequencer. https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code. In the example code in this link is exactly what zaros use here to configure their price adapter. 

   The price adapter is what configures all the relevant variables like the sequencer and price feed addresses and calls the getprice function from the chainlinkutil.sol library. if you look at the getprice function in the chainlinkutil.sol library, you will see that it queries the sequencer by calling its latestrounddata function and get the answer variable whoch only returns 0 or 1. 0 indicates that the sequencer is down and 1 means it is running. There is also a startedat variable which logs the timestamp of when the sequencer was last started and the rule from the chainlink docs which i linked above is that there needs to be a time period since the sequencer was last started before the price feed can be queried. This is why there is a SEQUENCER_GRACE_PERIOD_TIME constant set. This is the time period that needs to pass since the sequencer was last started before the price feed can be queried. This is all done to ensure that the price feed is reliable and secure. You can read about all this in the link i sent above. It is pretty straightforward and will explain everything that you see zaros implement in terms of their price adapter and how they get the price of assets.

 2. **Transaction Verification**
   - In rollups, smart contracts on Layer 1 validate the data submitted by the sequencer.
   - For example:
     - **Optimistic Rollups**: Use fraud proofs to ensure that the sequencer's submitted transactions are valid.
     - **ZK Rollups**: Use validity proofs (e.g., ZK-SNARKs) generated by the sequencer to confirm correctness.

 3. **Fallback Mechanisms**
   - Many L2 solutions implement on-chain fallback mechanisms, allowing users to interact with L1 directly if the sequencer goes offline or becomes malicious.

---

 **How Smart Contracts Interact with the Sequencer**
- **Data Feeds**: 
  Contracts like Chainlink's **Sequencer Uptime Feed** are queried to ensure the sequencer is live before relying on its data.

- **State Updates**: 
  The sequencer periodically submits batched transaction data to on-chain contracts for validation and finality.

- **Dispute Resolution**:
  If the sequencer submits incorrect or malicious data, smart contracts handle dispute resolution (e.g., fraud proofs in Optimistic Rollups).

---
Going back to the issue, the problem was because these were the inputs into the deposit for credit function CreditDelegationBranch::depositCreditForMarket(1, wstETH: [0x8c3aE1a8D635758eAEBbCC77ddC18F08749A707c], 136245227381904513329922062093133720046 [1.362e38]) [delegatecall]. this is where issue is coming from. asset was wsteth and this was amount. in reality, no one is ever going to deposit this amount of any asset but the problem was coming from a market.distributeDebtToVaults(ctx.marketUnrealizedDebtUsdX18, ctx.marketRealizedDebtUsdX18) function. In this function,it had the following line  self.realizedDebtUsdPerVaultShare = newRealizedDebtUsdX18.div(totalVaultSharesX18).intoInt256().toInt128();  when the value was casted to uint128, it caused the issue.

*/
```

# 13 INTRODUCTION TO THE IDEA OF USING ADAPTERS TO CONNECT COMPONENTS

While auditing zaeos, this idea of an adapter was used a lot and i know I spoke about price adapters above but there were also dex adapters used and this prompted me to discuss this idea of adapters and what they are used for as a whole. I talked about this idea in the feedistributionbranch.sol. See the comments I left below:

```solidity
 function convertAccumulatedFeesToWeth(
        uint128 marketId,
        address asset,
        uint128 dexSwapStrategyId,
        bytes calldata path
    )
        external
        onlyRegisteredSystemKeepers
    {
        // loads the collateral data storage pointer, collateral must be enabled
        Collateral.Data storage collateral = Collateral.load(asset);
        collateral.verifyIsEnabled();

        // loads the market data storage pointer
        Market.Data storage market = Market.loadExisting(marketId);

        // reverts if the market hasn't received any fees for the given asset
        (bool exists, uint256 receivedFees) = market.receivedFees.tryGet(asset);
        if (!exists) revert Errors.MarketDoesNotContainTheAsset(asset);
        if (receivedFees == 0) revert Errors.AssetAmountIsZero(asset);

        // working data, converted receivedFees uint256 -> UD60x18
        ConvertAccumulatedFeesToWethContext memory ctx;
        ctx.assetAmountX18 = ud60x18(receivedFees);

        // convert the asset amount to token amount
        ctx.assetAmount = collateral.convertUd60x18ToTokenAmount(ctx.assetAmountX18);

        // load weth address
        ctx.weth = MarketMakingEngineConfiguration.load().weth;

        // if asset is weth directly add to accumulated weth, else swap token for weth
        if (asset == ctx.weth) {
            // store the amount of weth
            ctx.receivedWethX18 = ctx.assetAmountX18;
        } else {
            // load the weth collateral data storage pointer
            Collateral.Data storage wethCollateral = Collateral.load(ctx.weth);

            // load custom swap path for asset if enabled
            AssetSwapPath.Data storage swapPath = AssetSwapPath.load(asset);

            // verify if the swap should be input multi-dex/custom swap path, single or multihop
            if (swapPath.enabled) {
                ctx.tokensSwapped = _performMultiDexSwap(swapPath, ctx.assetAmount);
            } else if (path.length == 0) {
                // loads the dex swap strategy data storage pointer
                DexSwapStrategy.Data storage dexSwapStrategy = DexSwapStrategy.loadExisting(dexSwapStrategyId);
                //c lets look at what this dexswapstrategy is doing. this is a new concept to me. it follows the tree proxy pattern we are used to seeing. so there is a dexswapstrategy leaf which contains the usual data struct and helper functions we expect. the data  struct contains a strategy id and a dex adapter address. these dex adapters are the contracts that actually make the swaps. you can see them in src/utils/dex-adapters. In that directory, there is a baseadapter.sol contract which provides helper functions that all the other dex adapters use. it contains functions to set slippage, get expected output amount etc which are all functions used by the dex adapters. especially setSwapAssetConfig and getexpectedoutput. what you will notice is that these 2 functions interact with the price adapter of the token which is used to get the price of the related tokens to get the expected output amount. 

                //c this gives us an insight to what a dex adapter actually is. we saw earlier how a price adapter simply combined the features of the sequencer and the price feed address. think about it. An adapter in hardware is a device, tool, or software that connects two different systems, components, or devices to make them compatible with each other. Think about when you have a charger. the adapter connects the wire to the socket to alloe electricity to pass from the socket to the wire. so we know that a dex adapter connects 2 components. looking in baseadapter.sol , we see that the price adapter is incorporated into the dex adapter. so the dex adapter is the device that connects the price adapter to the swap router. you will see the swap router contract used when you look at any of the dex adapters like the uniswapv2adapter.sol. So whenever you see an adapter used anywhere, you should know what the contract is supposed to do.

```

# 14 SINGLE HOP VS MULTI-HOP SWAPS 

This point is mostly based around the dex adapters which you can view in src/utils/dex-adapters. In there, you will see all the dex adapters that zaros use. Lets focus on the UniswapV2Adapter.sol. There are 2 functions here we want to look at and these are executeSwapExactInputSingle and executeSwapExactInput. These 2 functions show what single hops vs multi swaps are. Single hops are where used when you want to swap 2 tokens in the same pool. Multihops are where you want to swap tokens across 2 different pools. The 2 functions are as follows:

```solidity
/// @inheritdoc IDexAdapter
    function executeSwapExactInputSingle(SwapExactInputSinglePayload calldata swapPayload)
        external
        returns (uint256 amountOut)
    {
        // transfer the tokenIn from the send to this contract
        IERC20(swapPayload.tokenIn).transferFrom(msg.sender, address(this), swapPayload.amountIn);

        // aprove the tokenIn to the swap router
        address uniswapV2SwapStrategyRouterCache = uniswapV2SwapStrategyRouter;
        IERC20(swapPayload.tokenIn).approve(uniswapV2SwapStrategyRouterCache, swapPayload.amountIn);

        // get the expected output amount
        uint256 expectedAmountOut = getExpectedOutput(swapPayload.tokenIn, swapPayload.tokenOut, swapPayload.amountIn);

        // Calculate the minimum acceptable output based on the slippage tolerance
        uint256 amountOutMinimum = calculateAmountOutMin(expectedAmountOut);

        address[] memory path = new address[](2);
        path[0] = swapPayload.tokenIn;
        path[1] = swapPayload.tokenOut;

        uint256[] memory amountsOut = IUniswapV2Router02(uniswapV2SwapStrategyRouterCache).swapExactTokensForTokens({
            amountIn: swapPayload.amountIn,
            amountOutMin: amountOutMinimum,
            path: path,
            to: swapPayload.recipient,
            deadline: deadline
        });

        return amountsOut[1];
    }

    /// @inheritdoc IDexAdapter
    function executeSwapExactInput(SwapExactInputPayload calldata swapPayload) external returns (uint256 amountOut) {
        // transfer the tokenIn from the send to this contract
        IERC20(swapPayload.tokenIn).transferFrom(msg.sender, address(this), swapPayload.amountIn);

        // aprove the tokenIn to the swap router
        address uniswapV2SwapStrategyRouterCache = uniswapV2SwapStrategyRouter;
        IERC20(swapPayload.tokenIn).approve(uniswapV2SwapStrategyRouterCache, swapPayload.amountIn);

        // get the expected output amount
        uint256 expectedAmountOut = getExpectedOutput(swapPayload.tokenIn, swapPayload.tokenOut, swapPayload.amountIn);

        // Calculate the minimum acceptable output based on the slippage tolerance
        uint256 amountOutMinimum = calculateAmountOutMin(expectedAmountOut);

        // decode path as it is Uniswap V3 specific
        (address[] memory tokens,) = swapPayload.path.decodePath();

        // execute trade
        uint256[] memory amountsOut = IUniswapV2Router02(uniswapV2SwapStrategyRouterCache).swapExactTokensForTokens({
            amountIn: swapPayload.amountIn,
            amountOutMin: amountOutMinimum,
            path: tokens,
            to: swapPayload.recipient,
            deadline: deadline
        });

        // return the amount out of the last trade
        return amountsOut[tokens.length - 1];
    }
```
All of this can be viewed in the uniswap docs at https://docs.uniswap.org/contracts/v3/guides/swaps/multihop-swaps . The docs are very straightforward and tell you exactly how to perform each kind of swap and this is exactly what zaros do but they mix this with functions like getExpectedOutput and calculateAmountOutMin which are in Baseadapter.sol and are used to incorporate chainlink price feeds to get asset prices that are used to determine different values that are passed to the uniswap router which is what carries out the swap. 

It is rather straightforward but the only point I want to discuss is the swapPayload.path variable you see that is decoded in executeSwapExactInput and sent to the uniswap router. That path is simply the path you want the swap to take. With multihops, as I said , tokens are swapped across pools so this path variable encodes the path that you want to use to get from token A to token B. This is described very well in the uniswap docs I linked above. See this code block:

```solidity
    // SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity =0.7.6;
pragma abicoder v2;

import '@uniswap/v3-periphery/contracts/libraries/TransferHelper.sol';
import '@uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol';

contract SwapExamples {
    // For the scope of these swap examples,
    // we will detail the design considerations when using
    // `exactInput`, `exactInputSingle`, `exactOutput`, and  `exactOutputSingle`.

    // It should be noted that for the sake of these examples, we purposefully pass in the swap router instead of inherit the swap router for simplicity.
    // More advanced example contracts will detail how to inherit the swap router safely.

    ISwapRouter public immutable swapRouter;

    // This example swaps DAI/WETH9 for single path swaps and DAI/USDC/WETH9 for multi path swaps.

    address public constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address public constant WETH9 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    // For this example, we will set the pool fee to 0.3%.
    uint24 public constant poolFee = 3000;

    constructor(ISwapRouter _swapRouter) {
        swapRouter = _swapRouter;
    }

    /// @notice swapInputMultiplePools swaps a fixed amount of DAI for a maximum possible amount of WETH9 through an intermediary pool.
    /// For this example, we will swap DAI to USDC, then USDC to WETH9 to achieve our desired output.
    /// @dev The calling address must approve this contract to spend at least `amountIn` worth of its DAI for this function to succeed.
    /// @param amountIn The amount of DAI to be swapped.
    /// @return amountOut The amount of WETH9 received after the swap.
    function swapExactInputMultihop(uint256 amountIn) external returns (uint256 amountOut) {
        // Transfer `amountIn` of DAI to this contract.
        TransferHelper.safeTransferFrom(DAI, msg.sender, address(this), amountIn);

        // Approve the router to spend DAI.
        TransferHelper.safeApprove(DAI, address(swapRouter), amountIn);

        // Multiple pool swaps are encoded through bytes called a `path`. A path is a sequence of token addresses and poolFees that define the pools used in the swaps.
        // The format for pool encoding is (tokenIn, fee, tokenOut/tokenIn, fee, tokenOut) where tokenIn/tokenOut parameter is the shared token across the pools.
        // Since we are swapping DAI to USDC and then USDC to WETH9 the path encoding is (DAI, 0.3%, USDC, 0.3%, WETH9).
        ISwapRouter.ExactInputParams memory params =
            ISwapRouter.ExactInputParams({
                path: abi.encodePacked(DAI, poolFee, USDC, poolFee, WETH9),
                recipient: msg.sender,
                deadline: block.timestamp,
                amountIn: amountIn,
                amountOutMinimum: 0
            });

        // Executes the swap.
        amountOut = swapRouter.exactInput(params);
    }

    /// @notice swapExactOutputMultihop swaps a minimum possible amount of DAI for a fixed amount of WETH through an intermediary pool.
    /// For this example, we want to swap DAI for WETH9 through a USDC pool but we specify the desired amountOut of WETH9. Notice how the path encoding is slightly different in for exact output swaps.
    /// @dev The calling address must approve this contract to spend its DAI for this function to succeed. As the amount of input DAI is variable,
    /// the calling address will need to approve for a slightly higher amount, anticipating some variance.
    /// @param amountOut The desired amount of WETH9.
    /// @param amountInMaximum The maximum amount of DAI willing to be swapped for the specified amountOut of WETH9.
    /// @return amountIn The amountIn of DAI actually spent to receive the desired amountOut.
    function swapExactOutputMultihop(uint256 amountOut, uint256 amountInMaximum) external returns (uint256 amountIn) {
        // Transfer the specified `amountInMaximum` to this contract.
        TransferHelper.safeTransferFrom(DAI, msg.sender, address(this), amountInMaximum);
        // Approve the router to spend  `amountInMaximum`.
        TransferHelper.safeApprove(DAI, address(swapRouter), amountInMaximum);

        // The parameter path is encoded as (tokenOut, fee, tokenIn/tokenOut, fee, tokenIn)
        // The tokenIn/tokenOut field is the shared token between the two pools used in the multiple pool swap. In this case USDC is the "shared" token.
        // For an exactOutput swap, the first swap that occurs is the swap which returns the eventual desired token.
        // In this case, our desired output token is WETH9 so that swap happpens first, and is encoded in the path accordingly.
        ISwapRouter.ExactOutputParams memory params =
            ISwapRouter.ExactOutputParams({
                path: abi.encodePacked(WETH9, poolFee, USDC, poolFee, DAI),
                recipient: msg.sender,
                deadline: block.timestamp,
                amountOut: amountOut,
                amountInMaximum: amountInMaximum
            });

        // Executes the swap, returning the amountIn actually spent.
        amountIn = swapRouter.exactOutput(params);

        // If the swap did not require the full amountInMaximum to achieve the exact amountOut then we refund msg.sender and approve the router to spend 0.
        if (amountIn < amountInMaximum) {
            TransferHelper.safeApprove(DAI, address(swapRouter), 0);
            TransferHelper.safeTransferFrom(DAI, address(this), msg.sender, amountInMaximum - amountIn);
        }
    }
}

```
so the fees described here are simply the pool fees for the pool that you want to swap from . this example wants to swap DAI for WETH so the path variable encodes this as (tokenOut, fee, tokenIn/tokenOut, fee, tokenIn). so for the above example, it was (WETH9, poolFee, USDC, poolFee, DAI) where we are swapping DAI to USDC IN A DAI/USDC pool with a pool fee of poolFee and then swapping USDC for WETH in a WETH9/USDC pool with a pool fee value of poolFee. You can view all the pool fees on the uniswap pool. Notice how the poolfee is defined as 3000 which stands for 0.3% as we know that 10000 is the scaling factor used in most projects and we learnt about this scaling factor in one of the previous audits so you can look through your past audit notes to refamiliarise yourself. Besides that, there isnt much more to know here. You can read about how the uniswap routers work in the uniswap docs but you can kinda guess from when we did the tswap audit so you can also look at the code for that audit if you want to a quick refresher.

Lets discuss some findings you can report that are related to single and multi hop swaps. The finding below was found from one of the reports sent before the zaros audit started. See below:

[Medium-1] Hardcoded swap path 

Setting a constant swap path in Solidity for liquidity pool transactions could result in sub-optimal returns. The swap path determines the sequence of tokens to be swapped in a multi-token trade. If the path is constant, it won't adapt to market fluctuations or changes in liquidity distribution across different pools. This could mean missing out on a more profitable route.

Resolution: To ensure optimal returns, use a dynamic swap path strategy. This strategy calculates the most efficient path for each transaction, considering current liquidity distribution and token prices. It allows the user to take advantage of the best possible trading conditions at the time of the swap, maximizing returns and minimizing slippage. Implement this strategy with care, as it requires a deeper understanding of DeFi protocols and markets.

Num of instances: 1

 Findings 

```solidity
/// @notice Settles the given vaults' debt or credit by swapping assets to USDC or vice versa.
    /// @dev Converts ZLP Vaults unsettled debt to settled debt by:
    ///     - If in debt, swapping the vault's assets to USDC.
    ///     - If in credit, swapping the vault's available USDC to its underlying assets.
    /// exchange for their assets.
    /// @dev USDC acquired from onchain markets is deposited to vaults and used to cover future USD Token swaps, in
    /// case the vault is in debt.
    /// @dev If the vault is in net credit and doesn't own enough USDC to fully settle the due amount, it may have its
    /// assets rebalanced with other vaults through `CreditDelegation::rebalanceVaultsAssets`.
    /// @dev There isn't any issue settling debt of vaults of different engines in the same call, as the system
    /// allocates the USDC acquired in case of a debt settlement to the engine's USD Token accordingly.
    /// @param vaultsIds The vaults' identifiers to settle.
    function settleVaultsDebt(uint256[] calldata vaultsIds) external onlyRegisteredSystemKeepers {
        // first, we need to update the credit capacity of the vaults
        Vault.recalculateVaultsCreditCapacity(vaultsIds);

        // working data, cache usdc address
        SettleVaultDebtContext memory ctx;
        ctx.usdc = MarketMakingEngineConfiguration.load().usdc;

        // load the usdc collateral data storage pointer
        Collateral.Data storage usdcCollateralConfig = Collateral.load(ctx.usdc);

        for (uint256 i; i < vaultsIds.length; i++) {
            // load the vault storage pointer
            Vault.Data storage vault = Vault.loadExisting(vaultsIds[i].toUint128());

            // cache the vault's unsettled debt, if zero skip to next vault
            // amount in zaros internal precision
            ctx.vaultUnsettledRealizedDebtUsdX18 = vault.getUnsettledRealizedDebt();
            if (ctx.vaultUnsettledRealizedDebtUsdX18.isZero()) continue;

            // otherwise vault has debt to be settled, cache the vault's collateral asset
            ctx.vaultAsset = vault.collateral.asset;

            // loads the dex swap strategy data storage pointer
            DexSwapStrategy.Data storage dexSwapStrategy =
                DexSwapStrategy.loadExisting(vault.swapStrategy.assetDexSwapStrategyId);

            // if the vault is in debt, swap its assets to USDC
            if (ctx.vaultUnsettledRealizedDebtUsdX18.lt(SD59x18_ZERO)) {
                // get swap amount; both input and output in native precision
                ctx.swapAmount = calculateSwapAmount(
                    dexSwapStrategy.dexAdapter,
                    ctx.usdc,
                    ctx.vaultAsset,
                    usdcCollateralConfig.convertSd59x18ToTokenAmount(ctx.vaultUnsettledRealizedDebtUsdX18.abs())
                );

                // swap the vault's assets to usdc in order to cover the usd denominated debt partially or fully
                // both input and output in native precision
                ctx.usdcOut = _convertAssetsToUsdc(
                    vault.swapStrategy.usdcDexSwapStrategyId,
                    ctx.vaultAsset,
                    ctx.swapAmount,
                    vault.swapStrategy.usdcDexSwapPath,
                    address(this),
                    ctx.usdc
                );

                // sanity check to ensure we didn't somehow give away the input tokens
                if (ctx.usdcOut == 0) revert Errors.ZeroOutputTokens();

                // uint256 -> udc60x18 scaling native precision to zaros internal precision
                ctx.usdcOutX18 = usdcCollateralConfig.convertTokenAmountToUd60x18(ctx.usdcOut);

                // use the amount of usdc bought with assets to update the vault's state
                // note: storage updates must be done using zaros internal precision
                //
                // deduct the amount of usdc swapped for assets from the vault's unsettled debt
                vault.marketsRealizedDebtUsd -= ctx.usdcOutX18.intoUint256().toInt256().toInt128();

                // allocate the usdc acquired to back the engine's usd token
                UsdTokenSwapConfig.load().usdcAvailableForEngine[vault.engine] += ctx.usdcOutX18.intoUint256();

                // update the variables to be logged
                ctx.assetIn = ctx.vaultAsset;
                ctx.assetInAmount = ctx.swapAmount;
                ctx.assetOut = ctx.usdc;
                ctx.assetOutAmount = ctx.usdcOut;
                // since we're handling debt, we provide a positive value
                ctx.settledDebt = ctx.usdcOut.toInt256();
            } else {
                // else vault is in credit, swap its USDC previously accumulated
                // from market and vault deposits into its underlying asset

                // get swap amount; both input and output in native precision
                ctx.usdcIn = calculateSwapAmount(
                    dexSwapStrategy.dexAdapter,
                    ctx.vaultAsset,
                    ctx.usdc,
                    usdcCollateralConfig.convertSd59x18ToTokenAmount(ctx.vaultUnsettledRealizedDebtUsdX18.abs())
                );

                // get deposited USDC balance of the vault, convert to native precision
                ctx.vaultUsdcBalance = usdcCollateralConfig.convertUd60x18ToTokenAmount(ud60x18(vault.depositedUsdc));

                // if the vault doesn't have enough usdc use whatever amount it has
                // make sure we compare native precision values together and output native precision
                ctx.usdcIn = (ctx.usdcIn <= ctx.vaultUsdcBalance) ? ctx.usdcIn : ctx.vaultUsdcBalance;

                // swaps the vault's usdc balance to more vault assets and
                // send them to the ZLP Vault contract (index token address)
                // both input and output in native precision
                ctx.assetOutAmount = _convertUsdcToAssets(
                    vault.swapStrategy.assetDexSwapStrategyId,
                    ctx.vaultAsset,
                    ctx.usdcIn,
                    vault.swapStrategy.assetDexSwapPath,
                    vault.indexToken,
                    ctx.usdc
                );

                // sanity check to ensure we didn't somehow give away the input tokens
                if (ctx.assetOutAmount == 0) revert Errors.ZeroOutputTokens();

                // subtract the usdc amount used to buy vault assets from the vault's deposited usdc, thus, settling
                // the due credit amount (partially or fully).
                // note: storage updates must be done using zaros internal precision
                vault.depositedUsdc -= usdcCollateralConfig.convertTokenAmountToUd60x18(ctx.usdcIn).intoUint128();

                // update the variables to be logged
                ctx.assetIn = ctx.usdc;
                ctx.assetInAmount = ctx.usdcIn;
                ctx.assetOut = ctx.vaultAsset;
                // since we're handling credit, we provide a negative value
                ctx.settledDebt = -ctx.usdcIn.toInt256();
            }

             
            emit LogSettleVaultDebt(
                 vaultsIds[i].toUint128(),
                ctx.assetIn,
                ctx.assetInAmount,
                 ctx.assetOut,
                 ctx.assetOutAmount,
                 ctx.settledDebt)
         }}

```
This is an easy win you can get. If you see that the protocol implementing multihop swaps is hardcoding the path to the swaps, you can report this as a medium and suggest that having dynamic paths are always better. This is related to hardcoded values which is something that a lot of protocols do as you can see was also done in the benqi audit and you can look at the findings from that audit to see what I mean. Hardcoded values are usually an easy medium to find and report


# 15 INTRO TO GRIEFING ATTACKS

Before now, I had always heard the term griefing but never explored what it means unti i came across this block of code in ZlpVault::maxDeposit():

```solidity
 /// @notice Returns the maximum amount of assets that can be deposited into the ZLP Vault, taking into account the
    /// configured deposit cap.
    /// @dev Overridden and used in ERC4626.
    /// @return maxAssets The maximum amount of depositable assets.
    function maxDeposit(address) public view override returns (uint256 maxAssets) {
        // load the zlp vault storage pointer
        ZlpVaultStorage storage zlpVaultStorage = _getZlpVaultStorage();
        // cache the market making engine contract
        IMarketMakingEngine marketMakingEngine = IMarketMakingEngine(zlpVaultStorage.marketMakingEngine);

        // get the vault's deposit cap
        uint128 depositCap = marketMakingEngine.getDepositCap(zlpVaultStorage.vaultId);

        // cache the vault's total assets
        uint256 totalAssetsCached = totalAssets(); //q my inflation attack senses are tingling when i see something like this

        // underflow check here would be redundant
        unchecked {
            // we need to ensure that depositCap > totalAssets, otherwise, a malicious actor could grief deposits by
            // sending assets directly to the vault contract and bypassing the deposit cap
            maxAssets = depositCap > totalAssetsCached ? depositCap - totalAssetsCached : 0;
        }}
```
Fous on the comment in the unchecked block which said :

```
// we need to ensure that depositCap > totalAssets, otherwise, a malicious actor could grief deposits by
            // sending assets directly to the vault contract and bypassing the deposit cap
```
They speak about griefing deposits by a user sending assets directly to the zlpvault without calling vaultrouterbranch::deposit and filling up the depositcap which doesnt allow users who call vaultrouterbranch::deposit to deposit. This is a typical example of what a griefing attack looks like. So a user being able to perform an action that stops users who are eligible from doing what they are rightfully supposed to do.


# 16 FULL AUDIT PROCEDURE , IMPORTANCE OF GOING THROUGH FULL PROCEDURE, DONT GET COMFY WITH A FEW HIGHS. 

As you know from previous audits, you know the procedure is to find an entry point function, do a full first pass to understand what every line does. You can find bugs in the first pass a lot of the time but in the second pass which is the assumption analysis for that same entry point function is where most of the solo bugs will come from. The invariant tests come as a final step and there will be even more bugs you can cover from this next step.

What you need to note is that sometimes say you have found some bugs in the first pass. High vulnerabilities even. You then start to get all happy and excited and your mind naturally starts to chill out thinking you are in a good position. As a seciurity researcher, you need to train yourself so this does not happen. You must always be hungry. This sounds like it doesnt need noting but it does. There are more bugs and you need to find them. In fact when yu find one, you should be hungrier knowing that theres more. It is never enough. Do not get comfortable until the whole procedure is complete. even then you can redo the whole procedure or go a step further and do formal verification or break down all assumptions from second pass even more. You must keep going in the time you have allocated to auditing the codebase. Think about it like this if you have seen it, you have no way to know that 100 other people have not also seen it so you must be hungry to find the ones that no one else has seen and the deeper you go into this procedure with hunger, the higher your chances of seeing what others dont see.

# 17 IMPORTANCE OF AVOIDING LIMITING POC'S , NEVER ASSUME ALWAYS VERIFY
Also i want to stress the importance of writing non limiting pocs. You already know about the fact that you can end up discovering more bugs while writing poc's but the detail you put into the POC is extremely important as to whether you end up finding more bugs or not and I will show you an example of this. See the following test i wrote to prove a griefing attack:

```solidity
 function test_griefingdeposits(
        uint256 vaultId
    )
        external
    {
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
    
       
        //c user who will grief deposits
        address userA = users.naruto.account;
        deal(fuzzVaultConfig.asset, userA, 100e18);
        

        //c user trying to deposit into vault
        address userB = users.sasuke.account;
        deal(fuzzVaultConfig.asset, userB, 100e18);
     

        //c get deposit cap of vault
        uint128 depositCap = vaultsConfig[fuzzVaultConfig.vaultId].depositCap;
       

        //c get the deposit fee of the vault
        uint128 depositFee = uint128(vaultsConfig[fuzzVaultConfig.vaultId].depositFee);

        //c get assets sent to vault index token 
        uint256 assetminusfee = amount - uint256(depositFee);
         

        //c attempt to grief deposits
        vm.startPrank(userA);
        IERC20(fuzzVaultConfig.asset).transfer(fuzzVaultConfig.indexToken, depositCap);
        vm.stopPrank();


        //c check totalassets of vault
        uint256 totalAssets = IERC4626(fuzzVaultConfig.indexToken).totalAssets();
        console.log(totalAssets);

        //c user attempts to deposits into vault
        vm.startPrank(userB);
        vm.expectRevert(abi.encodeWithSelector(ERC4626Upgradeable.ERC4626ExceededMaxDeposit.selector,userB,assetminusfee,0));
        marketMakingEngine.deposit(fuzzVaultConfig.vaultId, 1e18, 0, "", false);
        vm.stopPrank(); 

        
    }
```

Just looking at this test, it works perfectly and proves the attack but if you look at the amount user B deposits into the vault , it is hardcoded as 1e18 which is fine but what that does is that the test only runs for a single amount to deposit and yes it works but this is known as a limiting POC. It does not open any doors for me to find any other exploits as I have made the test very rigid. This is something you need to watch out for. You need to learn to expand the range of your POC to where you are aiming for the same end result BUT you expand the range of the POC's by either checking variables changed due to the functions you are calling or/and implementing random values. This is how I expanded the range of the POC to potentially help me find more bugs. this is from deposit.t.sol:

```solidity
 function test_griefingdeposits(
        uint256 vaultId,
        uint256 amount
    )
        external
    {
        VaultConfig memory fuzzVaultConfig = getFuzzVaultConfig(vaultId);
       
        //c user who will grief deposits
        address userA = users.naruto.account;
        deal(fuzzVaultConfig.asset, userA, 100e18);
        

        //c user trying to deposit into vault
        address userB = users.sasuke.account;
        deal(fuzzVaultConfig.asset, userB, 100e18);
     

        //c get deposit cap of vault
        uint128 depositCap = vaultsConfig[fuzzVaultConfig.vaultId].depositCap;
       

        //c get the deposit fee of the vault
        uint128 depositFee = uint128(vaultsConfig[fuzzVaultConfig.vaultId].depositFee);

        

         amount =bound(amount, calculateMinOfSharesToStake(fuzzVaultConfig.vaultId), fuzzVaultConfig.depositCap / 2);
        //c get assets sent to vault index token 
         uint128 depositfeeforamount = uint128(amount)*depositFee/1e18;
        uint256 assetminusfee = amount - uint256(depositfeeforamount);
         

        //c attempt to grief deposits
        vm.startPrank(userA);
        IERC20(fuzzVaultConfig.asset).transfer(fuzzVaultConfig.indexToken, depositCap);
        vm.stopPrank();


        //c check totalassets of vault
        uint256 totalAssets = IERC4626(fuzzVaultConfig.indexToken).totalAssets();
        console.log(totalAssets);

        //c user attempts to deposits into vault
        vm.startPrank(userB);
        vm.expectRevert(abi.encodeWithSelector(ERC4626Upgradeable.ERC4626ExceededMaxDeposit.selector,userB,assetminusfee,0));
        marketMakingEngine.deposit(fuzzVaultConfig.vaultId, uint128(amount), 0, "", false);
        vm.stopPrank(); 

        
    }
```

This POC has more range now as I fuzzed the amount being deposited by user B which allowed me to see if anything would change with a different amount being deposited. With this, I am able to see different behaviours based on the random amount deposited and make sure all the behavior is the same. 

An extra point to note was that I initially had:
```solidity
amount =bound(amount, calculateMinOfSharesToStake(fuzzVaultConfig.vaultId), fuzzVaultConfig.depositCap / 2);
        //c get assets sent to vault index token 
        uint256 assetminusfee = amount - uint256(depositFee);
```

which led me to the following error:
```
[FAIL: Error != expected error: ERC4626ExceededMaxDeposit(0xa6E6dCb0884C1300154C8F9eFB324C1f844629C2, 989999995890653179 [9.899e17], 0) != ERC4626ExceededMaxDeposit(0xa6E6dCb0884C1300154C8F9eFB324C1f844629C2, 989999995849144625 [9.899e17], 0); counterexample: calldata=0xa5f0bb4e0000000000000000000000000000000000000d81b4f7363242e4e996ea1246b4000000000000000000000000000000000000000000000000000000000896f92f args=[273947763974192049499639943153332 [2.739e32], 144111919 [1.441e8]]] test_griefingdeposits(uint256,uint256) (runs: 0, μ: 0, ~: 0)
```

I saw this I was still getting the error I wanted but the values were not as expected and I was tempted to ignore this and ASSUME that everything was okay. This is where the golden rule of NEVER ASSUME ALWAYS VERIFY came in so i decided not to assume and figure out exactly why the values werent the same when they should be. The reason why you never assume is because the value differences could potentially lead me to find another bug. This is what led me to check what I was doing against what vaultrouterbranch::deposit was doing and I found that the issue was with my POC and the above code block was wrong. So i refactored it in the test i pasted above and it worked perfectly. You must follow NEVER ASSUME ALWAYS VERIFY at all times. If you dont, you will miss a lot of things.


# 17 CUSDV3 COMPOUND TOKEN WITH CUSTOM LOGIC EXPLOIT WHERE USER CAN DEPOSIT TYPE(UINT256).MAX INTO PROTOCOL , WBTC TOKEN 8 DECIMALS

This finding came from one of the reports I read up on zaros and it spoke about a cusdv3 token which is a token with custom logic that I havent really heard about so I thought it was worth noting so take a look at it below. CUSDV3 is a rebasing token which is used by compound to represent a user's present usdc position on their platform. What I mean by present is for example, for people who lend usdc, it represents the value the lenders deposited in the platform plus any accrued interest in usdc. To know more anbout this cusdv3 token and the contracts that mint this tioen, you can read this link https://www.rareskills.io/post/cusdc-v3-compound . It goes into detail about all the functionality and everything else you need to know. 

The finding is pretty easy to understand and is another easy one you can report in future audits. See below:

[High-2] Arbitrary staking/deposit on a arbitrary token with an arbitrary amount has no checks to ensure token amount isn't type(uint256).max thus allowing wrong stake amount for certain tokens with custom logic such as cUSDCv3

Certain tokens such as cUSDCv3 have transfer logic where when a transfer takes place their transfer functionality checks if the 'amount' to be transferred is type(uint256).max, if this is the case the balance of the sender is transferred. So if a user has a dust amount cUSDCv3 attempts to stake/deposit amount 'type(uint256).max'  in this protocol, the actual transfer amount will be procesed as the user's total balance of that token, not type(uint256).max. Thus the staking function will register/queue the stake as type(uint256).max but in actuality only cUSDCv3.balanceOf(msg.sender) will have been transferred. This can cause serious discrepancies with the protocols intended logic. To remedy this, there can be a check to prevent users from passing type(uint256).max as the stake/deposit amount. 

Num of instances: 2

Findings 


<details><summary>Click to show findings</summary>

['[181](https://github.com/Cyfrin/2025-01-zaros-part-2/tree/main/src/market-making/branches/CreditDelegationBranch.sol#L181-L181)']
```solidity
181:     function depositCreditForMarket( // <= FOUND
182:         uint128 marketId,
183:         address collateralAddr,
184:         uint256 amount
185:     )
186:         external
187:         onlyRegisteredEngine(marketId)
188:     {
189:         if (amount == 0) revert Errors.ZeroInput("amount");
190: 
191:         
192:         Collateral.Data storage collateral = Collateral.load(collateralAddr);
193:         collateral.verifyIsEnabled();
194: 
195:         
196:         
197:         
198:         Market.Data storage market = Market.loadLive(marketId);
199:         if (market.getTotalDelegatedCreditUsd().isZero()) {
200:             revert Errors.NoDelegatedCredit(marketId);
201:         }
202: 
203:         
204:         UD60x18 amountX18 = collateral.convertTokenAmountToUd60x18(amount);
205: 
206:         
207:         address usdToken = MarketMakingEngineConfiguration.load().usdTokenOfEngine[msg.sender];
208: 
209:         
210:         address usdc = MarketMakingEngineConfiguration.load().usdc;
211: 
212:         
213:         if (collateralAddr == usdToken) {
214:             
215:             market.updateNetUsdTokenIssuance(unary(amountX18.intoSD59x18()));
216:         } else {
217:             if (collateralAddr == usdc) {
218:                 market.settleCreditDeposit(address(0), amountX18);
219:             } else {
220:                 
221:                 
222:                 market.depositCredit(collateralAddr, amountX18);
223:             }
224:         }
225: 
226:         
227:         
228:         
229:         
230:         IERC20(collateralAddr).safeTransferFrom(msg.sender, address(this), amount);
231: 
232:         
233:         emit LogDepositCreditForMarket(msg.sender, marketId, collateralAddr, amount);
234:     }
```
['[82](https://github.com/Cyfrin/2025-01-zaros-part-2/tree/main/src/market-making/branches/FeeDistributionBranch.sol#L82-L82)']
```solidity
82:     function receiveMarketFee( // <= FOUND
83:         uint128 marketId,
84:         address asset,
85:         uint256 amount
86:     )
87:         external
88:         onlyRegisteredEngine(marketId)
89:     {
90:         
91:         if (amount == 0) revert Errors.ZeroInput("amount");
92: 
93:         
94:         Market.Data storage market = Market.loadExisting(marketId);
95: 
96:         
97:         Collateral.Data storage collateral = Collateral.load(asset);
98: 
99:         
100:         collateral.verifyIsEnabled();
101: 
102:         
103:         UD60x18 amountX18 = collateral.convertTokenAmountToUd60x18(amount);
104: 
105:         
106:         
107:         
108:         IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
109: 
110:         if (asset == MarketMakingEngineConfiguration.load().weth) {
111:             
112:             _handleWethRewardDistribution(market, address(0), amountX18);
113:         } else {
114:             
115:             market.depositFee(asset, amountX18);
116:         }
117: 
118:         
119:         emit LogReceiveMarketFee(asset, marketId, amount);
120:     }
```


</details>

Another thing to note from zaros is that I learnt that the WBTC token has 8 decimals and this is very important to note. Usdc isnt the only weird erc20 token with 6 decimals. WBTC has 8 decimals so keep this mind



# 18 DO NOT INGORE ANY FUNCTION JUST BECAUSE YOU FOUND A BUG IN IT, THERE MIGHT BE MORE BUGS IN THAT FUNCTION

This is a very important point and one you must keep in mind. When looking at a function, if you find a bug early when looking at the function, this doesnt mean you should move on to find bugs elsewhere and discard that function because there can be more bugs in the same function. You need to make sure you still look through the whole function to see what else you can find. A great example of this was one I had in zaros and you can see this by looking at the findings.md for my findings. I first identified the Vault's debt can never be settled as Vault::marketsRealizedDebtUsd is always returns zero vulnerability which affected CreditDelegationBranch::settlevaultsdebt. I saw it always returned 0 but I didnt stop there. I still went into the function to see that if it worked correctly using the fix I suggested in the above finding, would everything work as expected? 

It was good I asked that and kept looking into CreditDelegationBranch::settlevaultsdebt as it led me to 2 more bugs which were CreditDelegationBranch::settlevaultsdebt incorrectly swaps token from marketmakingengine and not directly from Zlpvault which breaks protocol intended functionality and Incorrect swap amount in CreditDelegationBranch::settleVaultsDebt improperly inflates the tokens to swap leading to DOS or/and oversettling vault debt.

If I just assumed the fact that vault debt would always be zero which causes CreditDelegationBranch::settlevaultsdebt to settle zero debt and then moved on, I would have missed 2 more highs which is criminal and something you cannot afford to do.

# 19 COMPARE SHARE CALCULATIONS TO ERC4626 CALCULATIONS FOR SHARES WITH VAULTS

This idea came from someone else's finding I saw on zaros and the main thing to note is that whenever a protocol is implementing a vault, they may not use the ERC4626 standard exactly. They might choose to adapt the formulae for assets to share ratio to serve whatever purposes they want. Zaros did exactly this as they decided to include the total debt in the formula when calculating how many shares a user should get when depositing assets. You can look in the ZpVault.sol contract to see what they did. 

By doing this, it exposed a way for an attacker to get a lot more shares for way less assets. Note that inflation attacks arent the only problems that could occur with vaults so this is something you have to keep in mind. See details of the finding below:

Summary

An attacker can take advantage of the vault’s debt dynamics to gain an unfair share allocation. When the vault has a high level of debt, the share price decreases, enabling the attacker to mint an excessive number of shares for a minimal deposit. Once the debt is repaid, the share price returns to its normal level, allowing the attacker to redeem their shares for a significantly higher value, extracting substantial profits at the expense of the vault and its honest participants.

 Impact

* **Funds Drain:** The attacker can drain funds from the vault, leading to significant losses for other depositors.
* **System Insolvency:** The vault may become insolvent, undermining trust in the protocol and causing long-term damage to its reputation.

<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/zlp/ZlpVault.sol#L190>

<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/zlp/ZlpVault.sol#L172>

***

 Attack Scenario

 **1. Initial Vault State (Healthy)**

```solidity
// Initial state
uint256 totalAssets = 1000; // Total assets in the vault
uint256 debt = 0; // Vault debt
uint256 totalAssetsMinusVaultDebt = totalAssets - debt; // 1000
uint256 totalShares = 1000; // Total shares issued
uint256 sharePrice = totalAssetsMinusVaultDebt / totalShares; // 1 asset per share
```

 **2. Debt Increase**

A malicious user monitors the protocol for opportunities to increase the vault's debt or actively participates in creating debt.

```solidity
uint256 debt = 999; // Artificially inflated debt
totalAssetsMinusVaultDebt = totalAssets - debt; // 1
uint256 newSharePrice = totalAssetsMinusVaultDebt / totalShares; // 0.001 assets per share
```

 **3. Attacker Deposits 1 Asset & Mints Shares**

Using the share minting formula:
<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L224>

```solidity
uint256 previewSharesOut = assetsIn.mulDiv(
    totalShares + 10 ** decimalOffset,
    totalAssetsMinusVaultDebt,
    MathOpenZeppelin.Rounding.Floor
);
```

* With `assetsIn = 1` and `totalAssetsMinusVaultDebt = 1`, the attacker mints:

  ```solidity
  uint256 mintedShares = 1 * (1000 / 1); // 1000 shares
  ```

 **4. Debt Decrease**

After a certain period, the vault's debt decreases.

```solidity
uint256 debt = 0; // Debt repaid
totalAssetsMinusVaultDebt = totalAssets - debt; // 1000
```

 **5. Attacker Redeems Shares for Assets**

Using the redemption formula:
<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L178>

```solidity
uint256 previewAssetsOut = sharesIn.mulDiv(
    totalAssetsMinusVaultDebt,
    totalShares + 10 ** decimalOffset,
    MathOpenZeppelin.Rounding.Floor
);
```

* Redeeming `1000 shares`:

  ```solidity
  uint256 redeemedAssets = 1000 * (1000 / 2000); // 500 assets
  ```
* **Final Profit:** The attacker deposited **1 asset** and redeemed **500 assets**, netting a profit of **499 assets**.

***

 Root Cause

The vulnerability arises because the share minting and redemption formulas are directly influenced by `totalAssetsMinusVaultDebt`. When `totalAssetsMinusVaultDebt` is artificially reduced (due to inflated debt), the share price drops significantly, allowing attackers to mint an excessive number of shares. Once the debt is repaid, the share price increases, enabling the attacker to redeem their shares at a much higher value, resulting in substantial profits.


# 20 STEPWISE JUMPS 

This was a finding you missed in the zaros audit that is very important and you cannot miss again. A stepwise jump exploit is a form of attack where an exploiter takes advantage of how rewards, interest rates, or emissions are updated in discrete steps rather than continuously. So say a protocol aims to distribute rewards to its holders, it then sends the fees to be distributed to holders in a function , what can happen is that an attacker can frontrun that transaction where fees are sent to the protocol and then become a holder just before the fees are sent to the protocol and then they will be able to get a portion of the fees sent to the contract and once they get this, they can immediately unstake the token and leave with the extracted fees.

This was exactly the case with zaros which was detailed in this finding which i didnt spot. See below:

 Summary

In VaultRouterBranch.sol:stake(), users stake their `Vault.indexTokens` to earn rewards based on their share. However malicious users can frontrun the reward distribution by quickly staking their tokens right before distribution, receiving the same proportional rewards as long-term stakers.

 Vulnerability Details

Basically fee distribution being initiated by `FeeConversionKeeper` by checking checkUpkeep() function and then calling performUpkeep() to convert accumulated fees to WETH:


```solidity
function checkUpkeep(bytes calldata /**/ ) external view returns (bool upkeepNeeded, bytes memory performData) {
    FeeConversionKeeperStorage memory self = _getFeeConversionKeeperStorage();

    uint128[] memory liveMarketIds = self.marketMakingEngine.getLiveMarketIds();

    bool distributionNeeded;
    uint128[] memory marketIds = new uint128[]();
    address[] memory assets = new address[]();
    uint256 index = 0;
    uint128 marketId;

    // Iterate over markets by id
    for (uint128 i; i < liveMarketIds.length; i++) {
        marketId = liveMarketIds[i];

        (address[] memory marketAssets, uint256[] memory feesCollected) =
            self.marketMakingEngine.getReceivedMarketFees(marketId);

        // Iterate over receivedMarketFees
        for (uint128 j; j < marketAssets.length; j++) {
            distributionNeeded = checkFeeDistributionNeeded(marketAssets[j], feesCollected[j]);

            if (distributionNeeded) {
                // set upkeepNeeded = true
                upkeepNeeded = true;

                // set marketId, asset
                marketIds[index] = marketId;
                assets[index] = marketAssets[j];

                index++;
            }
        }
    }

    if (upkeepNeeded) {
        performData = abi.encode(marketIds, assets);
    }
}

// call FeeDistributionBranch::convertAccumulatedFeesToWeth
function performUpkeep(bytes calldata performData) external override onlyForwarder {
    FeeConversionKeeperStorage memory self = _getFeeConversionKeeperStorage();

    IMarketMakingEngine marketMakingEngine = self.marketMakingEngine;

    // decode performData
    (uint128[] memory marketIds, address[] memory assets) = abi.decode(performData, (uint128[], address[]));

    // convert accumulated fees to weth for decoded markets and assets
    for (uint256 i; i < marketIds.length; i++) {
        marketMakingEngine.convertAccumulatedFeesToWeth(marketIds[i], assets[i], self.dexSwapStrategyId, "");
    }
}

```

In the marketMakingEngine.convertAccumulatedFeesToWeth() contract handles WETH reward distribution:


```solidity
function _handleWethRewardDistribution(
    Market.Data storage market,
    address assetOut,
    UD60x18 receivedWethX18
)
    internal
{
    // cache the total fee recipients shares as UD60x18
    UD60x18 feeRecipientsSharesX18 = ud60x18(MarketMakingEngineConfiguration.load().totalFeeRecipientsShares);

    // calculate the weth rewards for protocol and vaults
    UD60x18 receivedProtocolWethRewardX18 = receivedWethX18.mul(feeRecipientsSharesX18);
    UD60x18 receivedVaultsWethRewardX18 =
        receivedWethX18.mul(ud60x18(Constants.MAX_SHARES).sub(feeRecipientsSharesX18));

    // calculate leftover reward
    UD60x18 leftover = receivedWethX18.sub(receivedProtocolWethRewardX18).sub(receivedVaultsWethRewardX18);

    // add leftover reward to vault reward
    receivedVaultsWethRewardX18 = receivedVaultsWethRewardX18.add(leftover);

    // adds the weth received for protocol and vaults rewards using the assets previously paid by the engine
    // as fees, and remove its balance from the market's `receivedMarketFees` map
    market.receiveWethReward(assetOut, receivedProtocolWethRewardX18, receivedVaultsWethRewardX18);

    // recalculate markes' vaults credit delegations after receiving fees to push reward distribution
    Vault.recalculateVaultsCreditCapacity(market.getConnectedVaultsIds());
}

```

Due to the combination of market.receive Distribution.distributeValue() function is triggered, resulting in fees being distributed to users.

```solidity
function distributeValue(Data storage self, SD59x18 value) internal {
    if (value.eq(SD59x18_ZERO)) {
        return;
    }

    UD60x18 totalShares = ud60x18(self.totalShares);

    if (totalShares.eq(UD60x18_ZERO)) {
        revert Errors.EmptyDistribution();
    }

    SD59x18 deltaValuePerShare = value.div(totalShares.intoSD59x18());

    self.valuePerShare = sd59x18(self.valuePerShare).add(deltaValuePerShare).intoInt256();
}
```

So by checking `FeeConversionKeeper.sol:checkUpkeep()` frontrunner can understand when `Distribution.distributeValue()` will be called.
And by staking his tokens just before fee distribution he can receive rewards.

You can view the full finding at: https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/s/168 



# 21 ALWAYS DOUBLE CHECK PAUSED FUNCTIONALITY, SLITHER ENTRY POINTS USEFUL COMMAND

this was another easy finding I missed where zaros had paused functionality to be able to pause the contract and its functions at any point. Whenever you see any protocol that utilizes the pause functionality, it is so important to go and check all key functions if the pause modifier is applied to every key function because it seems mundane but a lot of protocols will forget especially when they have a lot of lines of code to write, it is easy to forget. You must always look out for this. See the finding below:

 Summary

The `claimFees` function in `FeeDistributionBranch.sol` uses `Vault.load()` instead of `Vault.loadLive()`, allowing users to claim fees from paused vaults. This bypasses the vault's pause mechanism which is designed to halt all vault operations during paused state.

Vulnerability Details

The `claimFees` function uses basic vault loading without status validation:
```solidity
function claimFees(uint128 vaultId) external {
    // Uses basic load without vault status check
@>  Vault.Data storage vault = Vault.load(vaultId);
    
    bytes32 actorId = bytes32(uint256(uint160(msg.sender)));
    
    // Checks shares and claims fees
    if (vault.wethRewardDistribution.actor[actorId].shares == 0) 
        revert Errors.NoSharesAvailable();
    
    UD60x18 amountToClaimX18 = vault.wethRewardDistribution.getActorValueChange(actorId).intoUD60x18();
    if (amountToClaimX18.isZero()) 
        revert Errors.NoFeesToClaim();
    
    // Updates distribution state and transfers WETH
    vault.wethRewardDistribution.accumulateActor(actorId);
    address weth = MarketMakingEngineConfiguration.load().weth;
    // ... WETH transfer logic
}
```

The issue arises because:

1. `Vault.load()` is used which doesn't check vault status
2. Fees can be claimed even when the vault is paused
3. This contradicts the vault's pause mechanism which should halt all operations

 Impact

Allows fee claims when vault operations should be frozen
 You can view the full finding at: https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/s/618

The way to avoidmissing simple findings like this is to use a very useful slither command which is:

```bash
slither . --print entry-points
```
The result of this will show you all of the entry points in the contracts and all modifiers and will help you see which functions are paussed, which have only owner, etc. This is a very useful tool.

# 22 USDT PARTIAL ALLOWANCE REVERT ON APPROVALS

 This was another cool finding from reported issues. The key takeaway point is that USDT doesnt enable incrreasing allowance. what this means is that if you set an approval for an amount of usdt token to another address and that address hasnt used up the whole approved amount and you try to approve more tokens, the tx will revert.

 You need to revoke the approval for that address (set it to 0) and then re-approve if you want to approve a different amount. This is important for protocols who run approvals before sending tokens to other contracts which can be key logic. If they support USDT, this check is something to consider and if they run their approvals for USDT without checking, then you have a solid finding there. See the report below:

 Partial‐Approve Revert Vulnerability in UniswapV2Adapter

The UniswapV2Adapter contract can revert on swap operations for tokens like USDT that have a non‐standard allowance mechanism. USDT historically reverts if you attempt to change an existing non‐zero allowance to another non‐zero value. Consequently, a leftover allowance or dust balance of USDT in the adapter can break all subsequent approvals.

Although the Zaros protocol vaults are not currently configured to accept USDT, the published documentation states that USDT support will be added. As of now, if USDT were used with the UniswapV2Adapter, it could cause DoS‐type reverts and prevent any further USDT swaps.

Vulnerability Details
USDT’s ERC‐20 implementation reverts on partial approval changes. Specifically, if the adapter already has a non‐zero allowance for USDT, a subsequent call to:

```solidity
IERC20(tokenIn).approve(uniswapV2SwapStrategyRouter, amountIn);
```

where amountIn != 0, reverts under USDT’s rules unless the allowance is first reset to zero. This means a malicious actor (or an accidental leftover) could deposit a small USDT amount into the adapter, leaving a non‐zero allowance. Next time the adapter tries to set a new allowance for USDT, the transaction fails.

As a result, all USDT swaps will become blocked, causing a denial of service for that token. This vulnerability is specific to USDT or any similarly non‐compliant token.

The UniswapV2Adapter code repeatedly does:

```solidity
IERC20(swapPayload.tokenIn).approve(router, swapPayload.amountIn);
```

With no zero‐allowance reset.

A malicious user can “dust” 1 unit of USDT or rely on leftover allowance, causing subsequent approvals to revert. 

You can view the full finding at: https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/s/110

# 23 PAY ATTENTION TO FOR LOOPS, SINGLE ITERATION VS FINAL RESULT

The details of this exploit aren't actually important as its nothing that we dont know about the reason why you missed it is because you didnt pay attention to single iteration vs final result.

So in this bug, the main idea is that there was a for loop which was supposed to do something and in each iteration, it did what it was supposed to do but when the final result came out, the cummulation of what each iteration didnt add up to what the for loop was supposed to do because one variable was passed in each iteration that affected each iteration and the sum of its effect on each iteration messed up the final result. If you remeber the msg.value bug that occured where msg.value used in a loop can be very problematic which i covered in one of the audit notes on github, this was the overall cause of the issue. The single iteration did what it was meant to but the overall result was devastatingly wrong. This is the same issue here but the effect is a bit less but still significant. This is something you need to watch out for whenever you see a for loop, your mind must go to single iteration vs final result exploit. 

You can read about the full issue here: https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/s/872 

I would normally paste code snippets and stuff but the detail of the bug is irrelevant. The main thing to note is the single iteration vs final result I stated above.

# 24 PAY ATTENTION TO ARRAYS INITIALIZED WITH ARBITRARY SIZES

The summary of this finding is that if you see any array being initialised anywhere in solidity, make sure that the array is not given an arbitrary size. i.e. a size that doesnt correspond to the amount of elements stored in the array. This is kinda hard to explain without looking at the exploit. So lets have a look:

 Summary

The function `performUpkeep` in the FeeConversionKeeper contract always reverts due to an improper memory allocation issue in checkUpkeep. Specifically, the code over-allocates memory for the marketIds and assets arrays.

 Vulnerability Details

The arbitrary 10 multiplication leads to uninitialized memory slots, causing invalid data access. performUpkeep attempts to process these uninitialized indexes, leading to a revert.
The error arises when trying to access `totalAssets()` on an uninitialized zero address, leading to a revert.

```solidity
/src/external/chainlink/keepers/fee-conversion-keeper/FeeConversionKeeper.sol:69
69:     function checkUpkeep(bytes calldata /**/ ) external view returns (bool upkeepNeeded, bytes memory performData) {
70:         FeeConversionKeeperStorage memory self = _getFeeConversionKeeperStorage();
71: 
72:         uint128[] memory liveMarketIds = self.marketMakingEngine.getLiveMarketIds();
73: 
74:         bool distributionNeeded;
75:         uint128[] memory marketIds = new uint128[](liveMarketIds.length * 10);
76:         address[] memory assets = new address[](liveMarketIds.length * 10);
```

Consider the following Proof of code:


```solidity
/test/integration/external/chainlink/keepers/fee-conversion/checkUpkeep/checkUpkeep.t.sol:32
32:     function test_WhenMarketsHaveMoreThanMinFeeForDistribution( // @audit POC
33:     )
34:         external
35:         givenCheckUpkeepIsCalled
36:     {
37:         uint256 marketId=3967010399546;
38:         uint256 amount=115792089237316195423570985008687907853269984665640564039457584007913129639932;
39:         uint256 minFeeDistributionValueUsd=9306648518645823426120026427;
40: 
41:         PerpMarketCreditConfig memory fuzzPerpMarketCreditConfig = getFuzzPerpMarketCreditConfig(marketId);
42: 
43:         minFeeDistributionValueUsd = bound({
44:             x: minFeeDistributionValueUsd,
45:             min: 1,
46:             max: convertUd60x18ToTokenAmount(address(usdc), USDC_DEPOSIT_CAP_X18)
47:         });
48: 
49:         amount = bound({
50:             x: amount,
51:             min: minFeeDistributionValueUsd,
52:             max: convertUd60x18ToTokenAmount(address(usdc), USDC_DEPOSIT_CAP_X18)
53:         });
54:         deal({ token: address(usdc), to: address(fuzzPerpMarketCreditConfig.engine), give: amount });
55: 
56:         changePrank({ msgSender: users.owner.account });
57: 
58:         configureFeeConversionKeeper(1, 1);
59: 
60:         changePrank({ msgSender: address(fuzzPerpMarketCreditConfig.engine) });
61: 
62:         marketMakingEngine.receiveMarketFee(fuzzPerpMarketCreditConfig.marketId, address(usdc), amount);
63: 
64:         (bool upkeepNeeded, bytes memory performData) = FeeConversionKeeper(feeConversionKeeper).checkUpkeep("");
65: 
66:         (uint128[] memory marketIds, address[] memory assets) = abi.decode(performData, (uint128[], address[]));
67:         console.log("marketIds" , marketIds.length);
68:         // it should return true
69:         assertTrue(upkeepNeeded);
70: 
71:         assertEq(assets[0], address(usdc));
72:         assertEq(marketIds[0], fuzzPerpMarketCreditConfig.marketId);
73: 
74:         FeeConversionKeeper(feeConversionKeeper).performUpkeep(performData);
75:     }
```

## Output

```shell

    │   │   │   │   ├─ [0] 0x0000000000000000000000000000000000000000::totalAssets() [staticcall]
    │   │   │   │   │   └─ ← [Stop] 
    │   │   │   │   └─ ← [Revert] EvmError: Revert

```

The revert occurs when accessing an uninitialized zero address index, causing `totalAssets()` to fail.


So as you can see, in the checkUpkeep function, a marketIds array was initialised with a random size that didnt correspond to the amount of marketIds that the vault actually had. What this meant was that when checkUpkeep called the function and looped through the marketIds array to get the totalAssets in the market, it would get to an element in the marketIds array which has a zero value so trying to call totalAssets on a zero address will obviously revert. So the moral of the story is that if you see any array initialised or looped through, make sure to check the size of the array and make sure that all elements in the array actually have values in them because if actions are performed on an element that has no values in it, it could lead to an unexpected revert.


