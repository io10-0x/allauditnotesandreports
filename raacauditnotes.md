# 1 LIQUIDITY INDEX AND INTEREST RATES, COMPOUNDING VS LINEAR

A **liquidity index** is a measure used in DeFi lending protocols to track the **cumulative interest accrued** over time for a given asset in a lending pool. It helps lenders and borrowers determine how much interest has been earned or owed since their deposit or loan was initiated.

---

**1️⃣ How Does the Liquidity Index Work?**

The **liquidity index** is a **cumulative multiplier** that represents the growth of an asset's value in a lending protocol **due to interest accrual**.

- It starts at **1e27 (or another base value)** when the protocol is initialized.
- It increases over time as interest accrues on deposits and loans.
- It updates every time **interest is compounded** (usually when a new deposit, withdrawal, or borrow action occurs).
- It ensures that all depositors and borrowers get the correct interest applied **based on when they interacted with the pool**.

---

**2️⃣ Liquidity Index Formula**

The formula typically used to calculate liquidity index was used in RAAC in the ReserveLibrary.sol contract and I will display it wit the comments I left on it so you can see how it works:

```solidity
   function calculateLinearInterest(
        uint256 rate,
        uint256 timeDelta,
        uint256 lastIndex
    ) internal pure returns (uint256) {
        uint256 cumulatedInterest = rate * timeDelta;
        //c so the interest rate(IN RAY) is multiplied by how much time has passed since the last update to get how much interest has been accumulated. so this value is the interest rate per second. Lets see an example to drill this home. so if interest rate is 20%, this will be 20% of 1e27 which is 5e25. So assume timedelta is is 100 seconds, then the interest accumulated will be 5e25 * 100 = 5e27. so total interest per second is 5e27 or 5% (5e27/1e27).
        cumulatedInterest = cumulatedInterest / SECONDS_PER_YEAR;
        //c then the interest rate per second is divided by the number of seconds in a year to get the interest rate per year. continuing from the above example, 5e27/31536000 = 1.58e21 or 0.00000158% annual interest rate
        return WadRayMath.RAY + cumulatedInterest;
        //c then the interest rate per year is added to 1e27 to make the cummulative interest have ray precision. continuing from the above example, 1e27 + 1.58e21 = 1.00000158e27 which is still 0.00000158% when you divide by 1e27 which is what happens in calculateliquidityindex function where rayMul is called. This will be the new cummulated interest from this update. To get the new liquidity index, this value will be multiplied by the old liquidity index. This is because the liquidity index is a cummulative value so it is the old value * the new interest rate.
    }

    function calculateLiquidityIndex(
        uint256 rate,
        uint256 timeDelta,
        uint256 lastIndex
    ) internal pure returns (uint128) {
        uint256 cumulatedInterest = calculateLinearInterest(
            rate,
            timeDelta,
            lastIndex
        );
        return cumulatedInterest.rayMul(lastIndex).toUint128();
        //c so the cummulated interest is multiplied by the old liquidity index to get the new liquidity index. This is because The liquidity index stores a historical record of accrued interest, and by multiplying it with cumulatedInterest, we ensure correct compounding over time. The reason why we multiplied instead of adding is detailed in notes.md

        //q possible unsafe casting ??? not sure as rayMul always returns 27 decimals so I doubt that would be the case
    }


```

Why is Multiplication Necessary?
If we only added the interest rate to the last index without multiplying, the interest would not compound. Multiplying ensures that past accrued interest also earns interest in future periods.

This follows the compound interest formula:

Final Amount=Principal×(1+r)^t
where:

Principal = lastIndex
r = rate
t = timeDelta / SECONDS_PER_YEAR.

So the liquidity index compounds but the interest is calculated linearly. To understand this, you need to understand the difference between compounding and linear increases. See below:

Linear Interest (Addition)
In linear interest, the interest is added at regular intervals without applying it to the previously accumulated amount.
This means that only the principal accrues interest over time.
Formula:

Total Amount=Principal times (1 + r)^t

r = interest rate per period
t = number of periods
Principal = initial amount

Example of Linear Interest
Suppose you deposit $1000 at a 5% annual interest rate, and you calculate interest linearly:

After 1 year:

1000+(1000×0.05×1)=1050

After 2 years:
1000+(1000×0.05×2)=1100
After 10 years:
1000+(1000×0.05×10)=1500

Key Insight: The interest is added linearly to the original amount, so it grows in a straight line.

Compound Interest (Multiplication)
In compound interest, interest is applied to both the principal and the previously accumulated interest.
This means each period’s interest is calculated on the updated total.
Formula:

Total Amount=Principal×(1+r)^t

Example of Compound Interest
Using the same $1000 at a 5% annual rate, but compounded each year:

After 1 year:
1000×(1.05)=1050

After 2 years:
1000×(1.05)^2 =1102.5

After 10 years:
1000×(1.05)^10 =1628.89

Key Insight: Instead of growing linearly, the value multiplies each period, leading to exponential growth.

Notice how r which is interest rate per period is different when calculating linearly and when compounding even though the interest rate is the same at 5%. When getting a linear calculation, the interest rate is only considered on the principal amount which is what was said above. So what it means is that only the principal gains interest which is why in the linear calculation, after 2 years, the interest is (0.05(5%) times 2) times 1000. So the interest is only on the original 1000 invested.

In compounding, if i have 1000 principal and it becomes 1050 after a year, in the second year, i am making 5% on my 1050 and not the original 1000 which is the difference between linear and compounding. So r when compounding takes into account the interest gained from the previous year which is why r is 1.05 and not 0.05 when compared to linear calculations.

So we compound the liquidity index by multiplying the previous index by the latest index which is what the idea of compounding is supposed to do. So keep this in mind. With linear calculation, we are always adding but when compounding, we multiply. This is the major take home point.

You might be asking why is interest is calculated linearly but the index gets compounded? This is because the protocol applies an interest rate that grows over time based on a fixed rate per second.

Interest is calculated using:
Interest Accrued=Rate×Time

This gives us a simple way to determine how much interest has accumulated since the last update.Linear interest calculation is used temporarily to find how much interest should be applied in this period. Compounding happens via the index, so the total value of deposits grows exponentially over multiple time periods.

This formula means that **the liquidity index increases as time passes**, based on the protocol’s interest rate.

---

**3️⃣ Example of How Liquidity Index Works**

**Step 1: Initial Conditions**

- The **liquidity index starts at 1e27**.
- A user deposits **100 DAI** into the pool.
- The **interest rate is 5% per year**.

  **Step 2: After 1 Year**

- Interest accumulates over time.
- The **liquidity index increases** based on the 5% rate as seen above.
- Suppose it increases to **1.05e27**.

Now, the user's **new balance** is:

100 times 1.05e27/1e27 = 105 DAI

**The liquidity index ensures that all users earn interest proportionally based on when they entered the market.**

---

**4️⃣ Use Cases of Liquidity Index**

1. **Interest Calculation:**

   - The **liquidity index** allows users to calculate accrued interest without storing individual interest amounts for each deposit.

2. **Accurate Accounting for Depositors & Borrowers:**

   - Users who deposit at **different times** will see their balances grow correctly based on when they entered.

3. **Reducing Storage Costs:**
   - Instead of tracking interest for every user separately, the liquidity index lets **each user compute their earnings using a single global multiplier**.

-

# 2 TAYLOR SERIES EXPANSION, FACTORIALS, MORE ON COMPOUNDING

In reservelibrary::calculateCompoundedInterest, there is the following line:

```solidity
 // Will use a taylor series expansion (7 terms)
 return WadRayMath.rayExp(exponent);
```

It says the rayExp function uses a taylor series expansion. A taylor expansion is a method of compounding interest. This is how pure compounding is done in solidity and if you look at the rayExp function in WayRadMath.sol, you will see how the function works. I will give a high level off how the taylor expansion compounds values. It is not so dissimilar to how general compounding works which i covered earlier in an example.

**Overview of Taylor Series Expansion**

The **Taylor series expansion** is a mathematical tool used to approximate functions as an infinite sum of terms calculated from their derivatives at a single point. It is widely used in numerical analysis, physics, and engineering to approximate complex functions.

For a function f(x), the Taylor series expansion around x = 0 (also called the **Maclaurin series**) is:

f(x) = (f(0) + f'(0)x) + (f''(0))(x^2)/2! + (f'''(0))(x^3)/3! + ...

Let me explain somethings before we go on. You can see the formula starts with f(0) and then f'(0) and the ' keeps increasing. What this means is f'(0) is the rate of change of f(0) and f''(0) is the rate of change of f'(0) and so on.

Also you can see that each time, we divide by 2! and then 3! and so on. 2! means 2x1 , 3! is 3x2x1, 4! is 4x3x2x1 and so on. This is known as a factorial.

This allows us to approximate functions like **exponential, sine, cosine, logarithm, etc.**, without having to compute them directly.

---

**Example: Approximating e^x with Taylor Series**

The exponential function e^x can be expanded using the Taylor series:

e^x = (1 + x) + x^2/2! + x^3/3! + x^4/4! + x^5/5! + ....

This formula is **especially useful for computing compounding growth**, as seen in finance and blockchain applications. So in the above case, f(x) is e^x.

We can specify how many terms in the taylor series we want to use. What I mean by terms is simply how many terms are in the formula. So if you literally count how many terms are in this formula, you will see there are 5 terms.

---

**Numerical Example: Approximating \( e^1 \)**

Let’s approximate e^1 (which is actually (approx 2.71828 )) using only the first **five** terms of the Taylor series:

e^x = (1 + 1) + 1^2/2! + 1^3/3! + 1^4/4! + 1^5/5! + ....

**Approximation:**

e^1 approx = 1 + 1 + 0.5 + 0.1667 + 0.0417 + 0.0083 = 2.7186. So the taylor series based on a 5 term expansion will give us the exponential of 1 as 2.7186

**Actual value of ( e^1 ) = 2.71828** when you actually use a calculator to get the value
✅ The approximation is **very close**, even with just a few terms using the taylor series! This is how powerful the taylor series is and how beneficial it is for computations.

---

**Applying Taylor Series to Blockchain (Compounded Interest)**

In **smart contracts**, particularly in DeFi applications, we use **Taylor expansion** to approximate exponential functions because Solidity lacks native floating-point support.

For **compounded interest**, we use:

e^rt = (1 + rt) + rt^2/2! + rt^3/3! + rt^4/4! + rt^5/5! + ....

Where:

- r = interest rate per second
- t = time elapsed

By using **Taylor series**, we can **efficiently approximate compounding** without needing complex exponentiation.

This is what RAAC uses to get the value of compounded interest.

# 3 NORMALIZED DEBT

This is another important concept in lending and borrowing protocols you need to understand. It is crucial to how protocols update debt in their protocols dynamically without having to update a user's debt each time it accrues interest.

Understanding Normalized Debt in Aave & DeFi Protocols

**1. What is Normalized Debt?**
Normalized debt is a way to represent a loan amount **without needing to update it every block** as interest accrues. Instead of storing the raw debt value, protocols like Aave store **normalized debt**, which **automatically scales** with the liquidity index.

The formula is:

Normalized Debt = Actual Debt at Borrowing Time/Liquidity Index at Borrowing Time

or rearranged:

Actual Debt = Normalized Debt times Current Liquidity Index

---

**2. Example: Borrowing with a Liquidity Index of 1.5**

- Suppose you borrow **1,000 DAI** when the **liquidity index is 1.5**.
- Instead of storing 1,000 DAI as debt, the protocol stores:

Normalized Debt = 1,000/1.5 = 666.67

- The protocol **doesn’t need to update this value over time**.
- Instead, the **actual debt is always computed dynamically** using the **current liquidity index**.

---

**3. What Happens Over Time?**
Let's say the **liquidity index increases** over time due to interest accrual:

| **Time** | **Liquidity Index** | **Normalized Debt** | **Actual Debt Owed** |
| -------- | ------------------- | ------------------- | -------------------- |
| Time     | **1.5**             | **666.67**          | **1,000 DAI**        |
| Year 1   | **1.8**             | **666.67**          | **1,200 DAI**        |
| Year 2   | **2.0**             | **666.67**          | **1,333.33 DAI**     |
| Year 3   | **2.5**             | **666.67**          | **1,666.67 DAI**     |

- **Notice that the normalized debt stays the same (666.67)**
- But the **actual debt grows** as the **liquidity index increases**.

---

**4. Why is This Done?**
**Simplifies Accounting**: Instead of updating everyone's debt each block, the protocol just updates the **liquidity index**.  
 **Instant Interest Accrual**: Borrowers' debt increases automatically as the liquidity index grows.  
**Gas Efficiency**: No need for storage updates on every block—reduces gas costs significantly.

---

**5. Key Takeaway**
The debt **doesn't go down**. Instead, the **normalized debt stays constant**, while the **actual debt increases over time** as the liquidity index grows.

# 4 LIQUIDATION CRUCIAL CHECKS THAT AID ASSUMPTION ANALYSIS

This is the first protocol I am auditing that I am looking into liquidation logic so it is a good time to go over some common liquidation bugs that can help you uncover exploits not just in RAAC but in any protocol logic. These are by no means all of them but just some to keep in mind.

- LIQUIDATION FUNCTION REVERTING

This is the most general but arguably most important check on any liquidation function. You need to make sure that there is no way that an attacker can manipulate the function in a way where it reverts when it is called for that address or just in general. There are many ways this can happen including with blacklisted tokens, logic in the function causing reverts that a user could capitalize on, etc.

If the liquidation can revert for any reason at all, it is something you should take a closer look at because if there is a way for a user to not be liquidated, they can borrow say $100 worth of USDC by depositing $150 worth of ETH. If the ETH price drops by half, I still have my 100 USDC but the ETH i deposited is now worth $75 so that means this position is undercollateralized by a lot. In fact, it is insolvent.

Insolvency is the idea that the user's collateral cannot pay off their debt. As a result, the protocol is left with bad debt that they cannot account for. They essentially lost money. Health factors and LTV's are used to prevent insolvency from ever happening but if a user can find a way to not get liquidated via an error, they can create insolvent positions.

This idea of insolvency leads to the next liquidation exploit that is important to note:

- INSOLVENT POSITIONS CANNOT BE LIQUIDATED

So lets carry on the example from above, say the user found a way to make their position insolvent, some protocols, do not have any functionality to liquidate insolvent positions. Most liquidate functions check that the collateral is enough to cover the debt but they dont usually handle the case where the position is insolvent. If it is insolvent, the protocol will still want to be able to take the user collateral to reduce the amount of bad debt they have incurred. If they cannot liquidate the position , the collateral ends up stuck in the contract which isnt good for anyone.

- LIQUIDATIONS SHOULD TAKE FEES INTO ACCOUNT

Usually, when a user deposits collateral into a contract, a fee is taken out of that collateral for many different reasons most especially as fees are usually the way these protocols make money. So when liquidating a user, the check is usually to make sure that their collateral is worth more than their debt or at least a percentage of their collateral is worth more than their debt (LTV, health factor, etc come into play here). What a lot of protocols don't do is that they dont take into account the fees they took from the user when calculating their liquidation amount.

For example, i deposit 0.05 ETH currently worth $100 as collateral, $5 worth of fees in ETH is taken from that but when checking if i should be liquidated, the protocol compares my debt to whatever my 0.05 ETH i deposited is now worth. In reality, it should compare my debt to 0.05 ETH less the fees that they took as that is the actual collateral value. This is a sneaky one but happens often.

- PARTIAL LIQUIDATION LEAVES USER IN A WORSE OFF POSITION THAN BEFORE (HEALTH FACTOR WEIGHTED AVERAGE)

This is a bit more complex but not by much. Lets take a look at it with an example. In this example, a protocol allows users to add different tokens as collateral and borrow against them. Different tokens have different LTV's. As you know, LTV is simply thr percentage of their collateral they can borrow. Assume that the health factor is set to 1 so if your health factor is less than 1, you are getting liquidated.

A user has:

- $1000 USDT in liabilities
- 6 ETH each worth $100 for 100% LTV ($600 borrowing power)
- 1 BTC each worth $400 for 50% LTV ($200 borrowing power)
- Health Factor: (600+200)/ 1000 = 0.8
  Health Factor is calculated by dividing the borrowing power by the amount of debt. So at this point, the user's health factor is below 1 so they are going to get partially liquidated for this example just to get their health factor back up to avoid full liquidation.

Usually, when protocols have many assets as collateral and they want to decide which to liquidate first, they go for the largest position the user has which is ETH in this user's case. So we want to take out half of this user's ETH and use it to pay off half of their debt. So:

- 50% of 1000 USDT = 500
- Proportionate supply = 500 USDT/ $100 (ETH Price) = 5 ETH
- 500 USDT in liabilities
- 1 ETH left for user after partial liquidation ($100 borrowing power)
- 1 BTC ($200 borrowing power)
- New Health Factor: (100+200)/ 500 = 0.6

As you can see, the health factor of the user is now lower than it was before the liquidation. In this position, the user will be liquidated again lol. This is a mistake a lot of protocols make when they have many different assets they accept as collateral. The solution is for the protocol to simulate the health factor post liquidation and carefully select the asset to liquidate which will ultimately increase the health factor or simply start with the asset with the lowest LTV as the math works out that the health factor increases when liquidating this way which is expected behavior. This is a key thing you need to watch for when looking at liquidations.

- USERS BEING ABLE TO INTENTIONALLY LIQUIDATE THEMSELVES

There have been many hacks that have to do with this. You need to make sure that there is no way that a user could game the protocol into intentionally reducing their own health factor and then liquidating themselves. A lot of protocols allow users to liquidate other users by paying off their debt and getting a reward for doing it which is usually a fee of some sort. If a user can find a way to get their health factor below 1 for example, they can liquidate themselves. You might be wondering, well why tf would I wanna bring myself to liquidation? n many lending protocols, liquidators are incentivized with a liquidation bonus (e.g., 5%-10%) for repaying bad debt. Normally, an attacker would not want to be liquidated, but if the liquidation bonus is high enough, a user could game the system. Here's how:

1️⃣ User Deposits & Borrows
Alice deposits 100 ETH as collateral.
Alice borrows 50 ETH worth of DAI (e.g., 50,000 DAI).
Health Factor (HF) = 1.5, so she's safe from liquidation.
2️⃣ Alice Uses a Flash Loan
Alice borrows 100,000 DAI via a flash loan.
She swaps all 100,000 DAI for ETH, temporarily pumping ETH’s price.
3️⃣ Artificially Reducing HF
With a higher ETH price, Alice borrows more DAI against her inflated ETH collateral.
The extra borrowed DAI increases her debt, reducing her health factor.
4️⃣ Triggering Self-Liquidation
Right after borrowing more DAI, the ETH price returns to normal (or is dumped by Alice).
This restores ETH to its true price, making Alice’s new debt unsustainable.
Her Health Factor drops below 1, making her eligible for liquidation.
5️⃣ Alice Liquidates Herself for Profit
Since liquidators receive a 10% bonus, Alice liquidates her own position.
She pays off her own debt and claims the liquidation reward.
If the liquidation bonus is greater than the flash loan fee, Alice profits.
6️⃣ Flash Loan Repayment
Alice repays the flash loan.
She keeps the excess liquidation bonus as profit.

So you need to scour the contract for any way a user could potentially liquidate themselves.

- INCENTIVES FOR LIQUIDATION (LIQUIDATION FEES)

As the point above stated, people are only motivated to liquidate other people because of the liquidation fee they gain from doing it. Otherwise, why would anyone want to liquidate another person. There is no incentive. You need to make sure that this incentive is in the protocol AND it is prioritized and by prioritized, I mean it should be one of the first things paid out in the liquidate function. If it isnt, then whatever accounting that may happen before the fee is paid may cause the liquidation fee to reduce and this can definitely be a problem so you MUST be on the lookout for things like this.

With this list of potential vulnerabilities, you can use this as a base to find even more exploits that can happen with liquidations but this will be good to get you warmed up for assumption analysis.

# 5 DELETING A MAPPING IN A STRUCT EXPLOIT

This is just a quick note about trying to delete a mapping from a struct. If you ever see any code that attempts to delete a map in a struct , it wont work. The value in the mapping will still be there so keep that in mind.

# 6 UTILIZATION RATE IN DEFI PROTOCOLS

The key thing to note about this is that when calculating the utilization rate, know that interest on debt and interest on liquidity can be taken into account but also it is simpler for protocols to not take it into consideration. Let's look at what utilization rate even is and the explanation of this point is made below. We will look at the comparison between the 2 methods and when one is better to be used over another.

What is Utilization Rate?
The utilization rate is a metric that describes the proportion of funds in a lending pool that are currently being borrowed. It is calculated as:

Utilization Rate= Total Borrowed/Total Available Liquidity+Total Borrowed

Total Borrowed: The total amount of funds currently borrowed from the pool.

Total Available Liquidity: The total amount of funds available for borrowing in the pool. The utilization rate is expressed as a percentage (e.g., 50% utilization means half of the funds in the pool are borrowed).

Does Utilization Rate Depend on Interest?
the answer to this is dependent on the chosen model being applied, the utilization rate is independent of interest in the simplified version. It is purely a function of:
The amount of funds borrowed and the amount of funds available in the pool.Interest rates, on the other hand, are often determined based on the utilization rate.

For example:
When the utilization rate is high, interest rates may increase to incentivize repayments and discourage further borrowing.When the utilization rate is low, interest rates may decrease to incentivize borrowing.However, the utilization rate itself does not factor in interest.

Simplicity and Real-Time Calculation:

The utilization rate is a simple, real-time metric that can be calculated instantly based on the current state of the pool. Including interest would complicate this calculation.

# 7 What Are Normalized Total Borrows and Normalized Total Liquidity?

Normalized Total Borrows:

This represents the total borrowed amount adjusted for accrued interest. It reflects the current value of the debt, including the interest that has accumulated over time.

For example, if users have borrowed 100 units at an interest rate of 10%, the normalized total borrows after one year would be 110 units.

Normalized Total Liquidity:

This represents the total available liquidity adjusted for interest earned by lenders. It reflects the current value of the funds deposited in the pool, including the interest earned over time.

For example, if lenders have deposited 1000 units at an interest rate of 5%, the normalized total liquidity after one year would be 1050 units.

How Would the Utilization Ratio Change?
The standard utilization ratio is calculated as:

Utilization Ratio= Total Borrowed/(Total Available Liquidity+Total Borrowed)
​

If we use normalized values, the formula becomes:

Utilization Ratio (Normalized)= Normalized Total Borrows/(Normalized Total Liquidity+Normalized Total Borrows)

Advantages of Using Normalized Values
Reflects True Economic State:

By accounting for interest accrual, the utilization ratio would better reflect the true economic state of the lending pool. It would show how much of the pool’s value (including interest) is being used for borrowing.

Better Risk Management:

A normalized utilization ratio could provide a more accurate measure of risk. For example, if interest rates are high, the normalized borrows would increase, signaling higher risk even if the raw borrowed amount hasn’t changed.

When Might This Approach Make Sense?
Using normalized values for the utilization ratio could make sense in certain scenarios:

Long-Term Lending Protocols: In protocols where loans are held for long periods, interest accrual becomes significant, and a normalized utilization ratio might provide a more accurate picture of the pool’s state.

Risk-Averse Protocols:Protocols that prioritize risk management might benefit from a utilization ratio that accounts for interest accrual, as it would better reflect the true economic risk.

Example Calculation
Suppose:

Total Borrowed: 100 units (raw)

Total Available Liquidity: 1000 units (raw)

Interest Rate on Borrows: 10%

Interest Rate on Liquidity: 5%

Time Elapsed: 1 year

Normalized Values:
Normalized Total Borrows: 100×(1+0.10)=110

Normalized Total Liquidity: 1000×(1+0.05)=1050

Utilization Ratio (Normalized):

Utilization Ratio (Normalized)= 110/1160 ≈9.48%

Compare this to the standard utilization ratio:

Utilization Ratio (Standard) =100/1100≈9.09%

In this case, the normalized utilization ratio is slightly higher because it accounts for the interest accrued on borrows.

Conclusion

Using normalized total borrows and normalized total liquidity to calculate the utilization ratio would make the metric more reflective of the true economic state of the lending pool. However, this approach introduces complexity, potential circular dependencies, and computational challenges. It might be suitable for advanced or long-term lending protocols but would require careful design to avoid unintended consequences. For most protocols, the standard utilization ratio (based on raw amounts) remains the preferred choice due to its simplicity and ease of understanding.

# 8 DIAMOND PROBLEM SOLIDITY

When trying to create a mock ERC4626 vault contract, I had the following code:

```solidity
//SDPX-license-Identifier: MIT

pragma solidity ^0.8.20;

import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockCrvVault is ERC20, ERC4626 {
    address public crv;
    error CannotDepositZeroShares();

    constructor(address _crv) ERC20("Vault", "VLT") ERC4626(IERC20(_crv)) {
        crv = _crv; // Initialize the crv variable
    }


    function deposit(
        uint256 assets,
        address receiver
    ) public override returns (uint256) {
        uint256 previewedShares = convertToShares(assets);
        if (previewedShares == 0) {
            revert CannotDepositZeroShares();
        }
        super.deposit(assets, receiver);
    }

    /** @dev See {IERC4626-convertToShares}. */
    function convertToShares(
        uint256 assets
    ) public view override returns (uint256) {
        if (totalSupply() == 0) {
            return assets;
        }
        if (totalAssets() == 0) {
            return 0;
        }
        super.convertToShares(assets);
    }
}

```

This produced the following error:

```
Derived contract must override function "decimals". Two or more base classes define function with same name and parameter types
```

The question then became why does the decimals parameter have to be overriden when it has a virtual keyword attached to it in ERC4626.sol. We know that virtual keyword means that the function can be overriden but doesnt mean that it must be. So i was wondering why that function in particular had to be overriden.

The reason you must override decimals() but not other virtual functions in ERC4626 is due to the diamond problem in Solidity, which happens when a function exists in multiple inherited contracts and Solidity needs explicit resolution.

Why decimals() Must Be Overridden

1. Multiple Base Contracts Define decimals()
   Both ERC20 and ERC4626 inherit from IERC20Metadata, which defines the decimals() function:

ERC20 has its own implementation of decimals().
ERC4626 also has an overridden decimals() function.
Because both contracts provide their own implementation of decimals(), Solidity requires you to explicitly specify how your contract should resolve the conflict.

2. Multiple Inheritance Conflict
   Since MockCrvVault inherits both ERC20 and ERC4626, Solidity does not automatically choose one decimals() implementation—it needs you to explicitly override and choose which logic should be used.

Why Other Virtual Functions Don't Need Overriding
Even though ERC4626 has many virtual functions, you don’t need to override them unless there's an inheritance conflict.

Functions like maxWithdraw(), convertToShares(), etc., do not exist in both ERC20 and ERC4626, so Solidity can just inherit them without issue.
Since only decimals() exists in multiple places, it is the only function requiring explicit override.

# 9 TRY-CATCH BLOCKS CAN BE SKIPPED. FUNCTION ATOMICITY CAN BE SKIPPED, 63/64 GAS RULE

While looking through previous findings, i saw something very interesting about try/catch blocks. The key point is that if you see a try/catch block, you need to look into how crucial the information in that try/catch block is because if it very important, what an attacker can do is call the function with just enough gas to make all the other parts of the transaction work but cause the code in the try block to fail but still be able to make the transaction succeed. This is something I didnt know was possible but this is a very key bit of information you MUST keep in mind. The finding explains this perfectly. See below:

Relevant GitHub Links

<https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-core/src/abstracts/SablierV2Lockup.sol#L394-L400>

Summary

Both `withdraw` and `renounce` implement a total of 3 callbacks.
In the case of `withdraw` there is a callback to the recipient and to the sender and in the case of `renounce`, only a callback to the recipient.
The issue here is that the caller of the function chooses how much gas he uses for the transaction, because of this he can specify an amount of gas that would be enough to execute the entire transaction, but not enough to execute the callbacks.

Vulnerability Details

Since `withdraw` is called by either the owner of the NFT or an approved operator, either one can call withdraw with an amount of gas that won't be enough for the sender callback to execute.
In the case of `renounce`, the sender can specify an amount of gas that won't be enough to execute the callback of the recipient.

Since the callback is wrapped in a `try/catch`, the caller can send enough gas so that the callback won't have enough to execute, reverting and entering the catch block, but the whole transaction will still have enough to execute. This happens because of the 63/64 rule, when an external call is made only 63/64 gas is forwarded to it, meaning there is always some leftover gas so the rest of the transaction can execute.

In our case, only an event is emitted after the callbacks, so we only need around 1600 gas left over.
This means that sender may calculate the gas, so the callback will receive `63 * 1600 = 100800`, which may not be enough for the complex operations inside the callback. This would result in failed callback execution, but success in `_withdraw`, function, because we have a `catch` block and another 1600 gas to emit the event after the callback:

```solidity
        if (msg.sender != sender && sender.code.length > 0 && sender != recipient) {
            try ISablierV2Sender(sender).onLockupStreamWithdrawn({
                streamId: streamId,
                caller: msg.sender,
                to: to,
                amount: amount
            }) { } catch { }
        }
```

As stated by the protocol, the callbacks can be complex and vital to both the `sender` and the `recipient`, so skipping them is not good, as logic that might be vital to them, won't execute and those 100800 units of gas may not be enough.

README statement:

```
In SablierV2Lockup, when onLockupStreamCanceled(), onLockupStreamWithdrawn() and onLockupStreamRenounced() hooks are called, there could be gas bomb attacks because we do not limit the gas transferred to the external contract. This is intended to support complex transactions implemented in those hooks.
```

Note that this issue differs from one, which enforces function caller to send more gas, because of the event emission after the callback. The fix for that, would even make the current attack path easier to implement for malicious actor, because he could make callback function receive even less gas and still accomplish successful `withdraw` transaction

Impact

Skipping important logic, breaking the atomicity of the system.

This finding talks about a 63/64 rule which i have never heard about till now but here is a quick overview of what this is:

The 63/64 rule in Solidity refers to gas forwarding limitations when making external contract calls using call, delegatecall, or staticcall.

1. What Is the 63/64 Rule?
   When a contract calls another contract via call, delegatecall, or staticcall, it can only forward 63/64 (≈98.4%) of the available gas to the recipient contract.

Why?
This rule was introduced in EIP-150 (Ethereum Hard Fork: "Tangerine Whistle").
It prevents "DoS attacks via gas exhaustion" by ensuring the calling contract always retains some gas to handle failures or reverts.
If the full gas amount were forwarded, the calling contract could be left with zero gas, making it unable to recover. 2. How Does It Work?
If the caller has X gas available, the external call will only forward:

63/64 time X

This means:

The called contract receives slightly less gas than expected.
The calling contract always retains at least X / 64 gas to prevent getting stuck.

1. Example of 63/64 Rule in Action
   Scenario 1: Calling a Contract Without Controlling Gas

```solidity
contract Caller {
    function callOtherContract(address target) external {
        (bool success, ) = target.call{gas: gasleft()}(""); // Forward all gas
        require(success, "Call failed");
    }
}
```

✅ Expected: Entire gasleft() is forwarded.
❌ Reality: Only 63/64 of the gas is actually sent!

Now you have this in your arsenal, if you see any try/catch blocks, you know to think about this 63/64 gas rule.

# 10 CREATE OPCODE CAN BE EXPLOITED WITH BLOCK REORG, ANY PROTOCOL ACTION/PREDICTION MADE USING A CONTRACT'S NONCE CAN BE EXPLOITED USING BLOCK REORG

Before I talk about this finding, lets look at the details of the submission so you understand what I am about to break down. You can view the full finding at: https://codehawks.cyfrin.io/c/2024-05-Sablier/s/28

Relevant GitHub Links

<https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-periphery/src/SablierV2MerkleLockupFactory.sol#L36>

<https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-periphery/src/SablierV2MerkleLockupFactory.sol#L71>

Summary

When a campaign initiator wants to create a linear or tranched airstream, he calls `SablierV2MerkleLockupFactory::createMerkleLL`, `SablierV2MerkleLockupFactory::createMerkleLT`, however these functions use the CREATE method (can be seen in the provided github permalinks) where the address derivation depends only on the `SablierV2MerkleLockupFactory` nonce. This is susceptible to reorg attacks.

## Vulnerability Details

As mentioned in the report's title, reorgs can occur in all EVM chains and most likely on L2's like Arbitrum or Polygon, and as stated in the protocol's README Sablier is compatible with "Any network which is EVM compatible", here are some reference links for some previous reorgs that happened in the past:

Ethereum: <https://decrypt.co/101390/ethereum-beacon-chain-blockchain-reorg> - 2 years ago

Polygon: <https://polygonscan.com/block/36757444/f?hash=0xf9aefee3ea0e4fc5f67aac48cb6e25912158ce9dca9ec6c99259d937433d6df8> - 2 years ago, this is with 120 blocks depth which means 4 minutes of re-written tx's since the block rate is \~2 seconds
<https://protos.com/polygon-hit-by-157-block-reorg-despite-hard-fork-to-reduce-reorgs/> - February last year, 157 blocks depth

Optimistic rollups (Optimism/Arbitrum) are also suspect to reorgs since if someone finds a fraud the blocks will be reverted, even though the user receives a confirmation.

These are the biggest events of reorgs that happened, here is a link for forked blocks, which means excluded blocks as a result of "Block Reorganizations" on Polygon: <https://polygonscan.com/blocks_forked?p=1>, where can be observed that at least two-digit block reorgs happen every month.

The vulnerability here is that airstream creators rely on address derivation in advance or when trying to deploy the same address on different chains, any funds sent to the airstream can be stolen.

Proof-Of-Concept:

Imagine the following scenario:

1. Alice deploys a new Airstream and funds it.
2. Bob has an active bot that observes the blockchain and alerts in reorg.
3. Bob calls one of the forementioned create functions
4. Thus an Airstream is created with an address to which Alice sends tokens.
5. Finally Alice's tx is executed and an Airstream is funded which Bob controls.
6. Bob immediately calls `SablierV2MerkleLockup::clawback` and transfers the tokens to himself.

Let’s talk about this finding a bit more. If you click the first link , it takes you to the contract factory where new airstreams can be deployed.

The line in question that causes the exploit is :

```solidity
 // Deploy the MerkleLockup contract with CREATE.
        merkleLL = new SablierV2MerkleLL(baseParams, lockupLinear, streamDurations);

```

and is located in a SablierV2MerkleLockupFactory.sol which is used to deploy airstreams using the above line.

You can see from the comment that the contract factory uses the CREATE opcode to deploy this contract. As you know from your assembly and fv course , CREATE is used when a contract is going to deploy another contract . Also CREATE2 can be used and this is where this bug becomes cool . This is why knowing about opcodes is super important in your blockchain journey.

We know that CREATE opcode uses the sender address and the nonce to determine what the contract address of the contract will be. See evm.codes where it says the following about the CREATE opcode :

```
The destination address is calculated as the rightmost 20 bytes (160 bits) of the Keccak-256 hash of the rlp encoding of the sender address followed by its nonce. That is:

address = keccak256(rlp([sender_address,sender_nonce]))[12:]
```

So this is how the destination contract address with the CREATE opcode is created. Notice that in this exploit , the sender address will always be the contract address of the factory contract that is making the deployments so this is deterministic , we always know what the address will be . What makes using the CREATE opcode contract address unpredictable is because of the nonce. The nonce of the contract as you know is simply a counter on how many transactions the contract has made and this number goes up by 1 each time and this nonce change will completely change what the keccak256 hash looks like where the address comes from. As you know, even a single change in the data being hashed with keccak256 can completely change the hash returned.

The exploit occurs when there is a block reorg. Lets have a quick look at what a block reorg is.

A block reorg (block reorganization) happens when the blockchain temporarily forks and the network switches to a longer chain, causing some previously confirmed blocks to be discarded (orphaned).

1. Why Do Block Reorgs Happen?
   A. Natural Network Behavior
   Ethereum (and other blockchains) operates on a longest chain rule:

If two miners find a valid block at the same time, the network temporarily forks.
Some nodes will see one block, while others see the other block.
The next miner who adds a new block extends one of these forks.
The longer chain becomes the "main" chain, and the other block is discarded (reorg).
B. High Network Latency
If a new block does not propagate fast enough, another miner might find a competing block.
The slower block gets discarded, leading to a reorg.
C. Validator Disagreements (Proof of Stake)
In Ethereum's Proof of Stake (PoS), validators sometimes propose blocks at the same time.
The network chooses one based on chain length and finalization rules.

🔹 Example:

Step 1: Miner A and Miner B both find a block at height 100.
Step 2: Network is split—some nodes accept A’s block, others accept B’s.
Step 3: A third miner extends chain A → Chain A becomes the "main" chain.
Step 4: Chain B’s block 100 is discarded (block reorg).
If your transaction was in block B, it is now in an orphaned block and needs to be re-mined.

Transactions from an orphaned block (a block that gets discarded in a reorg) can be re-mined in a different order when included in the new main chain.

1. Why Are Transactions Reordered in a Reorg?
   When a block is orphaned due to a reorg:

All transactions in the orphaned block return to the mempool (if they are not included in another valid block).
Miners/validators pick transactions from the mempool for the next block.
Transactions are re-selected based on priority mechanisms, which may be different from the original order.
Since the mempool is dynamic:

Some transactions may have higher gas fees than before, leading to new ordering.
New transactions may enter the mempool, pushing lower-priority transactions back.
Some transactions may be dropped entirely if they become invalid (e.g., they depend on a state that changed in the reorg).

So this happens where your transaction is already on the block and has been confirmed onchain as being on the block . This is different to a front run where i see a transaction on the mempool before it has been confirmed . In the case , the transaction has already been included jn a block but a block reorg happens and these transactions are in an orphaned block and have to be remined as explained above. In a situation like this, using the CREATE opcode can mean an attacker can predict a reorg and call the same createMerkleLL function in the contract factory that a user has called to deploy an airstream and since it is the same SablierV2MerkleLockupFactory.sol contract deploying the txs, the sender address is the same but the nonces will be different . In a block reorg , if the attackers tx is placed before the normal user’s which is very easy to do as attacker can set their tx as higher priority with higher gas, then obviously the nonces will change as well which means that the attackers airstream will have the contract address that the normal user thought they were going to get.

So in a situation where the user tries to predict what the address will be and use that for certain actions, say the user predicts the contract address of their airstream which apparently was necessary in this protocol and then funds it instead of waiting for block confirmations to see what contract address they will get, they predict the address and send funds to it. In a case like this, in a block re-org , the attacker will control the predicted contract address and be able to steal those funds as explained above.

The fix would be to simply use CREATE2 opcode instead of simply CREATE and the reason is that CREATE2 as you see from evm.codes takes a salt value from the stack that can be any bytes value and this allows for us to predict what the contract address is going to be as evm.codes tells us. So if the contract factory passes bytes(msg.sender) as the salt for create2, even in a block reorg, it won’t change who owns which contract because the nonce is not used with CREATE2. See CREATE2 instructions below:

```
Equivalent to CREATE, except the salt allows the new contract to be deployed at a consistent, deterministic address.

Should deployment succeed, the account's code is set to the return data resulting from executing the initialisation code.

The destination address is calculated as follows:

initialisation_code = memory[offset:offset+size]
address = keccak256(0xff + sender_address + salt + keccak256(initialisation_code))[12:]

Stack input
value: value in wei to send to the new account.
offset: byte offset in the memory in bytes, the initialisation code of the new account.
size: byte size to copy (size of the initialisation code).
salt: 32-byte value used to create the new account at a deterministic address.
Stack output
address: the address of the deployed contract, 0 if the deployment failed
```

This is such a cool finding and will help you a lot going forward . Now you know that when you see any contracts using the create opcode or trying to use the nonce to do any sort of calculation and predict anything based on the nonce , block reorg attacks will spring to mind immediately

## 11 DIFFERENT TYPES OF REENTRANCY , NONREENTRANT MODIFIER DOESNT STOP REENTRANCY IF CEI ISNT FOLLOWED

You already know about reentrancies so you might be wondering what is this point for? The main takeaway point is that there are different types fo reentrancies. The problem you have previously had was that whenever you saw a function that had the nonRentrant modifier, you automatically assumed that there was no way that the function could be reentered even if it didnt follow CEI methodology.

This was a limited mentality because it is true and not true at the same time. This is because I was not aware of all the different ways reentrancy could occur. The only one I was aware of was single function reentrancy and with a nonRentrant modifier, single function reentrancy is actually not possible BUT there are more types of reentrancy that can occur if the function doesnt follow the CEI methodology. These include multiple function reentrancy, cross contract reentrancy , cross chain reentrancy and the list goes on.

The main point is that even if a function has a nonRentrant modifier, we can still perform reentrancy by reentering another function before the rest of the state is updated in the vulnerable function. Lets look at a simplified example of a cross contract reentrancy. See below:

```solidity
pragma solidity 0.8.17;

abstract contract ReentrancyGuard {
    bool internal locked;

    modifier noReentrant() {
        require(!locked, "No re-entrancy");
        locked = true;
        _;
        locked = false;
    }
}

interface IMoonToken {
    function transferOwnership(address _newOwner) external;
    function transfer(address _to, uint256 _value) external returns (bool success);
    function transferFrom(address _from, address _to, uint256 _value) external returns (bool success);
    function balanceOf(address _owner) external view returns (uint256 balance);
    function approve(address _spender, uint256 _value) external returns (bool success);
    function allowance(address _owner, address _spender) external view returns (uint256 remaining);
    function mint(address _to, uint256 _value) external returns (bool success);
    function burnAccount(address _from) external returns (bool success);
}

contract MoonToken {
    uint256 private constant MAX_UINT256 = type(uint256).max;
    mapping (address => uint256) public balances;
    mapping (address => mapping (address => uint256)) public allowed;

    string public constant name = "MOON Token";
    string public constant symbol = "MOON";
    uint8 public constant decimals = 18;

    uint256 public totalSupply;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(owner == msg.sender, "You are not the owner");
        _;
    }

    function transferOwnership(address _newOwner) external onlyOwner {
        owner = _newOwner;
    }

    function transfer(address _to, uint256 _value)
        external
        returns (bool success)
    {
        require(balances[msg.sender] >= _value);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function transferFrom(
        address _from,
        address _to,
        uint256 _value
    ) external returns (bool success) {
        uint256 allowance_ = allowed[_from][msg.sender];
        require(balances[_from] >= _value && allowance_ >= _value);

        balances[_to] += _value;
        balances[_from] -= _value;

        if (allowance_ < MAX_UINT256) {
            allowed[_from][msg.sender] -= _value;
        }
        return true;
    }

    function balanceOf(address _owner)
        external
        view
        returns (uint256 balance)
    {
        return balances[_owner];
    }

    function approve(address _spender, uint256 _value)
        external
        returns (bool success)
    {
        allowed[msg.sender][_spender] = _value;
        return true;
    }

    function allowance(address _owner, address _spender)
        external
        view
        returns (uint256 remaining)
    {
        return allowed[_owner][_spender];
    }

    function mint(address _to, uint256 _value)
        external
        onlyOwner  // MoonVault must be the contract owner
        returns (bool success)
    {
        balances[_to] += _value;
        totalSupply += _value;
        return true;
    }

    function burnAccount(address _from)
        external
        onlyOwner  // MoonVault must be the contract owner
        returns (bool success)
    {
        uint256 amountToBurn = balances[_from];
        balances[_from] -= amountToBurn;
        totalSupply -= amountToBurn;
        return true;
    }
}
```

The ReentrancyGuard contains the noReentrant modifier that is used to prevent reentrancy attacks. The noReentrant (lines 6–11) is a simple lock that allows only a single entrance to the function applying it. If there is an attempt to do the reentrant, the transaction would be reverted by the require statement and none of this stuff should be new to you.

```solidity
pragma solidity 0.8.17;

import "./Dependencies.sol";

// InsecureMoonVault must be the contract owner of the MoonToken
contract InsecureMoonVault is ReentrancyGuard {
    IMoonToken public immutable moonToken;

    constructor(IMoonToken _moonToken) {
        moonToken = _moonToken;
    }

    function deposit() external payable noReentrant {  // Apply the noReentrant modifier
        bool success = moonToken.mint(msg.sender, msg.value);
        require(success, "Failed to mint token");
    }

    function withdrawAll() external noReentrant {  // Apply the noReentrant modifier
        uint256 balance = getUserBalance(msg.sender);
        require(balance > 0, "Insufficient balance");

        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Failed to send Ether");

        success = moonToken.burnAccount(msg.sender);
        require(success, "Failed to burn token");
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }

    function getUserBalance(address _user) public view returns (uint256) {
        return moonToken.balanceOf(_user);
    }
}
```

Since the InsecureMoonVault contract applies noReentrant modifier to the deposit and withdrawAll functions, both functions are safe from signle functio reentrancy attacks which we discussed above. Previously, you wouldnt look out for more chances of reentrancy after seeing the noReentrant modifier but this is where you would be wrong. Lets discuss how a cross contract reentrancy can occur here.

if you havent noticed it yet, the withdrawAll function is the vulnerable one but since it has the noRentrant modifer, how do we exploit this ?

The root cause of cross-contract reentrancy attack is typically caused by having multiple contracts mutually sharing the same state variable, and some of them update that variable insecurely. So whenever you are thinking about how to exploit reentrancy , you first need to find your entry point, which is how you will renter the contract which we already know is going to be the low level call function in withdrawAll as we know we can just have a contract with a receive function and reenter which is rookie stuff.

Now we know how to enter, the next thing we need to know is which states are updated AFTER our entry point and which other functions share these state variables ? So after our entry point, moonToken.burnAccount is called and this function updates the balancesmapping mainly. It also updates the totalsupply but we dont really need this for our exploit but if this protocol did some interesting stuff with totalSupply, we could mess with that as well but for this simplified example, lets focus on the balances mapping.

So which other functions share the balances state variable?? Well there are a lot, there is the mint function which is restricted to onlyOwner so nothing we can do there, there is balanceOf which is a view function so we cant change state with that BUT there is also a transfer function and once you see this, your eyes should light up because we know now that we can use our entry point to call the transfer function BEFORE THE moonToken.burnAccount is called and our balance is reset and we can drain the contract like this.

The entry point and the shared state variables after the entry point are the 2 ways you can reenter any contract that has functions that dont follow CEI. Admittedly, it is harder to spot than single function reentrancy but if you follow these 2 steps, you can find it easily enough. You can easily design a contract that can be used to exploit this bug with a function that deposits tokens , another function that calls withdrawAll once and then in the receive function, call transfer to transfer the tokens out of the contract and then call withdrawAll again and keep doing this until the contract is drained. This is a simple example and an easy 2 step process you can use once you spot CEI not being used in any function in any protocol.

## 12 UNDERSTANDING VOTING ESCROWS

Voting Escrows are a concept we havent really seen yet and you are likely to come across this many times going forward so you need to understand what is going on and how it works in order to easily navigate them the more you see them.

What exactly are they? To put it simply - users lock some of their funds for a period of time. They in return receive a balance proportional to both the deposited amount and the lock's duration.

In fact, a user's balance at any time can be derived from the following formula:

(amountLocked / MAX_TIME ) times (lockEndTime - block.timestamp)

it's that simple. However, in order to fully understand the codebase, we need to understand every single bit of it.

Something which seems hard to understand is the introduction of bias and slope. So what exactly are they?

slope is the rate per second at which a user's balance decreases. Think of this as the amount of tokens lost per second. It should always be equal to:

amountLocked / MAX_TIME

Now your next question is probably why do we use MAX_TIME ?? Surely the rate of a users balance decrease should be the amount they locked / how long they locked the tokens. Let me explain why this isnt the case.

**Understanding Slope Calculation in Voting Escrow (ve) Systems**

The formula:

slope = amountLocked/MAX_TIME

defines **the rate per second at which a user's balance (voting power) decreases** over time.

---

**Why MAX_TIME and Not the User’s Lock Duration?**

**1. Normalization for Consistency**

- **`MAX_TIME`** is the **longest possible lock duration** (e.g., 4 years in many ve-token systems).
- **All users’ slopes are normalized** against the same `MAX_TIME`, ensuring a **consistent, predictable rate of decay**.
- If different users had different time bases (e.g., `userLockTime` instead of `MAX_TIME`), the system would not be able to fairly compare or sum up their voting power.

---

**2. ve-System Mechanics: How Voting Power Decays Over Time**

- A user's **voting power (veTokens)** starts at its highest when they lock tokens.
- Over time, the **voting power decays linearly to 0** as the lock approaches its end which we will see with an example below.
- Using `MAX_TIME` ensures that the decay slope is **standardized across all users**, making calculations **easier** and ensuring the system remains **fair and predictable**.

---

**3. Example Calculation**

Assume:

- `MAX_TIME = 4 years = 1,261,440,000 seconds`
- Alice locks **400 tokens for 2 years** (`= 630,720,000 seconds`)
- Bob locks **400 tokens for 4 years** (`= 1,261,440,000 seconds`)

  **If we used each user’s lock time (`userLockTime`):**

- Alice’s slope would be:

  400/630,720,000 = 6.34 times 10^-7 tokens per second

- Bob’s slope would be:

  400/1,261,440,000 = 3.17 times 10^-7 tokens per second

The issue? **Alice’s slope is twice Bob’s, even though Bob locked for longer!**

- This would **overweight** shorter locks in the system.
- If we instead **normalize** everything using `MAX_TIME`, both users' voting power is **fairly distributed**.

  **Corrected approach using `MAX_TIME`:**

- Alice’s slope:

  400/1,261,440,000 = 3.17 times 10^-7

- Bob’s slope:

400/1,261,440,000 = 3.17 times 10^-7

Now, both users start **at the correct ratio based on their locks**, and the system can fairly **compare users with different lock durations**.

---

**4. The Key Idea: MAX_TIME Creates a Standardized Decay Rate**

- **Ensures fairness**: Users who lock for **shorter periods** start with **lower initial voting power**.
- **Maintains linearity**: Decay over time is **predictable** and **consistent** across all users.
- **Simplifies calculations**: Keeps all voting power computations **normalized** to a single time base.

bias is simply the user's balance at a given point. In fact, the formula I gave you earlier was the formula to calculate bias:

(amountLocked /MAX_TIME) times (lockendtime - block.timestamp)

which can be simplified to:

slope times (lockendtime - block.timestamp)

which makes sense as it is the amount of tokens lost per second multiplied by how many seconds are left of the user's lock.

This information is usually going to be in Point struct. In RAAC, the point struct is located in the RAACVoting.sol library. See below:

```solidity
  struct Point {
        int128 bias;       // Current voting power
        int128 slope;      // Rate of voting power decay
        uint256 timestamp; // Point creation timestamp
    }

```

As long as the user has a point in the past, we can always derive their balance from the following formula:

uservotingpower = point.bias - (point.slope times (block.timestamp - point.timestamp))

where (point.slope times (block.timestamp - point.timestamp)) is simply the amount of tokens the user has lost up to that point (call this rate of decay) and point.bias is the user's balance when they started a lock. Lets expand on what this formula is doing with an example.

Let’s assume the following:

A user locks 100 tokens for a duration of 1 year (365 days).

The lockEndTime is 365 days from the lock start time.

The point.timestamp is the time when the user locked their tokens (let’s call this t = 0).

The point.slope is calculated as:
slope = lockedTokens / MAX_TIME
= 100 / 365
≈ 0.27397 (tokens per day)

The point.bias is calculated as:
bias = slope times (lockEndTime - point.timestamp)
= 0.27397 times (365 - 0)
= 100
Now, let’s calculate the user’s voting power at different points in time:

At t = 0 (lock start time):
votingPower = point.bias - point.slope times (block.timestamp - point.timestamp)
= 100 - 0.27397 times (0 - 0)
= 100
The user has their full voting power of 100 at the start.

At t = 100 days:

votingPower = 100 - 0.27397 times (100 - 0)
= 100 - 27.397
≈ 72.603
After 100 days, the user’s voting power has decreased to ~72.603.

At t = 200 days:

votingPower = 100 - 0.27397 times (200 - 0)
= 100 - 54.794
≈ 45.206
After 200 days, the user’s voting power has decreased to ~45.206.

At t = 365 days (lock end time):
votingPower = 100 - 0.27397 times (365 - 0)
= 100 - 100
= 0
At the end of the lock period, the user’s voting power has decreased to 0.

Why This Formula Works:
Linear Decay: The voting power decreases linearly over time because the reduction is proportional to the elapsed time (block.timestamp - point.timestamp).

Tends to 0: As block.timestamp approaches lockEndTime, the term point.slope times (block.timestamp - point.timestamp) approaches point.bias, ensuring the voting power tends to 0.

So all is well and good but now we know that the user's bias (balance) always goes down as time passes, the total supply of all veTokens obviously can't be gotten using the usual totalSupply function that an ERC20 contract uses. This is actually a problem that happened right here with RAAC and I will use their protocol to show why.

So in veRAACToken, this is how users lock RAAC to get veRAAC:

```solidity
 /**
     * @notice Creates a new lock position for RAAC tokens
     * @dev Locks RAAC tokens for a specified duration and mints veRAAC tokens representing voting power
     * @param amount The amount of RAAC tokens to lock
     * @param duration The duration to lock tokens for, in seconds
     */
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        if (amount == 0) revert InvalidAmount();
        if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
        if (totalSupply() + amount > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded();
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION)
            revert InvalidLockDuration();

        // Do the transfer first - this will revert with ERC20InsufficientBalance if user doesn't have enough tokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);

        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;

        // Create lock position
        _lockState.createLock(msg.sender, amount, duration);
        _updateBoostState(msg.sender, amount);

        // Calculate initial voting power
        (int128 bias, int128 slope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            amount,
            unlockTime
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(bias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Mint veTokens
        _mint(msg.sender, newPower);

        emit LockCreated(msg.sender, amount, unlockTime);
    }

```

The main part we are focusing on is where they are minted veRAAC in the function. You can see that when they lock, their bias is calculated and then they are minted that bias which represents their balance. This is all well and good BUT that balance is only good until 1 second has passed. This is obviously because of the slope. The slope means that every second, the user's balance decreases so for example, say the bias was 100 and the slope (token decrease per second) was 1. So once the user is minted these veRAAC tokens, which are 100, once a second passes, that balance should be reduced to 99 to reflect their new bias. the totalsupply should also be updates to reflect this change which is very important because if the users lock duration is over but they havent withdraw yet, their bias is 0 but the totalSupply will still be 100 since they havent withdrawn which burns the tokens. This is a huge problem because if the total supply is used for calculations, then it doesnt correctly reflect the bias.

RAAC did not implement this and what this meant was that their totalSupply was inaccurate. This is how they calculated the totalvoting power in veRAACToken.sol

```solidity
function getTotalVotingPower() external view override returns (uint256) {
        return totalSupply();
    }

```

which as we discussed doesnt factor slope changes in the calculation. So the question is how do we actually get the totalsupply that accounts for each users decreasing balance ? We could loop through each user and calculate their bias and then return that as the total supply but this is EXTREEMLY gas inefficient especially if there are many users.

So the solution to calculate total supply is actually similar to calculating a user's balance at a given timestamp. Remember how we said:

uservotingpower = point.bias - (point.slope times (block.timestamp - point.timestamp))
where (point.slope times (block.timestamp - point.timestamp)) is simply the amount of tokens the user has lost up to that point (call this rate of decay) and point.bias is the user's balance when they started a lock.

Well we can do this globally and what I mean is that we can record a global bias and a global scope. the global bias will be the balance of all user's when they started a lock so in veRAACToken::lock, once a useer creates a lock , whatever bias is calculated for them is added to the global bias. This also happens is a user increases their lock. With this, we will have the total of all user biases.

Next up is that we need to have a global scope and the same idea will be prevalent with that but there will be some nuances and I will explain why. So like global bias, once a user starts a lock or increases a lock, their scope will also increase BUT what happens when a user's lock is over?

We dont actually reduce the global bias ever just like we never reduce the point.bias ever as this represents the user's balance when they locked so this isnt changing BUT the global scope will have to change because when a user's lock ends, their slope doesnt contribute to the amount of tokens that all users have lost up to that point anymore because the have no voting power anymore so for them, their point.bias - (point.slope times (block.timestamp - point.timestamp)) is 0.

So for our total supply calculation, we need to reflect this because if we dont, the rate of decay will be more than it needs to be because we are including the rate of decay of a user who has no voting power which doesnt make sense. So we need to find a way to remove a user's slope from the global scope once their voting power is 0 i.e. at the end of their lock time.

So to do this, we need to track the amount of slope we need to subtract from global scope at what time. There are 2 things that will allow us do this:

Weekly Lock End Times:

All lock end times are aligned to the end of a week (timestamp % 604800 == 0). So when a user creates a lock, the end time has to be at the end of a week. So we need to make sure that the timestamp for the end of the lock period is exactly divisible by 1 week which is 604800 in seconds. This way, we know that at the end of every week, we can check to see who's voting power is at 0. We do this using:

Slope Changes Mapping:

A mapping slopeChanges tracks the changes in slope that need to be applied at specific timestamps (end of weeks). When a user creates a lock, their slope is added to globalSlope, and their slope is also added to slopeChanges mapping at their lock end time.

Applying Slope Changes:

Whenever a week ends, the contract applies the slope changes stored in slopeChanges for that timestamp. This decreases the globalSlope by the appropriate amount, ensuring that the total supply is updated correctly.

Lets drill this point home with an example:

Example Walkthrough:
Let’s walk through an example to illustrate how this works.

Initial State:
globalBias = 0

globalSlope = 0

slopeChanges = {}

Assume MAX_TIME = 5 weeks

User A Locks Tokens:
User A locks 100 tokens for 2 weeks.

Their slope = 100 / (5 \* 604800) ≈ 0.00003307 (tokens per second).

At t = 0 (lock creation time), their bias is:
bias = slope _ (lockEndTime - block.timestamp)
= 0.00003307 _ (2 _ 604800 - 0)
= 0.00003307 _ 1209600
≈ 40

The contract:

Adds their bias to globalBias: globalBias = 40.

Adds their slope to globalSlope: globalSlope = 0.00003307.

Adds their slope to slopeChanges at the lock end time (t = 2 weeks):
slopeChanges[2 weeks] = 0.00003307

User B Locks Tokens:
User B locks 200 tokens for 3 weeks.

Their slope = 200 / (5 \* 604800) ≈ 0.00006614 (tokens per second).

At t = 0 (lock creation time), their bias is:
bias = slope _ (lockEndTime - block.timestamp)
= 0.00006614 _ (3 _ 604800 - 0)
= 0.00006614 _ 1814400
≈ 120

The contract:

Adds their bias to globalBias: globalBias = 40 + 120 = 160.

Adds their slope to globalSlope: globalSlope = 0.00003307 + 0.00006614 = 0.00009921.

Adds their slope to slopeChanges at the lock end time (t = 3 weeks):
slopeChanges[3 weeks] = 0.00006614

At t = 1 week:
The current timestamp is t = 604800 seconds (1 week).

User A’s remaining bias:
bias*A = slope_A * (lockEndTime - block.timestamp)
= 0.00003307 _ (2 _ 604800 - 604800)
= 0.00003307 \_ 604800
≈ 20

User B’s remaining bias:
bias*B = slope_B * (lockEndTime - block.timestamp)
= 0.00006614 _ (3 _ 604800 - 604800)
= 0.00006614 \_ 1209600
≈ 80

The total supply at t = 1 week:

totalSupply = globalBias - globalSlope _ (block.timestamp - lastCheckpointTimestamp)
= 160 - 0.00009921 _ 604800
≈ 160 - 60
= 100

At t = 2 weeks:
The contract applies the slope change for t = 2 weeks:

globalSlope += slopeChanges[2 weeks]
globalSlope = 0.00009921 - 0.00003307 = 0.00006614.
User A’s lock has expired, so their slope is no longer contributing to the total voting power.

User B’s remaining bias:
bias*B = slope_B * (lockEndTime - block.timestamp)
= 0.00006614 _ (3 _ 604800 - 2 \_ 604800)
= 0.00006614 \* 604800
≈ 40

The total supply at t = 2 weeks:

totalSupply = globalBias - globalSlope _ (block.timestamp - lastCheckpointTimestamp)
= 160 - 0.00009921 _ 1209600
≈ 160 - 120
= 40

This is how totalSupply should be tracked. Now you have this knowledge, you dive into any protocol that uses voting escrows with confidence. Another good tool in the toolbox to have.

# 13 RESPECT FIRST PASS FLOW, FOLLOW LEADS AND EXPLORE DIFFERENT IDEAS

When running a first pass, different functions will take you to different contracts to follow different leads. Do not hesitate to follow a lead all the way, as you don't know where that will take you. Most likely, if you have a hunch, there will either be a bug there or something to learn. Once you are satisfied that road didn't lead anywhere, then you can come back to your pass and carry on. Do not look at functions sequentially; you have to follow an attack path. Looking at functions sequentially will lead you nowhere. By sequentially, i mean what we did when we were first starting out by taking a contract and looking at each function in the contract one by one. You CANNOT do that. This is the worst way to audit and is very beginner level. You have to pick an entry point and follow it all the way through with a first pass. In that first pass,, different leads and ideas will come up and you need to follow those leads to the end. Once the first pass is done, then you can go through the same path again and perform assumption analysis.

# 14 CUSTOM ERROR CHECKS, COOL NEW WAY TO FIND EASY HIGH/MEDIUMS

So let me present a different way besides assumption analysis you can use to quickly find bugs. the idea is that when you go into a contract, look at the custom errors defined in the contract and perform a search on these errors in the file so you see when they are used and more importantly, where they are not used. A lot of the time, you will find that an error message that is used in one function that should be used in another, just isnt.

You can see this in H-16 of the RAACfindings.md file where i found a case where an error should have been used in the function but wasnt and the only way i knew that was because it was used in another function that did a similar thing but wasnt used in that function which lead to a high vulnerability error. If i did a global search on the custom error, I would have been able to spot the issue faster which is why this custom error checks can be useful if you dont have much time on your hands.

# DAOS AND GOVERNANCE EXPLOITS TO BE AWARE OF

As we as using voting escrows (ve mechanics), RAAC implemented the use of a DAO which is something we covered during the foundry course so you can go over that to for a recap but here we are going to cover some key things you need to look out for whnever you see voting escrows or DAO's being used in protocols.

1. LACK OF VOTING LOCKING IN PROPOSALS

The idea is to make sure that if a user votes on a proposal, their voting power is locked in that proposal and prevents the user from voting again with the same voting power. This was actually an issue I spotted in RAAC. Look at H-18 in the raacFindings.md to see the relevant finding so in any protocol that implements the ability to vote on a proposal, make sure users cannot double vote. There is also a situation where a user can vote with their tokens, then transfer to another address and vote again and rinse and repeat which can obviously also skew the voting and is another one you need to look out for.

2. 2 STEP FLASHLOAN ATTACK

For this attack to work, there are 2 things that need to be in place. The first is that we need to be able to get a flashloan which buys the governance token and gets voting power. We already know what a flashloan is and we know that whatever we do, we must pay the flashloan back in the same transaction where we took it. Usually with flash loand, an exploit is usually possible and we use the flashloan to amplify the impact of the exploit but in this case, the flashloan can be an inegral part of the attack and I will explain why shortly.

The second step is that we should be able to vote for the proposal and then execute the proposal in the same transaction. By executing the proposal in the same transaction, our tokens are unlocked and we can swap the governance token back to whatever token we flashloaned in order to pay back the flashloan.

So lets look at how this works, if we can get a flashloan, buy the governance token, gain voting power, vote on a proposal AND execute the proposal which releases our tokens in the saame transaction then we are in the money.

This sounds like a lot of steps at first but if we take out the aspect of the flashloan, the idea would simply be if we can vote and execute the proposal, i.e. end voting in the same transaction, then there is an issue. As you can imagine, the issue is that I can take a flashloan for a large amount, buy the governance token, vote on any proposal i want with those tokens, end the proposal so voting is done, then sell the tokens i bought and pay back my loan. So i can essentially skew the voting to whatever way i want with this attack. I actually reported something in similar in RAAC and you can read the finding in raacfindings.md in H-19. This finding didnt use a flashloan but you can apply the flashloan concept to that.

There was also a beanstalk hack where the hacker took 180 million by doing this exact thing. it was actually exactly this that happened and you can read about it online. We dont really need to explain it step by step because the exploit wasnt complicated. It was exactly as I explained above. You can read about the exploit at https://www.halborn.com/blog/post/explained-the-beanstalk-hack-april-2022 or any resources you find online.
