# 1 Misleading ltv value will lead to portfolio owners taking actions on incorrect data

Summary
VenusAssetHandler's getUserAccountData incorrectly calculates ltv using a formula intended for the liquidation threshold. Consequently, ltv always equals the current liquidation threshold, misleading users about their true borrowing risk.

Finding Description
VenusAssetHandler::getUserAccountData returns the user account data across all reserves in the Venus protocol. See function below:

```solidity
/**
   * @notice Returns the user account data across all reserves in the Venus protocol.
   * @param user The address of the user.
   * @param comptroller The address of the Venus Comptroller.
   * @return accountData A struct containing the user's account data.
   * @return tokenAddresses A struct containing the balances of the user's lending and borrowing tokens.
   */
  function getUserAccountData(
    address user,
    address comptroller,
    address[] memory
  )
    public
    view
    returns (
      FunctionParameters.AccountData memory accountData,
      FunctionParameters.TokenAddresses memory tokenAddresses
    )
  {
    (
      accountData.totalCollateral,
      accountData.totalDebt,
      accountData.availableBorrows,
      accountData.ltv,
      tokenAddresses.lendTokens,
      tokenAddresses.borrowTokens
    ) = getAccountPosition(comptroller, user);

    accountData.totalCollateral = accountData.totalCollateral / 1e10; // change the scale from 18 to 8
    accountData.totalDebt = accountData.totalDebt / 1e10; // change the scale from 18 to 8
    accountData.availableBorrows = accountData.availableBorrows / 1e10; // change the scale from 18 to 8
    accountData.currentLiquidationThreshold = accountData.ltv; // The average liquidation threshold is same with average collateral factor in Venus
    accountData.healthFactor = accountData.totalDebt == 0
      ? type(uint).max // Set health factor to max if no debt
      : (accountData.totalCollateral * accountData.ltv) / accountData.totalDebt;
  }
```

2 key variables returned are accountData.currentLiquidationThreshold and accountData.ltv which both tell the portfolio owner their current liquidation threshold and their current ltv (loan to value) rate. These variables are used to guide decisions on how to manage the portfolio with key actions like borrowing/lending. The natspec in FunctionParameters::AccountData specifies that the ltv is a variable rate and calculates the ltv for the account.

```solidity
/**
     * @dev Struct containing account-related data such as collateral, debt, and health factors.
     *
     * @param totalCollateral The total collateral value of the account.
     * @param totalDebt The total debt value of the account.
     * @param availableBorrows The total amount available for borrowing.
     * @param currentLiquidationThreshold The current liquidation threshold value of the account.
     * @param ltv The loan-to-value ratio of the account. <== //c see description
     * @param healthFactor The health factor of the account, used to determine its risk of liquidation.
     */
    struct AccountData {
        uint totalCollateral;
        uint totalDebt;
        uint availableBorrows;
        uint currentLiquidationThreshold;
        uint ltv;
        uint healthFactor;
    }
```

The issue lies in the way accountData.ltv is calculated.

```solidity
/**
   * @notice Internal function to calculate the user's account position in the Venus protocol.
   * @param comptroller The address of the Venus Comptroller.
   * @param account The address of the user account.
   * @return totalCollateral The total collateral value of the user.
   * @return totalDebt The total debt of the user.
   * @return availableBorrows The available borrowing power of the user.
   * @return ltv The loan-to-value ratio of the user's collateral.
   * @return lendTokens An array of addresses representing lent assets.
   * @return borrowTokens An array of addresses representing borrowed assets.
   */
  function getAccountPosition(
    address comptroller,
    address account
  )
    internal
    view
    returns (
      uint totalCollateral,
      uint totalDebt,
      uint availableBorrows,
      uint ltv,
      address[] memory lendTokens,
      address[] memory borrowTokens
    )
  {
    AccountLiquidityLocalVars memory vars; // Initialize the struct to hold calculation results

    address[] memory assets = IVenusComptroller(comptroller).getAssetsIn(
      account
    ); // Get all assets in the account
    uint assetsCount = assets.length; // Number of assets

    // Initialize structs to store token information
    TokenInfo memory lendInfo;
    lendInfo.tokens = new address[](assetsCount); // Initialize the array for lent assets

    TokenInfo memory borrowInfo;
    borrowInfo.tokens = new address[](assetsCount); // Initialize the array for borrowed assets

    // Process each asset to update the account position
    (vars, lendInfo, borrowInfo) = processAssets(
      comptroller,
      account,
      assets,
      assetsCount,
      vars,
      lendInfo,
      borrowInfo
    );

    // Handle VAIController logic separately
    vars.sumBorrowPlusEffects = handleVAIController(
      comptroller,
      account,
      vars.sumBorrowPlusEffects
    );

    // Resize the arrays to remove empty entries
    resizeArray(lendInfo.tokens, assetsCount - lendInfo.count);
    resizeArray(borrowInfo.tokens, assetsCount - borrowInfo.count);

    // Calculate and return the final account position
    return (
      vars.totalCollateral,
      vars.sumBorrowPlusEffects,
      vars.sumCollateral > vars.sumBorrowPlusEffects
        ? vars.sumCollateral - vars.sumBorrowPlusEffects
        : 0,
      vars.totalCollateral > 0
        ? divRound_(vars.sumCollateral * 1e4, vars.totalCollateral)
        : 0,
      lendInfo.tokens,
      borrowInfo.tokens
    );
  }
```

The current formula incorrectly calculates LTV by factoring in sumCollateral and applying a scaling factor, which is actually used for liquidation threshold calculations. LTV should be (loan amount / collateral value) * 100, representing the percentage of collateral that has been borrowed. The existing formula instead compares adjusted collateral (factoring in collateralFactorMantissa) with total collateral, which is used to determine the liquidation threshold rather than true LTV. As a result, a user retrieving their data, will always see their ltv value equal to the current liquidity threshold prompting them to rebalance the portfolio to avoid liquidation falsely.

Impact Explanation
Users rely on accountData.ltv to gauge borrowing risk. A faulty ltv leads to premature or unnecessary rebalancing actions. While it doesn't immediately create a direct exploit, it hampers user experience and may prompt suboptimal decisions, impacting user funds indirectly.

Likelihood Explanation
Likely – Any user querying their Venus portfolio via this function will encounter the incorrect ltv. Since the function is a standard way to retrieve account health, most integrations or dashboards using it will display misleading data.

Proof of Concept
The following test was run in test/Bsc/9_Borrowing.test.ts

```solidity
it("should borrow USDT using vBTC as collateral", async () => {
    //c for testing purposes
    let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
    let vault = await portfolio.vault();
    console.log(
      "DAI Balance before",
      await ERC20.attach(addresses.USDT).balanceOf(vault)
    );

    await rebalancing.borrow(
      addresses.vUSDT_Address,
      [addresses.vBTC_Address],
      addresses.USDT,
      addresses.corePool_controller,
      "10000000000000000000"
    );
    console.log(
      "DAI Balance after",
      await ERC20.attach(addresses.USDT).balanceOf(vault)
    );

    const comptroller: IVenusComptroller = await ethers.getContractAt(
      "IVenusComptroller",
      addresses.corePool_controller
    ); //c for testing purposes

    const [accountData, tokenAddresses] =
      await venusAssetHandler.getUserAccountData(
        vault,
        comptroller.address
      );
    console.log("AccountData", accountData.totalCollateral);

    //c get vBTC collateralFactor
    const markets = await comptroller.markets(addresses.vBTC_Address);
    const collateralFactor = markets.collateralFactorMantissa;
    console.log("CollateralFactor", collateralFactor);

    let sumCollateral = BigNumber.from(accountData.totalCollateral)
      .mul(BigNumber.from(collateralFactor)) // Use BigNumber multiplication
      .div(ethers.constants.WeiPerEther); // Scale down if collateralFactor is 18-decimal

    const correctLTV = BigNumber.from(accountData.totalDebt)
      .mul(ethers.constants.WeiPerEther) // Scale up numerator
      .div(sumCollateral); // Perform division

    const scaledCorrectLTV = correctLTV
      .div(ethers.constants.WeiPerEther)
      .mul(1e4);

    console.log("Correct LTV:", correctLTV.toString());

    console.log("Account LTV:", accountData.ltv.toString());
    console.log(
      "Current Liquidation Threshold:",
      accountData.currentLiquidationThreshold.toString()
    );
    console.log("Correct LTV:", correctLTV.toString());

    assert(scaledCorrectLTV.lt(accountData.ltv));
    assert(accountData.currentLiquidationThreshold.eq(accountData.ltv));
    assert(scaledCorrectLTV.lt(accountData.currentLiquidationThreshold));

    console.log("newtokens", await portfolio.getTokens());
});
```

Recommendation
Correct the ltv calculation to reflect the actual ratio of loan amount / collateral value. Specifically, replace the current formula with:

```solidity
ltv = (vars.sumBorrowPlusEffects * 1e4) / vars.sumCollateral;
```

(assuming consistent decimal scaling). This ensures ltv accurately reflects how much of the user's collateral is borrowed, preventing false alarms about liquidation.

# 2 The current exchange rate is not used to calculate the total collateral of assets held by portfolio

Finding Description
Users deposit tokens into a portfolio using VaultManager::multiTokenDeposit which calls VaultManager::_validateAndGetBalances. This calls TokenBalanceLibrary::getTokenBalancesOf which in turn calls TokenBalanceLibrary::_getAdjustedTokenBalance to get the token balances of all tokens in the vault

```solidity
/**
   * @notice Fetches the balance of a specific token held in a given vault.
   * @dev Retrieves the token balance using the ERC20 `balanceOf` function.
   * Throws if the token or vault address is zero to prevent erroneous queries.
   *
   * @param _token The address of the token whose balance is to be retrieved.
   * @param _vault The address of the vault where the token is held.
   * @return tokenBalance The current token balance within the vault.
   */
  function _getAdjustedTokenBalance(
    address _token,
    address _vault,
    IProtocolConfig _protocolConfig,
    ControllerData[] memory controllersData
  ) public view returns (uint256 tokenBalance) {
    if (_token == address(0) || _vault == address(0))
      revert ErrorLibrary.InvalidAddress(); // Ensures neither the token nor the vault address is zero.
    if (_protocolConfig.isBorrowableToken(_token)) {
      address controller = _protocolConfig.marketControllers(_token);
      ControllerData memory controllerData = findControllerData(
        controllersData,
        controller
      );

      uint256 rawBalance = _getTokenBalanceOf(_token, _vault);
      tokenBalance =
        (rawBalance * controllerData.unusedCollateralPercentage) /
        1e18;
    } else {
      tokenBalance = _getTokenBalanceOf(_token, _vault);
    }
  }
```

This function uses controllerData.unusedCollateralPercentage to adjust vToken balances to allow for any debt the portfolio has accumulated to that point. In calculation of the unusedCollateralPercentage, the totalCollateral and totalDebt are calculated. The totalCollateral is calculated as follows. First the accountData of the vault is retrieved from the comptroller via getAccountSnapshot:

```solidity
/**
   * @notice Internal function to update liquidity variables with asset snapshot data.
   * @param asset The Venus pool asset to process.
   * @param account The address of the user account.
   * @param vars The struct holding liquidity calculation variables.
   * @return shouldContinue Boolean indicating whether to continue the loop.
   */
  function updateVarsWithSnapshot(
    IVenusPool asset,
    address account,
    AccountLiquidityLocalVars memory vars
  ) internal view returns (bool shouldContinue) {
    (
      uint oErr,
      uint vTokenBalance,
      uint borrowBalance,
      uint exchangeRateMantissa
    ) = asset.getAccountSnapshot(account); // Get the snapshot of the account in the asset

    if (oErr != 0) {
      return true; // Indicate that the loop should continue and skip this asset if there was an error
    }

    // Update the variables with the snapshot data
    vars.vTokenBalance = vTokenBalance;
    vars.borrowBalance = borrowBalance;
    vars.exchangeRateMantissa = exchangeRateMantissa;

    return false; // No error, proceed with processing this asset
  }
```

These values are then used to calculate the totalCollateral. The issue lies above as exchangeRateMantissa is not the most up to date exchange rate from venus.

```solidity
/**
     * @notice Get a snapshot of the account's balances, and the cached exchange rate
     * @dev This is used by comptroller to more efficiently perform liquidity checks.
     * @param account Address of the account to snapshot
     * @return error Always NO_ERROR for compatibility with Venus core tooling
     * @return vTokenBalance User's balance of vTokens
     * @return borrowBalance Amount owed in terms of underlying
     * @return exchangeRate Stored exchange rate
     */
    function getAccountSnapshot(
        address account
    )
        external
        view
        override
        returns (uint256 error, uint256 vTokenBalance, uint256 borrowBalance, uint256 exchangeRate)
    {
        return (NO_ERROR, accountTokens[account], _borrowBalanceStored(account), _exchangeRateStored());
    }
```

_exchangeRateStored() is used to calculate the exchange rate in this function but this exchange rate does not take the interest accrued from the last borrow into account. The following function from the venus code show this below:

```solidity
/**
     * @notice Calculates the exchange rate from the underlying to the VToken
     * @dev This function does not accrue interest before calculating the exchange rate
     * @return exchangeRate Calculated exchange rate scaled by 1e18
     */
    function exchangeRateStored() external view override returns (uint256) {
        return _exchangeRateStored();
    }

    /**
     * @notice Accrue interest then return the up-to-date exchange rate
     * @return exchangeRate Calculated exchange rate scaled by 1e18
     */
    function exchangeRateCurrent() public override nonReentrant returns (uint256) {
        accrueInterest();
        return _exchangeRateStored();
    }
```

As a result, the exchange rate used by Velvet to calculate the totalCollateral is not the most up to date rate accounting for latest user actions. This means that the totalCollateral value calculated will be skewed which in turn affects controllerData.unusedCollateralPercentage which is directly used to calculate balances that are used to calculate appropriate ratios for user deposits. If a large enough user action on venus accrues enough interest in a single transaction, this interest will not be reflected in the totalCollateral and controllerData.unusedCollateralPercentage. This will affect the amount of tokens a user can deposit into a portfolio as a minimum ratio is calculated which determines how much a user can transfer into a portfolio. If the exchange rate used does not account for up to date venus market user actions, it could over/underestimate this ratio which will directly affect how much vToken capital a portfolio can take in at certain contract states.

Impact Explanation
The inaccurate exchange rate can impact how much collateral is recognized for each user deposit. While it doesn't directly allow malicious exploits, it causes misleading calculations, potentially inflating or deflating amounts users can deposit in the portfolio. This mismatch can hinder users' experience, or produce unexpected behaviors in portfolio management.

Likelihood Explanation
High – Every time a user mints or borrows from a Venus market without a fresh exchangeRateCurrent() call, the stored exchange rate remains outdated. Since these actions occur frequently, it's almost certain the contract will, at some point, rely on a stale exchange rate for calculations, leading to consistent skew in the recognized collateral value.

Proof of Concept
```solidity
it("lack of exchange rate update can reduce ratio on user deposits ", async () => {
    //c for testing purposes

    let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
    let vault = await portfolio.vault();
    console.log(
      "DAI Balance before",
      await ERC20.attach(addresses.USDT).balanceOf(vault)
    );

    await rebalancing.borrow(
      addresses.vUSDT_Address,
      [addresses.vBTC_Address],
      addresses.USDT,
      addresses.corePool_controller,
      "10000000000000000000"
    );
    console.log(
      "DAI Balance after",
      await ERC20.attach(addresses.USDT).balanceOf(vault)
    );

    const comptroller: IVenusComptroller = await ethers.getContractAt(
      "IVenusComptroller",
      addresses.corePool_controller
    ); //c for testing purposes

    //c get a user to borrow a lot of btc from market
    let wBTCHolder = "0x659DDDcC02115946e142E856c1F7Cd7E39d03D3D";

    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [wBTCHolder],
    });
    const wBTCSigner = await ethers.getSigner(wBTCHolder);

    const vBTC: IVenusPool = await ethers.getContractAt(
      "IVenusPool",
      addresses.vBTC_Address
    ); //c for testing purposes

    vBTC.connect(wBTCSigner).mint("200000");
    vBTC.connect(wBTCSigner).borrow("100000"); //c for testing purposes

    //c get the exchange rate stored
    const exchangeRateStored = await vBTC.exchangeRateStored();
    console.log("ExchangeRateStored", exchangeRateStored);

    //c get the current exchange rate (callstatic helps return values from non view functions)
    const exchangeRate = await vBTC.callStatic.exchangeRateCurrent();
    console.log("ExchangeRateCurrent", exchangeRate);

    assert(exchangeRateStored.lt(exchangeRate));
});
```

The test reflects this in a minimal way showing the exchange rate difference but the impact can be scaled higher depending on how much the exchange rate is moved by a single user mint/borrow in the relevant venus market.

Recommendation
Use exchangeRateCurrent() instead of exchangeRateStored() to ensure the protocol calculates collateral with the most up-to-date interest accrual.

# 3 _minPortfolioTokenHoldingAmount not upheld via lack of entry fee considerations and transfer mechanics

Finding Description
When a portfolio is created, the owner sets a _minPortfolioTokenHoldingAmount which indicates the minimum amount of portfolio tokens that can be held and can be minted according to the natspec regarding the variable. This value is enforced when a user deposits and withdraws tokens from the portfolio to maintain a base amount of portfolio tokens that a user can hold. For the purposes of this explanation, I will focus on the deposit logic. VaultManager::_depositAndMint calls VaultCalculations::_getTokenAmountToMint which is where _minPortfolioTokenHoldingAmount is enforced.

```solidity
/**
   * @notice Calculates the amount of portfolio tokens to mint based on the deposit ratio and the total supply of portfolio tokens.
   * @param _depositRatio The ratio of the user's deposit to the total value of the vault.
   * @param _totalSupply The current total supply of portfolio tokens.
   * @return The amount of portfolio tokens to mint for the given deposit.
   */
  function _getTokenAmountToMint(
    uint256 _depositRatio,
    uint256 _totalSupply,
    IAssetManagementConfig _assetManagementConfig
  ) internal view returns (uint256) {
    uint256 mintAmount = _calculateMintAmount(_depositRatio, _totalSupply);
    if (mintAmount < _assetManagementConfig.minPortfolioTokenHoldingAmount()) {
      revert ErrorLibrary.MintedAmountIsNotAccepted();
    }
    return mintAmount;
  }
```

The issue lies in the flow following this check which happens in VaultManager::_mintTokenAndSetCooldown.

```solidity
/**
     * @notice Mints new portfolio tokens, considering the entry fee, if applicable, and assigns them to the specified address.
     * @param _to Address to which the minted tokens will be assigned.
     * @param _mintAmount Amount of portfolio tokens to mint.
     * @return The amount of tokens minted after deducting any entry fee.
     */
    function _mintTokenAndSetCooldown(
        address _to,
        uint256 _mintAmount,
        IAssetManagementConfig _assetManagementConfig
    ) internal returns (uint256) {
        uint256 entryFee = _assetManagementConfig.entryFee();

        if (_mintAndBurnCheck(entryFee, _to, _assetManagementConfig)) {
            _mintAmount = feeModule()._chargeEntryOrExitFee(
                _mintAmount,
                entryFee
            );
        }

        _mint(_to, _mintAmount);
        

        // Updates the cooldown period based on the minting action.
        userCooldownPeriod[_to] = _calculateCooldownPeriod(
            balanceOf(_to),
            _mintAmount,
            protocolConfig().cooldownPeriod(),
            userCooldownPeriod[_to],
            userLastDepositTime[_to]
        );
        userLastDepositTime[_to] = block.timestamp; 

        return _mintAmount;
    }
```

In this function, the entry fee is subtracted from the user's mintAmount and then the user is minted the remaining tokens. As a result, this can lead to the user ending up with less tokens than _minPortfolioTokenHoldingAmount which is a direct violation of what the variable is meant to enforce.

Another issue arises where a user is freely allowed to transfer portfolio tokens. Velvet set a global _minPortfolioTokenHoldingAmount amount that governs the minimum amount a user must hold. Portfolio creators cannot set their internal _minPortfolioTokenHoldingAmount lower than this amount which is enforced in PortfolioSettings::__PortfolioSettings_init.

```solidity
function __PortfolioSettings_init(
        address _protocolConfig,
        uint256 _initialPortfolioAmount,
        uint256 _minPortfolioTokenHoldingAmount,
        bool _publicPortfolio,
        bool _transferable,
        bool _transferableToPublic
    ) internal onlyInitializing {
        if (_protocolConfig == address(0)) revert ErrorLibrary.InvalidAddress();
        protocolConfig = IProtocolConfig(_protocolConfig);

        if (
            _initialPortfolioAmount < protocolConfig.minInitialPortfolioAmount()
        ) {
            revert ErrorLibrary.InvalidMinPortfolioAmountByAssetManager();
        }

        if (
            _minPortfolioTokenHoldingAmount <
            protocolConfig.minPortfolioTokenHoldingAmount()
        ) {
            revert ErrorLibrary.InvalidMinAmountByAssetManager();
        }

        initialPortfolioAmount = _initialPortfolioAmount;
        minPortfolioTokenHoldingAmount = _minPortfolioTokenHoldingAmount;
        publicPortfolio = _publicPortfolio; 
        _setTransferability(_transferable, _transferableToPublic);
    }
```

Due to the ability of user's to freely transfer tokens between accounts after a cooldown period, a user can violate both protocolConfig.minInitialPortfolioAmount() and the portfolio's minInitialPortfolioAmount() check which breaks key protocol functionality.

Impact Explanation
It undermines the intended protections of the portfolio token system. Users can inadvertently or deliberately violate minimum holding constraints, eroding key protocol functionality. Given the protective nature of these limits, their circumvention negatively affects asset management integrity and portfolio economics.

Likelihood Explanation
High – Every deposit with a non-zero entry fee currently risks minting below the defined minimum holding amount. Additionally, the freedom of token transfers without revalidating minimum holding requirements increases the probability users will unintentionally or intentionally violate the protocol's rules, particularly in portfolios with smaller balances or frequent transactions.

Proof of Concept
This test was run in test/Bsc/9_Borrowing.test.ts

```solidity
it("should swap tokens for user using native token 1", async () => {
        //c for testing purposes
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            //c this function is defined in IntentCalculations.ts so have a look there to see what it does
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "50000000000000000" //c changed from 100000000000000000 for testing purposes
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch
          .connect(nonOwner)
          .multiTokenSwapETHAndTransfer(
            {
              _minMintAmount: 0,
              _depositAmount: "1000000000000000000", //c changed from 1000000000000000000 for testing purposes. From looking at depositbatch.sol, it doesnt look like this variable actually does anything
              _target: portfolio.address,
              _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
              _callData: postResponse,
            },
            {
              value: "500000000000000000", //c changed from 1000000000000000000 for testing purposes
            }
          );

        //c get user portfolioToken after depositing
        const depositorBal = await portfolio.balanceOf(nonOwner.address);
        console.log("depositorBal", depositorBal);

        //c get entry fee
        const entryFee = await feeModule0.txEntryFee();
        console.log("entryFee", entryFee);

        const protocolPortfolioTokenHoldingAmount =
          await protocolConfig.minPortfolioTokenHoldingAmount();
        console.log(
          "protocolPortfolioTokenHoldingAmount",
          protocolPortfolioTokenHoldingAmount
        );

        //c prove that user is now holding less than _minPortfolioTokenHoldingAmount
        const assetManagementConfig =
          await portfolioFactory.getassetManagementConfig(0);
        let AssetManagementConfig = await ethers.getContractFactory(
          "AssetManagementConfig"
        );
        const assetManagementConfigImplement =
          await AssetManagementConfig.attach(assetManagementConfig);
        const _minPortfolioTokenHoldingAmount =
          await assetManagementConfigImplement.minPortfolioTokenHoldingAmount();
        console.log(
          "_minPortfolioTokenHoldingAmount",
          _minPortfolioTokenHoldingAmount
        );
        assert(depositorBal.lt(_minPortfolioTokenHoldingAmount));

        await ethers.provider.send("evm_increaseTime", [86500]);

        //c user decides to transfer tokens to another user
        await portfolio
          .connect(nonOwner)
          .transfer(owner.address, "49745725570581380000");

        //c get user portfolioToken after transfer for a different test
        const depositorBalAfterTransfer = await portfolio.balanceOf(
          nonOwner.address
        );
        console.log("depositorBalAfterTransfer", depositorBalAfterTransfer);

        assert(
          depositorBalAfterTransfer.lt(protocolPortfolioTokenHoldingAmount)
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });
```

To run the test successfully, the portfolio init data was updated as follows:

```solidity
const portfolioFactoryCreate =
        await portfolioFactory.createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "20",
          _performanceFee: "2500",
          _entryFee: "50", //c changed from 0 for testing purposes
          _exitFee: "50", //c changes from 0 for testing purposes
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "49900000000000000000", //c changed from 1000000000000000000 for testing purposes
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: true,
          _transferableToPublic: true,
          _whitelistTokens: false,
          _externalPositionManagementWhitelisted: true,
        });
```

The following function was also added to PortfolioFactory.sol to retrieve the assetManagementConfig proxy of the relevant portfolio:

```solidity
function getassetManagementConfig(
        uint256 portfoliofundId
    ) external view virtual returns (address) {
        return
            address(PortfolioInfolList[portfoliofundId].assetManagementConfig);
    }
```

Recommendation
Adjust the _getTokenAmountToMint logic to factor in entry fees when enforcing the minimum holding amount. Implement an additional check within the token transfer function to ensure no user can hold fewer tokens than the minimum defined by both portfolio-specific and global protocol settings.


# 4 multiTokenSwapETHAndTransfer and multiTokenSwapAndDeposit will revert with overflow when depositToken is not native token

Summary
The issue involves a vulnerability in the DepositBatch::multiTokenSwapETHAndTransfer function where incorrect handling of calldata for data._depositToken leads to a potential denial-of-service (DOS) due to an overflow in deposit calculations.

Finding Description
DepositBatch::multiTokenSwapETHAndTransfer is a function that allows users to convert the native token of the chain (BNB in this example) into multiple portfolio tokens and simultaneously deposit these tokens into the portfolio.

```solidity
/**
   * @notice Performs a multi-token swap and deposit operation for the user.
   * @param data Struct containing parameters for the batch handler.
   */
  function multiTokenSwapETHAndTransfer(
    FunctionParameters.BatchHandler memory data
  ) external payable nonReentrant {
    if (msg.value == 0) {
      revert ErrorLibrary.InvalidBalance();
    }

    address user = msg.sender;

    _multiTokenSwapAndDeposit(data, user);

    (bool sent, ) = user.call{ value: address(this).balance }("");
    if (!sent) revert ErrorLibrary.TransferFailed();
  }

 function _multiTokenSwapAndDeposit(
    FunctionParameters.BatchHandler memory data,
    address user
  ) internal {
    address[] memory tokens = IPortfolio(data._target).getTokens();
    address _depositToken = data._depositToken;
    address target = data._target;
    uint256 tokenLength = tokens.length;
    uint256[] memory depositAmounts = new uint256[](tokenLength);

    if (data._callData.length != tokenLength)
      revert ErrorLibrary.InvalidLength();

    // Perform swaps and calculate deposit amounts for each token
    for (uint256 i; i < tokenLength; i++) {
      address _token = tokens[i];
      uint256 balance;
      if (_token == _depositToken) {
        //Sending encoded balance instead of swap calldata
        balance = abi.decode(data._callData[i], (uint256));
      } else {
        uint256 balanceBefore = _getTokenBalance(_token, address(this));
        (bool success, ) = SWAP_TARGET.delegatecall(data._callData[i]);
        if (!success) revert ErrorLibrary.DepositBatchCallFailed();
        uint256 balanceAfter = _getTokenBalance(_token, address(this));
        balance = balanceAfter - balanceBefore;
      }
      if (balance == 0) revert ErrorLibrary.InvalidBalanceDiff();

      TransferHelper.safeApprove(_token, target, 0);
      TransferHelper.safeApprove(_token, target, balance);

      depositAmounts[i] = balance;
    }

    IPortfolio(target).multiTokenDepositFor(
      user,
      depositAmounts,
      data._minMintAmount
    );

    //Return any leftover vault token dust to the user
    for (uint256 i; i < tokenLength; i++) {
      address _token = tokens[i];
      uint256 portfoliodustReturn = _getTokenBalance(_token, address(this));
      if (portfoliodustReturn > 0) {
        TransferHelper.safeTransfer(_token, user, portfoliodustReturn);
      }
    }
  }
```

The flow is as follows. The user calls multiTokenSwapETHAndTransfer with the FunctionParameters.BatchHandler struct called data . _multiTokenSwapAndDeposit is then called using these parameters. In the data struct, there is a calldata array that contains all the calldata related to each swap to be performed. In the case of multiTokenSwapAndDeposit , the user calls multiTokenSwapAndDeposit with the FunctionParameters.BatchHandler struct called data and the address they would like to deposit on behalf of. _multiTokenSwapAndDeposit is then called using these parameters. This calldata can either be generated by the user or for ease, IntentCalculations::createEnsoCallDataRoute can be called which generates the required calldata optimised for the best paths for the swap by enso. The calldata is designed to be delegate called on the enso swap handler which is hardcoded into DepositBatch.sol. All calldata generated by enso calls an executeShortcut(bytes32, bytes32[], bytes[]) function which has a seperate flow that is used to carry out the calldata.

The issue is highlighted in a scenario where a user enters data._depositToken as a token that is initialised by the portfolio. The way DepositBatch::_multiTokenSwapAndDeposit handles this scenario is as follows:

```solidity
if (_token == _depositToken) {
    //Sending encoded balance instead of swap calldata
    balance = abi.decode(data._callData[i], (uint256)); 
}
```

The intended idea is that if the user calls multiTokenSwapETHAndTransfer or multiTokenSwapAndDeposit with data._depositToken as any token that is one of the portfolio's allowed tokens, the calldata related to that token is to be decoded and set as the required depositAmount for that token. The implementation of this is wrong because the calldata gotten from the enso handler in a situation where multiTokenSwapETHAndTransfer is called and data._depositToken is any token that is one of the portfolio's allowed tokens will look like the following:

```solidity
0x8fd8d1bbef338d7b4cee3edf847339e8fc216ef03268b89c8a9d5884e3155ea01169c30c000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000e00000000000000000000000000000000000000000000000000000000000000003d0e30db00300ffffffffffffbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c6e7a43a3010001ffffffff017e7d64d987cab6eed08a191c4c2459daf2f8ed0b241c59120101ffffffffffff7e7d64d987cab6eed08a191c4c2459daf2f8ed0b000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000016345785d8a00000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000014a6701dc1c8000
```

This is sample calldata returned by the enso handler. This calldata always starts with the function selector (0x8fd8d1bb) and also contains all the parameters for all functions that will be called. Decoding all the calldata as a uint256 returns the following value :

```solidity
65063823845086575781701663574583048641295546983125765704444912785086492270240
```

This value is not the intended value that the user wanted to deposit into the portfolio. The incorrect value is then sent in the depositAmounts array to IPortfolio(target).multiTokenDepositFor which calls TokenCalculations::_getDepositToVaultBalanceRatio and due to the calculations done in this function, it will overflow which causes a DOS for the user trying to deposit into the portfolio using these functions.

Impact Explanation
The issue completely prevents users from successfully executing multiTokenSwapAndDeposit. The intended functionality—swapping user tokens into portfolio tokens—is rendered unusable, significantly limiting user interaction with portfolios.

Likelihood Explanation
This will always occur when a user calls multiTokenSwapAndDeposit with data._depositToken being the same as a token the portfolio accepts. However, it may not always occur in multiTokenSwapETHAndTransfer as it would ideally be a misjudgement from the user to call multiTokenSwapETHAndTransfer with data._depositToken as a non native token address but even in such a scenario, DepositBatch::_multiTokenSwapAndDeposit was designed to handle such a scenario by accurately decoding the calldata and getting the required amount but the implementation used is incorrect which leads to the DOS.

Proof of Concept
This test was run in test/Bsc/9_Borrowing.test.ts

```solidity
it("multiTokenSwapETHAndTransfer will revert with overflow when depositToken is not the native token due to abi.decode error", async () => {
        //c for testing purposes

        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
         
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: iaddress.ethAddress,
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );
      });
```

Recommendation
The issue can be fixed by correctly decoding the calldata by explicitly accessing only the exact 32 bytes in the calldata that represent the amount the user intended to deposit.

# # 5 Portfolio debt repayment functionality incorrectly handling multiple debt tokens

Summary
The Rebalancing::repay function incorrectly handles multiple debt tokens when a single flash loan token is used. This leads to insufficient flash loan funds for swap operations or an out-of-bounds array access error (Panic(0x32)). The bug occurs due to the assumption that a single flash loan amount can cover multiple debt repayments, while the flash loan only provides flashLoanAmount[0].

Finding Description
Portfolio owners have access to a rebalancing contract which allows for debt repayments via Rebalancing::repay. See the full call flow below:

```solidity
function repay(
  address _controller,
  FunctionParameters.RepayParams calldata repayData
) external onlyAssetManager nonReentrant protocolNotPaused {
  // Attempt to repay the debt through the borrowManager
  // Returns true if the token's debt is fully repaid, false if partial repayment
  bool isTokenFullyRepaid = borrowManager.repayVault(_controller, repayData);

  // If the token's debt is fully repaid, decrement the count of borrowed token types
  // This helps maintain an accurate count for the MAX_BORROW_TOKEN_LIMIT check
  if (isTokenFullyRepaid) tokensBorrowed--;
  emit TokenRepayed(repayData);
}
```
BorrowManager::repayVault:

```solidity
function repayVault(
  address _controller,
  FunctionParameters.RepayParams calldata repayData
) external onlyRebalancerContract returns (bool) {
  IAssetHandler assetHandler = IAssetHandler(
    _protocolConfig.assetHandlers(_controller)
  );

  beforeRepayVerification(
    repayData._factory,
    repayData._solverHandler,
    repayData._swapHandler,
    repayData._bufferUnit
  );

  // Get the number of borrowed tokens before the flash loan repayment
  uint256 borrowedLengthBefore = (
    assetHandler.getBorrowedTokens(_vault, _controller)
  ).length;

  // Prepare the data for the flash loan execution
  // This includes the function selector and parameters needed for the vault flash loan
  bytes memory data = abi.encodeWithSelector(
    IAssetHandler.executeVaultFlashLoan.selector,
    address(this),
    repayData
  );

  // Set flash loan flag to true to allow callback execution and prevent unauthorized callbacks
  _isFlashLoanActive = true;

  // Perform a delegatecall to the asset handler
  // delegatecall allows the asset handler's code to be executed in the context of this contract
  // maintaining this contract's storage while using the asset handler's logic
  (bool success, ) = address(assetHandler).delegatecall(data);

  // Check if the delegatecall was successful
  if (!success) revert ErrorLibrary.RepayVaultCallFailed();

  // Get the number of borrowed tokens after the flash loan repayment
  uint256 borrowedLengthAfter = (
    assetHandler.getBorrowedTokens(_vault, _controller)
  ).length;

  // Transfer any remaining dust balance back to the vault
  TransferHelper.safeTransfer(
    repayData._flashLoanToken,
    _vault,
    IERC20Upgradeable(repayData._flashLoanToken).balanceOf(address(this))
  );

  // Return true if we successfully reduced the number of borrowed tokens
  // This indicates that at least one borrow position was fully repaid
  return borrowedLengthAfter < borrowedLengthBefore;
}
```

VenusAssetHandler::executeVaultFlashLoan:

```solidity
function executeVaultFlashLoan(
  address _receiver,
  FunctionParameters.RepayParams calldata repayData
) external {
  // Getting pool address dynamically based on the token pair
  address _poolAddress = IThena(repayData._factory).poolByPair(
    repayData._token0,
    repayData._token1
  );

  // Defining the data to be passed in the flash loan, including the amount and pool key
  FunctionParameters.FlashLoanData memory flashData = FunctionParameters
    .FlashLoanData({
      flashLoanToken: repayData._flashLoanToken,
      debtToken: repayData._debtToken,
      protocolTokens: repayData._protocolToken,
      bufferUnit: repayData._bufferUnit,
      solverHandler: repayData._solverHandler,
      swapHandler: repayData._swapHandler,
      poolAddress: _poolAddress,
      flashLoanAmount: repayData._flashLoanAmount,
      debtRepayAmount: repayData._debtRepayAmount,
      poolFees: repayData._poolFees,
      firstSwapData: repayData.firstSwapData,
      secondSwapData: repayData.secondSwapData,
      isMaxRepayment: repayData.isMaxRepayment,
      isDexRepayment: repayData.isDexRepayment
    });

  // Initiate the flash loan from the Algebra pool
  IAlgebraPool(_poolAddress).flash(
    _receiver, // Recipient of the flash loan
    repayData._token0 == repayData._flashLoanToken
      ? repayData._flashLoanAmount[0]
      : 0, // Amount of token0 to flash loan
    repayData._token1 == repayData._flashLoanToken
      ? repayData._flashLoanAmount[0]
      : 0, // Amount of token1 to flash loan
    abi.encode(flashData) // Encode flash loan data to pass to the callback
  );
}
```

BorrowManager::algebraPoolFlashLoanCallback:

```solidity
function algebraFlashCallback(
  uint256 fee0,
  uint256 fee1,
  bytes calldata data
) external override {
  //Ensure flash loan is active to prevent unauthorized callbacks
  if (!_isFlashLoanActive) revert ErrorLibrary.FlashLoanIsInactive();

  FunctionParameters.FlashLoanData memory flashData = abi.decode(
    data,
    (FunctionParameters.FlashLoanData)
  ); // Decode the flash loan data

  IAssetHandler assetHandler = IAssetHandler(
    _protocolConfig.assetHandlers(flashData.protocolTokens[0])
  ); // Get the asset handler for the protocol tokens
  address controller = _protocolConfig.marketControllers(
    flashData.protocolTokens[0]
  ); // Get the market controller for the protocol token

  // Get user account data, including total collateral and debt
  (
    FunctionParameters.AccountData memory accountData,
    FunctionParameters.TokenAddresses memory tokenBalances
  ) = assetHandler.getUserAccountData(
      _vault,
      controller,
      _portfolio.getTokens()
    );

  // Process the loan to generate the transactions needed for repayment and swaps
  (
    IAssetHandler.MultiTransaction[] memory transactions,
    uint256 totalFlashAmount
  ) = assetHandler.loanProcessing(
      _vault,
      address(_portfolio),
      controller,
      address(this),
      tokenBalances.lendTokens,
      accountData.totalCollateral,
      IThena(msg.sender).globalState().fee,
      flashData
    );

  // Execute each transaction in the sequence
  uint256 transactionsLength = transactions.length;
  for (uint256 i; i < transactionsLength; i++) {
    (bool success, ) = transactions[i].to.call(transactions[i].txData); // Execute the transaction
    if (!success) revert ErrorLibrary.FlashLoanOperationFailed(); // Revert if the call fails
  }

  // Calculate the fee based on the token0 and fee0/fee1,using the Algebra Pool
  uint256 fee = IAlgebraPool(msg.sender).token0() == flashData.flashLoanToken
    ? fee0
    : fee1;

  // Calculate the amount owed including the fee
  uint256 amountOwed = totalFlashAmount + fee;

  // Transfer the amount owed back to the flashLoan provider
  TransferHelper.safeTransfer(
    flashData.flashLoanToken,
    msg.sender,
    amountOwed
  );
  
  //Reset the flash loan state to prevent subsequent unauthorized callbacks
  _isFlashLoanActive = false;
}
```

VenusAssetHandler::loanProcessing:

```solidity
function loanProcessing(
  address vault,
  address executor,
  address controller,
  address receiver,
  address[] memory lendTokens,
  uint256 totalCollateral,
  uint fee,
  FunctionParameters.FlashLoanData memory flashData
)
  external
  view
  returns (MultiTransaction[] memory transactions, uint256 totalFlashAmount)
{
  if (flashData.isDexRepayment) {
    (transactions, totalFlashAmount) = loanProcessingDex(
      vault,
      executor,
      controller,
      receiver,
      lendTokens,
      totalCollateral,
      fee,
      flashData
    );
  } else {
    // Process swaps and transfers during the loan
    (
      MultiTransaction[] memory swapTransactions,
      uint256 flashLoanAmount
    ) = swapAndTransferTransactions(vault, flashData);

    // Handle repayment transactions
    MultiTransaction[] memory repayLoanTransaction = repayTransactions(
      executor,
      vault,
      flashData
    );

    // Handle withdrawal transactions
    MultiTransaction[] memory withdrawTransaction = withdrawTransactions(
      executor,
      vault,
      controller,
      receiver,
      lendTokens,
      totalCollateral,
      fee,
      flashData
    );

    // Combine all transactions into one array
    transactions = new MultiTransaction[](
      swapTransactions.length +
        repayLoanTransaction.length +
        withdrawTransaction.length
    );
    uint256 count;

    // Add swap transactions to the final array
    uint256 swapTransactionsLength = swapTransactions.length;
    for (uint i = 0; i < swapTransactionsLength; ) {
      transactions[count].to = swapTransactions[i].to;
      transactions[count].txData = swapTransactions[i].txData;
      count++;
      unchecked {
        ++i;
      }
    }

    // Add repay transactions to the final array
    uint256 repayLoanTransactionLength = repayLoanTransaction.length;
    for (uint i = 0; i < repayLoanTransactionLength; ) {
      transactions[count].to = repayLoanTransaction[i].to;
      transactions[count].txData = repayLoanTransaction[i].txData;
      count++;
      unchecked {
        ++i;
      }
    }

    // Add withdrawal transactions to the final array
    uint256 withdrawTransactionLength = withdrawTransaction.length;
    for (uint i = 0; i < withdrawTransactionLength; ) {
      transactions[count].to = withdrawTransaction[i].to;
      transactions[count].txData = withdrawTransaction[i].txData;
      count++;
      unchecked {
        ++i;
      }
    }

    return (transactions, flashLoanAmount); // Return the final array of transactions and total flash loan amount
  }
}
```
The issue lies in the way the flash loan amount array is used in FunctionParameters::RepayParams:

```solidity
/**
   * @dev Struct containing the parameters required for repaying a debt using a flash loan.
   *
   * @param _factory The address of the factory contract responsible for creating necessary contracts.
   * @param _token0 The address of the first token in the swap pair (e.g., USDT).
   * @param _token1 The address of the second token in the swap pair (e.g., USDC).
   * @param _flashLoanToken The address of the token to be borrowed in the flash loan.
   * @param _debtToken The addresses of the tokens representing the debt to be repaid.
   * @param _protocolToken The addresses of the protocol-specific tokens, such as lending tokens (e.g., vTokens for Venus protocol).
   * @param _solverHandler The address of the contract handling the execution of swaps and other logic.
   * @param _swapHandler The address of the contract handling encoded data 
   * @param _bufferUnit Buffer unit for collateral amount
   * @param _flashLoanAmount The amounts of the flash loan to be taken for each corresponding `_flashLoanToken`.
   * @param _debtRepayAmount The amounts of debt to be repaid for each corresponding `_debtToken`.
   * @param _poolFees The fees for v3 pools of dexes.
   * @param firstSwapData The encoded data for the first swap operation, used for repaying the debt.
   * @param secondSwapData The encoded data for the second swap operation, used for further adjustments after repaying the debt.
   * @param isMaxRepayment Boolean flag to determine if the maximum borrowed amount should be repaid.
   * @param isDexRepayment Boolean flag to deteremine whether to repay using solver or dex.
   */
  struct RepayParams {
    address _factory;
    address _token0; //USDT   --> need to change token0 and token1 to single address
    address _token1; //USDC
    address _flashLoanToken;
    address[] _debtToken;
    address[] _protocolToken; // lending token in case of venus
    address _solverHandler;
    address _swapHandler;
    uint256 _bufferUnit;
    uint256[] _flashLoanAmount;
    uint256[] _debtRepayAmount;
    uint256[] _poolFees;
    bytes[] firstSwapData;
    bytes[] secondSwapData;
    bool isMaxRepayment;
    bool isDexRepayment;
  }
FunctionParameters::_flashLoanAmount is described in the above natspec as the amounts of the flash loan to be taken for each corresponding _flashLoanToken. This implies that for each _flashLoanToken, there should be a corresponding _flashLoanAmount. For example, if the chosen flashLoanToken is USDT, then the flashLoanAmount should be the corresponding loan amount for usdt that should be taken to repay the borrow amount.

The reason is because in VenusAssetHandler::executeVaultFlashLoan, there is the following check:

  // Initiate the flash loan from the Algebra pool
        IAlgebraPool(_poolAddress).flash(
            _receiver, // Recipient of the flash loan
            repayData._token0 == repayData._flashLoanToken
                ? repayData._flashLoanAmount[0]
                : 0, // Amount of token0 to flash loan
            repayData._token1 == repayData._flashLoanToken
                ? repayData._flashLoanAmount[0]
                : 0, // Amount of token1 to flash loan
            abi.encode(flashData) // Encode flash loan data to pass to the callback
This code block is what initiates the flash loan and the code block checks whether token0 or token1 is the flash loan token and if the flash loan token is token0 or token1 , then the flashloanamount[0] is sent to the BorrowManager contract as the flash loan. As a result of this, the BorrowManager will only have flashloanamount[0] of flash loan token.

The bug appears in VenusAssetHandler::swapAndTransferTransactions:

```solidity
/**
   * @notice Internal function to handle swaps and transfers during loan processing.
   * @param vault The address of the vault.
   * @param flashData A struct containing flash loan data.
   * @return transactions An array of transactions to execute.
   * @return totalFlashAmount The total amount of flash loan used.
   */
  function swapAndTransferTransactions(
    address vault,
    FunctionParameters.FlashLoanData memory flashData
  )
    internal
    pure
    returns (MultiTransaction[] memory transactions, uint256 totalFlashAmount)
  {
    uint256 tokenLength = flashData.debtToken.length; // Get the number of debt tokens
    transactions = new MultiTransaction[](tokenLength * 2); // Initialize the transactions array
    uint count;

    // Loop through the debt tokens to handle swaps and transfers
    for (uint i; i < tokenLength; ) {
      // Check if the flash loan token is different from the debt token
      if (flashData.flashLoanToken != flashData.debtToken[i]) {
        // Transfer the flash loan token to the solver handler
        transactions[count].to = flashData.flashLoanToken;
        transactions[count].txData = abi.encodeWithSelector(
          bytes4(keccak256("transfer(address,uint256)")),
          flashData.solverHandler, // recipient
          flashData.flashLoanAmount[i]
        );
        count++;

        // Swap the token using the solver handler
        transactions[count].to = flashData.solverHandler;
        transactions[count].txData = abi.encodeWithSelector(
          bytes4(keccak256("multiTokenSwapAndTransfer(address,bytes)")),
          vault,
          flashData.firstSwapData[i]
        );
        count++;
      }
      // Handle the case where the flash loan token is the same as the debt token
      else {
        // Transfer the token directly to the vault
        transactions[count].to = flashData.flashLoanToken;
        transactions[count].txData = abi.encodeWithSelector(
          bytes4(keccak256("transfer(address,uint256)")),
          vault, // recipient
          flashData.flashLoanAmount[i]
        );
        count++;
      }

      totalFlashAmount += flashData.flashLoanAmount[i]; // Update the total flash loan amount
      unchecked {
        ++i;
      }
    }
    // Resize the transactions array to remove unused entries
    uint unusedLength = ((tokenLength * 2) - count);
    assembly {
      mstore(transactions, sub(mload(transactions), unusedLength))
    }
  }
```

VenusAssetHandler::swapAndTransferTransactions loops through tokenLength which is flashData.debtToken.length. As per FunctionParameters::FlashLoanData describes the debt token as the addresses of the tokens representing the debt to be repaid. The idea is that the loop goes through each debt token and for each token, if the debt token is not the same as the flash loan token, then 2 transactions are included. The first is to transfer the flash loan token to the swap handler and the second is to swap the flash loan amount for the borrowed token to pay back the loan as seen above.

The bug occurs because since the BorrowManager contract will only have flashloanamount[0] of flash loan token, when the loop has to run more than once where there is more than 1 debt token to repay, the BorrowManager will not have enough flash loan token to send to the swap handler which will cause the transaction to fail. Alternatively, this bug can be presented another way. Ideally, there should be a single flash loan amount per flash loan token as explained above. If a user enters a single flashloanamount to repay more than 1 debt token, flashData.flashLoanAmount[i] where i >0 will not exist and will cause a Panic(0x32) error for trying to access an out of bounds index in an array.

Impact Explanation
Transaction Failures: Users trying to repay multiple debts using a flash loan will face failed transactions due to missing flash loan amounts.

Likelihood Explanation
The bug is highly likely to occur because:

The protocol allows users to borrow multiple tokens, making multi-token debt repayments a common scenario. The logic explicitly loops through multiple debt tokens, meaning the issue will manifest anytime more than one debt token is being repaid. There is no validation ensuring the flash loan amount array is correctly sized before being used in swapAndTransferTransactions, making an out-of-bounds read inevitable.

Proof of Concept
This test was run by replacing all the contents of 9_Borrowing.test.ts with the following:
```javascript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect, use } from "chai";
import "@nomicfoundation/hardhat-chai-matchers";
import { ethers, network, upgrades } from "hardhat";
import { BigNumber, Contract } from "ethers";
import VENUS_CHAINLINK_ORACLE_ABI from "../abi/venus_chainlink_oracle.json";

import {
  createEnsoCallData,
  createEnsoCallDataRoute,
} from "./IntentCalculations";

import { tokenAddresses, IAddresses, priceOracle } from "./Deployments.test";

import {
  Portfolio,
  Portfolio__factory,
  ProtocolConfig,
  Rebalancing__factory,
  PortfolioFactory,
  PancakeSwapHandler,
  VelvetSafeModule,
  FeeModule,
  FeeModule__factory,
  EnsoHandler,
  EnsoHandlerBundled,
  AccessController__factory,
  TokenExclusionManager__factory,
  TokenBalanceLibrary,
  BorrowManagerVenus,
  DepositBatch,
  DepositManager,
  WithdrawBatch,
  WithdrawManager,
  VenusAssetHandler,
  IAssetHandler,
  IVenusComptroller,
  IVenusPool,
} from "../../typechain";

import { chainIdToAddresses } from "../../scripts/networkVariables";
import { assert } from "console";

var chai = require("chai");
const axios = require("axios");
const qs = require("qs");
//use default BigNumber
chai.use(require("chai-bignumber")());

describe.only("Tests for Deposit", () => {
  let accounts;
  let iaddress: IAddresses;
  let vaultAddress: string;
  let velvetSafeModule: VelvetSafeModule;
  let portfolio: any;
  let portfolio1: any;
  let portfolioCalculations: any;
  let tokenExclusionManager: any;
  let tokenExclusionManager1: any;
  let borrowManager: BorrowManagerVenus;
  let ensoHandler: EnsoHandler;
  let depositBatch: DepositBatch;
  let depositManager: DepositManager;
  let venusAssetHandler: VenusAssetHandler;
  let withdrawBatch: WithdrawBatch;
  let withdrawManager: WithdrawManager;
  let portfolioContract: Portfolio;
  let comptroller: Contract;
  let portfolioFactory: PortfolioFactory;
  let swapHandler: PancakeSwapHandler;
  let rebalancing: any;
  let rebalancing1: any;
  let tokenBalanceLibrary: TokenBalanceLibrary;
  let protocolConfig: ProtocolConfig;
  let positionWrapper: any;
  let fakePortfolio: Portfolio;
  let txObject;
  let owner: SignerWithAddress;
  let treasury: SignerWithAddress;
  let _assetManagerTreasury: SignerWithAddress;
  let nonOwner: SignerWithAddress;
  let depositor1: SignerWithAddress;
  let addr2: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addrs: SignerWithAddress[];
  let feeModule0: FeeModule;
  const assetManagerHash = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes("ASSET_MANAGER")
  );

  let swapVerificationLibrary: any;
  let portfolioInfo: any;

  const provider = ethers.provider;
  const chainId: any = process.env.CHAIN_ID;
  const addresses = chainIdToAddresses[chainId];

  function delay(ms: number) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  describe.only("Tests for Deposit", () => {
    before(async () => {
      accounts = await ethers.getSigners();
      [
        owner,
        depositor1,
        nonOwner,
        treasury,
        _assetManagerTreasury,
        addr1,
        addr2,
        ...addrs
      ] = accounts;

      iaddress = await tokenAddresses();

      const EnsoHandler = await ethers.getContractFactory("EnsoHandler");
      ensoHandler = await EnsoHandler.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await ensoHandler.deployed();
      console.log("EnsoHandler deployed to:", ensoHandler.address);

      const DepositBatch = await ethers.getContractFactory("DepositBatch");
      depositBatch = await DepositBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await depositBatch.deployed();
      console.log("DepositBatch deployed to:", depositBatch.address);

      const DepositManager = await ethers.getContractFactory("DepositManager");
      depositManager = await DepositManager.deploy(depositBatch.address);
      await depositManager.deployed();
      console.log("DepositManager deployed to:", depositManager.address);

      const WithdrawBatch = await ethers.getContractFactory("WithdrawBatch");
      withdrawBatch = await WithdrawBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await withdrawBatch.deployed();
      console.log("WithdrawBatch deployed to:", withdrawBatch.address);

      const WithdrawManager = await ethers.getContractFactory(
        "WithdrawManager"
      );
      withdrawManager = await WithdrawManager.deploy();
      await withdrawManager.deployed();
      console.log("WithdrawManager deployed to:", withdrawManager.address);

      const SwapVerificationLibrary = await ethers.getContractFactory(
        "SwapVerificationLibraryAlgebra"
      );
      swapVerificationLibrary = await SwapVerificationLibrary.deploy();
      await swapVerificationLibrary.deployed();
      console.log(
        "SwapVerificationLibrary deployed to:",
        swapVerificationLibrary.address
      );

      const TokenBalanceLibrary = await ethers.getContractFactory(
        "TokenBalanceLibrary"
      );

      tokenBalanceLibrary = await TokenBalanceLibrary.deploy();
      tokenBalanceLibrary.deployed();
      console.log(
        "TokenBalanceLibrary deployed to:",
        tokenBalanceLibrary.address
      );

      const PositionWrapper = await ethers.getContractFactory(
        "PositionWrapper"
      );
      const positionWrapperBaseAddress = await PositionWrapper.deploy();
      await positionWrapperBaseAddress.deployed();
      console.log(
        "positionWrapperBaseAddress deployed to:",
        positionWrapperBaseAddress.address
      );

      const ProtocolConfig = await ethers.getContractFactory("ProtocolConfig");

      const _protocolConfig = await upgrades.deployProxy(
        ProtocolConfig,
        [treasury.address, priceOracle.address],
        { kind: "uups" }
      );
      console.log("protocolConfig deployed to:", _protocolConfig.address);

      const chainLinkOracle = "0x1B2103441A0A108daD8848D8F5d790e4D402921F";

      let oracle = new ethers.Contract(
        chainLinkOracle,
        VENUS_CHAINLINK_ORACLE_ABI,
        owner.provider
      );

      let oracleOwner = await oracle.owner();

      await network.provider.request({
        method: "hardhat_impersonateAccount",
        params: [oracleOwner],
      });
      const oracleSigner = await ethers.getSigner(oracleOwner);

      const tx = await oracle.connect(oracleSigner).setTokenConfigs([
        {
          asset: "0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB",
          feed: "0x0567F2323251f0Aab15c8dFb1967E4e8A7D42aeE",
          maxStalePeriod: "31536000",
        },
        {
          asset: "0x1AF3F329e8BE154074D8769D1FFa4eE058B1DBc3",
          feed: "0x132d3C0B1D2cEa0BC552588063bdBb210FDeecfA",
          maxStalePeriod: "31536000",
        },
        {
          asset: "0x7130d2A12B9BCbFAe4f2634d864A1Ee1Ce3Ead9c",
          feed: "0x264990fbd0A4796A3E3d8E37C4d5F87a3aCa5Ebf",
          maxStalePeriod: "31536000",
        },
      ]);
      await tx.wait();

      const PancakeSwapHandler = await ethers.getContractFactory(
        "PancakeSwapHandler"
      );
      swapHandler = await PancakeSwapHandler.deploy();
      await swapHandler.deployed();
      console.log("swapHandler deployed to:", swapHandler.address);

      protocolConfig = ProtocolConfig.attach(_protocolConfig.address);
      await protocolConfig.setCoolDownPeriod("70");
      await protocolConfig.enableSolverHandler(ensoHandler.address);
      await protocolConfig.enableSwapHandler(swapHandler.address);

      const Rebalancing = await ethers.getContractFactory("Rebalancing");
      const rebalancingDefult = await Rebalancing.deploy();
      await rebalancingDefult.deployed();
      console.log("rebalancingDefult deployed to:", rebalancingDefult.address);

      const AssetManagementConfig = await ethers.getContractFactory(
        "AssetManagementConfig"
      );
      const assetManagementConfig = await AssetManagementConfig.deploy();
      await assetManagementConfig.deployed();
      console.log(
        "assetManagementConfig deployed to:",
        assetManagementConfig.address
      );

      const TokenExclusionManager = await ethers.getContractFactory(
        "TokenExclusionManager"
      );
      const tokenExclusionManagerDefault = await TokenExclusionManager.deploy();
      await tokenExclusionManagerDefault.deployed();
      console.log(
        "tokenExclusionManagerDefault deployed to:",
        tokenExclusionManagerDefault.address
      );

      const Portfolio = await ethers.getContractFactory("Portfolio", {
        libraries: {
          TokenBalanceLibrary: tokenBalanceLibrary.address,
        },
      });
      portfolioContract = await Portfolio.deploy();
      await portfolioContract.deployed();
      console.log("portfolioContract deployed to:", portfolioContract.address);

      const VenusAssetHandler = await ethers.getContractFactory(
        "VenusAssetHandler"
      );
      venusAssetHandler = await VenusAssetHandler.deploy();
      await venusAssetHandler.deployed();
      console.log("venusAssetHandler deployed to:", venusAssetHandler.address);

      const BorrowManager = await ethers.getContractFactory(
        "BorrowManagerVenus"
      );
      borrowManager = await BorrowManager.deploy();
      await borrowManager.deployed();
      console.log("borrowManager deployed to:", borrowManager.address);

      await protocolConfig.setAssetHandlers(
        [
          addresses.vBNB_Address,
          addresses.vBTC_Address,
          addresses.vDAI_Address,
          addresses.vUSDT_Address,
          addresses.vLINK_Address,
          addresses.vUSDT_DeFi_Address,
          addresses.corePool_controller,
        ],
        [
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
        ]
      );

      await protocolConfig.setSupportedControllers([
        addresses.corePool_controller,
      ]);

      await protocolConfig.setSupportedFactory(addresses.thena_factory);

      await protocolConfig.setAssetAndMarketControllers(
        [
          addresses.vBNB_Address,
          addresses.vBTC_Address,
          addresses.vDAI_Address,
          addresses.vUSDT_Address,
          addresses.vLINK_Address,
        ],
        [
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
        ]
      );

      let whitelistedTokens = [
        iaddress.usdcAddress,
        iaddress.btcAddress,
        iaddress.ethAddress,
        iaddress.wbnbAddress,
        iaddress.usdtAddress,
        iaddress.dogeAddress,
        iaddress.daiAddress,
        iaddress.cakeAddress,
        addresses.LINK_Address,
        addresses.DOT,
        addresses.vBTC_Address,
        addresses.vETH_Address,
      ];

      let whitelist = [owner.address];

      const PositionManager = await ethers.getContractFactory(
        "PositionManagerAlgebra",
        {
          libraries: {
            SwapVerificationLibraryAlgebra: swapVerificationLibrary.address,
          },
        }
      );
      const positionManagerBaseAddress = await PositionManager.deploy();
      await positionManagerBaseAddress.deployed();
      console.log(
        "positionManagerBaseAddress deployed to:",
        positionManagerBaseAddress.address
      );

      const FeeModule = await ethers.getContractFactory("FeeModule");
      const feeModule = await FeeModule.deploy();
      await feeModule.deployed();
      console.log("feeModule deployed to:", feeModule.address);

      const TokenRemovalVault = await ethers.getContractFactory(
        "TokenRemovalVault"
      );
      const tokenRemovalVault = await TokenRemovalVault.deploy();
      await tokenRemovalVault.deployed();
      console.log("tokenRemovalVault deployed to:", tokenRemovalVault.address);

      fakePortfolio = await Portfolio.deploy();
      await fakePortfolio.deployed();
      console.log("fakePortfolio deployed to:", fakePortfolio.address);

      const VelvetSafeModule = await ethers.getContractFactory(
        "VelvetSafeModule"
      );
      velvetSafeModule = await VelvetSafeModule.deploy();
      await velvetSafeModule.deployed();
      console.log("velvetSafeModule deployed to:", velvetSafeModule.address);

      const ExternalPositionStorage = await ethers.getContractFactory(
        "ExternalPositionStorage"
      );
      const externalPositionStorage = await ExternalPositionStorage.deploy();
      await externalPositionStorage.deployed();
      console.log(
        "externalPositionStorage deployed to:",
        externalPositionStorage.address
      );

      const PortfolioFactory = await ethers.getContractFactory(
        "PortfolioFactory"
      );

      const portfolioFactoryInstance = await upgrades.deployProxy(
        PortfolioFactory,
        [
          {
            _basePortfolioAddress: portfolioContract.address,
            _baseTokenExclusionManagerAddress:
              tokenExclusionManagerDefault.address,
            _baseRebalancingAddres: rebalancingDefult.address,
            _baseAssetManagementConfigAddress: assetManagementConfig.address,
            _feeModuleImplementationAddress: feeModule.address,
            _baseTokenRemovalVaultImplementation: tokenRemovalVault.address,
            _baseVelvetGnosisSafeModuleAddress: velvetSafeModule.address,
            _basePositionManager: positionManagerBaseAddress.address,
            _baseExternalPositionStorage: externalPositionStorage.address,
            _baseBorrowManager: borrowManager.address,
            _gnosisSingleton: addresses.gnosisSingleton,
            _gnosisFallbackLibrary: addresses.gnosisFallbackLibrary,
            _gnosisMultisendLibrary: addresses.gnosisMultisendLibrary,
            _gnosisSafeProxyFactory: addresses.gnosisSafeProxyFactory,
            _protocolConfig: protocolConfig.address,
          },
        ],
        { kind: "uups" }
      );

      portfolioFactory = PortfolioFactory.attach(
        portfolioFactoryInstance.address
      );
      console.log("portfolioFactory deployed to:", portfolioFactory.address);

      await withdrawManager.initialize(
        withdrawBatch.address,
        portfolioFactory.address
      );

      console.log("portfolioFactory address:", portfolioFactory.address);
      const portfolioFactoryCreate =
        await portfolioFactory.createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "20",
          _performanceFee: "2500",
          _entryFee: "0",
          _exitFee: "0",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: true,
          _transferableToPublic: true,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });

      const portfolioFactoryCreate2 = await portfolioFactory
        .connect(nonOwner)
        .createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "200",
          _performanceFee: "2500",
          _entryFee: "10",
          _exitFee: "10",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: false,
          _transferableToPublic: false,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });
      const portfolioAddress = await portfolioFactory.getPortfolioList(0);
      portfolioInfo = await portfolioFactory.PortfolioInfolList(0);
      console.log("portfolioInfo", portfolioInfo);

      const portfolioAddress1 = await portfolioFactory.getPortfolioList(1);
      const portfolioInfo1 = await portfolioFactory.PortfolioInfolList(1);

      portfolio = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress
      );
      const PortfolioCalculations = await ethers.getContractFactory(
        "PortfolioCalculations",
        {
          libraries: {
            TokenBalanceLibrary: tokenBalanceLibrary.address,
          },
        }
      );
      feeModule0 = FeeModule.attach(await portfolio.feeModule());
      portfolioCalculations = await PortfolioCalculations.deploy();
      await portfolioCalculations.deployed();
      console.log(
        "portfolioCalculations deployed to:",
        portfolioCalculations.address
      );

      portfolio1 = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress1
      );
      console.log("portfolio1 deployed to:", portfolio1.address);

      rebalancing = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo.rebalancing
      );
      console.log("rebalancing deployed to:", rebalancing.address);

      rebalancing1 = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo1.rebalancing
      );
      console.log("rebalancing1 deployed to:", rebalancing1.address);

      tokenExclusionManager = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo.tokenExclusionManager
      );
      console.log(
        "tokenExclusionManager deployed to:",
        tokenExclusionManager.address
      );

      tokenExclusionManager1 = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo1.tokenExclusionManager
      );
      console.log(
        "tokenExclusionManager1 deployed to:",
        tokenExclusionManager1.address
      );

      console.log("portfolio deployed to:", portfolio.address);

      console.log("rebalancing:", rebalancing1.address);
      const code = await ethers.provider.getCode(
        "0xE71E810998533fB6F32E198ea11C9Dd3Ffdd2Cac"
      );
      console.log(code !== "0x" ? "This is a contract" : "This is an EOA");
    });

    describe("Deposit Tests", function () {
      it("should init tokens", async () => {
        await portfolio.initToken([
          iaddress.wbnbAddress,
          iaddress.btcAddress,
          iaddress.ethAddress,
          iaddress.dogeAddress,
          iaddress.usdcAddress,
          iaddress.cakeAddress,
        ]);
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "1000000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch
          .connect(nonOwner)
          .multiTokenSwapETHAndTransfer(
            {
              _minMintAmount: 0,
              _depositAmount: "1000000000000000000",
              _target: portfolio.address,
              _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
              _callData: postResponse,
            },
            {
              value: "100000000000000000000",
            }
          );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should rebalance to lending token vBNB", async () => {
        let tokens = await portfolio.getTokens();
        console.log(tokens);
        let sellToken = tokens[1];
        let buyToken = addresses.vBNB_Address;

        let newTokens = [
          tokens[0],
          buyToken,
          tokens[2],
          tokens[3],
          tokens[4],
          tokens[5],
        ];

        let vault = await portfolio.vault();

        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let balance = BigNumber.from(
          await ERC20.attach(sellToken).balanceOf(vault)
        ).toString();

        let balanceToSwap = BigNumber.from(balance).toString();
        console.log("Balance to rebalance", balanceToSwap);

        const postResponse = await createEnsoCallDataRoute(
          ensoHandler.address,
          ensoHandler.address,
          sellToken,
          buyToken,
          balanceToSwap
        );

        const encodedParameters = ethers.utils.defaultAbiCoder.encode(
          [
            " bytes[][]", // callDataEnso
            "bytes[]", // callDataDecreaseLiquidity
            "bytes[][]", // callDataIncreaseLiquidity
            "address[][]", // increaseLiquidityTarget
            "address[]", // underlyingTokensDecreaseLiquidity
            "address[]", // tokensIn
            "address[]", // tokens
            " uint256[]", // minExpectedOutputAmounts
          ],
          [
            [[postResponse.data.tx.data]],
            [],
            [[]],
            [[]],
            [],
            [sellToken],
            [buyToken],
            [0],
          ]
        );

        await rebalancing.updateTokens({
          _newTokens: newTokens,
          _sellTokens: [sellToken],
          _sellAmounts: [balanceToSwap],
          _handler: ensoHandler.address,
          _callData: encodedParameters,
        });

        console.log(
          "balance after sell",
          await ERC20.attach(sellToken).balanceOf(vault)
        );
        console.log(
          "balance after buy",
          await ERC20.attach(buyToken).balanceOf(vault)
        );
      });

      it("should rebalance to lending token vETH", async () => {
        //c for testing purposes
        let tokens = await portfolio.getTokens();
        let sellToken = tokens[2];
        let buyToken = addresses.vETH_Address;

        let newTokens = [
          tokens[0],
          tokens[1],
          buyToken,
          tokens[3],
          tokens[4],
          tokens[5],
        ];

        let vault = await portfolio.vault();
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let balance = BigNumber.from(
          await ERC20.attach(sellToken).balanceOf(vault)
        ).toString();

        let balanceToSwap = BigNumber.from(balance).toString();

        console.log("Balance to rebalance", balanceToSwap);

        const postResponse = await createEnsoCallDataRoute(
          ensoHandler.address,
          ensoHandler.address,
          sellToken,
          buyToken,
          balanceToSwap
        );

        const encodedParameters = ethers.utils.defaultAbiCoder.encode(
          [
            " bytes[][]", // callDataEnso
            "bytes[]", // callDataDecreaseLiquidity
            "bytes[][]", // callDataIncreaseLiquidity
            "address[][]", // increaseLiquidityTarget
            "address[]", // underlyingTokensDecreaseLiquidity
            "address[]", // tokensIn
            "address[]", // tokens
            " uint256[]", // minExpectedOutputAmounts
          ],
          [
            [[postResponse.data.tx.data]],
            [],
            [[]],
            [[]],
            [],
            [sellToken],
            [buyToken],
            [0],
          ]
        );

        await rebalancing.updateTokens({
          _newTokens: newTokens,
          _sellTokens: [sellToken],
          _sellAmounts: [balanceToSwap],
          _handler: ensoHandler.address,
          _callData: encodedParameters,
        });

        console.log(
          "balance after sell",
          await ERC20.attach(sellToken).balanceOf(vault)
        );
        console.log(
          "balance after buy",
          await ERC20.attach(buyToken).balanceOf(vault)
        );
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();
        console.log(tokens);

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should set tokens as collateral", async () => {
        let tokens = [addresses.vBNB_Address];
        let vault = await portfolio.vault();
        await rebalancing.enableCollateralTokens(
          tokens,
          addresses.corePool_controller
        );
        const comptroller: IVenusComptroller = await ethers.getContractAt(
          "IVenusComptroller",
          addresses.corePool_controller
        );

        let assets = await comptroller.getAssetsIn(vault);
        expect(assets).to.include(addresses.vBNB_Address);
      });

      it("should remove tokens as collateral", async () => {
        let tokens = [addresses.vBNB_Address];
        let vault = await portfolio.vault();
        await rebalancing.disableCollateralTokens(
          tokens,
          addresses.corePool_controller
        );
        const comptroller: IVenusComptroller = await ethers.getContractAt(
          "IVenusComptroller",
          addresses.corePool_controller
        );

        let assets = await comptroller.getAssetsIn(vault);
        expect(assets).to.not.include(addresses.vBNB_Address);
      });

      it("borrow should revert if pool is not supported", async () => {
        await expect(
          rebalancing.borrow(
            addresses.vUSDT_DeFi_Address,
            [addresses.vBNB_Address],
            addresses.USDT,
            addresses.corePool_controller,
            "2000000000000000000"
          )
        ).to.be.revertedWithCustomError(rebalancing, "InvalidAddress");
      });

      it("borrow should revert if protocol is paused", async () => {
        await protocolConfig.setProtocolPause(true);

        await expect(
          rebalancing.borrow(
            addresses.vUSDT_Address,
            [addresses.vBNB_Address],
            addresses.USDT,
            addresses.corePool_controller,
            "2000000000000000000"
          )
        ).to.be.revertedWithCustomError(rebalancing, "ProtocolIsPaused");

        await protocolConfig.setProtocolPause(false);
      });

      it("should borrow DAI using vBNB as collateral", async () => {
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let vault = await portfolio.vault();
        console.log(
          "DAI Balance before",
          await ERC20.attach(addresses.DAI_Address).balanceOf(vault)
        );

        await rebalancing.borrow(
          addresses.vDAI_Address,
          [addresses.vBNB_Address],
          addresses.DAI_Address,
          addresses.corePool_controller,
          "10000000000000000000"
        );
        console.log(
          "DAI Balance after",
          await ERC20.attach(addresses.DAI_Address).balanceOf(vault)
        );

        console.log("newtokens", await portfolio.getTokens());
      });

      it("should borrow LINK using vBNB as collateral", async () => {
        //c for testing purposes
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let tokens = await portfolio.getTokens();
        let vault = await portfolio.vault();

        const userData = await venusAssetHandler.getUserAccountData(
          vault,
          addresses.corePool_controller,
          tokens
        );
        const lendTokens = userData[1].lendTokens;
        console.log("lendTokens", lendTokens);

        const comptroller: IVenusComptroller = await ethers.getContractAt(
          "IVenusComptroller",
          addresses.corePool_controller
        );

        console.log(
          "LINK Balance before",
          await ERC20.attach(addresses.LINK_Address).balanceOf(vault)
        );

        await rebalancing.borrow(
          addresses.vLINK_Address,
          [addresses.vBNB_Address],
          addresses.LINK_Address,
          addresses.corePool_controller,
          "1000000000000000000"
        );
        console.log(
          "LINK Balance after",
          await ERC20.attach(addresses.LINK_Address).balanceOf(vault)
        );

        console.log("newtokens", await portfolio.getTokens()); //c there is an issue here because it seems like when i comment out this test, vETH isnt recognized as a lending token unless i use it to borrow something else
      });

      it("protocol owner should be able to set new max borrow token limit", async () => {
        await protocolConfig.updateMaxBorrowTokenLimit(3);
      });

      it("should fail if non protocol owner, is trying to set new max borrow token limit", async () => {
        await expect(
          protocolConfig.connect(nonOwner).updateMaxBorrowTokenLimit(3)
        ).to.be.reverted;
      });

      it("should fail if protocol owner tried to update borrow token limit, more then max limit(i.e 20)", async () => {
        expect(
          await protocolConfig.updateMaxBorrowTokenLimit(3)
        ).to.be.revertedWithCustomError(protocolConfig, "ExceedsBorrowLimit");
      });

      it("repay complete borrowed DAI and LINK using flashloan", async () => {
        //c for testing purposes

        //c get vBNB balance
        const pool: IVenusPool = await ethers.getContractAt(
          "IVenusPool",
          addresses.vBNB_Address
        );

        let vault = await portfolio.vault();
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let tokens = await portfolio.getTokens();

        const comptroller: IVenusComptroller = await ethers.getContractAt(
          "IVenusComptroller",
          addresses.corePool_controller
        );

        const vBNBBalance = await pool.balanceOf(vault);
        console.log("vBNBBalance", vBNBBalance.toString());
        let flashloanBufferUnit = 32; //Flashloan buffer unit in 1/10000
        let bufferUnit = 353; //Buffer unit for collateral amount in 1/100000
        let borrowedToken = [addresses.DAI_Address, addresses.LINK_Address];
        let borrowedProtocolToken = [
          addresses.vDAI_Address,
          addresses.vLINK_Address,
        ];

        let balanceBorrowed =
          await portfolioCalculations.getVenusTokenBorrowedBalance(
            borrowedProtocolToken,
            vault
          );

        const userData = await venusAssetHandler.getUserAccountData(
          vault,
          addresses.corePool_controller,
          tokens
        );
        const lendTokens = [userData[1].lendTokens[0]];
        console.log("userData", userData);

        console.log("balanceBorrowed before repay", balanceBorrowed);

        const balanceToRepayDAI = balanceBorrowed[0].toString();
        const balanceToRepayLINK = BigNumber.from(balanceBorrowed[1])
          .div(2)
          .toString();

        const balanceToSwapDAI =
          await portfolioCalculations.calculateFlashLoanAmountForRepayment(
            borrowedProtocolToken[0],
            addresses.vUSDT_Address, //FlashLoanProtocolToken
            addresses.corePool_controller,
            balanceToRepayDAI,
            flashloanBufferUnit
          );

        //c only repay half LINK to show where bug is
        const balanceToSwapLINK =
          await portfolioCalculations.calculateFlashLoanAmountForRepayment(
            borrowedProtocolToken[1],
            addresses.vUSDT_Address, //FlashLoanProtocolToken
            addresses.corePool_controller,
            balanceToRepayLINK,
            flashloanBufferUnit
          );

        const balanceToSwap = [
          balanceToSwapDAI.toString(),
          balanceToSwapLINK.toString(),
        ];
        const balancetoSwapAmount = BigNumber.from(balanceToSwapDAI).add(
          BigNumber.from(balanceToSwapLINK)
        );
        console.log("balancetoSwapAmount", balancetoSwapAmount);

        console.log("balanceToRepay", balanceToRepayDAI);
        console.log("balanceToSwap", balanceToSwap);

        let encodedParameters = [];

        for (let a = 0; a < borrowedToken.length; a++) {
          const postResponse = await createEnsoCallDataRoute(
            ensoHandler.address,
            ensoHandler.address,
            addresses.USDT, //FlashLoan Token
            borrowedToken[a],
            balanceToSwap[a]
          );

          encodedParameters.push(
            ethers.utils.defaultAbiCoder.encode(
              ["bytes[]", "address[]", "uint256[]"],
              [[postResponse.data.tx.data], [addresses.USDT], [0]]
            )
          );
        }
        console.log("1");

        let encodedParameters1 = [];
        //Because repay(rebalance) is one borrow token at a time

        const amounToSell =
          await portfolioCalculations.getCollateralAmountToSell(
            vault,
            addresses.corePool_controller,
            venusAssetHandler.address,
            borrowedProtocolToken,
            tokens,
            [balanceToRepayDAI, balanceToRepayLINK],
            "10", //Flash loan fee
            bufferUnit //Buffer unit for collateral amount
          );

        console.log("amounToSell", amounToSell);
        console.log("lendTokens", lendTokens);

        for (let j = 0; j < lendTokens.length; j++) {
          const postResponse1 = await createEnsoCallDataRoute(
            ensoHandler.address,
            ensoHandler.address,
            lendTokens[j],
            addresses.USDT, //FlashLoanToken
            amounToSell[j].toString() //Need calculation here //c changed from amounToSell[j].toString() for testing purposes
          );

          encodedParameters1.push(
            ethers.utils.defaultAbiCoder.encode(
              ["bytes[]", "address[]", "uint256[]"],
              [[postResponse1.data.tx.data], [addresses.USDT], [0]]
            )
          );
        }


        console.log("BEFOREREPAY");
        const tx = await rebalancing.repay(addresses.corePool_controller, {
          _factory: addresses.thena_factory,
          _token0: addresses.USDT, //USDT - Pool token
          _token1: addresses.USDC_Address, //USDC - Pool token
          _flashLoanToken: addresses.USDT, //Token to take flashlaon
          _debtToken: borrowedToken, //Token to pay debt of
          _protocolToken: borrowedProtocolToken, // lending token in case of venus
          _solverHandler: ensoHandler.address, //Handler to swap
          _bufferUnit: bufferUnit, //Buffer unit for collateral amount
          _swapHandler: swapHandler.address,
          _flashLoanAmount: [
            balancetoSwapAmount.toString(),
            balancetoSwapAmount.toString(),
          ],
          _debtRepayAmount: [balanceToRepayDAI, balanceToRepayLINK],
          firstSwapData: encodedParameters,
          secondSwapData: encodedParameters1,
          isMaxRepayment: false,
          _poolFees: [500, 500, 500],
          isDexRepayment: false,
        });

        const txreceipt = await tx.wait();
        console.log("txreceipt", txreceipt);

        balanceBorrowed =
          await portfolioCalculations.getVenusTokenBorrowedBalance(
            [addresses.vDAI_Address, addresses.vLINK_Address],
            vault
          );

        console.log("balanceBorrowed after repay", balanceBorrowed);
      });
```

This was the result from the "repay complete borrowed DAI and LINK using flashloan" test showing that the ErrorLibrary.RepayVaultCallFailed() error was triggered as a result of the transfer failure.

1) Tests for Deposit
       Tests for Deposit
         Deposit Tests
           repay complete borrowed DAI and LINK using flashloan:
     Error: VM Exception while processing transaction: reverted with an unrecognized custom error (return data: 0x6416e5ee)
    at <UnrecognizedContract>.<unknown> (0xbdcb16c381a7447f2c9173e57e66f402811e1249)
    at ERC1967Proxy._delegate (@openzeppelin/contracts/proxy/Proxy.sol:31)
    at ERC1967Proxy._fallback (@openzeppelin/contracts/proxy/Proxy.sol:60)
    at Rebalancing.repay (contracts/rebalance/Rebalancing.sol:262)
    at ERC1967Proxy._delegate (@openzeppelin/contracts/proxy/Proxy.sol:31)
    at ERC1967Proxy._fallback (@openzeppelin/contracts/proxy/Proxy.sol:60)
    at EdrProviderWrapper.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:444:41)
    at async TracerWrapper.request (node_modules/hardhat-tracer/src/wrapper.ts:66:16)
    at async EthersProviderWrapper.send (node_modules/@nomiclabs/hardhat-ethers/src/internal/ethers-provider-wrapper.ts:13:20)
  
Recommendation
To fix the issue, update VenusAssetHandler::executeVaultFlashLoan and VenusAssetHandler::swapAndTransferTransactions to correctly handle multiple debt tokens



# 6 Fees from uni v3 lp's will never be reinvested

Summary
When a user calls increaseLiquidity, any accrued Uniswap V3 fees are unintentionally transferred to the user instead of being reinvested. This is due to a logic flaw in how dust and collected fees are handled, which breaks the intended yield aggregation logic.

Finding Description
PositionManagerAbstract.sol is a base contract inherited by PositionManagerAbstractUniswap and PositionManagerUniswap to base implementation position manager that is cloned to produce a position manager for a given portfolio. The position manager provides a wrapper for uniswap's NonfungiblePositionManager and allows portfolio owners to manage liquidity positions for themselves and other users.

The key functions where the bug occurs is in the combination of PositionManagerAbstract::_collectFeesAndReinvest and PositionManagerAbstract::increaseLiquidity. See below:

  /**
   * @notice Increases liquidity in an existing Uniswap V3 position and mints corresponding wrapper tokens.
   * @param _params Struct containing parameters necessary for adding liquidity and minting tokens.
   * @dev Handles the transfer of tokens, adds liquidity to Uniswap V3, and mints wrapper tokens proportionate to the added liquidity.
   */
  function increaseLiquidity(
    WrapperFunctionParameters.WrapperDepositParams memory _params
  ) external notPaused nonReentrant {
    if (
      address(_params._positionWrapper) == address(0) ||
      _params._dustReceiver == address(0)
    ) revert ErrorLibrary.InvalidAddress();

    uint256 tokenId = _params._positionWrapper.tokenId();
    address token0 = _params._positionWrapper.token0();
    address token1 = _params._positionWrapper.token1();

    // Reinvest any collected fees back into the pool before adding new liquidity.
    _collectFeesAndReinvest(
      _params._positionWrapper,
      tokenId,
      token0,
      token1,
      _params._tokenIn,
      _params._tokenOut,
      _params._amountIn
    );

    // Track token balances before the operation to calculate dust later.
    uint256 balance0Before = IERC20Upgradeable(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20Upgradeable(token1).balanceOf(address(this));

    // Transfer the desired liquidity tokens from the caller to this contract.
    _transferTokensFromSender(
      token0,
      token1,
      _params._amount0Desired,
      _params._amount1Desired
    );

    uint256 balance0After = IERC20Upgradeable(token0).balanceOf(address(this));
    uint256 balance1After = IERC20Upgradeable(token1).balanceOf(address(this));

    _params._amount0Desired = balance0After - balance0Before;
    _params._amount1Desired = balance1After - balance1Before;

    // Approve the Uniswap manager to use the tokens for liquidity.
    _approveNonFungiblePositionManager(
      token0,
      token1,
      _params._amount0Desired,
      _params._amount1Desired
    );

    // Increase liquidity at the position.
    (uint128 liquidity, , ) = uniswapV3PositionManager.increaseLiquidity(
      INonfungiblePositionManager.IncreaseLiquidityParams({
        tokenId: tokenId,
        amount0Desired: _params._amount0Desired,
        amount1Desired: _params._amount1Desired,
        amount0Min: _params._amount0Min,
        amount1Min: _params._amount1Min,
        deadline: block.timestamp
      })
    );

    // Mint wrapper tokens corresponding to the liquidity added.
    _mintTokens(_params._positionWrapper, tokenId, liquidity);

    // Calculate token balances after the operation to determine any remaining dust.
    balance0After = IERC20Upgradeable(token0).balanceOf(address(this));
    balance1After = IERC20Upgradeable(token1).balanceOf(address(this));

    // Return any dust to the caller.
    _returnDust(
      _params._dustReceiver,
      token0,
      token1,
      balance0After,
      balance1After
    );

    emit LiquidityIncreased(msg.sender, liquidity);
  }

 function _collectFeesAndReinvest(
    IPositionWrapper _positionWrapper,
    uint256 _tokenId,
    address _token0,
    address _token1,
    address tokenIn,
    address tokenOut,
    uint256 amountIn
  ) internal {
    // Collect all available fees for the position to this contract
    uniswapV3PositionManager.collect(
      INonfungiblePositionManager.CollectParams({
        tokenId: _tokenId,
        recipient: address(this),
        amount0Max: type(uint128).max,
        amount1Max: type(uint128).max
      })
    );

    (int24 tickLower, int24 tickUpper) = _getTicksFromPosition(_tokenId);

    (uint256 feeCollectedT0, uint256 feeCollectedT1) = _swapTokensForAmount(
      WrapperFunctionParameters.SwapParams({
        _positionWrapper: _positionWrapper,
        _tokenId: _tokenId,
        _amountIn: amountIn,
        _token0: _token0,
        _token1: _token1,
        _tokenIn: tokenIn,
        _tokenOut: tokenOut,
        _tickLower: tickLower,
        _tickUpper: tickUpper
      })
    );

    // Reinvest fees if they exceed the minimum threshold for reinvestment
    if (
      feeCollectedT0 > MIN_REINVESTMENT_AMOUNT &&
      feeCollectedT1 > MIN_REINVESTMENT_AMOUNT
    ) {
      // Approve the Uniswap manager to use the tokens for liquidity.
      _approveNonFungiblePositionManager(
        _token0,
        _token1,
        feeCollectedT0,
        feeCollectedT1
      );

      // Increase liquidity using all collected fees
      uniswapV3PositionManager.increaseLiquidity(
        INonfungiblePositionManager.IncreaseLiquidityParams({
          tokenId: _tokenId,
          amount0Desired: feeCollectedT0,
          amount1Desired: feeCollectedT1,
          amount0Min: 1,
          amount1Min: 1,
          deadline: block.timestamp
        })
      );
    }
  }

When a user calls increaseLiquidity to add liquidity to a position, PositionManager::_returnDust is called which returns any refunded tokens from uni v3 back to the caller. increaseLiquidity also calls _collectFeesAndReinvest to reinvest any accumulated fees above a minimum threshold back into the position.

 function _returnDust(
    address _dustReceiver,
    address _token0,
    address _token1,
    uint256 _amount0,
    uint256 _amount1
  ) internal {
    if (_amount0 > 0)
      TransferHelper.safeTransfer(_token0, _dustReceiver, _amount0);
    if (_amount1 > 0)
      TransferHelper.safeTransfer(_token1, _dustReceiver, _amount1);
  }
The issue is that when a user increases liquidity, the balance of both input tokens (token0 and token1) for the positionmanager is calculated and passed to the _returnDust function which transfers both the uni v3 unused tokens and whatever fees have been accumulated by the position up to that point. As a result, the fees will never be reinvested as any collected fees from _collectFeesAndReinvest are transfered to the next caller of PositionManager::increaseLiquidity which is not the intended behaviour of the function.

Describe which security guarantees it breaks and how it breaks them. If this bug does not automatically happen, showcase how a malicious input would propagate through the system to the part of the code where the issue occurs.

Impact Explanation
This bug breaks expected protocol behavior:

LP fees are not reinvested as expected. LP compounding is disabled, reducing user and protocol yield. Any user can extract unearned rewards from the pool just by calling increaseLiquidity, even with minimal or no deposits.

Likelihood Explanation
This code path is actively used. Any user increasing liquidity will unintentionally or maliciously trigger the bug.

Proof of Concept
This test was run in test/Arbitrum/7_PositionWrapper.test.ts. To carry out the test, i added the following helper functions to PositionManagerAbstract:

```solidity
function collectFees(
        IPositionWrapper _positionWrapper,
        uint256 _tokenId,
        address _token0,
        address _token1,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) public payable {
        //c for testing purposes
        uniswapV3PositionManager.collect(
            INonfungiblePositionManager.CollectParams({
                tokenId: _tokenId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            })
        );

        (int24 tickLower, int24 tickUpper) = _getTicksFromPosition(_tokenId);

        (uint256 feeCollectedT0, uint256 feeCollectedT1) = _swapTokensForAmount(
            WrapperFunctionParameters.SwapParams({
                _positionWrapper: _positionWrapper,
                _tokenId: _tokenId,
                _amountIn: amountIn,
                _token0: _token0,
                _token1: _token1,
                _tokenIn: tokenIn,
                _tokenOut: tokenOut,
                _tickLower: tickLower,
                _tickUpper: tickUpper
            })
        );
    }

    function collectFeesandReinvest( //c for testing purposes
        IPositionWrapper _positionWrapper,
        uint256 _tokenId,
        address _token0,
        address _token1,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) public payable {
        _collectFeesAndReinvest(
            _positionWrapper,
            _tokenId,
            _token0,
            _token1,
            tokenIn,
            tokenOut,
            amountIn
        );
    }
```

The following helper was also added to PositionManagaerAbstractUniswap:

```solidity
 function pokePosition(uint256 tokenId) public {
        //c for testing purposes
        INonfungiblePositionManager.DecreaseLiquidityParams
            memory decreaseLiq = INonfungiblePositionManager
                .DecreaseLiquidityParams({
                    tokenId: tokenId,
                    liquidity: 10,
                    amount0Min: 0,
                    amount1Min: 0,
                    deadline: block.timestamp + 10000
                });
        INonfungiblePositionManager(address(uniswapV3PositionManager))
            .decreaseLiquidity(decreaseLiq);
    }
```

```javascript
it("fees are never reinvested", async () => {
        //c for testing purposes

        //c get the current liquidity held by position manager
        const tokenId = await positionWrapper.tokenId();
        const liquidity = await positionManager.getExistingLiquidity(tokenId);
        console.log("liquidity", liquidity);

        //c ensure there are currently no fees for either token
        const fees = await positionManager.getTokensOwed(tokenId);
        console.log("fees", fees);

        //c perform some swaps to generate fees
        //c get address with usdt and usdc
        let usdtHolder = "0xeCc2950b36293a138a2ba9d39Fc42adB56Dd86B4";
        let usdcHolder = "0x74300Ca8BC0e6b70C9A7f7E788FF6893196a83B6";

        //c get balances of both holders for usdt and usdc
        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        const tokens = await portfolio.getTokens();

        const usdtHolderBal = await ERC20.attach(addresses.USDT).balanceOf(
          usdtHolder
        );
        console.log("usdtHolderBal", usdtHolderBal.toString());

        const usdcHolderBal = await ERC20.attach(addresses.USDC).balanceOf(
          usdcHolder
        );
        console.log("usdcHolderBal", usdcHolderBal.toString());

        await network.provider.request({
          method: "hardhat_impersonateAccount",
          params: [usdtHolder],
        });

        await network.provider.request({
          method: "hardhat_impersonateAccount",
          params: [usdcHolder],
        });

        const usdtSigner = await ethers.getSigner(usdtHolder);
        const usdcSigner = await ethers.getSigner(usdcHolder);

        //c user sends native ETH to depositBatch contract
        await nonOwner.sendTransaction({
          value: "10000000000000000000",
          to: usdtHolder,
        });

        await nonOwner.sendTransaction({
          value: "10000000000000000000",
          to: usdcHolder,
        });

        const amountToSwap = "700000000000";

        //c approve USDT and USDC to swapHandler
        await ERC20.attach(addresses.USDT)
          .connect(usdtSigner)
          .approve(swapHandlerV3.address, amountToSwap);

        await ERC20.attach(addresses.USDC)
          .connect(usdcSigner)
          .approve(swapHandlerV3.address, amountToSwap);

        for (let a = 0; a < 20; a++) {
          await swapHandlerV3
            .connect(usdtSigner)
            .swapTokenToToken(
              addresses.USDT,
              addresses.USDC,
              "100",
              "10000000000"
            );
          await swapHandlerV3
            .connect(usdcSigner)
            .swapTokenToToken(
              addresses.USDC,
              addresses.USDT,
              "100",
              "10000000000"
            );
        }
        await positionManager.pokePosition(tokenId);

        //c ensure there are currently fees for both tokens
        const fees1 = await positionManager.getTokensOwed(tokenId);
        console.log("fees", fees1);

        const token0 = addresses.USDC;
        const token1 = addresses.USDT;

        //c collect fees
        await positionManager.collectFees(
          positionWrapper.address,
          tokenId,
          token0,
          token1,
          token0,
          token1,
          0
        );

        //c get balance of position manager to make sure fees are collected
        const balToken0 = await ERC20.attach(token0).balanceOf(
          positionManager.address
        );
        console.log("balToken0", balToken0.toString());
        const balToken1 = await ERC20.attach(token1).balanceOf(
          positionManager.address
        );
        console.log("balToken1", balToken1.toString());

        assert(balToken0.eq(fees1[0]));
        assert(balToken1.eq(fees1[1]));

        //c prove that fees get sent to whoever calls PositionManager::increaseLiquidity
        await increaseLiquidity(
          nonOwner,
          positionManager.address,
          swapHandler.address,
          token0,
          token1,
          position1,
          "100000000000000000",
          "100000000000000000"
        );

        //c get balance of position manager to make sure fees are collected
        const postIncreaseBalToken0 = await ERC20.attach(token0).balanceOf(
          positionManager.address
        );
        console.log("postIncreaseBalToken0", postIncreaseBalToken0.toString());
        const postIncreaseBalToken1 = await ERC20.attach(token1).balanceOf(
          positionManager.address
        );
        console.log("postIncreaseBalToken1", postIncreaseBalToken1.toString());
        assert(postIncreaseBalToken0.eq(0));
        assert(postIncreaseBalToken1.eq(0));
      });
```

Recommendation
Store fees collected in temporary variables and subtract those from the “dust” calculation before transferring tokens to the user.


# 7 Incorrect SwapVerificationLibraryUniswap::verifyOneSidedRatio formula allows incorrect swap ratio

Summary
The verifyOneSidedRatio function is intended to ensure that nearly all of the input token (tokenIn) is swapped to the output token (tokenOut) during liquidity reinvestment, allowing only a small dust remainder (e.g., 0.5%). However, the current implementation inverts the logic and allows up to 99.5% of the input token to remain, rendering the check ineffective. This can result in incorrect liquidity ratios, failed transactions, and unclaimed funds being unintentionally returned to other users.

Finding Description
This issue stems from PositionManagerAbstract::_collectFeesAndReinvest which is a function meant to collect fees from the uniswap v3 position and reinvest the fees back into the position. This function calls PositionManagerAbstractUniswap::_swapTokensForAmount which performs the relevant swaps to rebalance the tokens to have a ratio similar to the uniswap pool ratio in the relevant tick range. See below:

```solidity
 /**
   * @dev Handles swapping tokens to achieve a desired pool ratio.
   * @param _params Parameters including tokens and amounts for the swap.
   * @return balance0 Updated balance of token0.
   * @return balance1 Updated balance of token1.
   */
  function _swapTokensForAmount(
    WrapperFunctionParameters.SwapParams memory _params
  ) internal override returns (uint256 balance0, uint256 balance1) {
    // Swap tokens to the token0 or token1 pool ratio
    if (_params._amountIn > 0) {
      (balance0, balance1) = _swapTokenToToken(_params);
    } else {
      (uint128 tokensOwed0, uint128 tokensOwed1) = _getTokensOwed(
        _params._tokenId
      );
      SwapVerificationLibraryUniswap.verifyZeroSwapAmountForReinvestFees(
        protocolConfig,
        _params,
        address(uniswapV3PositionManager),
        tokensOwed0,
        tokensOwed1
      );
    }
  }
```

The path we will focus on is where _params._amountIn > 0 and _swapTokenToToken is called:


```solidity
  function _swapTokenToToken(
    WrapperFunctionParameters.SwapParams memory _params
  ) internal override returns (uint256 balance0, uint256 balance1) {
    if (
      _params._tokenIn == _params._tokenOut ||
      !(_params._tokenOut == _params._token0 ||
        _params._tokenOut == _params._token1) ||
      !(_params._tokenIn == _params._token0 ||
        _params._tokenIn == _params._token1)
    ) {
      revert ErrorLibrary.InvalidTokenAddress();
    }

    TransferHelper.safeApprove(_params._tokenIn, address(router), 0);
    TransferHelper.safeApprove(
      _params._tokenIn,
      address(router),
      _params._amountIn
    );

    uint256 balanceTokenInBeforeSwap = IERC20Upgradeable(_params._tokenIn)
      .balanceOf(address(this));

    uint256 balanceTokenOutBeforeSwap = IERC20Upgradeable(_params._tokenOut)
      .balanceOf(address(this));

    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
      .ExactInputSingleParams({
        tokenIn: _params._tokenIn,
        tokenOut: _params._tokenOut,
        fee: 100,
        recipient: address(this),
        deadline: block.timestamp,
        amountIn: _params._amountIn,
        amountOutMinimum: 0,
        sqrtPriceLimitX96: 0
      });

    router.exactInputSingle(params);

    SwapVerificationLibraryUniswap.verifySwap(
      _params._tokenIn,
      _params._tokenOut,
      _params._amountIn,
      IERC20Upgradeable(_params._tokenOut).balanceOf(address(this)) -
        balanceTokenOutBeforeSwap,
      protocolConfig.acceptedSlippageFeeReinvestment(),
      IPriceOracle(protocolConfig.oracle())
    );

    uint256 balanceTokenInAfterSwap = IERC20Upgradeable(_params._tokenIn)
      .balanceOf(address(this));

    (balance0, balance1) = SwapVerificationLibraryUniswap.verifyRatioAfterSwap(
      protocolConfig,
      _params._positionWrapper,
      address(uniswapV3PositionManager),
      _params._tickLower,
      _params._tickUpper,
      _params._token0,
      _params._token1,
      _params._tokenIn,
      balanceTokenInBeforeSwap,
      balanceTokenInAfterSwap
    );
  }
```

This function calls SwapVerificationLibraryUniswap.verifyRatioAfterSwap which is used to compare the ratio after the tokenIn to tokenOut swap to the current uniswap pool ratio at the specified tick range (between tick lower and tick upper).

```solidity
  function verifyRatioAfterSwap(
    IProtocolConfig _protocolConfig,
    IPositionWrapper _positionWrapper,
    address _nftManager,
    int24 _tickLower,
    int24 _tickUpper,
    address _token0,
    address _token1,
    address _tokenIn,
    uint256 _balanceBeforeSwap,
    uint256 _balanceAfterSwap
  ) external returns (uint256 balance0, uint256 balance1) {
    // Calculate the ratio after the swap and the expected pool ratio
    uint256 ratioAfterSwap;
    uint256 poolRatio;
    address tokenZeroBalance;
    (
      balance0,
      balance1,
      ratioAfterSwap,
      poolRatio,
      tokenZeroBalance
    ) = calculateRatios(
      _positionWrapper,
      _nftManager,
      _tickLower,
      _tickUpper,
      _token0,
      _token1
    );

    // If the pool ratio is zero, verify using a one-sided check, otherwise use a standard ratio check
    if (poolRatio == 0) {
      if (_tokenIn != tokenZeroBalance) revert ErrorLibrary.InvalidSwapToken();
      verifyOneSidedRatio(
        _protocolConfig,
        _balanceBeforeSwap,
        _balanceAfterSwap
      );
    } else {
      _verifyRatio(_protocolConfig, poolRatio, ratioAfterSwap);
    }
  }

function verifyOneSidedRatio(
    IProtocolConfig _protocolConfig,
    uint256 _balanceBeforeSwap,
    uint256 _balanceAfterSwap
  ) public view {
    uint256 allowedRatioDeviationBps = _protocolConfig
      .allowedRatioDeviationBps();
    // Calculate the maximum allowed balance after swap based on the deviation
    // This ensures that most of the tokenIn has been sold, leaving only a small dust value
    uint256 dustAllowance = (_balanceBeforeSwap *
      (TOTAL_WEIGHT - allowedRatioDeviationBps)) / TOTAL_WEIGHT;

    if (_balanceAfterSwap > dustAllowance) {
      revert ErrorLibrary.InvalidSwapAmount();
    }
  }
```

The error occurs in the verifyOneSidedRatio function which is called when the poolRatio is 0 which implies that the current tick is outside the tick range specified when calling PositionManagerAbstract::_collectFeesAndReinvest which means all the liquidity reinvested in the position will only be in one token. verifyOneSidedRatio's aim is to make sure that the tokenIn we swapped in PositionManagerAbstractUniswap::_swapTokenToToken is completely swapped for tokenOut apart from a dust amount which depends on the allowedRatioDeviationBps which is set to 50 in basis points which is 0.5%. The idea is to make sure that the tokenIn left after the swap is less than 0.5% of the tokenIn that was swapped. However, this is not what this function is doing. Instead, it is calculating dustAllowance as 99.5% of the tokenIn that was swapped and comparing it to the balanceafterswap which is not the intended functionality.

As a result of this, when poolRatio is 0 , the amount of tokenIn is allowed to be more than the allowed dust amount and up to 99.5% of the initial balance. This can lead to one of 2 errors. The first is that if there is enough tokenIn and tokenOut to increaseLiquidity in the position regardless of the failed check, uniswapv3 will refund the extra amount of tokenIn to the position manager which can and will be received by the next user who calls increaseLiquidity which is a seperate issue raised in issue 237 with POC. The second is that if the ratio after swap leaves more tokenIn and less tokenOut to fulfill the liquidity to increase the position, the tx will revert due to insufficient funds and dust amount check will not fulfill its intended purpose

Impact Explanation
Liquidity Inaccuracy: The reinvested liquidity will not reflect the intended pool ratio, potentially causing failed transactions or inefficient capital usage.

Fund Leakage Between Users: Since unswapped tokens remain on the contract and are not tracked as user-specific, they may be mistakenly sent to a different user during future calls to increaseLiquidity. This breaks token accounting and fair attribution of yield, violating user isolation.

Likelihood Explanation
It affects all reinvestments where the current price is outside the active tick range (a common scenario). The incorrect dust calculation always passes due to the inverted logic. This function is called automatically in increaseLiquidity, meaning it doesn’t require special user behavior to trigger.

Proof of Concept
This test was run in test/Bsc/7_PositionWrapper.test.ts. It shows the described miscalculation in the SwapVerificationLibraryUniswap::verifyOneSidedRatio.

```javascript
 it("verifyOneSidedRatio bug", async () => {
        //c for testing purposes
        const balBeforeSwap = BigNumber.from("100000000000000000000"); //c 1e20
        const ratioDeviation = BigNumber.from("50");

        //c the verifyOneSidedRatio function should revert if balAfterSwap is more than 0.5% of balBeforeSwap according to natspec but as we will see below, it will not revert
        const balAfterSwap = BigNumber.from("10000000000000000000"); //c 1e19 which is more than 0.5% of balBeforeSwap

        await swapVerificationLibrary.verifyOneSidedRatioTest(
          protocolConfig.address,
          balBeforeSwap,
          balAfterSwap
        );
      });
```

To run this test, I created the following helper function in SwapVerificationLibraryUniswap.sol:
```solidity
function verifyOneSidedRatioTest(
        address _protocolConfig,
        uint256 _balanceBeforeSwap,
        uint256 _balanceAfterSwap
    ) public view {
        //c added for testing purposes
        uint256 allowedRatioDeviationBps = IProtocolConfig(_protocolConfig)
            .allowedRatioDeviationBps();
        // Calculate the maximum allowed balance after swap based on the deviation
        // This ensures that most of the tokenIn has been sold, leaving only a small dust value
        uint256 dustAllowance = (_balanceBeforeSwap *
            (TOTAL_WEIGHT - allowedRatioDeviationBps)) / TOTAL_WEIGHT;

        if (_balanceAfterSwap > dustAllowance) {
            revert ErrorLibrary.InvalidSwapAmount();
        }
    }
```

Recommendation
Fix the dust check logic in verifyOneSidedRatio to correctly reflect the intended 0.5% threshold.

# 8 Oracle Will Return the Wrong Price for Asset if Underlying Aggregator Hits minAnswer

Summary
FeeModule::chargePerformanceFee function relies on on-chain price oracles to determine vault performance, but does not account for Chainlink aggregator min/max price limits. This creates a risk where stale or artificial prices can be used to calculate inflated fees.

Finding Description
FeeModule::chargePerformanceFee is a function that is used for asset managers to be able to charge performance fees based on the vaults performance.

```solidity
 /**
   * @dev Calculates and mints performance fees based on the vault's performance relative to a high watermark.
   * Can only be called by the asset manager when the protocol is not in emergency pause.
   */
  function chargePerformanceFee()
    external
    onlyAssetManager
    protocolNotEmergencyPaused
    nonReentrant
  {
    uint256 totalSupply = portfolio.totalSupply();

    uint256 vaultBalance = portfolio.getVaultValueInUSD(
      IPriceOracle(protocolConfig.oracle()),
      portfolio.getTokens(),
      totalSupply,
      portfolio.vault()
    );
    uint256 currentPrice = _getCurrentPrice(vaultBalance, totalSupply);

    if (totalSupply == 0 || vaultBalance == 0 || highWatermark == 0) {
      _updateHighWaterMark(currentPrice);
      return;
    }

    uint256 performanceFee = _calculatePerformanceFeeToMint(
      currentPrice,
      highWatermark,
      totalSupply,
      vaultBalance,
      assetManagementConfig.performanceFee()
    );

    (uint256 protocolFee, uint256 assetManagerFee) = _splitFee(
      performanceFee,
      protocolConfig.protocolFee()
    );

    _mintProtocolAndManagementFees(assetManagerFee, protocolFee);
    _updateHighWaterMark(
      _getCurrentPrice(vaultBalance, portfolio.totalSupply())
    );
  }
```

This function calls VaultCalculations::getVaultValueInUSD to be able to get the vault's balance in usd.

```solidity
 function getVaultValueInUSD(
    IPriceOracle _oracle,
    address[] memory _tokens,
    uint256 _totalSupply,
    address _vault
  ) external view returns (uint256 vaultValue) {
    if (_totalSupply == 0) return 0;

    uint256 _tokenBalanceInUSD;
    uint256 tokensLength = _tokens.length;
    for (uint256 i; i < tokensLength; i++) {
      address _token = _tokens[i];
      if (!protocolConfig().isTokenEnabled(_token))
        revert ErrorLibrary.TokenNotEnabled();
      _tokenBalanceInUSD = _oracle.convertToUSD18Decimals(
        _token,
        IERC20Upgradeable(_token).balanceOf(_vault)
      );

      vaultValue += _tokenBalanceInUSD;
    }
```

This is done by calling PriceOracleAbstract::convertToUSD18Decimals which uses chainlink aggregators to get prices of assets via chainlink oracles.

```solidity
  function convertToUSD18Decimals(
    address _base,
    uint256 amountIn
  ) external view returns (uint256 amountOut) {
    uint256 output = uint256(_latestRoundData(_base, Denominations.USD));
    uint256 decimalChainlink = decimals(_base, Denominations.USD);
    IERC20MetadataUpgradeable token = IERC20MetadataUpgradeable(_base);
    uint8 decimal = token.decimals();

    uint256 diff = 18 - decimal;

    amountOut = (output * amountIn * (10 ** diff)) / (10 ** decimalChainlink);
  }
```

The issue lies in chainlink's aggregators have a built-in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash), the price of the oracle will continue to return the minPrice instead of the actual price of the asset. See chainlink docs at https://docs.chain.link/data-feeds#overview which states that for monitoring purposes, you must decide what limits are acceptable for your application. Configure your application to detect when the reported answer is close to reaching reasonable minimum and maximum limits so it can alert you to potential market events. Assets like LINK and other price feed addresses, have aggregators with min and max price set which can be viewed at https://arbiscan.io/address/0x9b8DdcF800a7BfCdEbaD6D65514dE59160a2C9CC#readContract.

The aggregators used in PriceOracleAbstract::_latestRoundData are not configured to enforce min and max values that Velvet would tolerate and as a result, it can lead to asset managers being able to inflate the vault's balance with assets whose prices have hit the aggregators minPrice or maxPrice. As a result, performance fees can be inflated based on wrongly priced asset. As referenced above, this exact situation occured with venus and blizz finance during the LUNA crash. See https://rekt.news/venus-blizz-rekt.

Impact Explanation
The vulnerability allows asset managers to extract performance fees based on inaccurate vault valuations. This directly results in user loss and protocol misbehavior. Even a single mispriced asset (especially a low-liquidity or crashing token) can significantly skew the overall vault value and fee calculations.

Likelihood Explanation
it's plausible that tokens held in the vault could experience sharp devaluations — especially during market volatility or black swan events. A determined or opportunistic asset manager could intentionally manipulate vault holdings to include tokens with frozen prices or wait for oracle thresholds to be hit in a crash.

Proof of Concept
In the below POC, I have shown how the getVaultValueInUSD function works via a helper that mimics the function behaviour in the "show getvaultvalueinusd function utilizing price oracle" test. It shows the process of how the vault value is retrieved via chainlink oracles. A full POC would require me deploying a seperate mock aggregator and showing how this value can manipulate performance fees. I feel like that would be unneccessary as the explanation clearly describes the issue and its impact. If requested, i can provide a more detailed POC.

The helper function was created in VaultCalculations.sol:
```solidity
function getVaultValueInUSDTest(
    address _oracle,
    address[] memory _tokens,
    uint256 _totalSupply,
    address _vault
  ) external view returns (uint256 vaultValue) {
    if (_totalSupply == 0) return 0;

    uint256 _tokenBalanceInUSD;
    uint256 tokensLength = _tokens.length;
    for (uint256 i; i < tokensLength; i++) {
      address _token = _tokens[i];
      if (!protocolConfig().isTokenEnabled(_token))
        revert ErrorLibrary.TokenNotEnabled();
      _tokenBalanceInUSD = IPriceOracle(_oracle).convertToUSD18Decimals(
        _token,
        IERC20Upgradeable(_token).balanceOf(_vault)
      );

      vaultValue += _tokenBalanceInUSD;
    }
  }
```
```javascript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect } from "chai";
import "@nomicfoundation/hardhat-chai-matchers";
import { ethers, upgrades, network } from "hardhat";
import { BigNumber, Contract } from "ethers";
import {
  PERMIT2_ADDRESS,
  AllowanceTransfer,
  MaxAllowanceTransferAmount,
  PermitBatch,
} from "@uniswap/permit2-sdk";

import { IAddresses, priceOracle, protocolConfig } from "./Deployments.test";

import {
  Portfolio,
  Portfolio__factory,
  ProtocolConfig,
  Rebalancing__factory,
  PortfolioFactory,
  EnsoHandler,
  VelvetSafeModule,
  FeeModule,
  UniswapV2Handler,
  TokenBalanceLibrary,
  BorrowManagerAave,
  TokenExclusionManager,
  TokenExclusionManager__factory,
  PositionWrapper,
  AssetManagementConfig,
  PositionManagerUniswap,
  SwapHandlerV3,
  INonfungiblePositionManager,
  IProtocolConfig,
} from "../../typechain";

import { chainIdToAddresses } from "../../scripts/networkVariables";

import {
  calcuateExpectedMintAmount,
  createEnsoDataElement,
} from "../calculations/DepositCalculations.test";

import {
  swapTokensToLPTokens,
  increaseLiquidity,
  decreaseLiquidity,
  calculateSwapAmountUpdateRange,
} from "./IntentCalculations";
import { assert } from "console";

var chai = require("chai");
const axios = require("axios");
const qs = require("qs");
//use default BigNumber
chai.use(require("chai-bignumber")());

describe.only("Tests for Deposit + Withdrawal", () => {
  let accounts;
  let velvetSafeModule: VelvetSafeModule;
  let portfolio: any;
  let portfolioCalculations: any;
  let ensoHandler: EnsoHandler;
  let portfolioContract: Portfolio;
  let portfolioFactory: PortfolioFactory;
  let swapHandler: UniswapV2Handler;
  let protocolConfig: ProtocolConfig;
  let borrowManager: BorrowManagerAave;
  let tokenBalanceLibrary: TokenBalanceLibrary;
  let positionManager: PositionManagerUniswap;
  let assetManagementConfig: AssetManagementConfig;
  let positionWrapper: any;
  let txObject;
  let owner: SignerWithAddress;
  let treasury: SignerWithAddress;
  let assetManagerTreasury: SignerWithAddress;
  let nonOwner: SignerWithAddress;
  let depositor1: SignerWithAddress;
  let addr2: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addrs: SignerWithAddress[];
  let zeroAddress: any;
  let swapHandlerV3: SwapHandlerV3;
  let swapVerificationLibrary: any;

  let position1: any;

  const provider = ethers.provider;
  const chainId: any = process.env.CHAIN_ID;
  const addresses = chainIdToAddresses[chainId];

  const uniswapV3ProtocolHash = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes("UNISWAP-V3")
  );

  function delay(ms: number) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  describe.only("Tests for Position Manager + Wrapper", () => {
    const uniswapV3Router = "0xE592427A0AEce92De3Edee1F18E0157C05861564";

    /// @dev The minimum tick that may be passed to #getSqrtRatioAtTick computed from log base 1.0001 of 2**-128
    const MIN_TICK = 0; //c changed from -887272 for testing purposes
    /// @dev The maximum tick that may be passed to #getSqrtRatioAtTick computed from log base 1.0001 of 2**128
    const MAX_TICK = 2; //c changed from -MIN_TICK for testing purposes

    before(async () => {
      accounts = await ethers.getSigners();
      [
        owner,
        depositor1,
        nonOwner,
        treasury,
        assetManagerTreasury,
        addr1,
        addr2,
        ...addrs
      ] = accounts;

      const provider = ethers.getDefaultProvider();

      const SwapVerificationLibrary = await ethers.getContractFactory(
        "SwapVerificationLibraryUniswap"
      );
      swapVerificationLibrary = await SwapVerificationLibrary.deploy();
      await swapVerificationLibrary.deployed();

      const TokenBalanceLibrary = await ethers.getContractFactory(
        "TokenBalanceLibrary"
      );

      tokenBalanceLibrary = await TokenBalanceLibrary.deploy();
      await tokenBalanceLibrary.deployed();

      const EnsoHandler = await ethers.getContractFactory("EnsoHandler");
      ensoHandler = await EnsoHandler.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await ensoHandler.deployed();

      const SwapHandlerV3 = await ethers.getContractFactory("SwapHandlerV3");
      swapHandlerV3 = await SwapHandlerV3.deploy(
        uniswapV3Router,
        addresses.WETH
      );
      await swapHandlerV3.deployed();

      const PortfolioCalculations = await ethers.getContractFactory(
        "PortfolioCalculations",
        {
          libraries: {
            TokenBalanceLibrary: tokenBalanceLibrary.address,
          },
        }
      );
      portfolioCalculations = await PortfolioCalculations.deploy();
      await portfolioCalculations.deployed();

      const PositionWrapper = await ethers.getContractFactory(
        "PositionWrapper"
      );
      const positionWrapperBaseAddress = await PositionWrapper.deploy();
      await positionWrapperBaseAddress.deployed();

      const PositionManager = await ethers.getContractFactory(
        "PositionManagerUniswap",
        {
          libraries: {
            SwapVerificationLibraryUniswap: swapVerificationLibrary.address,
          },
        }
      );
      const positionManagerBaseAddress = await PositionManager.deploy();
      await positionManagerBaseAddress.deployed();

      const BorrowManager = await ethers.getContractFactory(
        "BorrowManagerAave"
      );
      borrowManager = await BorrowManager.deploy();
      await borrowManager.deployed();

      const ProtocolConfig = await ethers.getContractFactory("ProtocolConfig");

      const _protocolConfig = await upgrades.deployProxy(
        ProtocolConfig,
        [treasury.address, priceOracle.address],
        { kind: "uups" }
      );

      protocolConfig = ProtocolConfig.attach(_protocolConfig.address);
      await protocolConfig.setCoolDownPeriod("70");
      await protocolConfig.enableSolverHandler(ensoHandler.address);
      await protocolConfig.setSupportedFactory(ensoHandler.address);

      await protocolConfig.enableProtocol(
        uniswapV3ProtocolHash,
        "0xC36442b4a4522E871399CD717aBDD847Ab11FE88",
        "0xE592427A0AEce92De3Edee1F18E0157C05861564",
        positionWrapperBaseAddress.address
      );

      const TokenExclusionManager = await ethers.getContractFactory(
        "TokenExclusionManager"
      );
      const tokenExclusionManagerDefault = await TokenExclusionManager.deploy();
      await tokenExclusionManagerDefault.deployed();

      const AssetManagementConfig = await ethers.getContractFactory(
        "AssetManagementConfig"
      );
      const assetManagementConfigBase = await AssetManagementConfig.deploy();
      await assetManagementConfigBase.deployed();

      const Portfolio = await ethers.getContractFactory("Portfolio", {
        libraries: {
          TokenBalanceLibrary: tokenBalanceLibrary.address,
        },
      });
      portfolioContract = await Portfolio.deploy();
      await portfolioContract.deployed();
      const PancakeSwapHandler = await ethers.getContractFactory(
        "UniswapV2Handler"
      );
      swapHandler = await PancakeSwapHandler.deploy();
      await swapHandler.deployed();

      swapHandler.init(addresses.SushiSwapRouterAddress);

      await protocolConfig.enableSwapHandler(swapHandler.address);

      let whitelistedTokens = [
        addresses.ARB,
        addresses.WBTC,
        addresses.WETH,
        addresses.DAI,
        addresses.ADoge,
        addresses.USDCe,
        addresses.USDT,
        addresses.CAKE,
        addresses.SUSHI,
        addresses.aArbUSDC,
        addresses.aArbUSDT,
        addresses.MAIN_LP_USDT,
        addresses.USDC,
        
      ];

      let whitelist = [owner.address];

      zeroAddress = "0x0000000000000000000000000000000000000000";

      const FeeModule = await ethers.getContractFactory("FeeModule");
      const feeModule = await FeeModule.deploy();
      await feeModule.deployed();

      const Rebalancing = await ethers.getContractFactory("Rebalancing");
      const rebalancingDefult = await Rebalancing.deploy();
      await rebalancingDefult.deployed();

      const TokenRemovalVault = await ethers.getContractFactory(
        "TokenRemovalVault"
      );
      const tokenRemovalVault = await TokenRemovalVault.deploy();
      await tokenRemovalVault.deployed();

      const VelvetSafeModule = await ethers.getContractFactory(
        "VelvetSafeModule"
      );
      velvetSafeModule = await VelvetSafeModule.deploy();
      await velvetSafeModule.deployed();

      const ExternalPositionStorage = await ethers.getContractFactory(
        "ExternalPositionStorage"
      );
      const externalPositionStorage = await ExternalPositionStorage.deploy();
      await externalPositionStorage.deployed();

      const PortfolioFactory = await ethers.getContractFactory(
        "PortfolioFactory"
      );

      const portfolioFactoryInstance = await upgrades.deployProxy(
        PortfolioFactory,
        [
          {
            _basePortfolioAddress: portfolioContract.address,
            _baseTokenExclusionManagerAddress:
              tokenExclusionManagerDefault.address,
            _baseRebalancingAddres: rebalancingDefult.address,
            _baseAssetManagementConfigAddress:
              assetManagementConfigBase.address,
            _feeModuleImplementationAddress: feeModule.address,
            _baseTokenRemovalVaultImplementation: tokenRemovalVault.address,
            _baseVelvetGnosisSafeModuleAddress: velvetSafeModule.address,
            _baseBorrowManager: borrowManager.address,
            _basePositionManager: positionManagerBaseAddress.address,
            _baseExternalPositionStorage: externalPositionStorage.address,
            _gnosisSingleton: addresses.gnosisSingleton,
            _gnosisFallbackLibrary: addresses.gnosisFallbackLibrary,
            _gnosisMultisendLibrary: addresses.gnosisMultisendLibrary,
            _gnosisSafeProxyFactory: addresses.gnosisSafeProxyFactory,
            _protocolConfig: protocolConfig.address,
          },
        ],
        { kind: "uups" }
      );

      portfolioFactory = PortfolioFactory.attach(
        portfolioFactoryInstance.address
      );

      await portfolioFactory.setPositionManagerAddresses(
        "0xC36442b4a4522E871399CD717aBDD847Ab11FE88",
        "0xE592427A0AEce92De3Edee1F18E0157C05861564"
      );

      console.log("portfolioFactory address:", portfolioFactory.address);
      const portfolioFactoryCreate =
        await portfolioFactory.createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "20",
          _performanceFee: "2500",
          _entryFee: "0",
          _exitFee: "0",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: true,
          _transferableToPublic: true,
          _whitelistTokens: true,
          _witelistedProtocolIds: [uniswapV3ProtocolHash],
        });

      const portfolioAddress = await portfolioFactory.getPortfolioList(0);

      portfolio = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress
      );

      const config = await portfolio.assetManagementConfig();

      assetManagementConfig = AssetManagementConfig.attach(config);

      await assetManagementConfig.enableUniSwapV3Manager(uniswapV3ProtocolHash);

      let positionManagerAddress =
        await assetManagementConfig.positionManager();

      positionManager = PositionManager.attach(positionManagerAddress);
    });

    describe("Position Wrapper Tests", function () {
      it("non owner should not be able to enable the uniswapV3 position manager", async () => {
        await expect(
          assetManagementConfig
            .connect(nonOwner)
            .enableUniSwapV3Manager(uniswapV3ProtocolHash)
        ).to.be.revertedWithCustomError(
          assetManagementConfig,
          "CallerNotAssetManager"
        );
      });

      it("owner should not be able to enable the uniswapV3 position manager after it has already been enabled", async () => {
        await expect(
          assetManagementConfig.enableUniSwapV3Manager(uniswapV3ProtocolHash)
        ).to.be.revertedWithCustomError(
          assetManagementConfig,
          "ProtocolManagerAlreadyEnabled"
        );
      });

      it("non owner should not be able to create a new position", async () => {
        // UniswapV3 position
        const token0 = addresses.USDC;
        const token1 = addresses.USDT;

        await expect(
          positionManager
            .connect(nonOwner)
            .createNewWrapperPosition(
              token0,
              token1,
              "Test",
              "t",
              "100",
              MIN_TICK,
              MAX_TICK
            )
        ).to.be.revertedWithCustomError(
          positionManager,
          "CallerNotAssetManager"
        );
      });

      it("owner should not be able to create a new position with a non-whitelisted token", async () => {
        // UniswapV3 position
        const token0 = addresses.USDC;
        const token1 = addresses.LINK;

        await expect(
          positionManager.createNewWrapperPosition(
            token0,
            token1,
            "Test",
            "t",
            "100",
            MIN_TICK,
            MAX_TICK
          )
        ).to.be.revertedWithCustomError(positionManager, "TokenNotWhitelisted");
      });

      it("owner should not be able to create a new position with disabled tokens", async () => {
        // UniswapV3 position
        const token0 = addresses.USDC;
        const token1 = addresses.USDT;

        await expect(
          positionManager.createNewWrapperPosition(
            token0,
            token1,
            "Test",
            "t",
            "100",
            MIN_TICK,
            MAX_TICK
          )
        ).to.be.revertedWithCustomError(positionManager, "TokenNotEnabled");
      });

      it("protocol owner should enable tokens", async () => {
        await protocolConfig.enableTokens([
          addresses.USDT,
          addresses.USDC,
          addresses.WBTC,
          addresses.WETH,
          addresses.LINK,
          addresses.ARB,
          addresses.USDCe,
        ]);
      });

      it("owner should create new position", async () => {
        // UniswapV3 position
        const token0 = addresses.USDC;
        const token1 = addresses.USDT;

        await positionManager.createNewWrapperPosition(
          token0,
          token1,
          "Test",
          "t",
          "100",
          MIN_TICK,
          MAX_TICK
        );

        position1 = await positionManager.deployedPositionWrappers(0);
        console.log("position1", position1);

        const PositionWrapper = await ethers.getContractFactory(
          "PositionWrapper"
        );
        positionWrapper = PositionWrapper.attach(position1);
        const owner = await positionWrapper.owner();
        console.log("owner", owner);
        console.log("positionmanager", positionManager.address);
      });

      it("should init tokens should fail if the list includes a non-whitelisted token", async () => {
        await expect(
          portfolio.initToken([
            addresses.LINK,
            addresses.WBTC,
            addresses.USDCe,
            position1,
          ])
        ).to.be.revertedWithCustomError(portfolio, "TokenNotWhitelisted");
      });

      it("should init tokens", async () => {
        await portfolio.initToken([
          addresses.ARB,
          addresses.WBTC,
          addresses.USDCe,
          position1, //c the reason why this position can be initialised is because TokenWhitelistManagement::isTokenWhitelisted checks for wrapped positions as well as regular tokens added to whitelist
        ]);
      });

      it("owner should approve tokens to permit2 contract", async () => {
        const tokens = await portfolio.getTokens();
        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        for (let i = 0; i < tokens.length; i++) {
          await ERC20.attach(tokens[i]).approve(
            PERMIT2_ADDRESS,
            MaxAllowanceTransferAmount
          );
        }
      });

      it("nonOwner should approve tokens to permit2 contract", async () => {
        const tokens = await portfolio.getTokens();
        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        for (let i = 0; i < tokens.length; i++) {
          await ERC20.attach(tokens[i])
            .connect(nonOwner)
            .approve(PERMIT2_ADDRESS, MaxAllowanceTransferAmount);
        }
      });

      it("should deposit multi-token into fund (First Deposit)", async () => {
        function toDeadline(expiration: number) {
          return Math.floor((Date.now() + expiration) / 1000);
        }

        let tokenDetails = [];
        // swap native token to deposit token
        let amounts = [];

        const permit2 = await ethers.getContractAt(
          "IAllowanceTransfer",
          PERMIT2_ADDRESS
        );

        const supplyBefore = await portfolio.totalSupply();

        const tokens = await portfolio.getTokens();
        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        for (let i = 0; i < tokens.length; i++) {
          let { nonce } = await permit2.allowance(
            owner.address,
            tokens[i],
            portfolio.address
          );
          if (i < tokens.length - 1) {
            await swapHandler.swapETHToTokens("500", tokens[i], owner.address, {
              value: "100000000000000000",
            });
          } else {
            // UniswapV3 position
            const token0 = addresses.USDC;
            const token1 = addresses.USDT;

            let { swapResult0, swapResult1 } = await swapTokensToLPTokens(
              owner,
              positionManager.address,
              swapHandler.address,
              token0,
              token1,
              "100000000000000000",
              "100000000000000000"
            );

            await positionManager.initializePositionAndDeposit(
              owner.address,
              position1,
              {
                _amount0Desired: swapResult0,
                _amount1Desired: swapResult1,
                _amount0Min: "0",
                _amount1Min: "0",
                _deployer: zeroAddress,
              }
            );
          }

          let balance = await ERC20.attach(tokens[i]).balanceOf(owner.address);
          let detail = {
            token: tokens[i],
            amount: balance,
            expiration: toDeadline( 1000 * 60 * 60 * 30),
            nonce,
          };
          amounts.push(balance);
          tokenDetails.push(detail);
        }

        console.log(
          "user balance after first deposit, first mint",
          await positionWrapper.balanceOf(owner.address)
        );

        console.log("total supply", await positionWrapper.totalSupply());

        const permit: PermitBatch = {
          details: tokenDetails,
          spender: portfolio.address,
          sigDeadline: toDeadline( 1000 * 60 * 60 * 30),
        };

        const { domain, types, values } = AllowanceTransfer.getPermitData(
          permit,
          PERMIT2_ADDRESS,
          chainId
        );
        const signature = await owner._signTypedData(domain, types, values);

        await portfolio.multiTokenDeposit(amounts, "0", permit, signature);

        const supplyAfter = await portfolio.totalSupply();

        expect(Number(supplyAfter)).to.be.greaterThan(Number(supplyBefore));
        expect(Number(supplyAfter)).to.be.equals(
          Number("100000000000000000000")
        );
        console.log("supplyAfter", supplyAfter);
      });

      it("should deposit multi-token into fund (Second Deposit)", async () => {
        let amounts = [];
        let newAmounts: any = [];
        let leastPercentage = 0;
        function toDeadline(expiration: number) {
          return Math.floor((Date.now() + expiration) / 1000);
        }

        let tokenDetails = [];
        // swap native token to deposit token

        const permit2 = await ethers.getContractAt(
          "IAllowanceTransfer",
          PERMIT2_ADDRESS
        );

        const supplyBefore = await portfolio.totalSupply();

        const tokens = await portfolio.getTokens();
        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        for (let i = 0; i < tokens.length; i++) {
          let { nonce } = await permit2.allowance(
            owner.address,
            tokens[i],
            portfolio.address
          );
          if (i < tokens.length - 1) {
            await swapHandler.swapETHToTokens("500", tokens[i], owner.address, {
              value: "150000000000000000",
            });
          } else {
            // UniswapV3 position
            const token0 = addresses.USDC;
            const token1 = addresses.USDT;

            await increaseLiquidity(
              owner,
              positionManager.address,
              swapHandler.address,
              token0,
              token1,
              position1,
              "100000000000000000",
              "100000000000000000"
            );
          }
          let balance = await ERC20.attach(tokens[i]).balanceOf(owner.address);
          let detail = {
            token: tokens[i],
            amount: balance,
            expiration: toDeadline( 1000 * 60 * 60 * 30),
            nonce,
          };
          amounts.push(balance);
          tokenDetails.push(detail);
        }

        console.log(
          "user balance",
          await positionWrapper.balanceOf(owner.address)
        );

        console.log("total supply", await positionWrapper.totalSupply());

        const permit: PermitBatch = {
          details: tokenDetails,
          spender: portfolio.address,
          sigDeadline: toDeadline( 1000 * 60 * 60 * 30),
        };

        const { domain, types, values } = AllowanceTransfer.getPermitData(
          permit,
          PERMIT2_ADDRESS,
          chainId
        );
        const signature = await owner._signTypedData(domain, types, values);

        // Calculation to make minimum amount value for user---------------------------------
        let result = await portfolioCalculations.getUserAmountToDeposit(
          amounts,
          portfolio.address
        );
        //-----------------------------------------------------------------------------------

        newAmounts = result[0];
        leastPercentage = result[1];

        let inputAmounts = [];
        for (let i = 0; i < newAmounts.length; i++) {
          inputAmounts.push(ethers.BigNumber.from(newAmounts[i]).toString());
        }

        console.log("leastPercentage ", leastPercentage);
        console.log("totalSupply ", await portfolio.totalSupply());

        let mintAmount =
          (await calcuateExpectedMintAmount(
            leastPercentage,
            await portfolio.totalSupply()
          )) * 0.98; // 2% entry fee

        // considering 1% slippage
        await portfolio.multiTokenDeposit(
          inputAmounts,
          mintAmount.toString(),
          permit,
          signature // slippage 1%
        );

        const supplyAfter = await portfolio.totalSupply();
        expect(Number(supplyAfter)).to.be.greaterThan(Number(supplyBefore));
        console.log("supplyAfter", supplyAfter);
      });

      it("show getvaultvalueinusd function utilizing price oracle", async () => { //c for testing purposes
        let tokens = await portfolio.getTokens();
        let totalSupply = await portfolio.totalSupply();
       const vaultValue = await portfolio.getVaultValueInUSDTest(priceOracle.address, tokens.slice(0, -1), totalSupply, await portfolio.vault());
       console.log("Vault Value in USD", vaultValue);
      });
```

These are the relevant logs from the test:

Vault Value in USD BigNumber { value: "1052678146534543265097" }
        ✔ show getvaultvalueinusd function utilizing price oracle

Recommendation
Implement safeguards in convertToUSD18Decimals to validate that the price returned from the oracle is within reasonable expected bounds. This can include:

Enforcing protocol-defined min/max price thresholds per token (configurable via governance). Reverting if the returned oracle value matches the min or max price configured in the Chainlink aggregator, as that likely indicates circuit breaker activation.

# 9 Performance fees cannot be charged to portfolios with aToken or vToken

Summary
The chargePerformanceFee function fails when the vault contains aTokens or vTokens (interest-bearing tokens from Aave or Venus), since these tokens cannot be enabled in the system due to their lack of price feeds in the oracle. This prevents performance fees from being charged under normal conditions when such tokens are present in the vault.

Finding Description
FeeModule::chargePerformanceFee is a function that is used for asset managers to be able to charge performance fees based on the vaults performance.

```solidity
 /**
   * @dev Calculates and mints performance fees based on the vault's performance relative to a high watermark.
   * Can only be called by the asset manager when the protocol is not in emergency pause.
   */
  function chargePerformanceFee()
    external
    onlyAssetManager
    protocolNotEmergencyPaused
    nonReentrant
  {
    uint256 totalSupply = portfolio.totalSupply();

    uint256 vaultBalance = portfolio.getVaultValueInUSD(
      IPriceOracle(protocolConfig.oracle()),
      portfolio.getTokens(),
      totalSupply,
      portfolio.vault()
    );
    uint256 currentPrice = _getCurrentPrice(vaultBalance, totalSupply);

    if (totalSupply == 0 || vaultBalance == 0 || highWatermark == 0) {
      _updateHighWaterMark(currentPrice);
      return;
    }

    uint256 performanceFee = _calculatePerformanceFeeToMint(
      currentPrice,
      highWatermark,
      totalSupply,
      vaultBalance,
      assetManagementConfig.performanceFee()
    );

    (uint256 protocolFee, uint256 assetManagerFee) = _splitFee(
      performanceFee,
      protocolConfig.protocolFee()
    );

    _mintProtocolAndManagementFees(assetManagerFee, protocolFee);
    _updateHighWaterMark(
      _getCurrentPrice(vaultBalance, portfolio.totalSupply())
    );
  }
```

This function calls VaultCalculations::getVaultValueInUSD to be able to get the vault's balance in usd.

```solidity
 function getVaultValueInUSD(
    IPriceOracle _oracle,
    address[] memory _tokens,
    uint256 _totalSupply,
    address _vault
  ) external view returns (uint256 vaultValue) {
    if (_totalSupply == 0) return 0;

    uint256 _tokenBalanceInUSD;
    uint256 tokensLength = _tokens.length;
    for (uint256 i; i < tokensLength; i++) {
      address _token = _tokens[i];
      if (!protocolConfig().isTokenEnabled(_token))
        revert ErrorLibrary.TokenNotEnabled();
      _tokenBalanceInUSD = _oracle.convertToUSD18Decimals(
        _token,
        IERC20Upgradeable(_token).balanceOf(_vault)
      );

      vaultValue += _tokenBalanceInUSD;
    }
```

VaultCalculations::getVaultValueInUSD calls TokenManagement::isTokenEnabled to make sure the token is enabled before the oracle can interact with it. Tokens can be enabled via TokenManagement::enableTokens:

```solidity
 function enableTokens(address[] calldata _tokens) external onlyProtocolOwner {
    uint256 tokensLength = _tokens.length;
    for (uint256 i; i < tokensLength; i++) {
      address token = _tokens[i];
      if (token == address(0)) revert ErrorLibrary.InvalidTokenAddress();
      // Ensures token has a valid price in the price oracle before enabling
      if (!(priceOracle.convertToUSD18Decimals(token, 1 ether) > 0))
        revert ErrorLibrary.TokenNotInPriceOracle();

      isEnabled[token] = true;
    }
    emit TokensEnabled(_tokens);
  }
```

This function aims to check if the oracle can fetch a price for the asset. If this is not the case, the tx will revert and the token cannot be enabled. The issue lies in the fact that portfolio owners are allowed to rebalance tokens to aave or venus tokens representing portfolio's collateral on either platform. These protocol tokens represent underlying assets and these underlying assets have value that must be considered when calculating the vault balance in FeeModule::chargeperformancefees. Not including this value can massively deflate the vault's value and as a result, the performance fees that can be charged by portfolio owners.

In summary, portfolio owners who hold a or vTokens in their vault will never be able to call FeeModule::chargePerformanceFee as it will always revert due to the aToken being one of their tokens in the vault. The only way a portfolio owner can do this is by calling Rebalancing::removePortfolioToken or Rebalancing::updateTokens and in both of these cases, it requires removing protocol assets from the vault which the portfolio owner may not want to do and can lead to adverse effects like liquidations, etc.

Impact Explanation
Reduces protocol utility and adoption for advanced strategies that rely on a/vTokens. Forces users to make suboptimal choices (like withdrawing tokens or removing them from the vault) just to access fee mechanisms.

Likelihood Explanation
Use of aTokens/vTokens is expected and encouraged by the protocol (e.g., via Rebalancing::borrow or updateTokens). These interest-bearing tokens are frequently present in vaults, especially those using automated lending or borrowing strategies.

Since the failure is deterministic (always reverts when such tokens are present), it will consistently affect portfolio owners who utilize these assets.

Proof of Concept
There are 2 tests to take note of which are "attempt to charge performance fee 1" and "attempt to charge performance fee 2". The first shows feemodule.performancefee reverting with TokenNotEnabled() which shows that without enabling the token (which cannot happen for a or vTokens), the performance fee function will not work. The second shows a scenario where a portfolio owner removes the protocol token and then attempts to charge performance fees. It displays the discrepancy between the vault balance calculated and what the vault balance would have been with the inclusion of the aToken. In the current code, aTokens are not considered as explained above hence why the test shows their impact and why they must be included.

```javascript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect, assert } from "chai";
import "@nomicfoundation/hardhat-chai-matchers";
import { ethers, upgrades } from "hardhat";
import { BigNumber, Contract } from "ethers";

import {
  createEnsoCallData,
  createEnsoCallDataRoute,
} from "./IntentCalculations";

import { IAddresses, priceOracle } from "./Deployments.test";

import {
  Portfolio,
  Portfolio__factory,
  ProtocolConfig,
  Rebalancing__factory, 
  PortfolioFactory,
  EnsoHandler,
  VelvetSafeModule,
  FeeModule,
  UniswapHandler,
  DepositBatch,
  DepositManager,
  TokenBalanceLibrary,
  BorrowManagerAave,
  TokenExclusionManager,
  TokenExclusionManager__factory,
  WithdrawBatch,
  WithdrawManager,
  AaveAssetHandler,
  IAavePool,
  IPoolDataProvider,
  ERC20,
} from "../../typechain";

import { chainIdToAddresses } from "../../scripts/networkVariables";
import { AbiCoder } from "ethers/lib/utils";


var chai = require("chai");
const axios = require("axios");
const qs = require("qs");
//use default BigNumber
chai.use(require("chai-bignumber")());

describe.only("Tests for Deposit + Withdrawal", () => {
  let accounts;
  let iaddress: IAddresses;
  let vaultAddress: string;
  let velvetSafeModule: VelvetSafeModule;
  let portfolio: any;
  let portfolio1: any;
  let portfolioCalculations: any;
  let tokenExclusionManager: any;
  let tokenExclusionManager1: any;
  let ensoHandler: EnsoHandler;
  let depositBatch: DepositBatch;
  let depositManager: DepositManager;
  let withdrawBatch: WithdrawBatch;
  let withdrawManager: WithdrawManager;
  let borrowManager: BorrowManagerAave;
  let tokenBalanceLibrary: TokenBalanceLibrary;
  let portfolioContract: Portfolio;
  let portfolioFactory: PortfolioFactory;
  let swapHandler: UniswapHandler;
  let rebalancing: any;
  let rebalancing1: any;
  let protocolConfig: ProtocolConfig;
  let fakePortfolio: Portfolio;
  let txObject;
  let owner: SignerWithAddress;
  let treasury: SignerWithAddress;
  let assetManagerTreasury: SignerWithAddress;
  let nonOwner: SignerWithAddress;
  let aaveAssetHandler: AaveAssetHandler;
  let depositor1: SignerWithAddress;
  let addr2: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addrs: SignerWithAddress[];
  let feeModule0: FeeModule;
  let zeroAddress: any;
  let approve_amount = ethers.constants.MaxUint256; //(2^256 - 1 )
  let token;

  const provider = ethers.provider;
  const chainId: any = process.env.CHAIN_ID;
  const addresses = chainIdToAddresses[chainId];

  function delay(ms: number) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
  describe.only("Tests for Deposit + Withdrawal", () => {
    before(async () => {
      accounts = await ethers.getSigners();
      [
        owner,
        depositor1,
        nonOwner,
        treasury,
        assetManagerTreasury,
        addr1,
        addr2,
        ...addrs
      ] = accounts;

      const provider = ethers.getDefaultProvider();

      const TokenBalanceLibrary = await ethers.getContractFactory(
        "TokenBalanceLibrary"
      );

      tokenBalanceLibrary = await TokenBalanceLibrary.deploy();
      await tokenBalanceLibrary.deployed();

      const EnsoHandler = await ethers.getContractFactory("EnsoHandler");
      ensoHandler = await EnsoHandler.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await ensoHandler.deployed();

      const DepositBatch = await ethers.getContractFactory("DepositBatch");
      depositBatch = await DepositBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await depositBatch.deployed();

      const DepositManager = await ethers.getContractFactory("DepositManager");
      depositManager = await DepositManager.deploy(depositBatch.address);
      await depositManager.deployed();

      const WithdrawBatch = await ethers.getContractFactory("WithdrawBatch");
      withdrawBatch = await WithdrawBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await withdrawBatch.deployed();

      const WithdrawManager = await ethers.getContractFactory(
        "WithdrawManager"
      );
      withdrawManager = await WithdrawManager.deploy();
      await withdrawManager.deployed();

      const PositionWrapper = await ethers.getContractFactory(
        "PositionWrapper"
      );
      const positionWrapperBaseAddress = await PositionWrapper.deploy();
      await positionWrapperBaseAddress.deployed();

      const ProtocolConfig = await ethers.getContractFactory("ProtocolConfig");
      const _protocolConfig = await upgrades.deployProxy(
        ProtocolConfig,
        [
          treasury.address,
          priceOracle.address],
        { kind: "uups" }
      );

      const UniSwapHandler = await ethers.getContractFactory(
        "UniswapHandler"
      );
      swapHandler = await UniSwapHandler.deploy();
      await swapHandler.deployed();


      protocolConfig = ProtocolConfig.attach(_protocolConfig.address);
      await protocolConfig.setCoolDownPeriod("60");
      await protocolConfig.enableSolverHandler(ensoHandler.address);
      await protocolConfig.enableSwapHandler(swapHandler.address);
      await protocolConfig.setSupportedFactory(addresses.aavePool);

      const Rebalancing = await ethers.getContractFactory("Rebalancing");
      const rebalancingDefult = await Rebalancing.deploy();
      await rebalancingDefult.deployed();

      const TokenExclusionManager = await ethers.getContractFactory(
        "TokenExclusionManager"
      );
      const tokenExclusionManagerDefault = await TokenExclusionManager.deploy();
      await tokenExclusionManagerDefault.deployed();

      const AssetManagementConfig = await ethers.getContractFactory(
        "AssetManagementConfig"
      );
      const assetManagementConfig = await AssetManagementConfig.deploy();
      await assetManagementConfig.deployed();

      const AaveAssetHandler = await ethers.getContractFactory(
        "AaveAssetHandler"
      );
      aaveAssetHandler = await AaveAssetHandler.deploy();
      await aaveAssetHandler.deployed();

      const BorrowManager = await ethers.getContractFactory(
        "BorrowManagerAave"
      );
      borrowManager = await BorrowManager.deploy();
      await borrowManager.deployed();

      await protocolConfig.setAssetHandlers(
        [
          addresses.aArbDAI,
          addresses.aArbUSDC,
          addresses.aArbLINK,
          addresses.aArbUSDT,
          addresses.aArbWBTC,
          addresses.aArbWETH,
          addresses.aavePool,
          addresses.aArbARB,
        ],
        [
          aaveAssetHandler.address,
          aaveAssetHandler.address,
          aaveAssetHandler.address,
          aaveAssetHandler.address,
          aaveAssetHandler.address,
          aaveAssetHandler.address,
          aaveAssetHandler.address,
          aaveAssetHandler.address,
        ]
      );

      await protocolConfig.setAssetAndMarketControllers(
        [
          addresses.aArbDAI,
          addresses.aArbUSDC,
          addresses.aArbLINK,
          addresses.aArbUSDT,
          addresses.aArbWBTC,
          addresses.aArbWETH,
          addresses.aArbARB,
          addresses.aavePool,
        ],
        [
          addresses.aavePool,
          addresses.aavePool,
          addresses.aavePool,
          addresses.aavePool,
          addresses.aavePool,
          addresses.aavePool,
          addresses.aavePool,
          addresses.aavePool,
        ]
      );

      await protocolConfig.setSupportedControllers([addresses.aavePool]);

      const Portfolio = await ethers.getContractFactory("Portfolio", {
        libraries: {
          TokenBalanceLibrary: tokenBalanceLibrary.address,
        },
      });
      portfolioContract = await Portfolio.deploy();
      await portfolioContract.deployed();

      let whitelistedTokens = [
        addresses.ARB,
        addresses.WBTC,
        addresses.WETH,
        addresses.DAI,
        addresses.ADoge,
        addresses.USDCe,
        addresses.USDT,
        addresses.aArbUSDC,
        addresses.aArbUSDT,
        addresses.MAIN_LP_USDT,
      ];

      let whitelist = [owner.address];

      zeroAddress = "0x0000000000000000000000000000000000000000";

      const SwapVerificationLibrary = await ethers.getContractFactory(
        "SwapVerificationLibraryUniswap"
      );
      const swapVerificationLibrary = await SwapVerificationLibrary.deploy();
      await swapVerificationLibrary.deployed();

      const PositionManager = await ethers.getContractFactory(
        "PositionManagerUniswap",
        {
          libraries: {
            SwapVerificationLibraryUniswap: swapVerificationLibrary.address,
          },
        }
      );
      const positionManagerBaseAddress = await PositionManager.deploy();
      await positionManagerBaseAddress.deployed();

      const FeeModule = await ethers.getContractFactory("FeeModule", {});
      const feeModule = await FeeModule.deploy();
      await feeModule.deployed();

      const TokenRemovalVault = await ethers.getContractFactory(
        "TokenRemovalVault"
      );
      const tokenRemovalVault = await TokenRemovalVault.deploy();
      await tokenRemovalVault.deployed();

      fakePortfolio = await Portfolio.deploy();
      await fakePortfolio.deployed();

      const VelvetSafeModule = await ethers.getContractFactory(
        "VelvetSafeModule"
      );
      velvetSafeModule = await VelvetSafeModule.deploy();
      await velvetSafeModule.deployed();

      const ExternalPositionStorage = await ethers.getContractFactory(
        "ExternalPositionStorage"
      );
      const externalPositionStorage = await ExternalPositionStorage.deploy();
      await externalPositionStorage.deployed();

      const PortfolioFactory = await ethers.getContractFactory(
        "PortfolioFactory"
      );

      const portfolioFactoryInstance = await upgrades.deployProxy(
        PortfolioFactory,
        [
          {
            _basePortfolioAddress: portfolioContract.address,
            _baseTokenExclusionManagerAddress:
              tokenExclusionManagerDefault.address,
            _baseRebalancingAddres: rebalancingDefult.address,
            _baseAssetManagementConfigAddress: assetManagementConfig.address,
            _feeModuleImplementationAddress: feeModule.address,
            _baseTokenRemovalVaultImplementation: tokenRemovalVault.address,
            _baseVelvetGnosisSafeModuleAddress: velvetSafeModule.address,
            _baseBorrowManager: borrowManager.address,
            _basePositionManager: positionManagerBaseAddress.address,
            _baseExternalPositionStorage: externalPositionStorage.address,
            _gnosisSingleton: addresses.gnosisSingleton,
            _gnosisFallbackLibrary: addresses.gnosisFallbackLibrary,
            _gnosisMultisendLibrary: addresses.gnosisMultisendLibrary,
            _gnosisSafeProxyFactory: addresses.gnosisSafeProxyFactory,
            _protocolConfig: protocolConfig.address,
          },
        ],
        { kind: "uups" }
      );

      portfolioFactory = PortfolioFactory.attach(
        portfolioFactoryInstance.address
      );

      await withdrawManager.initialize(
        withdrawBatch.address,
        portfolioFactory.address
      );

      console.log("portfolioFactory address:", portfolioFactory.address);
      const portfolioFactoryCreate =
        await portfolioFactory.createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "20",
          _performanceFee: "2500",
          _entryFee: "0",
          _exitFee: "0",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: true,
          _transferableToPublic: true,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });

      const portfolioFactoryCreate2 = await portfolioFactory
        .connect(nonOwner)
        .createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "200",
          _performanceFee: "2500",
          _entryFee: "0",
          _exitFee: "0",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: false,
          _transferableToPublic: false,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });
      const portfolioAddress = await portfolioFactory.getPortfolioList(0);
      const portfolioInfo = await portfolioFactory.PortfolioInfolList(0);
      console.log("portfolioinfo", portfolioInfo);

      const portfolioAddress1 = await portfolioFactory.getPortfolioList(1);
      const portfolioInfo1 = await portfolioFactory.PortfolioInfolList(1);

      portfolio = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress
      );
      const PortfolioCalculations = await ethers.getContractFactory(
        "PortfolioCalculations",
        {
          libraries: {
            TokenBalanceLibrary: tokenBalanceLibrary.address,
          },
        }
      );
      feeModule0 = FeeModule.attach(await portfolio.feeModule());
      portfolioCalculations = await PortfolioCalculations.deploy();
      await portfolioCalculations.deployed();

      portfolio1 = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress1
      );

      rebalancing = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo.rebalancing
      );

      rebalancing1 = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo1.rebalancing
      );

      tokenExclusionManager = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo.tokenExclusionManager
      );

      tokenExclusionManager1 = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo1.tokenExclusionManager
      );

      console.log("portfolio deployed to:", portfolio.address);

      console.log("rebalancing:", rebalancing1.address);
    });

    describe("Deposit Tests", function () {
      it("should init tokens", async () => {
        await portfolio.initToken([
          addresses.WETH,
          addresses.WBTC,
          addresses.USDT,
          addresses.USDC,
          addresses.DAI,
          addresses.USDCe,
        ]);
      });

    

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "600000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "600000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch
          .connect(nonOwner)
          .multiTokenSwapETHAndTransfer(
            {
              _minMintAmount: 0,
              _depositAmount: "600000000000000000",
              _target: portfolio.address,
              _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
              _callData: postResponse,
            },
            {
              value: "600000000000000000",
            }
          );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should rebalance to lending token aArbLINK", async () => {
        let tokens = await portfolio.getTokens();
        let sellToken = tokens[3];
        let buyToken = addresses.aArbLINK;

        let newTokens = [
          tokens[0],
          tokens[1],
          tokens[2],
          buyToken,
          tokens[4],
          tokens[5],
        ];

        let vault = await portfolio.vault();

        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let balance = BigNumber.from(
          await ERC20.attach(sellToken).balanceOf(vault)
        ).toString();

        let balanceToSwap = BigNumber.from(balance).toString();
        console.log("Balance to rebalance", balanceToSwap);

        const postResponse = await createEnsoCallDataRoute(
          ensoHandler.address,
          ensoHandler.address,
          sellToken,
          buyToken,
          balanceToSwap
        );

        const encodedParameters = ethers.utils.defaultAbiCoder.encode(
          [
            " bytes[][]", // callDataEnso
            "bytes[]", // callDataDecreaseLiquidity
            "bytes[][]", // callDataIncreaseLiquidity
            "address[][]", // increaseLiquidityTarget
            "address[]", // underlyingTokensDecreaseLiquidity
            "address[]", // tokensIn
            "address[]", // tokens
            " uint256[]", // minExpectedOutputAmounts
          ],
          [
            [[postResponse.data.tx.data]],
            [],
            [[]],
            [[]],
            [],
            [sellToken],
            [buyToken],
            [0],
          ]
        );

        await rebalancing.updateTokens({
          _newTokens: newTokens,
          _sellTokens: [sellToken],
          _sellAmounts: [balanceToSwap],
          _handler: ensoHandler.address,
          _callData: encodedParameters,
        });

        console.log(
          "balance after sell",
          await ERC20.attach(sellToken).balanceOf(vault)
        );
        console.log(
          "balance after buy",
          await ERC20.attach(buyToken).balanceOf(vault)
        );

      const tx =  await rebalancing.enableCollateralTokens([addresses.LINK], addresses.AavePool);
     
      });

      

     it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "600000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "600000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("protocol owner should enable tokens", async () => {
        await protocolConfig.enableTokens([
          addresses.USDT,
          addresses.USDC,
          addresses.WBTC,
          addresses.WETH,
          addresses.LINK,
          addresses.ARB,
          addresses.USDCe,
          addresses.DAI,
        ]);
      });

      it("attempt to charge performance fee 1", async () => {
        //c for testing purposes
        let tokens = await portfolio.getTokens();
        console.log("tokens", tokens);
       await expect( feeModule0.chargePerformanceFee()).to.be.revertedWithCustomError(feeModule0,"TokenNotEnabled");
      });

      it("attempt to charge performance fee 2", async () => {
        let tokens = await portfolio.getTokens();
        console.log("tokens", tokens);

        let vault = await portfolio.vault();
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
      let aArbLINKBalance = await ERC20.attach(addresses.aArbLINK).balanceOf(vault);
      console.log("aArbLINKBalance", aArbLINKBalance.toString());

      const userData = await aaveAssetHandler.getUserAccountData(vault, "0x794a61358D6845594F94dc1DB02A252b5b4814aD", tokens);
      console.log("totalCollateral", userData[0].totalCollateral.toString());
      console.log("lendTokens", userData[1].lendTokens);
        await rebalancing.removePortfolioToken(tokens[3]);

      let newTokens = await portfolio.getTokens();
      console.log("newTokens", newTokens);
      
      // Execute the transaction 
      await feeModule0.chargePerformanceFee();
      console.log("Performance fee charged"); 
      
     
      // Get the vault balance using a portfolio method
      const vaultValue = await portfolio.getVaultValueInUSDTest(
        priceOracle.address,
        newTokens,
        await portfolio.totalSupply(),
        vault
      );
      
      //c this value excludes the value of the aArbLINK collateral which undervalues the value in the vault 
      console.log("vaultValue", vaultValue.toString());
      
      //c this is the correct vault balance that should include the value of the underlying collateral held in the portfolio
      const actualVaultBalance = vaultValue.add(userData[0].totalCollateral);
      console.log("actualVaultBalance", actualVaultBalance.toString());
      assert(actualVaultBalance.gt(vaultValue));
      });

```

Recommendation
Whitelist a/vTokens for fee calculation, and allow them to be enabled manually with an override — even if the oracle cannot fetch their price. The system could fetch the underlying asset’s value (e.g., from the protocol’s API or by decoding internal accounting data).


# 10 Users do not get correct shares on withdrawals on venus enabled portfolios

Summary
The VaultManager::multiTokenWithdrawal function may result in incomplete withdrawals when the portfolio holds vBNB. This happens because vBNB does not revert on failed transfers, instead returning error codes — which the protocol’s try/catch mechanism does not catch, causing silent failures.

Finding Description
VaultManager::multiTokenWithdrawal allows a user who has deposited tokens into a portfolio to withdraw their tokens. Portfolio owners are allowed to borrow assets from protocols like aave and venus as part of managing portfolios. The withdrawal logic has the following code:

```solidity
  function _processTokenWithdrawal(
    address _token,
    address _tokenReceiver,
    uint256 _portfolioTokenAmount,
    uint256 totalSupplyPortfolio,
    address[] memory _exemptionTokens,
    uint256 exemptionIndex,
    TokenBalanceLibrary.ControllerData[] memory controllersData
  ) private returns (uint256, uint256) {
    // Calculate the proportion of each token to return based on the burned portfolio tokens.
    uint256 tokenBalance = TokenBalanceLibrary._getAdjustedTokenBalance(
      _token,
      vault,
      _protocolConfig,
      controllersData
    );
    tokenBalance =
      (tokenBalance * _portfolioTokenAmount) /
      totalSupplyPortfolio;

    // Prepare the data for ERC20 token transfer
    bytes memory inputData = abi.encodeWithSelector(
      IERC20Upgradeable.transfer.selector,
      _tokenReceiver,
      tokenBalance
    );

    // Execute the transfer through the safe module and check for success
    try IVelvetSafeModule(safeModule).executeWallet(_token, inputData) {
      // Check if the token balance is zero and the current token is not an exemption token, revert with an error.
      // This check is necessary because if there is any rebase token or the protocol sets the balance to zero,
      // we need to be able to withdraw other tokens. The balance for a withdrawal should always be >0,
      // except when the user accepts to lose this token.
      if (tokenBalance == 0) {
        if (_exemptionTokens[exemptionIndex] == _token) exemptionIndex += 1;
        else revert ErrorLibrary.WithdrawalAmountIsSmall();
      }
      return (tokenBalance, exemptionIndex);
    } catch {
      // Checking if exception token was mentioned in exceptionToken array
      if (_exemptionTokens[exemptionIndex] != _token) {
        revert ErrorLibrary.InvalidExemptionTokens();
      }
      return (0, exemptionIndex + 1);
    }
  }
```

This allows any user to withdraw their share of all portfolio tokens which include any protocol tokens (a and vTokens). The issue occurs when the portfolio is holding any vTokens. The design of the vToken contract does not revert even when a transfer is not allowed. Instead, it returns the error instead and allows the rest of the execution to continue. See https://bscscan.com/address/0xa07c5b74c9b40447a954e1466938b865b6bbea36#code

```solidity
 function transferTokens(address spender, address src, address dst, uint tokens) internal returns (uint) {
        /* Fail if transfer not allowed */
        uint allowed = comptroller.transferAllowed(address(this), src, dst, tokens);
        if (allowed != 0) {
            return failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.TRANSFER_COMPTROLLER_REJECTION, allowed);
        }
```

As a result, when a user withdraws tokens, if the transfer is not allowed by the vToken contract for any reason, the rest of the VaultManager::multiTokenWithdrawal continues which leaves the user with less than their fair share of tokens returned by the portfolio.

Impact Explanation
Loss of funds: Users can be short-changed on withdrawals without realizing it, especially if the token was substantial in value (e.g., holding a large amount of vBNB).

Silent failure: The logic assumes success based on try/catch, but vBNB does not revert, masking the error.

Likelihood Explanation
Venus tokens are commonly used in DeFi strategies which makes this vulnerability have a high likelihood The current logic does not validate that the token was actually transferred — it relies only on try/catch, which is ineffective in this scenario.

Proof of Concept

```javascript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect, use } from "chai";
import "@nomicfoundation/hardhat-chai-matchers";
import { ethers, network, upgrades } from "hardhat";
import { BigNumber, Contract } from "ethers";
import VENUS_CHAINLINK_ORACLE_ABI from "../abi/venus_chainlink_oracle.json";

import {
  createEnsoCallData,
  createEnsoCallDataRoute,
} from "./IntentCalculations";

import { tokenAddresses, IAddresses, priceOracle } from "./Deployments.test";

import {
  Portfolio,
  Portfolio__factory,
  ProtocolConfig,
  Rebalancing__factory,
  PortfolioFactory,
  PancakeSwapHandler,
  VelvetSafeModule,
  FeeModule,
  FeeModule__factory,
  EnsoHandler,
  EnsoHandlerBundled,
  AccessController__factory,
  TokenExclusionManager__factory,
  TokenBalanceLibrary,
  BorrowManagerVenus,
  DepositBatch,
  DepositManager,
  WithdrawBatch,
  WithdrawManager,
  VenusAssetHandler,
  IAssetHandler,
  IVenusComptroller,
  IVenusPool,
} from "../../typechain";

import { chainIdToAddresses } from "../../scripts/networkVariables";
import { assert } from "console";

var chai = require("chai");
const axios = require("axios");
const qs = require("qs");
//use default BigNumber
chai.use(require("chai-bignumber")());

describe.only("Tests for Deposit", () => {
  let accounts;
  let iaddress: IAddresses;
  let vaultAddress: string;
  let velvetSafeModule: VelvetSafeModule;
  let portfolio: any;
  let portfolio1: any;
  let portfolioCalculations: any;
  let tokenExclusionManager: any;
  let tokenExclusionManager1: any;
  let borrowManager: BorrowManagerVenus;
  let ensoHandler: EnsoHandler;
  let depositBatch: DepositBatch;
  let depositManager: DepositManager;
  let venusAssetHandler: VenusAssetHandler;
  let withdrawBatch: WithdrawBatch;
  let withdrawManager: WithdrawManager;
  let portfolioContract: Portfolio;
  let comptroller: Contract;
  let portfolioFactory: PortfolioFactory;
  let swapHandler: PancakeSwapHandler;
  let rebalancing: any;
  let rebalancing1: any;
  let tokenBalanceLibrary: TokenBalanceLibrary;
  let protocolConfig: ProtocolConfig;
  let positionWrapper: any;
  let fakePortfolio: Portfolio;
  let txObject;
  let owner: SignerWithAddress;
  let treasury: SignerWithAddress;
  let _assetManagerTreasury: SignerWithAddress;
  let nonOwner: SignerWithAddress;
  let depositor1: SignerWithAddress;
  let addr2: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addrs: SignerWithAddress[];
  let feeModule0: FeeModule;
  const assetManagerHash = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes("ASSET_MANAGER")
  );

  let swapVerificationLibrary: any;
  let portfolioInfo: any;

  const provider = ethers.provider;
  const chainId: any = process.env.CHAIN_ID;
  const addresses = chainIdToAddresses[chainId];

  function delay(ms: number) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  describe.only("Tests for Deposit", () => {
    before(async () => {
      accounts = await ethers.getSigners();
      [
        owner,
        depositor1,
        nonOwner,
        treasury,
        _assetManagerTreasury,
        addr1,
        addr2,
        ...addrs
      ] = accounts;

      iaddress = await tokenAddresses();

      const EnsoHandler = await ethers.getContractFactory("EnsoHandler");
      ensoHandler = await EnsoHandler.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await ensoHandler.deployed();
      console.log("EnsoHandler deployed to:", ensoHandler.address);

      const DepositBatch = await ethers.getContractFactory("DepositBatch");
      depositBatch = await DepositBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await depositBatch.deployed();
      console.log("DepositBatch deployed to:", depositBatch.address);

      const DepositManager = await ethers.getContractFactory("DepositManager");
      depositManager = await DepositManager.deploy(depositBatch.address);
      await depositManager.deployed();
      console.log("DepositManager deployed to:", depositManager.address);

      const WithdrawBatch = await ethers.getContractFactory("WithdrawBatch");
      withdrawBatch = await WithdrawBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await withdrawBatch.deployed();
      console.log("WithdrawBatch deployed to:", withdrawBatch.address);

      const WithdrawManager = await ethers.getContractFactory(
        "WithdrawManager"
      );
      withdrawManager = await WithdrawManager.deploy();
      await withdrawManager.deployed();
      console.log("WithdrawManager deployed to:", withdrawManager.address);

      const SwapVerificationLibrary = await ethers.getContractFactory(
        "SwapVerificationLibraryAlgebra"
      );
      swapVerificationLibrary = await SwapVerificationLibrary.deploy();
      await swapVerificationLibrary.deployed();
      console.log(
        "SwapVerificationLibrary deployed to:",
        swapVerificationLibrary.address
      );

      const TokenBalanceLibrary = await ethers.getContractFactory(
        "TokenBalanceLibrary"
      );

      tokenBalanceLibrary = await TokenBalanceLibrary.deploy();
      tokenBalanceLibrary.deployed();
      console.log(
        "TokenBalanceLibrary deployed to:",
        tokenBalanceLibrary.address
      );

      const PositionWrapper = await ethers.getContractFactory(
        "PositionWrapper"
      );
      const positionWrapperBaseAddress = await PositionWrapper.deploy();
      await positionWrapperBaseAddress.deployed();
      console.log(
        "positionWrapperBaseAddress deployed to:",
        positionWrapperBaseAddress.address
      );

      const ProtocolConfig = await ethers.getContractFactory("ProtocolConfig");

      const _protocolConfig = await upgrades.deployProxy(
        ProtocolConfig,
        [treasury.address, priceOracle.address],
        { kind: "uups" }
      );
      console.log("protocolConfig deployed to:", _protocolConfig.address);

      const chainLinkOracle = "0x1B2103441A0A108daD8848D8F5d790e4D402921F";

      let oracle = new ethers.Contract(
        chainLinkOracle,
        VENUS_CHAINLINK_ORACLE_ABI,
        owner.provider
      );

      let oracleOwner = await oracle.owner();

      await network.provider.request({
        method: "hardhat_impersonateAccount",
        params: [oracleOwner],
      });
      const oracleSigner = await ethers.getSigner(oracleOwner);

      const tx = await oracle.connect(oracleSigner).setTokenConfigs([
        {
          asset: "0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB",
          feed: "0x0567F2323251f0Aab15c8dFb1967E4e8A7D42aeE",
          maxStalePeriod: "31536000",
        },
        {
          asset: "0x1AF3F329e8BE154074D8769D1FFa4eE058B1DBc3",
          feed: "0x132d3C0B1D2cEa0BC552588063bdBb210FDeecfA",
          maxStalePeriod: "31536000",
        },
        {
          asset: "0x7130d2A12B9BCbFAe4f2634d864A1Ee1Ce3Ead9c",
          feed: "0x264990fbd0A4796A3E3d8E37C4d5F87a3aCa5Ebf",
          maxStalePeriod: "31536000",
        },
      ]);
      await tx.wait();

      const PancakeSwapHandler = await ethers.getContractFactory(
        "PancakeSwapHandler"
      );
      swapHandler = await PancakeSwapHandler.deploy();
      await swapHandler.deployed();
      console.log("swapHandler deployed to:", swapHandler.address);

      protocolConfig = ProtocolConfig.attach(_protocolConfig.address);
      await protocolConfig.setCoolDownPeriod("70");
      await protocolConfig.enableSolverHandler(ensoHandler.address);
      await protocolConfig.enableSwapHandler(swapHandler.address);

      const Rebalancing = await ethers.getContractFactory("Rebalancing");
      const rebalancingDefult = await Rebalancing.deploy();
      await rebalancingDefult.deployed();
      console.log("rebalancingDefult deployed to:", rebalancingDefult.address);

      const AssetManagementConfig = await ethers.getContractFactory(
        "AssetManagementConfig"
      );
      const assetManagementConfig = await AssetManagementConfig.deploy();
      await assetManagementConfig.deployed();
      console.log(
        "assetManagementConfig deployed to:",
        assetManagementConfig.address
      );

      const TokenExclusionManager = await ethers.getContractFactory(
        "TokenExclusionManager"
      );
      const tokenExclusionManagerDefault = await TokenExclusionManager.deploy();
      await tokenExclusionManagerDefault.deployed();
      console.log(
        "tokenExclusionManagerDefault deployed to:",
        tokenExclusionManagerDefault.address
      );

      const Portfolio = await ethers.getContractFactory("Portfolio", {
        libraries: {
          TokenBalanceLibrary: tokenBalanceLibrary.address,
        },
      });
      portfolioContract = await Portfolio.deploy();
      await portfolioContract.deployed();
      console.log("portfolioContract deployed to:", portfolioContract.address);

      const VenusAssetHandler = await ethers.getContractFactory(
        "VenusAssetHandler"
      );
      venusAssetHandler = await VenusAssetHandler.deploy();
      await venusAssetHandler.deployed();
      console.log("venusAssetHandler deployed to:", venusAssetHandler.address);

      const BorrowManager = await ethers.getContractFactory(
        "BorrowManagerVenus"
      );
      borrowManager = await BorrowManager.deploy();
      await borrowManager.deployed();
      console.log("borrowManager deployed to:", borrowManager.address);

      await protocolConfig.setAssetHandlers(
        [
          addresses.vBNB_Address,
          addresses.vBTC_Address,
          addresses.vDAI_Address,
          addresses.vUSDT_Address,
          addresses.vLINK_Address,
          addresses.vUSDT_DeFi_Address,
          addresses.corePool_controller,
        ],
        [
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
        ]
      );

      await protocolConfig.setSupportedControllers([
        addresses.corePool_controller,
      ]);

      await protocolConfig.setSupportedFactory(addresses.thena_factory);

      await protocolConfig.setAssetAndMarketControllers(
        [
          addresses.vBNB_Address,
          addresses.vBTC_Address,
          addresses.vDAI_Address,
          addresses.vUSDT_Address,
          addresses.vLINK_Address,
        ],
        [
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
        ]
      );

      let whitelistedTokens = [
        iaddress.usdcAddress,
        iaddress.btcAddress,
        iaddress.ethAddress,
        iaddress.wbnbAddress,
        iaddress.usdtAddress,
        iaddress.dogeAddress,
        iaddress.daiAddress,
        iaddress.cakeAddress,
        addresses.LINK_Address,
        addresses.DOT,
        addresses.vBTC_Address,
        addresses.vETH_Address,
      ];

      let whitelist = [owner.address];

      const PositionManager = await ethers.getContractFactory(
        "PositionManagerAlgebra",
        {
          libraries: {
            SwapVerificationLibraryAlgebra: swapVerificationLibrary.address,
          },
        }
      );
      const positionManagerBaseAddress = await PositionManager.deploy();
      await positionManagerBaseAddress.deployed();
      console.log(
        "positionManagerBaseAddress deployed to:",
        positionManagerBaseAddress.address
      );

      const FeeModule = await ethers.getContractFactory("FeeModule");
      const feeModule = await FeeModule.deploy();
      await feeModule.deployed();
      console.log("feeModule deployed to:", feeModule.address);

      const TokenRemovalVault = await ethers.getContractFactory(
        "TokenRemovalVault"
      );
      const tokenRemovalVault = await TokenRemovalVault.deploy();
      await tokenRemovalVault.deployed();
      console.log("tokenRemovalVault deployed to:", tokenRemovalVault.address);

      fakePortfolio = await Portfolio.deploy();
      await fakePortfolio.deployed();
      console.log("fakePortfolio deployed to:", fakePortfolio.address);

      const VelvetSafeModule = await ethers.getContractFactory(
        "VelvetSafeModule"
      );
      velvetSafeModule = await VelvetSafeModule.deploy();
      await velvetSafeModule.deployed();
      console.log("velvetSafeModule deployed to:", velvetSafeModule.address);

      const ExternalPositionStorage = await ethers.getContractFactory(
        "ExternalPositionStorage"
      );
      const externalPositionStorage = await ExternalPositionStorage.deploy();
      await externalPositionStorage.deployed();
      console.log(
        "externalPositionStorage deployed to:",
        externalPositionStorage.address
      );

      const PortfolioFactory = await ethers.getContractFactory(
        "PortfolioFactory"
      );

      const portfolioFactoryInstance = await upgrades.deployProxy(
        PortfolioFactory,
        [
          {
            _basePortfolioAddress: portfolioContract.address,
            _baseTokenExclusionManagerAddress:
              tokenExclusionManagerDefault.address,
            _baseRebalancingAddres: rebalancingDefult.address,
            _baseAssetManagementConfigAddress: assetManagementConfig.address,
            _feeModuleImplementationAddress: feeModule.address,
            _baseTokenRemovalVaultImplementation: tokenRemovalVault.address,
            _baseVelvetGnosisSafeModuleAddress: velvetSafeModule.address,
            _basePositionManager: positionManagerBaseAddress.address,
            _baseExternalPositionStorage: externalPositionStorage.address,
            _baseBorrowManager: borrowManager.address,
            _gnosisSingleton: addresses.gnosisSingleton,
            _gnosisFallbackLibrary: addresses.gnosisFallbackLibrary,
            _gnosisMultisendLibrary: addresses.gnosisMultisendLibrary,
            _gnosisSafeProxyFactory: addresses.gnosisSafeProxyFactory,
            _protocolConfig: protocolConfig.address,
          },
        ],
        { kind: "uups" }
      );

      portfolioFactory = PortfolioFactory.attach(
        portfolioFactoryInstance.address
      );
      console.log("portfolioFactory deployed to:", portfolioFactory.address);

      await withdrawManager.initialize(
        withdrawBatch.address,
        portfolioFactory.address
      );

      console.log("portfolioFactory address:", portfolioFactory.address);
      const portfolioFactoryCreate =
        await portfolioFactory.createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "20",
          _performanceFee: "2500",
          _entryFee: "0",
          _exitFee: "0",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: true,
          _transferableToPublic: true,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });

      const portfolioFactoryCreate2 = await portfolioFactory
        .connect(nonOwner)
        .createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "200",
          _performanceFee: "2500",
          _entryFee: "10",
          _exitFee: "10",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: false,
          _transferableToPublic: false,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });
      const portfolioAddress = await portfolioFactory.getPortfolioList(0);
      portfolioInfo = await portfolioFactory.PortfolioInfolList(0);
      console.log("portfolioInfo", portfolioInfo);

      const portfolioAddress1 = await portfolioFactory.getPortfolioList(1);
      const portfolioInfo1 = await portfolioFactory.PortfolioInfolList(1);

      portfolio = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress
      );
      const PortfolioCalculations = await ethers.getContractFactory(
        "PortfolioCalculations",
        {
          libraries: {
            TokenBalanceLibrary: tokenBalanceLibrary.address,
          },
        }
      );
      feeModule0 = FeeModule.attach(await portfolio.feeModule());
      portfolioCalculations = await PortfolioCalculations.deploy();
      await portfolioCalculations.deployed();
      console.log(
        "portfolioCalculations deployed to:",
        portfolioCalculations.address
      );

      portfolio1 = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress1
      );
      console.log("portfolio1 deployed to:", portfolio1.address);

      rebalancing = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo.rebalancing
      );
      console.log("rebalancing deployed to:", rebalancing.address);

      rebalancing1 = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo1.rebalancing
      );
      console.log("rebalancing1 deployed to:", rebalancing1.address);

      tokenExclusionManager = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo.tokenExclusionManager
      );
      console.log(
        "tokenExclusionManager deployed to:",
        tokenExclusionManager.address
      );

      tokenExclusionManager1 = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo1.tokenExclusionManager
      );
      console.log(
        "tokenExclusionManager1 deployed to:",
        tokenExclusionManager1.address
      );

      console.log("portfolio deployed to:", portfolio.address);

      console.log("rebalancing:", rebalancing1.address);
      const code = await ethers.provider.getCode(
        "0xE71E810998533fB6F32E198ea11C9Dd3Ffdd2Cac"
      );
      console.log(code !== "0x" ? "This is a contract" : "This is an EOA");
    });

    describe("Deposit Tests", function () {
      it("should init tokens", async () => {
        await portfolio.initToken([
          iaddress.wbnbAddress,
          iaddress.btcAddress,
          iaddress.ethAddress,
          iaddress.dogeAddress,
          iaddress.usdcAddress,
          iaddress.cakeAddress,
        ]);
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "1000000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch
          .connect(nonOwner)
          .multiTokenSwapETHAndTransfer(
            {
              _minMintAmount: 0,
              _depositAmount: "1000000000000000000",
              _target: portfolio.address,
              _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
              _callData: postResponse,
            },
            {
              value: "100000000000000000000",
            }
          );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should rebalance to lending token vBNB", async () => {
        let tokens = await portfolio.getTokens();
        console.log(tokens);
        let sellToken = tokens[1];
        let buyToken = addresses.vBNB_Address;

        let newTokens = [
          tokens[0],
          buyToken,
          tokens[2],
          tokens[3],
          tokens[4],
          tokens[5],
        ];

        let vault = await portfolio.vault();

        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let balance = BigNumber.from(
          await ERC20.attach(sellToken).balanceOf(vault)
        ).toString();

        let balanceToSwap = BigNumber.from(balance).toString();
        console.log("Balance to rebalance", balanceToSwap);

        const postResponse = await createEnsoCallDataRoute(
          ensoHandler.address,
          ensoHandler.address,
          sellToken,
          buyToken,
          balanceToSwap
        );

        const encodedParameters = ethers.utils.defaultAbiCoder.encode(
          [
            " bytes[][]", // callDataEnso
            "bytes[]", // callDataDecreaseLiquidity
            "bytes[][]", // callDataIncreaseLiquidity
            "address[][]", // increaseLiquidityTarget
            "address[]", // underlyingTokensDecreaseLiquidity
            "address[]", // tokensIn
            "address[]", // tokens
            " uint256[]", // minExpectedOutputAmounts
          ],
          [
            [[postResponse.data.tx.data]],
            [],
            [[]],
            [[]],
            [],
            [sellToken],
            [buyToken],
            [0],
          ]
        );

        await rebalancing.updateTokens({
          _newTokens: newTokens,
          _sellTokens: [sellToken],
          _sellAmounts: [balanceToSwap],
          _handler: ensoHandler.address,
          _callData: encodedParameters,
        });

        console.log(
          "balance after sell",
          await ERC20.attach(sellToken).balanceOf(vault)
        );
        console.log(
          "balance after buy",
          await ERC20.attach(buyToken).balanceOf(vault)
        );
      });

    
      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();
        console.log(tokens);

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.connect(nonOwner).multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

     
      it("should borrow DAI using vBNB as collateral", async () => {
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let vault = await portfolio.vault();
        let tokens = await portfolio.getTokens();
       

       
        console.log(
          "DAI Balance before",
          await ERC20.attach(addresses.DAI_Address).balanceOf(vault)
        );

        await rebalancing.borrow(
          addresses.vDAI_Address,
          [addresses.vBNB_Address],
          addresses.DAI_Address,
          addresses.corePool_controller,
          "500000000000000000000"
        );
        console.log(
          "DAI Balance after",
          await ERC20.attach(addresses.DAI_Address).balanceOf(vault)
        );

        console.log("newtokens", await portfolio.getTokens());
        const userData = await venusAssetHandler.getUserAccountData(
          vault,
          addresses.corePool_controller,
          tokens
        );
        const totalCollateral = userData[0].totalCollateral;
        const liqThreshold = userData[0].ltv;
        const totalDebt = userData[0].totalDebt;
        console.log("totalCollateral", totalCollateral);
        console.log("liqThreshold", liqThreshold);
        console.log("totalDebt", totalDebt);

        const currentthreshold = BigNumber.from(totalDebt).mul(100).div(totalCollateral);
        console.log("currentthreshold", currentthreshold);
        const vBNBBalanceBefore = await ERC20.attach(addresses.vBNB_Address).balanceOf(
          nonOwner.address
         );
         console.log("vBNBBalanceBefore", vBNBBalanceBefore);
      });

      
      it("should withdraw in multitoken for nonOwner", async () => {
        await ethers.provider.send("evm_increaseTime", [70]);

        const supplyBefore = await portfolio.totalSupply();
        const amountPortfolioToken = await portfolio.balanceOf(nonOwner.address);
        const amountPortToken40Percent = BigNumber.from(amountPortfolioToken).mul(40).div(100);
        

        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        const tokens = await portfolio.getTokens();
        const token0BalanceBefore = await ERC20.attach(tokens[0]).balanceOf(
          nonOwner.address
        );
        const token1BalanceBefore = await ERC20.attach(tokens[1]).balanceOf(
          nonOwner.address
        );
        const token2BalanceBefore = await ERC20.attach(tokens[2]).balanceOf(
          nonOwner.address
        );

        const token3BalanceBefore = await ERC20.attach(tokens[3]).balanceOf(
          nonOwner.address
        );

        const token4BalanceBefore = await ERC20.attach(tokens[4]).balanceOf(
          nonOwner.address
        );

        const token5BalanceBefore = await ERC20.attach(tokens[5]).balanceOf(
          nonOwner.address
        );
        let vault = await portfolio.vault();

        const vBNBBalanceBefore = await ERC20.attach(addresses.vBNB_Address).balanceOf(
          vault
        );
        console.log("vBNBBalanceBefore", vBNBBalanceBefore);
       
        const zeroAddress = ethers.constants.AddressZero;

       const tx =  await portfolio
          .connect(nonOwner)
          .multiTokenWithdrawal(
            BigNumber.from(amountPortToken40Percent),
            {
              _factory: addresses.thena_factory,
              _token0: zeroAddress, //USDT - Pool token
              _token1: zeroAddress, //USDC - Pool token
              _flashLoanToken: zeroAddress, //Token to take flashlaon
              _bufferUnit: "0",
              _solverHandler: ensoHandler.address, //Handler to swap
              _flashLoanAmount: [0],
              firstSwapData: ["0x"],
              secondSwapData: ["0x"],
              isDexRepayment: false,
              _poolFees: [0, 0, 0],
              _swapHandler: swapHandler.address,
            }
          );
        const txreceipt = await tx.wait();
        console.log("txreceipt", txreceipt);

        const supplyAfter = await portfolio.totalSupply();

        const token0BalanceAfter = await ERC20.attach(tokens[0]).balanceOf(
          nonOwner.address
        );
        const token1BalanceAfter = await ERC20.attach(tokens[1]).balanceOf(
          nonOwner.address
        );
        const token2BalanceAfter = await ERC20.attach(tokens[2]).balanceOf(
          nonOwner.address
        );
        const token3BalanceAfter = await ERC20.attach(tokens[3]).balanceOf(
          nonOwner.address
        );
        const token4BalanceAfter = await ERC20.attach(tokens[4]).balanceOf(
          nonOwner.address
        );
        const token5BalanceAfter = await ERC20.attach(tokens[5]).balanceOf(
          nonOwner.address
        );

        expect(Number(supplyBefore)).to.be.greaterThan(Number(supplyAfter));
        expect(Number(token0BalanceAfter)).to.be.greaterThan(
          Number(token0BalanceBefore)
        );
       
        expect(Number(token2BalanceAfter)).to.be.greaterThan(
          Number(token2BalanceBefore)
        );

        expect(Number(token3BalanceAfter)).to.be.greaterThan(
          Number(token3BalanceBefore)
        );

        expect(Number(token4BalanceAfter)).to.be.greaterThan(
          Number(token4BalanceBefore)
        );

        expect(Number(token5BalanceAfter)).to.be.greaterThan(
          Number(token5BalanceBefore)
        );

        const userData = await venusAssetHandler.getUserAccountData(
          vault,
          addresses.corePool_controller,
          tokens
        );
        const totalCollateral = userData[0].totalCollateral;
        const liqThreshold = userData[0].ltv;
        const totalDebt = userData[0].totalDebt;
        console.log("totalCollateral", totalCollateral);
        console.log("liqThreshold", liqThreshold);
        console.log("totalDebt", totalDebt);

        //c notice how the threshold does not change and neither does the vBNB balance of the vault which signals that no vBNB tokens were sent to nonOwner
        const currentthreshold = BigNumber.from(totalDebt).mul(10000).div(totalCollateral);
        console.log("currentthreshold", currentthreshold);
        const vBNBBalanceAfter = await ERC20.attach(addresses.vBNB_Address).balanceOf(
         vault
        );
        console.log("vBNBBalanceAfter", vBNBBalanceAfter);
      });

```

Recommendation
Update _processTokenWithdrawal to explicitly check the return value of the token transfer if interacting with non-reverting tokens like vBNB

# 11 Malicious users can liquidate vault positions in volatile market

Summary
The current withdrawal logic in VaultManager::multiTokenWithdrawal allows users to extract their proportional share of portfolio-held assets, including protocol tokens like vTokens. However, this withdrawal mechanism does not consider the systemic risk it may pose to active protocol positions such as Venus borrows, potentially leaving the vault under-collateralized and vulnerable to liquidation through malicious timing.

Finding Description
VaultManager::multiTokenWithdrawal allows a user who has deposited tokens into a portfolio to withdraw their tokens. Portfolio owners are allowed to borrow assets from protocols like aave and venus as part of managing portfolios. The withdrawal logic has the following code:

```solidity
  function _processTokenWithdrawal(
    address _token,
    address _tokenReceiver,
    uint256 _portfolioTokenAmount,
    uint256 totalSupplyPortfolio,
    address[] memory _exemptionTokens,
    uint256 exemptionIndex,
    TokenBalanceLibrary.ControllerData[] memory controllersData
  ) private returns (uint256, uint256) {
    // Calculate the proportion of each token to return based on the burned portfolio tokens.
    uint256 tokenBalance = TokenBalanceLibrary._getAdjustedTokenBalance(
      _token,
      vault,
      _protocolConfig,
      controllersData
    );
    tokenBalance =
      (tokenBalance * _portfolioTokenAmount) /
      totalSupplyPortfolio;

    // Prepare the data for ERC20 token transfer
    bytes memory inputData = abi.encodeWithSelector(
      IERC20Upgradeable.transfer.selector,
      _tokenReceiver,
      tokenBalance
    );

    // Execute the transfer through the safe module and check for success
    try IVelvetSafeModule(safeModule).executeWallet(_token, inputData) {
      // Check if the token balance is zero and the current token is not an exemption token, revert with an error.
      // This check is necessary because if there is any rebase token or the protocol sets the balance to zero,
      // we need to be able to withdraw other tokens. The balance for a withdrawal should always be >0,
      // except when the user accepts to lose this token.
      if (tokenBalance == 0) {
        if (_exemptionTokens[exemptionIndex] == _token) exemptionIndex += 1;
        else revert ErrorLibrary.WithdrawalAmountIsSmall();
      }
      return (tokenBalance, exemptionIndex);
    } catch {
      // Checking if exception token was mentioned in exceptionToken array
      if (_exemptionTokens[exemptionIndex] != _token) {
        revert ErrorLibrary.InvalidExemptionTokens();
      }
      return (0, exemptionIndex + 1);
    }
  }
```

This allows any user to withdraw their share of all portfolio tokens which include any protocol tokens (a and vTokens). vTokens are transferable and will be transferred to any user who has the appropriate amount of portfolio tokens.

The issue occurs where a portfolio has an open position on venus for example with active borrows. A malicious user with a deposit in the portfolio can initiate a withdrawal that leaves the portfolio in a position where it is dangerously close to the liquidation threshold as vTokens from the vault will be transferred to the user. This will reduce the safety of the vault's venus position if the user has significant shares of the portfolio token. The malicious user can withdraw enough tokens that will leave the portfolio's position at the liquidation threshold. The user can time this withdrawal to occur when markets are volatile and this means that if the vault is at the liquidation threshold, any slight price movement will mean that the vault's position will be in shortfall, ripe for liquidation. The malicious user knowing this, can then liquidate the vault's position once it's shortfall > 0.

Impact Explanation
Forced liquidation of the vault's positions, leading to partial loss of protocol-held collateral. Profit extraction by malicious users who act both as vault token holders and liquidators. Protocol instability, particularly during volatile market conditions.

Likelihood Explanation
On-chain transparency: Health factor and liquidity data from the Venus Comptroller are publicly accessible. Any user can monitor the vault’s collateralization status in real time and identify when it approaches the liquidation threshold.

Low barrier to withdrawal: There are no current restrictions or safeguards in place that prevent withdrawals when the vault's borrow position is at risk. Any user holding portfolio tokens can initiate a withdrawal without protocol-level checks on post-withdrawal solvency.

Proof of Concept
The important test here is the "malicious user can liquidate vault" test which shows that a user with enough portfolio tokens can bring the vault's position to be very close/meet the liquidation threshold, making it ripe for liquidation with any slight price movements. Once the price movements occur, the user can then liquidate the vault's position and profit.

```javascript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect, use } from "chai";
import "@nomicfoundation/hardhat-chai-matchers";
import { ethers, network, upgrades } from "hardhat";
import { BigNumber, Contract } from "ethers";
import VENUS_CHAINLINK_ORACLE_ABI from "../abi/venus_chainlink_oracle.json";

import {
  createEnsoCallData,
  createEnsoCallDataRoute,
} from "./IntentCalculations";

import { tokenAddresses, IAddresses, priceOracle } from "./Deployments.test";

import {
  Portfolio,
  Portfolio__factory,
  ProtocolConfig,
  Rebalancing__factory,
  PortfolioFactory,
  PancakeSwapHandler,
  VelvetSafeModule,
  FeeModule,
  FeeModule__factory,
  EnsoHandler,
  EnsoHandlerBundled,
  AccessController__factory,
  TokenExclusionManager__factory,
  TokenBalanceLibrary,
  BorrowManagerVenus,
  DepositBatch,
  DepositManager,
  WithdrawBatch,
  WithdrawManager,
  VenusAssetHandler,
  IAssetHandler,
  IVenusComptroller,
  IVenusPool,
  IResilientOracle,
} from "../../typechain";

import { chainIdToAddresses } from "../../scripts/networkVariables";
import { assert } from "console";

var chai = require("chai");
const axios = require("axios");
const qs = require("qs");
//use default BigNumber
chai.use(require("chai-bignumber")());

describe.only("Tests for Deposit", () => {
  let accounts;
  let iaddress: IAddresses;
  let vaultAddress: string;
  let velvetSafeModule: VelvetSafeModule;
  let portfolio: any;
  let portfolio1: any;
  let portfolioCalculations: any;
  let tokenExclusionManager: any;
  let tokenExclusionManager1: any;
  let borrowManager: BorrowManagerVenus;
  let ensoHandler: EnsoHandler;
  let depositBatch: DepositBatch;
  let depositManager: DepositManager;
  let venusAssetHandler: VenusAssetHandler;
  let withdrawBatch: WithdrawBatch;
  let withdrawManager: WithdrawManager;
  let portfolioContract: Portfolio;
  let comptroller: Contract;
  let portfolioFactory: PortfolioFactory;
  let swapHandler: PancakeSwapHandler;
  let rebalancing: any;
  let rebalancing1: any;
  let tokenBalanceLibrary: TokenBalanceLibrary;
  let protocolConfig: ProtocolConfig;
  let positionWrapper: any;
  let fakePortfolio: Portfolio;
  let txObject;
  let owner: SignerWithAddress;
  let treasury: SignerWithAddress;
  let _assetManagerTreasury: SignerWithAddress;
  let nonOwner: SignerWithAddress;
  let depositor1: SignerWithAddress;
  let addr2: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addrs: SignerWithAddress[];
  let feeModule0: FeeModule;
  const assetManagerHash = ethers.utils.keccak256(
    ethers.utils.toUtf8Bytes("ASSET_MANAGER")
  );

  let swapVerificationLibrary: any;
  let portfolioInfo: any;

  const provider = ethers.provider;
  const chainId: any = process.env.CHAIN_ID;
  const addresses = chainIdToAddresses[chainId];

  function delay(ms: number) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  describe.only("Tests for Deposit", () => {
    before(async () => {
      accounts = await ethers.getSigners();
      [
        owner,
        depositor1,
        nonOwner,
        treasury,
        _assetManagerTreasury,
        addr1,
        addr2,
        ...addrs
      ] = accounts;

      iaddress = await tokenAddresses();

      const EnsoHandler = await ethers.getContractFactory("EnsoHandler");
      ensoHandler = await EnsoHandler.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await ensoHandler.deployed();
      console.log("EnsoHandler deployed to:", ensoHandler.address);

      const DepositBatch = await ethers.getContractFactory("DepositBatch");
      depositBatch = await DepositBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await depositBatch.deployed();
      console.log("DepositBatch deployed to:", depositBatch.address);

      const DepositManager = await ethers.getContractFactory("DepositManager");
      depositManager = await DepositManager.deploy(depositBatch.address);
      await depositManager.deployed();
      console.log("DepositManager deployed to:", depositManager.address);

      const WithdrawBatch = await ethers.getContractFactory("WithdrawBatch");
      withdrawBatch = await WithdrawBatch.deploy(
        "0x38147794ff247e5fc179edbae6c37fff88f68c52"
      );
      await withdrawBatch.deployed();
      console.log("WithdrawBatch deployed to:", withdrawBatch.address);

      const WithdrawManager = await ethers.getContractFactory(
        "WithdrawManager"
      );
      withdrawManager = await WithdrawManager.deploy();
      await withdrawManager.deployed();
      console.log("WithdrawManager deployed to:", withdrawManager.address);

      const SwapVerificationLibrary = await ethers.getContractFactory(
        "SwapVerificationLibraryAlgebra"
      );
      swapVerificationLibrary = await SwapVerificationLibrary.deploy();
      await swapVerificationLibrary.deployed();
      console.log(
        "SwapVerificationLibrary deployed to:",
        swapVerificationLibrary.address
      );

      const TokenBalanceLibrary = await ethers.getContractFactory(
        "TokenBalanceLibrary"
      );

      tokenBalanceLibrary = await TokenBalanceLibrary.deploy();
      tokenBalanceLibrary.deployed();
      console.log(
        "TokenBalanceLibrary deployed to:",
        tokenBalanceLibrary.address
      );

      const PositionWrapper = await ethers.getContractFactory(
        "PositionWrapper"
      );
      const positionWrapperBaseAddress = await PositionWrapper.deploy();
      await positionWrapperBaseAddress.deployed();
      console.log(
        "positionWrapperBaseAddress deployed to:",
        positionWrapperBaseAddress.address
      );

      const ProtocolConfig = await ethers.getContractFactory("ProtocolConfig");

      const _protocolConfig = await upgrades.deployProxy(
        ProtocolConfig,
        [treasury.address, priceOracle.address],
        { kind: "uups" }
      );
      console.log("protocolConfig deployed to:", _protocolConfig.address);

      const chainLinkOracle = "0x1B2103441A0A108daD8848D8F5d790e4D402921F";

      let oracle = new ethers.Contract(
        chainLinkOracle,
        VENUS_CHAINLINK_ORACLE_ABI,
        owner.provider
      );

      let oracleOwner = await oracle.owner();

      await network.provider.request({
        method: "hardhat_impersonateAccount",
        params: [oracleOwner],
      });
      const oracleSigner = await ethers.getSigner(oracleOwner);

      const tx = await oracle.connect(oracleSigner).setTokenConfigs([
        {
          asset: "0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB",
          feed: "0x0567F2323251f0Aab15c8dFb1967E4e8A7D42aeE",
          maxStalePeriod: "31536000",
        },
        {
          asset: "0x1AF3F329e8BE154074D8769D1FFa4eE058B1DBc3",
          feed: "0x132d3C0B1D2cEa0BC552588063bdBb210FDeecfA",
          maxStalePeriod: "31536000",
        },
        {
          asset: "0x7130d2A12B9BCbFAe4f2634d864A1Ee1Ce3Ead9c",
          feed: "0x264990fbd0A4796A3E3d8E37C4d5F87a3aCa5Ebf",
          maxStalePeriod: "31536000",
        },
      ]);
      await tx.wait();

      const PancakeSwapHandler = await ethers.getContractFactory(
        "PancakeSwapHandler"
      );
      swapHandler = await PancakeSwapHandler.deploy();
      await swapHandler.deployed();
      console.log("swapHandler deployed to:", swapHandler.address);

      protocolConfig = ProtocolConfig.attach(_protocolConfig.address);
      await protocolConfig.setCoolDownPeriod("70");
      await protocolConfig.enableSolverHandler(ensoHandler.address);
      await protocolConfig.enableSwapHandler(swapHandler.address);

      const Rebalancing = await ethers.getContractFactory("Rebalancing");
      const rebalancingDefult = await Rebalancing.deploy();
      await rebalancingDefult.deployed();
      console.log("rebalancingDefult deployed to:", rebalancingDefult.address);

      const AssetManagementConfig = await ethers.getContractFactory(
        "AssetManagementConfig"
      );
      const assetManagementConfig = await AssetManagementConfig.deploy();
      await assetManagementConfig.deployed();
      console.log(
        "assetManagementConfig deployed to:",
        assetManagementConfig.address
      );

      const TokenExclusionManager = await ethers.getContractFactory(
        "TokenExclusionManager"
      );
      const tokenExclusionManagerDefault = await TokenExclusionManager.deploy();
      await tokenExclusionManagerDefault.deployed();
      console.log(
        "tokenExclusionManagerDefault deployed to:",
        tokenExclusionManagerDefault.address
      );

      const Portfolio = await ethers.getContractFactory("Portfolio", {
        libraries: {
          TokenBalanceLibrary: tokenBalanceLibrary.address,
        },
      });
      portfolioContract = await Portfolio.deploy();
      await portfolioContract.deployed();
      console.log("portfolioContract deployed to:", portfolioContract.address);

      const VenusAssetHandler = await ethers.getContractFactory(
        "VenusAssetHandler"
      );
      venusAssetHandler = await VenusAssetHandler.deploy();
      await venusAssetHandler.deployed();
      console.log("venusAssetHandler deployed to:", venusAssetHandler.address);

      const BorrowManager = await ethers.getContractFactory(
        "BorrowManagerVenus"
      );
      borrowManager = await BorrowManager.deploy();
      await borrowManager.deployed();
      console.log("borrowManager deployed to:", borrowManager.address);

      await protocolConfig.setAssetHandlers(
        [
          addresses.vBNB_Address,
          addresses.vBTC_Address,
          addresses.vDAI_Address,
          addresses.vUSDT_Address,
          addresses.vLINK_Address,
          addresses.vUSDT_DeFi_Address,
          addresses.corePool_controller,
        ],
        [
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
          venusAssetHandler.address,
        ]
      );

      await protocolConfig.setSupportedControllers([
        addresses.corePool_controller,
      ]);

      await protocolConfig.setSupportedFactory(addresses.thena_factory);

      await protocolConfig.setAssetAndMarketControllers(
        [
          addresses.vBNB_Address,
          addresses.vBTC_Address,
          addresses.vDAI_Address,
          addresses.vUSDT_Address,
          addresses.vLINK_Address,
        ],
        [
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
          addresses.corePool_controller,
        ]
      );

      let whitelistedTokens = [
        iaddress.usdcAddress,
        iaddress.btcAddress,
        iaddress.ethAddress,
        iaddress.wbnbAddress,
        iaddress.usdtAddress,
        iaddress.dogeAddress,
        iaddress.daiAddress,
        iaddress.cakeAddress,
        addresses.LINK_Address,
        addresses.DOT,
        addresses.vBTC_Address,
        addresses.vETH_Address,
      ];

      let whitelist = [owner.address];

      const PositionManager = await ethers.getContractFactory(
        "PositionManagerAlgebra",
        {
          libraries: {
            SwapVerificationLibraryAlgebra: swapVerificationLibrary.address,
          },
        }
      );
      const positionManagerBaseAddress = await PositionManager.deploy();
      await positionManagerBaseAddress.deployed();
      console.log(
        "positionManagerBaseAddress deployed to:",
        positionManagerBaseAddress.address
      );

      const FeeModule = await ethers.getContractFactory("FeeModule");
      const feeModule = await FeeModule.deploy();
      await feeModule.deployed();
      console.log("feeModule deployed to:", feeModule.address);

      const TokenRemovalVault = await ethers.getContractFactory(
        "TokenRemovalVault"
      );
      const tokenRemovalVault = await TokenRemovalVault.deploy();
      await tokenRemovalVault.deployed();
      console.log("tokenRemovalVault deployed to:", tokenRemovalVault.address);

      fakePortfolio = await Portfolio.deploy();
      await fakePortfolio.deployed();
      console.log("fakePortfolio deployed to:", fakePortfolio.address);

      const VelvetSafeModule = await ethers.getContractFactory(
        "VelvetSafeModule"
      );
      velvetSafeModule = await VelvetSafeModule.deploy();
      await velvetSafeModule.deployed();
      console.log("velvetSafeModule deployed to:", velvetSafeModule.address);

      const ExternalPositionStorage = await ethers.getContractFactory(
        "ExternalPositionStorage"
      );
      const externalPositionStorage = await ExternalPositionStorage.deploy();
      await externalPositionStorage.deployed();
      console.log(
        "externalPositionStorage deployed to:",
        externalPositionStorage.address
      );

      const PortfolioFactory = await ethers.getContractFactory(
        "PortfolioFactory"
      );

      const portfolioFactoryInstance = await upgrades.deployProxy(
        PortfolioFactory,
        [
          {
            _basePortfolioAddress: portfolioContract.address,
            _baseTokenExclusionManagerAddress:
              tokenExclusionManagerDefault.address,
            _baseRebalancingAddres: rebalancingDefult.address,
            _baseAssetManagementConfigAddress: assetManagementConfig.address,
            _feeModuleImplementationAddress: feeModule.address,
            _baseTokenRemovalVaultImplementation: tokenRemovalVault.address,
            _baseVelvetGnosisSafeModuleAddress: velvetSafeModule.address,
            _basePositionManager: positionManagerBaseAddress.address,
            _baseExternalPositionStorage: externalPositionStorage.address,
            _baseBorrowManager: borrowManager.address,
            _gnosisSingleton: addresses.gnosisSingleton,
            _gnosisFallbackLibrary: addresses.gnosisFallbackLibrary,
            _gnosisMultisendLibrary: addresses.gnosisMultisendLibrary,
            _gnosisSafeProxyFactory: addresses.gnosisSafeProxyFactory,
            _protocolConfig: protocolConfig.address,
          },
        ],
        { kind: "uups" }
      );

      portfolioFactory = PortfolioFactory.attach(
        portfolioFactoryInstance.address
      );
      console.log("portfolioFactory deployed to:", portfolioFactory.address);

      await withdrawManager.initialize(
        withdrawBatch.address,
        portfolioFactory.address
      );

      console.log("portfolioFactory address:", portfolioFactory.address);
      const portfolioFactoryCreate =
        await portfolioFactory.createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "20",
          _performanceFee: "2500",
          _entryFee: "0",
          _exitFee: "0",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: true,
          _transferableToPublic: true,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });

      const portfolioFactoryCreate2 = await portfolioFactory
        .connect(nonOwner)
        .createPortfolioNonCustodial({
          _name: "PORTFOLIOLY",
          _symbol: "IDX",
          _managementFee: "200",
          _performanceFee: "2500",
          _entryFee: "10",
          _exitFee: "10",
          _initialPortfolioAmount: "100000000000000000000",
          _minPortfolioTokenHoldingAmount: "10000000000000000",
          _assetManagerTreasury: _assetManagerTreasury.address,
          _whitelistedTokens: whitelistedTokens,
          _public: true,
          _transferable: false,
          _transferableToPublic: false,
          _whitelistTokens: false,
          _witelistedProtocolIds: [],
        });
      const portfolioAddress = await portfolioFactory.getPortfolioList(0);
      portfolioInfo = await portfolioFactory.PortfolioInfolList(0);
      console.log("portfolioInfo", portfolioInfo);

      const portfolioAddress1 = await portfolioFactory.getPortfolioList(1);
      const portfolioInfo1 = await portfolioFactory.PortfolioInfolList(1);

      portfolio = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress
      );
      const PortfolioCalculations = await ethers.getContractFactory(
        "PortfolioCalculations",
        {
          libraries: {
            TokenBalanceLibrary: tokenBalanceLibrary.address,
          },
        }
      );
      feeModule0 = FeeModule.attach(await portfolio.feeModule());
      portfolioCalculations = await PortfolioCalculations.deploy();
      await portfolioCalculations.deployed();
      console.log(
        "portfolioCalculations deployed to:",
        portfolioCalculations.address
      );

      portfolio1 = await ethers.getContractAt(
        Portfolio__factory.abi,
        portfolioAddress1
      );
      console.log("portfolio1 deployed to:", portfolio1.address);

      rebalancing = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo.rebalancing
      );
      console.log("rebalancing deployed to:", rebalancing.address);

      rebalancing1 = await ethers.getContractAt(
        Rebalancing__factory.abi,
        portfolioInfo1.rebalancing
      );
      console.log("rebalancing1 deployed to:", rebalancing1.address);

      tokenExclusionManager = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo.tokenExclusionManager
      );
      console.log(
        "tokenExclusionManager deployed to:",
        tokenExclusionManager.address
      );

      tokenExclusionManager1 = await ethers.getContractAt(
        TokenExclusionManager__factory.abi,
        portfolioInfo1.tokenExclusionManager
      );
      console.log(
        "tokenExclusionManager1 deployed to:",
        tokenExclusionManager1.address
      );

      console.log("portfolio deployed to:", portfolio.address);

      console.log("rebalancing:", rebalancing1.address);
      const code = await ethers.provider.getCode(
        "0xE71E810998533fB6F32E198ea11C9Dd3Ffdd2Cac"
      );
      console.log(code !== "0x" ? "This is a contract" : "This is an EOA");
    });

    describe("Deposit Tests", function () {
      it("should init tokens", async () => {
        await portfolio.initToken([
          iaddress.wbnbAddress,
          iaddress.btcAddress,
          iaddress.ethAddress,
          iaddress.dogeAddress,
          iaddress.usdcAddress,
          iaddress.cakeAddress,
        ]);
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "1000000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch
          .connect(nonOwner)
          .multiTokenSwapETHAndTransfer(
            {
              _minMintAmount: 0,
              _depositAmount: "1000000000000000000",
              _target: portfolio.address,
              _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
              _callData: postResponse,
            },
            {
              value: "100000000000000000000",
            }
          );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

      it("should rebalance to lending token vBNB", async () => {
        let tokens = await portfolio.getTokens();
        console.log(tokens);
        let sellToken = tokens[1];
        let buyToken = addresses.vBNB_Address;

        let newTokens = [
          tokens[0],
          buyToken,
          tokens[2],
          tokens[3],
          tokens[4],
          tokens[5],
        ];

        let vault = await portfolio.vault();

        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let balance = BigNumber.from(
          await ERC20.attach(sellToken).balanceOf(vault)
        ).toString();

        let balanceToSwap = BigNumber.from(balance).toString();
        console.log("Balance to rebalance", balanceToSwap);

        const postResponse = await createEnsoCallDataRoute(
          ensoHandler.address,
          ensoHandler.address,
          sellToken,
          buyToken,
          balanceToSwap
        );

        const encodedParameters = ethers.utils.defaultAbiCoder.encode(
          [
            " bytes[][]", // callDataEnso
            "bytes[]", // callDataDecreaseLiquidity
            "bytes[][]", // callDataIncreaseLiquidity
            "address[][]", // increaseLiquidityTarget
            "address[]", // underlyingTokensDecreaseLiquidity
            "address[]", // tokensIn
            "address[]", // tokens
            " uint256[]", // minExpectedOutputAmounts
          ],
          [
            [[postResponse.data.tx.data]],
            [],
            [[]],
            [[]],
            [],
            [sellToken],
            [buyToken],
            [0],
          ]
        );

        await rebalancing.updateTokens({
          _newTokens: newTokens,
          _sellTokens: [sellToken],
          _sellAmounts: [balanceToSwap],
          _handler: ensoHandler.address,
          _callData: encodedParameters,
        });

        console.log(
          "balance after sell",
          await ERC20.attach(sellToken).balanceOf(vault)
        );
        console.log(
          "balance after buy",
          await ERC20.attach(buyToken).balanceOf(vault)
        );
      });

    
      it("should swap tokens for user using native token", async () => {
        let tokens = await portfolio.getTokens();
        console.log(tokens);

        console.log("SupplyBefore", await portfolio.totalSupply());

        let postResponse = [];

        for (let i = 0; i < tokens.length; i++) {
          let response = await createEnsoCallDataRoute(
            depositBatch.address,
            depositBatch.address,
            "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            tokens[i],
            "100000000000000000"
          );
          postResponse.push(response.data.tx.data);
        }

        const data = await depositBatch.connect(nonOwner).multiTokenSwapETHAndTransfer(
          {
            _minMintAmount: 0,
            _depositAmount: "1000000000000000000",
            _target: portfolio.address,
            _depositToken: "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
            _callData: postResponse,
          },
          {
            value: "1000000000000000000",
          }
        );

        console.log("SupplyAfter", await portfolio.totalSupply());
      });

     
      it("should borrow DAI using vBNB as collateral", async () => {
        let ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        let vault = await portfolio.vault();
        let tokens = await portfolio.getTokens();
       

       
        console.log(
          "DAI Balance before",
          await ERC20.attach(addresses.DAI_Address).balanceOf(vault)
        );

        await rebalancing.borrow(
          addresses.vDAI_Address,
          [addresses.vBNB_Address],
          addresses.DAI_Address,
          addresses.corePool_controller,
          "500000000000000000000"
        );
        console.log(
          "DAI Balance after",
          await ERC20.attach(addresses.DAI_Address).balanceOf(vault)
        );

        console.log("newtokens", await portfolio.getTokens());
        const userData = await venusAssetHandler.getUserAccountData(
          vault,
          addresses.corePool_controller,
          tokens
        );
        const totalCollateral = userData[0].totalCollateral;
        const liqThreshold = userData[0].ltv;
        const totalDebt = userData[0].totalDebt;
        console.log("totalCollateral", totalCollateral);
        console.log("liqThreshold", liqThreshold);
        console.log("totalDebt", totalDebt);

        const currentthreshold = BigNumber.from(totalDebt).mul(100).div(totalCollateral);
        console.log("currentthreshold", currentthreshold);
        const vBNBBalanceBefore = await ERC20.attach(addresses.vBNB_Address).balanceOf(
          nonOwner.address
         );
         console.log("vBNBBalanceBefore", vBNBBalanceBefore);
      });

      it("malicious user can liquidate vault", async () => {
        await ethers.provider.send("evm_increaseTime", [70]);

        const supplyBefore = await portfolio.totalSupply();
        const amountPortfolioToken = await portfolio.balanceOf(nonOwner.address);
        const amountPortToken33Percent = BigNumber.from(amountPortfolioToken).mul(36).div(100);

        const ERC20 = await ethers.getContractFactory("ERC20Upgradeable");
        const tokens = await portfolio.getTokens();
        
        let vault = await portfolio.vault();

        const vBNBBalanceBefore = await ERC20.attach(addresses.vBNB_Address).balanceOf(
          vault
        );
        console.log("vBNBBalanceBefore", vBNBBalanceBefore);
       
        const zeroAddress = ethers.constants.AddressZero;

         await portfolio
          .connect(nonOwner)
          .multiTokenWithdrawal(
            BigNumber.from(amountPortToken33Percent),
            {
              _factory: addresses.thena_factory,
              _token0: zeroAddress, //USDT - Pool token
              _token1: zeroAddress, //USDC - Pool token
              _flashLoanToken: zeroAddress, //Token to take flashlaon
              _bufferUnit: "0",
              _solverHandler: ensoHandler.address, //Handler to swap
              _flashLoanAmount: [0],
              firstSwapData: ["0x"],
              secondSwapData: ["0x"],
              isDexRepayment: false,
              _poolFees: [0, 0, 0],
              _swapHandler: swapHandler.address,
            }
          );
          

        const userData = await venusAssetHandler.getUserAccountData(
          vault,
          addresses.corePool_controller,
          tokens
        );
        const totalCollateral = userData[0].totalCollateral;
        const liqThreshold = userData[0].ltv;
        const totalDebt = userData[0].totalDebt;
        console.log("totalCollateral", totalCollateral);
        console.log("liqThreshold", liqThreshold);
        console.log("totalDebt", totalDebt);

        const currentthreshold = BigNumber.from(totalDebt).mul(10000).div(totalCollateral);
        console.log("currentthreshold", currentthreshold);
        const vBNBBalanceAfter = await ERC20.attach(addresses.vBNB_Address).balanceOf(
         vault
        );
        console.log("vBNBBalanceAfter", vBNBBalanceAfter);

        const comptroller: IVenusComptroller = await ethers.getContractAt(
          "IVenusComptroller",
          addresses.corePool_controller
        );
       
       const [error, liquidity, shortfall] = await comptroller.getAccountLiquidity(vault);
        console.log("shortfall", shortfall.toString());
        console.log("liquidity", liquidity.toString());

        //c once vault is in shortfall which can easily happen in volatile markets, malicious user will liquidate the vault
      });
```

Looking at the logs from the test, it shows the following:

vBNBBalanceBefore BigNumber { value: "4838700267" }
totalCollateral BigNumber { value: "59005325596" }
liqThreshold BigNumber { value: "7800" }
totalDebt BigNumber { value: "30002672700" }
currentthreshold BigNumber { value: "7734" }
vBNBBalanceAfter BigNumber { value: "4357892025" }
shortfall 0

Notice how although the shortfall is 0, the current threshold is now dangerously close to the liquidation threshold and can tip over at any moment which makes the attack very viable. A better representation of this POC would be to create a mock oracle and show how the liquidation would work but I feel that is not necessary as the explanation is intuitive.

Recommendation
The protocol should implement witdrawal safety checks to ensure the vault’s solvency and health factor remains safe post-withdrawal.


# 12 Users cannot unpause protocol

Summary
Due to how setProtocolPause is restricted by onlyProtocolOwner and the forced false assignment to _unpauseProtocol for non-owners, users can never unpause the protocol even after 4 weeks, contradicting the intended functionality.

Finding Description
SystemSettings::setEmergencyPause allows protocol owner to set the emergency pause state of the protocol. It also allows users to unpause the protocol after a 4 week period has passed. See below:

```solidity
/**
   * @notice Sets the protocol pause state.
   * @param _paused The new pause state.
   */
  function setProtocolPause(bool _paused) public onlyProtocolOwner {
    if (isProtocolEmergencyPaused && !_paused)
      revert ErrorLibrary.ProtocolEmergencyPaused();
    isProtocolPaused = _paused;
    emit ProtocolPaused(_paused);
  }

  /**
   * @notice Allows the protocol owner to set the emergency pause state of the protocol.
   * @param _state Boolean parameter to set the pause (true) or unpause (false) state of the protocol.
   * @param _unpauseProtocol Boolean parameter to determine if the protocol should be unpaused.
   * @dev This function can be called by the protocol owner at any time, or by any user if the protocol has been
   *      paused for at least 4 weeks. Users can only unpause the protocol and are restricted from pausing it.
   *      The function includes a 5-minute cooldown between unpauses to prevent rapid toggling.
   * @dev Emits a state change to the emergency pause status of the protocol.
   */
  function setEmergencyPause(
    bool _state,
    bool _unpauseProtocol
  ) external virtual {
    bool callerIsOwner = _owner() == msg.sender;
    require(
      callerIsOwner ||
        (isProtocolEmergencyPaused &&
          block.timestamp - lastEmergencyPaused >= 4 weeks),
      "Unauthorized"
    );

    if (!callerIsOwner) {
      lastUnpausedByUser = block.timestamp;
      _unpauseProtocol = false;
    }
    if (_state) {
      if (block.timestamp - lastUnpausedByUser < 5 minutes)
        revert ErrorLibrary.TimeSinceLastUnpauseNotElapsed();
      lastEmergencyPaused = block.timestamp;
      setProtocolPause(true);
    }
    isProtocolEmergencyPaused = _state;

    if (!_state && _unpauseProtocol) {
      setProtocolPause(false);
    }
  }
```

Users can cannot call SystemSettings::setemergencyPause to unpause the protocol for 2 reasons. The first is that the setProtocolPause function has an onlyProtocolOwner modifier which means only the owner of the contract can call the function. Also, in SystemSettings::setemergencyPause, the if block that contains logic checks if _state is false && _unpauseprotocol is true which can never be the case for a user who is not the owner as _unpausepProtocol is forced to false in the if block. As a result, the if condition is never triggered which stops users from being able to unpause the protocol after the 4 week buffer has passed.

Impact Explanation
This flaw prevents the protocol from being unpaused by users after the 4-week buffer. If the owner is unresponsive or unavailable, the protocol remains paused indefinitely, potentially disrupting services and user funds.

Proof of Concept
This test was run in test/Arbitrum/1_IndexConfig.test.ts.

```javascript
it("users cannot unpause protocol via setEmergrncyPause", async () => {
        //c for testing purposes
        //c successfully pause the protocol
        await protocolConfig.setEmergencyPause(true, true);

        let protocolState = await protocolConfig.isProtocolPaused();
        expect(protocolState).to.be.true;

        //c wait 4 weeks
        await ethers.provider.send("evm_increaseTime", [2419200]);

        //c non owner calls function with _state as false
        await protocolConfig.connect(addr1).setEmergencyPause(false, true);

        /*c protocol is not unpaused as expected because
         if (!_state && _unpauseProtocol) {
            
            setProtocolPause(false);
        }
            is in setemergencyPause function and since _unpause is false, this if block actually never runs and the below assert fails
        */
        let protocolState1 = await protocolConfig.isProtocolPaused();
        expect(protocolState1).to.be.false;
      });
```
Recommendation
Allow non-owners to actually unpause by removing the forced _unpauseProtocol = false or introducing a separate logic path for non-owner unpausing. Relax onlyProtocolOwner or provide a dedicated internal call that checks if 4 weeks have elapsed, enabling non-owners to unpause.