# General Info

To get scope, I read the gogopool litepaper at https://docs.gogopool.com/protocol/gogopool-litepaper so i could understand what was going on
check //c (comments), //q questions and //bug (potential issues) i found in all the contracts below. There are also a lot of learning point below so this is worth a review

# Storage Contract

Storage contract is used to register contracts and store other data related to gogo protocol. contains different mappings and other things that the protocol might want to store. this is a very important contract as is is used multiple times across the whole protocol

contains this natspec This contract will be deployed via create2 proxy, so msg.sender will not work. this is because in the constructor, they set a guardian and they want to set it to the address that deployed the contract, they use a proxy so msg.sender will be the proxy contract which we dont want so they used tx.origin to get the person who first deployed the contract

# BaseTest Contract

starts the protocol off by deploying all relevant contract and storing them in storage contract.
if you look at the setup function, they call register contract function which simply sets up the contract in the correct mappings in the storage contract.

the protocol uses a rialto multisig which i am guessing is a wallet that interacts between avax chains. the p chain is where stakin is managed and people can stake their avax to different validators and where validators/delegators are made. c chain is the customer chain where all smart contract interactions happen. so this rialto multisig manages the interaction between 2 chains and allows people stake on the c chain and it moves it to the p chain and manages their delegations and everything else. in the base test, they create a simulator of that multisig that we can use to test which doesnt actually do any of that c chain / p chain stuff but it acts like it does just for testing purposes.

# Staking contract, APPROVE AND PULL METHODOLOGY VS DOUBLE TRANSFER GAS SAVINGS

when a user stakes ggp, this contract holds the ggp that the user sent but the user balances are all updated in the storage.sol contract. The staking contract deposits the token to the vault contract which is what holds all the avax and ggp sent to the protocol. the vault contract then updates the staking contract balance to reflect the amount staked.

There is potential misinformation in the vault contract because they say that that because the staking contract approves the amount of tokens to send to the vault contract and then calls deposit and then in that function the vault contract calls transferfrom, they save gas by avoiding a double transfer on the network token. What this means is that the staking contract didnt call a transfer function to transfer the ggp to the vault. it called deposit function which calls a transferfrom to get the tokens from the contract. this method is called approve and pull and is used by protocols to avoid spending extra gas by calling 2 transfers but the only way this saves gas is if the staking contract approves a large amount of ggp the first time so it wont have to call approve everytime a user wants to stake ggp. If a large amount of tokens is approved by the staking contract to vault contract the first time, then whenever a user calls stakeggp, it wont have to approve an amount again . it can just call the deposit function in the vault contract and this saves gas over double transfers because the staking contract doesnt have to call the second transfer everytime a user wants to stake to send the tokens to the vault which can be gas intensive. in this case though, when the approval is done in the staking contract, only the amount that the user is staking is approved everytime so this means that once the amount is transferred to the vault token, the allowance resets to 0 and when the next user calls stakeggp, the approval has to be done again which kinda defeats the point of the approve and pull over double transfer. i need to report this

see the approve and pull method below:

```javascript
/// @notice Stakes GGP in the protocol
	/// @param stakerAddr The C-chain address of a GGP staker in the protocol
	/// @param amount The amount of GGP being staked
	function _stakeGGP(address stakerAddr, uint256 amount) internal whenNotPaused {
		emit GGPStaked(stakerAddr, amount);

		// Deposit GGP tokens from this contract to vault
		Vault vault = Vault(getContractAddress("Vault"));
		TokenGGP ggp = TokenGGP(getContractAddress("TokenGGP"));
		ggp.approve(address(vault), amount); //bug amount is going to be approved everytime this function is called which defeats the gas saving incentive of using approve and pull method
		//ggp.safeTransfer(address(vault), amount); comment out the lines above and below and uncomment this and run testGetTotalGGPStake2 in staking.t.sol and you will see that using safeTransfer has this result  console::log("Gas used for 12 stakes: %d", 598993 [5.989e5]) and using approve and pull has this result console::log("Gas used for 12 stakes: %d", 1009447 [1.009e6]) so the gas saving incentive is not being achieved at all. in fact it has the opposite effect
		vault.depositToken("Staking", ggp, amount);
		int256 stakerIndex = getIndexOf(stakerAddr);
		if (stakerIndex == -1) {
			// create index for the new staker
			stakerIndex = int256(getUint(keccak256("staker.count"))); //c this hash counts the amount of stakers in the protocol and increments it by 1 in the next line to signify a new staker
			addUint(keccak256("staker.count"), 1);
			setUint(keccak256(abi.encodePacked("staker.index", stakerAddr)), uint256(stakerIndex + 1)); //c this gives the staker an index based on the amount of stakers in the protocol + 1
			setAddress(keccak256(abi.encodePacked("staker.item", stakerIndex, ".stakerAddr")), stakerAddr); //c this should be stakerindex+1 to be consistent with the index set in the previous line. it can be confusing for people looking at this. do this and run test_staker test and it will pass with better consistency as getindexof will return the correct user index rather than subtracting 1 from it. see the comment i raised in the getindexof function. this is known by the protocol already though so it is not an issue
			increaseGGPStake(stakerAddr, amount);
		}
	}

```

this was in the staking contract and the vault contract had the following:

```javascript
/// @notice Accept a token deposit and assign its balance to a network contract
	/// @dev (saves a large amount of gas this way through not needing a double token transfer via a network contract first)
	/// @param networkContractName Name of the contract that the token will be assigned to
	/// @param tokenContract The contract of the token being deposited
	/// @param amount How many tokens being deposited
	function depositToken(string memory networkContractName, ERC20 tokenContract, uint256 amount) external guardianOrRegisteredContract {
		//bug why is guardian allowed to bypass registered network contract and deposit. guardian can manipulate the balances by passing any registered network contract name
		// Valid Amount?
		if (amount == 0) {
			revert InvalidAmount();
		}
		// Make sure the network contract is valid (will revert if not)
		getContractAddress(networkContractName);
		// Make sure we accept this token
		if (!allowedTokens[address(tokenContract)]) {
			revert InvalidToken();
		}
		// Get contract key
		bytes32 contractKey = keccak256(abi.encodePacked(networkContractName, address(tokenContract)));
		// Emit token transfer event
		emit TokenDeposited(contractKey, address(tokenContract), amount);
		// Send tokens to this address now, safeTransfer will revert if it fails
		tokenContract.safeTransferFrom(msg.sender, address(this), amount);
		// Update balances
		tokenBalances[contractKey] = tokenBalances[contractKey] + amount; //c not reentrancy because this isnt native avax transfer and token isnt erc777
	}
```

You can see the line in the natspec where they say it saves a massive amount of gas but it evidently doesnt after i ran the test I spoke about in testGetTotalGGPStake2 in staking.t.sol and i showed the results in the bug comment i made in the code from the staking contract i pasted above.

I also left a comment in the getindexof function in the staking contract. it is more informational than anything but it allows the index to be set consistently across the code. I might try to escalate this but in the stakeggp function, there is the following line:

```javascript
setUint(
	keccak256(abi.encodePacked("staker.index", stakerAddr)),
	uint256(stakerIndex + 1)
); //c this gives the staker an index based on the amount of stakers in the protocol + 1
setAddress(
	keccak256(abi.encodePacked("staker.item", stakerIndex, ".stakerAddr")),
	stakerAddr
); //bug this should be stakerindex+1 to be consistent with the index set in the previous line. it can be confusing for people looking at this. do this and run test_staker test and it will pass with better consistency as getindexof will return the correct user index rather than subtracting 1 from it. see the bug i raised in the getindexof function
```

since getindexof always subtracts 1 when getting the index, this doesnt hurt right now but there is a discrepancy between the staker index set for the address(first line ) and the address set for the staker index mapping (second line). this cannot be right and could lead to issues. i need to find where this can be a bigger problem

# MinipoolManager contract, PERCENTAGE SCALING FACTOR

the idea of this contract is that it allows people create minipools and people can stake avax to these minipools. these minipools are like validators on the avax network as per the gogo docs.

the same issue i found with the index seems to be present here in this contract. the getindex function is defined the same way as it was in the staking.sol contract.

// Current rule is matched funds must be 1:1 nodeOp:LiqStaker this bit of documentation is in this file and this is a potential invariant to test

In this contract, there is the following line in the createminipoolonbehalfof function:

```javascript
if (delegationFee < 20_000 || delegationFee > 1_000_000) {
			//bug magic number used here can confuse the reader. natspec of this function says 2% is 20_000 and earlier natspec said minipool.item<index>.delegationFee = node operator specified fee (must be between 0 and 1 ether) 2% is 0.2 ether , this isnt true 2% is 20_000 as it said in the natspec of this function so if a user enters delegation fee of 0.2 ether, that wont work based on this reasoning
			revert DelegationFeeOutOfBounds();
		}


```

the natspec on this function defines the delegationfee as /// @param delegationFee Percentage delegation fee in units of ether (2% is 20_000). The reason 2% is 20000 is because both numerator and denominator are multiplied by a scaling factor of 10000. This is because we want to represent smaller percentages like 0.1 % or 0.01% . In solidity, we cannot type 0.1 as it doesnt accept decimals so instead, we can multiply both numerator and denominator by 10000 so 0.1/100 now becomes 1000/1000000 which is the same. In reality, the scaling factor doesnt have to be 10000. It is just the one that has been chosen by the community as the most efficient percentage scaling factor so we can represent percentages as small as 0.0001% and if you want to represent smaller percentages, you can change the scaling factor as you choose.

dont forget about bug escalation. this will teach you a lot. if you see a mistake, go and find where else the code the variabe or function was used or could be used and see how you can attack attack attack attack. Typical example is the the following:

```javascript
setUint(
	keccak256(
		abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpInitialAmt")
	),
	msg.value
); //bug when this value is reset, this value will change to the new value, this is a problem because the value of the initial deposit shouldnt change as per the natspec at the start of this documentation which says minipool.item<index>.avaxNodeOpInitialAmt = avax deposited by node operator for the **first** validation cycle
```

this line was in the createminipoolonbehalfof function and when i noticed this, i immediately searched the code for any other place where this .avaxNodeOpInitialAmt was used to see if i could find a bigger issue just in case this was an actual issue.

There is another function called recordstaking which is used to recordallthevalidator stats once their staking cycle is over. The function shows us how much the validaot and the ggavax holders get paid. See relevant section of the function below:

```javascript
/// @notice Records the nodeID's validation period end
	/// @param nodeID 20-byte Avalanche node ID
	/// @param endTime The time the node ID stopped validating Avalanche
	/// @param avaxTotalRewardAmt The rewards the node received from Avalanche for being a validator
	/// @dev Rialto will xfer back all staked avax + avax rewards. Also handles the slashing of node ops GGP bond.
	function recordStakingEnd(address nodeID, uint256 endTime, uint256 avaxTotalRewardAmt) public payable { //c so we have to trust rialto to call this function with the correct parameters which presents a security risk
		int256 minipoolIndex = onlyValidMultisig(nodeID);//c they keep tryna get me with this multisig check. i am excited thinking i have found a bug but i havent
		requireValidStateTransition(minipoolIndex, MinipoolStatus.Withdrawable);

		Minipool memory mp = getMinipool(minipoolIndex);
		if (endTime <= mp.startTime || endTime > block.timestamp) {
			revert InvalidEndTime();
		}

		uint256 totalAvaxAmt = mp.avaxNodeOpAmt + mp.avaxLiquidStakerAmt;
		if (msg.value != totalAvaxAmt + avaxTotalRewardAmt) {
			revert InvalidAmount();
		}

		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".status")), uint256(MinipoolStatus.Withdrawable));
		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".endTime")), endTime);
		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxTotalRewardAmt")), avaxTotalRewardAmt);

		// Calculate rewards splits (these will all be zero if no rewards were recvd)
		// NOTE: Commission fee amount fails to persist for Node Operators across cycling minipools.
		//       Currently, setting MinipoolNodeCommissionFeePct to 0 (as of 2/23/2024) avoids the issue,
		//       ensuring a 50/50 reward split. Revisit this logic if we want to reinstate a commission fee
		uint256 avaxHalfRewards = avaxTotalRewardAmt / 2;

		// Node operators recv an additional commission fee if dao.getMinipoolNodeCommissionFeePct() is not 0 which it is right now
		ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
		uint256 avaxLiquidStakerRewardAmt = avaxHalfRewards - avaxHalfRewards.mulWadDown(dao.getMinipoolNodeCommissionFeePct()); //c rewards from a validator are split 50/50 between the node operator and the liquid staker
		uint256 avaxNodeOpRewardAmt = avaxTotalRewardAmt - avaxLiquidStakerRewardAmt;

		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpRewardAmt")), avaxNodeOpRewardAmt);
		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerRewardAmt")), avaxLiquidStakerRewardAmt);
```

This isnt the full function but the aim was to highlight how the rewards are calculated. We can run an invariant test to check that those rewards are always the same. the rewards between liquid stakers and the validator (creator the minipool should be the same). The totalrewards sent back from rialto are entered into this function as parameters which is kinda worrying because what if they make a mistake when entering the parameters. This means rewards can be manipulated by the rialto multisig. These half rewards are sent to the ggavax contract as rewards via the depositfromstaking function in the erc4262 contract which i covered below.

# Vault contract

deposittoken function has a guardianOrRegisteredContract modifier and is weirdly the only function in the whole contract with that . this presents the risk of a malicious guardian being able to deposit tokens cheaper into the protocol without going via the network contract. why is that the case. the rest have onlyregisteredcontract modifiers for normal network contracts. this allows the guardian manipulate the balances by passing any registered network contract name and depositing tokens into the vault.

# TokenggAvax contract, ERC4626 TOKENS, VAULT TOKENS, INFLATION ATTACKS

this contract allows people swap avax for ggavax. ggavax is an interest bearing token that follows the EIP-4626 standard and you can read about this standard at:
https://eips.ethereum.org/EIPS/eip-4626. Reading and understanding this will give you scope on how the ggavax token contract was constructed.

Notice how it inherits from an erc4626upgradeable token to show that it is upgradeable but we already know about upgradeability. What i want to discuss is the idea of tokens using erc4246 methodology and how the ggavax token uses it. If you read the erc4626 link above, you should mostly understand what is happening there.

There is an asset which is supposed to be an erc20 token and this is very important to note because in the tokenggavax contract, say in the depositavax function, this is how ggavax handles it:

```javascript
/// @notice Allows users to deposit AVAX and receive ggAVAX
	/// @return shares The amount of ggAVAX minted
	function depositAVAX() public payable returns (uint256 shares) {
		uint256 assets = msg.value;
		// Check for rounding error since we round down in previewDeposit.
		if ((shares = previewDeposit(assets)) == 0) {
			revert ZeroShares();
		}

		emit Deposit(msg.sender, msg.sender, assets, shares);

		IWAVAX(address(asset)).deposit{value: assets}(); //c ERC4626 implementation
		_mint(msg.sender, shares);
		afterDeposit(assets, shares);
		//bug no value was returned to this function
	}
```

So the user deposits raw avax as a value when calling this function. you will see that this function then takes the avax the user deposited and deposited it into an asset address implementing a WAVAX interface. The asset that the ggavax token is using is the WAVAX token. They couldnt use the raw avax because raw avax isnt an erc20 and this protocol is supposed to allow users to deposit their raw avax and get ggavax. so for proper erc4626 implementation, they take the users deposit and deposit it as WAVAX for a 1-1 swap as you know, wavax has the same value of avax but as an erc20 which solves that problem. this is why we deposit the user avax to change it to wavax to use as our erc4626 asset.

there is also the concept of the shares in erc4626 and the shares are simply the yield bearing token that we want to exchange for the asset. in this case, the shares are ggavax. the assets are swapped for shares with an exchange rate and this exchange rate is based on the ratio of shares to assets in the contract. the depositing starts at a 1-1 exchange rate so the first depositor gets a huge advantage over the rest and can potentially drain other users assets via something known as an inflation attack but we will look at this shortly.

more on the exchange rate, the ggavax token inherits from an erc4626 upgradeable contract from solmate whch contained the logic for calculating the exchange rate. see below:

```javascript
	function convertToShares(uint256 assets) public view virtual returns (uint256) {
		uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

		return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
	}

	function convertToAssets(uint256 shares) public view virtual returns (uint256) {
		uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

		return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
	}

	function previewDeposit(uint256 assets) public view virtual returns (uint256) {
		return convertToShares(assets);
	}

```

Exchange Rate in Vaults:

The relationship between the total shares and total assets is:
Exchange Rate = Total Shares/Total Assets to get how much 1 asset is worth in shares and TotalAssets/ Total Shares to get how much 1 share is in assets. This is how the exchange rate is calculated. This is why there is a converttoassets formula and converttoshares function as you see above. This is pretty intuitive when you think about it. So whenever a user wants to deposit assets to the the contract, we are converting the assets they are depositing to shares and giving them ggavax shares. all we do is multiply the amount of assets they want to deposit by the exchange rate (how much 1 share is in assets) and that gives us the amount of shares to give to the user. the previewdeposit function is one that is suggested in the eip (which i linked above and you shouldve read) and is used to allow users estimate how many shares they are going to get based on how many assets they are depositing at the current exchange rate. this function is also very important as we will come to see. A default you will notice is that the division always rounds the exchange rate down for deposits so if assets \* supply/assets is a decimal, it will round the value down so if the new exchange rate is 1.5, it will round down to 1. we know this already as solidity does this by default. This is going to be key in the inflation attack so keep this mind.

Now lets look at how an attacker can drain assets from this contract. This is known as the inflation attack.

https://docs.openzeppelin.com/contracts/4.x/erc4626

The attack is detailed in the above link but I will break it down. It starts by an attacker first depositing a very small amount of tokens like 1 asset token into the pool (call this a0). Since the exchange rate starts at 1:1, the user gets 1 share for their 1 token. The attacker then 'donates' assets to the pool so they send asset tokens to the contract without calling the deposit function so they just send asset tokens to the contract. Lets call the amount of tokens they donated a1. This majorly skews the exchange rate. The exchange rate after this donation is a0/a0+a1
which is what 1 asset token is worth in share tokens which is what openzeppelin link above is saying. Now, the attacker has all the current shares and has messed with the exchange rate, say userB wants to call deposit function to deposit asset tokens and they want to deposit u mount of assets. so the amount of shares they will get is amount of assets multiplied by how much one asset is worth in shares (exchange rate).

Remember the attacker has manipulated the exchange rate to a0/a0+a1 so the amount of shares the user will get is u multiplied by a0/a0+a1. So if the attacker wants take all of the users deposited assets, all they have to make sure is that a0 and a1 are significant enough to make u multiplied by a0/a0+a1 < 1. According to the openzeppelin link above, using a0 =1 and a1 = u would be enough to do this. you can see this by doing some simple math to refactor u multiplied by a0/a0+a1 < 1 to u < 1+ a1/a0 and a0 = 1 and a1 = u will satisfy this condition. The reason for this is simple. if the amount of shares the user is going to get is a decimal less than 1, then solidity will round this value down based on our converttoshares function which i talked about earlier. this means that the shares value will be rounded down to 0 and 0 shares will be sent to the user but their deposit of u tokens is now in the contract and the attacker has a0 shares in the contract and now, a0 is worth + u as the user has deposited assets but got no shares back. So the attacker can change their shares for asset tokens and will end up with their earlier deposit + u. So the attacker waits for the users transaction in the mempool, sees the value of u and implements this strategy to drain their assets. This is known as the inflation attack. More is spoken about this in the link above so make sure to read that.

There are many ways to prevent this attack and tokenggavax implements all the known ones. If you look at the depositAVAX function i added from the ggavax contract above, you will see that it has a conditional to check if shares are 0. This is why the check is there. so if a user wants to deposit avax for ggavax and the ggavax they get back is 0 due to someone trying the inflation attack, the function will revert which mitigates that.

Furthermore, this is the receive function in that contract:

```javascript
/// @notice only accept AVAX via fallback from the WAVAX contract
	receive() external payable {
		require(msg.sender == address(asset)); //q need to look at if possible to send avax from wavax contract to this contract  but why would any attacker want to do that as this receive function doesnt do anything
		//c the reason for this was to avoid an inflation attack which is common with erc4626 tokens. see notes.md
	}
```

So the contract doesnt accept random avax deposits unless it comes the wavax contract which is the asset contract so no random person can send avax to this contract. This is more of a safeguard though and doesnt actually stop an inflation attack because as we have learnt, the inflation attack is meant to skew the asset in the contract and the asset here is wavax and the only way to do that would be to skew the totalassets part of the exchange rate calculation. As we know, 1 asset is worth shares/total assets so this is the exchange rate as we sa earlier. If total assets was calculated in the contract using IWAVAX(address(asset)).balanceOf(address(this)) ,then it would be easy to skew the total assets calculation by 'donating' WAVAX to the contract as I explained earlier. This isnt the case though as totalAssets is calculated using a totalassets function and this function uses variables that are only incremented whenever a deposit or withdrawal of avax is made. so at no point is IWAVAX(address(asset)).balanceOf(address(this)) used. If it was, then it would be easy to skew the exchange rate via totalassets and perform an inflation attack.

Regardless of these safeguards, the first depositor is going to get a way better exchange rate which gives them a massive advantage over other depositors so the ggavax contract tries to even further mitigate this by allowing the protocol to deposit first into the protocol and get the first shares to further prevent attackers front running and gaining user funds because even though there is a zero check on shares, the previewdeposit function still rounds down and the attacker may not be able to take all the user assets by setting their share amount to 0 but they can take a part of their assets via that amount that solidity rounds down. The openzeppelin link talking about the inflation attack which I spoke about earlier and you shouldve read covers this using the u/n example. To mitigate against this, the initialize function in the ggavax token had the following:

```javascript
function initialize(Storage storageAddress, ERC20 asset, uint256 initialDeposit) public initializer {
		__ERC4626Upgradeable_init(asset, "GoGoPool Liquid Staking Token", "ggAVAX");
		__BaseUpgradeable_init(storageAddress);

		version = 1;

		// sacrifice initial seed of shares to prevent front-running early deposits
		if (initialDeposit > 0) {
			deposit(initialDeposit, address(this));
		}

		rewardsCycleLength = 14 days;
		// Ensure it will be evenly divisible by `rewardsCycleLength`.
		rewardsCycleEnd = (block.timestamp.safeCastTo32() / rewardsCycleLength) * rewardsCycleLength; //c to make a numberA divisible by another numberB, all you have to do is divide numberA by number B and multiply it by numberB
	}

```

look at the if statement in the above function to see what I am talking about. They definitely went through everything to prevent inflation attacks lol.

the contract contains a withdrawforstaking function which sends avax to the minipool manager contract and this avax is matched with the minipool owners avax amount to help them become a validator. this creates an invariant that we can test as there is also a depositfromstaking function that is called once the staked avax is returned with the rewards from the rialto multisig staking the coins on the p-chain. The invariant is to make sure that the amount of avax returned to the tokenggavax contract is always more than the amount withdrawn. This is a very useful invariant and an example of how you can easily define new invariants when reviewing code. this is something you should always have in mind.

# RESEARCHING MATH HEAVY PROTOCOLS - IF YOU CAN MAKE THE FORMULA PRODUCE AN UNEXPECTED RESULT, YOU CAN EXPLOIT IT, FRONTRUNNING MENTALITY

Based on everything we have just learnt about the inflation attack, it is worth noting how this bug was found and the thought process you should have if you want to find similar bugs or not miss as much going forward. The inflation attack focused on the exchange rate formula and tried to find a way via the contract to skew the exchange rate which they did by 'donating' assets rather than depositing. To do that, you had to understand how the formula worked. There are a lot of bugs that can be found by diving deep into formulas and seeing how you can make it behave differently via the contract. So whenever you see a formula, you must understand everything the formula does and then think about what actions you can make via the contract to skew the formula and the effect that your actions have on the formula result. More times than not, you can find a way to exploit it if you can make the formula produce an unexpected result. This is one of the most important rule of thumbs in auditing. You have to try to exploit everything and this includes the formulae.

Edge cases like first actor logic (what happens if I am the first person to call the function that activates this formula), last depositor logic (what happens if I am the last person to call the function that activates this formula), etc will help you find loopholes in formulas because a lot of formulae have edge cases you can explore and exploit.

There is also the idea of front running which you need to engrain into every function you look at as this will help you think differently when you see any function. Just keep in mind that someone can see the transaction details before the transaction gets processed. You need to think about what can be done by the person who can see the tx details and if there is anything they can do to attack the transaction based on the logic you are looking at. Thinking like this will give you superpowers.
