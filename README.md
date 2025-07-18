Audit Analysis Overview
# STEP 1 - FIRST PASS AND ASSUMPTION ANALYSIS

# STEP 2 - ZOOM OUT ANALYSIS (FROM VELVET V4 AUDIT NOTES)
This section will introduce a step further than assumption analysis. So far, we have a first pass and assumption analysis. Now I will be introducing the zoom out methodology which must come immediately after assumption analysis. The fact that you didnt do this, led to missing this bug which was very easily spotted. I will give some background on the bug first before explaining the zoom out methodology:

The vault computes a global unusedCollateralPercentage for each controller (such as Aave) based on the total collateral and total debt across all tokens. This percentage is calculated using:

TokenBalanceLibrary::getControllersData
```solidity
      uint256 unusedCollateralPercentage;
      if (accountData.totalCollateral == 0) {
        unusedCollateralPercentage = 1e18; // 100% unused if no collateral
      } else {
        unusedCollateralPercentage =
          ((accountData.totalCollateral - accountData.totalDebt) * 1e18) /
          accountData.totalCollateral;
      }
```
This computed percentage is then uniformly applied to the balance of every token marked as borrowable, irrespective of whether each individual token has been borrowed against or how much it has been borrowed.

TokenBalanceLibrary::_getAdjustedTokenBalance
```solidity
  function _getAdjustedTokenBalance(
    address _token,
    address _vault,
    IProtocolConfig _protocolConfig,
    ControllerData[] memory controllersData
  ) public view returns (uint256 tokenBalance) {
    if (_token == address(0) || _vault == address(0))
      revert ErrorLibrary.InvalidAddress(); // Ensures neither the token nor the vault address is zero.
 _protocolConfig.isBorrowableToken(_token));
    if (_protocolConfig.isBorrowableToken(_token)) {
      address controller = _protocolConfig.marketControllers(_token);
controller);
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
Thus, even if a specific token has not been borrowed or has been borrowed at a different proportion, it still incorrectly receives the same global collateral usage percentage.

You can view the full finding at https://cantina.xyz/code/8cf9c7a0-a7a6-446a-8577-1e2c254eb5a8/findings?severity=medium,high&q=unused+collateral&finding=465

This bug seems very easy to spot so why did you not spot it? The reason is because you didnt apply the zoom out methodology. A lot of times, you tend to zoom into different functions as that is the point of a first pass and assumption analysis. You are going through every function in a user path, assumption analyzing each line and going as deep as you can which is great and what you are supposed to do. 

The problem is that the job isnt done there though and this is where the zoom out methodology becomes important. After doing a first pass and assumption analysis, you need to take the function or the path that you have just analysed and see where else this function is used in the codebase. Just because a path/function seems happy and everything works fine, when you see how that path is applied to other parts of the codebase, it may introduce bugs and this is the highest level of bugs you can find. In this zoom out phase is where you will find a lot of bugs.

Lets look at how zooming out would've helped us spot this bug. I initially looked at the whole flow of TokenBalanceLibrary::getControllersData and assessed the unusedCollateralPercentage calculation with numeric examples and all but the problem was that after doing the first pass, when i saw the unusedCollateralPercentage used in TokenBalanceLibrary::_getAdjustedTokenBalance, i assumed that since the calculation was correct for the percentage, every where it would be used, would also be correct which obviously was the same assumption the devs had and as you can see from this finding, that is definitely not the case.

If you used zoom out analysis, after assessing the unusedCollateralPercentage calculation, you would have them zoomed out and asked, if TokenBalanceLibrary::_getAdjustedTokenBalance happens, where unusedCollateralPercentage is used, will it still work as intended? This is the idea of zooming out. So you look at a function with a first pass or/and assumption analysis, , you then use the brain trigger "if x happens, does this function/variable still work as intended?". Ypou know your brain works very well with brain triggers. so this brain trigger is what fires the zoom out analysis. This is what zooming out means. With this brain trigger, thinking about what x could be will help you discover bugs and x can literally be anything. Dont limit your thinking to what is in the codebase. Also think about x being anything that could happen with any external integrations that the codebase uses. x can even be anything that can happen on the chain being deployed on. This is how far you can zoom out. This is the core of the zoom out methodology. Use this brain trigger to figure out all the things that x could possibly be and see how many more bugs you will find.

To break this down, what I mean is that if i used zoom out analysis, I would have zoomed out of the unusedCollateralPercentage calculation to search for where it was used in the codebase and I wouldve found TokenBalanceLibrary::_getAdjustedTokenBalance. Then i wouldve asked if unusedCollateralPercentage was used correctly here. On asking this question, I wouldve realised that the answer was no because as the finding details, This computed percentage is then uniformly applied to the balance of every token marked as borrowable, irrespective of whether each individual token has been borrowed against or how much it has been borrowed. This is how I wouldve found this bug. 

This zoom out methodology would also have been extremely useful for another finding I missed which will probably turn out to be a rare issue. Lets have a look at it:

When a Portfolio undergoes full liquidation of one of its assets, that asset cannot be removed through the removePortfolioToken() function, because it checks for the token balance:

```solidity
function removePortfolioToken(
    address _token
  ) external onlyAssetManager nonReentrant protocolNotPaused {
    address[] memory currentTokens = _getCurrentTokens();
    //@audit it is a portfolio token but the full liquidated
    if (!_isPortfolioToken(_token, currentTokens))
      revert ErrorLibrary.NotPortfolioToken();
		...
		
    uint256 tokenBalance = _getTokenBalanceOf(_token, _vault);
    //@audit After going through a full liquidation the balance is 0 
    _tokenRemoval(_token, tokenBalance);
  }
  
  function _tokenRemoval(address _token, uint256 _tokenBalance) internal {
		//@audit will revert here
    if (_tokenBalance == 0) revert ErrorLibrary.BalanceOfVaultIsZero();
	  ...
	}
  ```
Having a token with zero balance as a portfolio token prevents any deposits from occurring, temporarily DOSing deposits. The only recovery method is to donate tokens to restore the balance before removing the token.

Users can still withdraw from the portfolio through emergency withdrawal.

With this finding, zoom out methodology would have easily shown you that when a vault is liquidated, the balance becomes 0, you looked at the _tokenRemoval function which in isolation, when you zoomed in with the first pass and the assumption analysis, looked good but if you zoomed out and asked where is this function used ? You would have seen that it can be used to remove tokens that have been liquidated and if the tokens are liquidated, the balance is 0 and _tokenRemoval doesnt allow 0 token removals as the code above shows and that would have been another easy vulnverabilty. So to summarise, just because a function works as intended, doesnt mean it cannot cause a vulnerability. You need to zoom out and question where the function/path/variable is used in the codebase and what the implications of this are.

Once zoom out analysis is used for a function, the next thing is function state change analysis which is a concept I introduced in RAACV2AuditNotes so check that out.

# STEP 3 - FUNCTION STATE CHANGE ANALYSIS
