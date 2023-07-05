# Exit10 yAcademy Block 5 Audit

Report link: https://reports.yaudit.dev/reports/04-2023-Exit10

This audit was performed as a part of yAcademy Block 5. Below are the findings I contributed personally.

# Findings:

## [Medium - Incorrect EXIT_DISCOUNT values](https://reports.yaudit.dev/reports/04-2023-Exit10/#2-medium---incorrect-exit_discount-values)

The .env environment file sets the EXIT_DISCOUNT value to 5, which when combined with the PERCENT_BASE constant value of 10_000, results in a 0.05% discount on EXIT tokens. This contradicts the documentation which states that the discount is 5%.

### Technical Details

The docs page describing the EXIT token describes a 5% discount on the EXIT token. But this value is not used consistently in the code.

The value set in the .env file combined with the constant PERCENT_BASE value results in a much lower discount, only 0.05%.

### Impact

Medium. The exit discount in the code does not match the docs.

### Recommendation

Revise the documentation and the .env file so that the EXIT token discount implementation is consistent with the documentation.

# [Medium - Problematic MasterchefExit rewards distribution to first EXIT staker](https://reports.yaudit.dev/reports/04-2023-Exit10/#2-medium---incorrect-exit_discount-values)

MasterChefExit.sol has a deposit() function for users to deposit their reward tokens. This function has issue that could lead to a MEV competition between users resulting in some users losing out on rewards.

### Technical Details

There are three problems with MasterchefExit.deposit():

1. Only the first staker to call deposit() receives rewards, which can lead to frontrunning of this function.
2. A zero amount is permitted in deposit(), meaning the caller does not need to have any tokens.
3. Because a zero amount is permitted in deposit(), pool.totalStaked will not increase after deposit() is called the first time with a zero amount, so deposit() can keep getting called with a zero amount by anyone until there is a non-zero value staked.

### Impact

Medium. There is a risk of frontrunning and the logic allows a zero amount.

### Recommendation

Add a check to revert deposit() when it is called with a zero amount value.

## [Low - High hardcoded slippage](https://reports.yaudit.dev/reports/04-2023-Exit10/#6-low---high-hardcoded-slippage)

FeeSplitter.sol has a constant slippage of 1%. This is a high slippage for a deep liquidity pool like USDC/ETH and could be prone to sandwich attacks.

### Technical Details

The slippage can be changed to a lower value. It won’t cause the protocol to become stuck because FeeSplitter.sol uses function updateFees(uint256 swapAmountOut) to swap which has an input parameter. If the liquidity gets too low, and defined slippage is too high, input swapAmountOut can be decreased to complete the swaps.

### Impact

Low. Too high slippage can result in users receiving less value than expected.

### Recommendation

Set slippage to lower value.
