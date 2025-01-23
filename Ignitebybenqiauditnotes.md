ADD NOTES ABOUT ACCESSCONTROLUPGRADEABLE CONTRACT . WHOEVER IS GRANTED A ROLE CAN REVOKE ROLES. THIS IS VERY IMPORTANT

# 1 CALCULATING APR TO CALCULATE THE REWARD FOR AN ASSET

Explanation of `_calculateRewardAmount` Function

The `_calculateRewardAmount` function calculates the **reward amount** for a validator based on the duration of validation, the stake amount, and the target APR (Annual Percentage Rate). Below is a detailed explanation.

---

**Final Formula**

The formula used in the function can be expressed as:

{Reward Amount} = {stakeAmount} times {targetApr} times {validationDuration} / {10,000 times 60 times 60 times 24 times 365} this denominator is what converts the annual reward into a reward per second:
/ 60 / 60: Converts seconds to hours.
/ 24: Converts hours to days.
/ 365: Converts days to years.

---

**Explanation with an Example**

**Scenario**:

- `validationDuration` = 7 days = 7 times 24 times 60 times 60 = 604,800seconds
- `stakeAmount` = 1,000 tokens
- `targetApr` = ( 500 bps (5% APR)). bps means basis points and target apr is 500 because scaling factor for percentages in solidity is 10000 so 500/10000 is 5%. See gogopool notes.md

---

**Calculation**:

1. **Calculate the Annual Reward**:

   {Annual Reward} = {stakeAmount}times{targetApr}/{10,000} (scaling factor)

   Substituting values:

   {Annual Reward} = {1,000 times 500}/{10,000} = 50 tokens

2. **Prorate the Reward for the Active Duration**:

   {Reward for 7 days} = {{Annual Reward} times {validationDuration}}/{60 times 60 times 24 times 365}

   Substituting values:

   {Reward for 7 days} = {50 times 604,800}/{60 times 60 times 24 times 365}.

3. **Simplify the Denominator**:

   - 60 times 60 times 24 times 365 = 31,536,000 seconds in a year.

4. **Final Calculation**:

Reward for 7 days = 50 times 604,800/31,536,000 = 0.96 tokens

---

**Key Insights**

1. **Proportionality**:

   - The reward amount is proportional to:
     - The **stake amount** (`stakeAmount`).
     - The **target APR** (`targetApr`).
     - The **validation duration** (`validationDuration`).

2. **Assumptions**:

   - Rewards accrue **linearly** over time.
   - `validationDuration` is measured in **seconds**.

3. **Use Case**:
   - This formula is commonly used in staking or validator reward systems to calculate fair rewards based on participation.
   -

# 2 CONVERTING TOKEN AMOUNTS TO EQUIVALENT AMOUNTS OF ANOTHER TOKEN USING TOKEN DECIMALS EFFECTIVELY (UNLESS TOKEN HAS HIGHER THAN 18 DECIMALS)

Explanation of tokenamount function in `Ignite:: registerWithErc20Fee`

This calculation determines the number of tokens needed to cover a registration fee denominated in AVAX, considering the prices of AVAX and the token, as well as the token's decimals.
In this case, we want to convert an amount of avax tokens to an equivalent amount of another token. so if we have 5 avax and we want to know what 5 avax is worth in terms of another token.

---

Given:

- **`avaxPrice = 2e8`** (price of 1 AVAX is 2 in 8-decimal format, e.g., 2 USD).
- **`tokenPrice = 1e8`** (price of 1 QI token is 1 in 8-decimal format, e.g., 1 USD).
- **`registrationFee = 1e18`** (registration fee is 1 AVAX, in 18 decimals).
- **`token.decimals() = 18`** (QI token has 18 decimals).

We aim to compute how many QI tokens are required for the registration fee.

---

**Step-by-Step Calculation**

**1. Multiply AVAX Price by Registration Fee**

Value of AVAX in token units = {avaxPrice} times {registrationFee}

Substitute values:

2e8 times 1e18 = 2e26

---

**2. Divide by Token Price**

Token Equivalent of Registration Fee = Value of AVAX/tokenPrice

Substitute values:

2e26/1e8 = 2e18

**3. Adjust for Token Decimals**

The final step adjusts for the token’s decimals:

Final Token Amount = Token Equivalent of Registration Fee/10^(18 -token.decimals)

Since **`token.decimals() = 18`**, the adjustment becomes:

10^{18 - 18} = 10^0 = 1

No scaling is applied, so the result remains:
2e18

---

**Final Result**

- **QI Tokens Required**: ( 2e18 ) (equivalent to 2 QI tokens in 18-decimal format).

---

**What Changes if Token Decimals = 6 (e.g., USDC)?**

If `token.decimals()` is 6, like USDC, the calculation changes as follows:

**Adjustment for Token Decimals**

10^{18 - 6} = 10^{12}

**Final Token Amount**
Final Token Amount = 2e18/10^{12} = 2e6

**Final Result for USDC**

- **USDC Tokens Required**: ( 2e6) (equivalent to 2 USDC in 6-decimal format).

---

**Key Insights**

1. **For QI (18 decimals)**:(2e18) tokens are required.
2. **For USDC (6 decimals)**: ( 2e6 ) tokens are required.
3. The adjustment using 10^(18 - token.decimals ) ensures compatibility across tokens with different decimal formats.
4. If the token has higher than 18 decimals , this function wont work and some weird erc20's have higher than 18 decimal places. I raised this in the code.

# 3 DELETING ELEMENTS FROM AN ARRAY WHILE MAINTAINING COMPACTNESS AND AVOIDING ERRORS PROPERLY IN SOLIDITY

Explanation of `_deleteRegistration` Function's `if` Statements

The `_deleteRegistration` function removes a node registration and ensures data integrity by reindexing affected mappings and arrays. The two `if` statements in the function are critical for maintaining the compactness of arrays and updating mappings to reflect changes.

---

**First `if` Statement**

Code

```solidity
if (accountRegistrationIndex != accountRegistrationLastIndex) {
    string memory lastNodeId = registeredNodeIdsByAccount[registerer][
        accountRegistrationLastIndex
    ];
    registeredNodeIdsByAccount[registerer][
        accountRegistrationIndex
    ] = lastNodeId;
    accountRegistrationIndicesByNodeId[
        lastNodeId
    ] = accountRegistrationIndex;
}
```

Purpose
Ensures the registeredNodeIdsByAccount array remains compact after a deletion by swapping the last element into the position of the deleted element.
Key Components
accountRegistrationIndex:
Index of the node being deleted in the registeredNodeIdsByAccount array.

accountRegistrationLastIndex:
Index of the last element in the registeredNodeIdsByAccount array.

If the node being deleted is not the last element in the array:
Retrieve the last node ID from the array, move the last node ID into the position of the node to be deleted. Update the mapping accountRegistrationIndicesByNodeId to reflect the new index of the moved node. After this, there will be duplicates in the registeredNodeIdsByAccount array which is why the last element is deleted using the pop function. Same for the accountRegistrationIndicesByNodeId mapping. there will be duplicates as the last node id and the node id we want to delete will now be pointing to the same index number in the array which is wrong but since we delete teh mapping of the node id, this issue is mitigated.

Why It's Needed
Prevents gaps in the registeredNodeIdsByAccount array by keeping it contiguous. Ensures the mappings remain consistent with the updated array.
Example
Before Deletion:
registeredNodeIdsByAccount[registerer] = [Node1, Node2, Node3].
Node2 is at index 1, Node3 is at index 2.
After Deletion:
Node3 replaces Node2: [Node1, Node3].
The mapping updates: accountRegistrationIndicesByNodeId[Node3] = 1.

Second if Statement
Code

```solidity

if (registrationIndex != totalRegistrations) {
    string memory lastNodeId = registrations[totalRegistrations].nodeId;
    registrations[registrationIndex] = registrations[totalRegistrations];
    registrationIndicesByNodeId[lastNodeId] = registrationIndex;
}
```

Purpose
Ensures the registrations array remains compact by swapping the last registration into the position of the deleted registration.

Key Components
registrationIndex:
Index of the node being deleted in the registrations array.

totalRegistrations:
Index of the last element in the registrations array.

What It Does
If the node being deleted is not the last element in the registrations array:
Retrieve the last registration record.
Move the last record into the position of the deleted record.
Update the mapping registrationIndicesByNodeId to reflect the new index of the moved record. Same idea as the first if statement I explained above.

Why It's Needed
Prevents gaps in the registrations array by keeping it contiguous.
Ensures the mappings remain consistent with the updated array.

Example
Before Deletion:
registrations = [Reg1(Node1), Reg2(Node2), Reg3(Node3)].
Reg2 is at index 1, Reg3 is at index 2.
After Deletion:
Reg3 replaces Reg2: [Reg1(Node1), Reg3(Node3)].
The mapping updates: registrationIndicesByNodeId[Node3] = 1.

# 4 VS CODE OPEN TO THE SIDE OPTION, ALT + ARROW HELPFUL TIPS

When auditing code, you might want to look at 2 different functions at the same time and you dont want to keep scrolling up and down in the same file. You might also want to look some code from one function in another contract and compare it to code in a contract you are looking at. Instead of scrolling back and forth between screens, in the file explorer of vs code , right click the file you want to open and click open to the side and you will see both files side by side which will help you easier navigate files. This is a very useful tip that will help you a lot. Also, if you hav navigated to a part of the file and you want to go back to where you were teh cursor was previously, the commant is alt + left arrow and to go forward, it is alt + right arrow.

# 5 ONLY FUNCTIONS MARKED WITH PAYABLE CAN RECEIVE RAW ETHER

This is something you should know but only functions marked as payable can receive ether. You based a huge 'bug' on the fact that you could send raw avax to the registerwithoutcollateral function but that wasnt possible. You shouldve known this but now you are reminded, you can do better next time.

# 6 ASSUMPTION ANALYSIS - FINDING H/M's EASILY. EACH LINE HAS AN ASSUMPTION IN IT. EACH ASSUMPTION HAS MONEY IN IT. GET THE MONEY

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

# 7 LOOKING AT PREVIOUS FINDINGS ON SIMILAR PROTOCOLS

If you are looking at a protocol's code and you arent finding anything, a good way to get inspiration is to look for similar protocols on solodit and look at different H/M findings related to similar protocols. These findings could help you find similar issues in the protocol you are looking at. If the exploit you are looking at is something brand new that you havent seen before, add it to the notes.md attached to the audit you are doing as you would usually do.
