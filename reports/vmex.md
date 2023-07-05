# VMEX yAcademy Block 5 Audit

Report link: https://reports.yaudit.dev/reports/06-2023-VMEX/

This audit was performed as a part of yAcademy Block 5. Below are the findings I contributed personally.

# Findings:

## [Critical - Tranche admin can DOS their tranche by setting treasury address to address(0)](https://reports.yaudit.dev/reports/06-2023-VMEX/#2-critical---tranche-admin-can-dos-their-tranche-by-setting-treasury-address-to-address0)

Tranche admins are semi-privileged yet untrusted actors in the VMEX protocol. They are responsible for claiming and managing specific asset tranches and are able to configure parameters for their tranche within bounds set by VMEX. One of these is a fee parameter; the tranche admin can specify an address where they’d like to receive a fee that is paid out of their tranche’s yield.

This fee is paid by minting the appropriate amount of aTokens to the tranche admin’s fee address whenever the state is updated. However, the aToken’s \_mint() function will revert if the recipient address is 0x0. This means that a tranche admin can DOS the tranche by setting their fee address to 0x0, preventing any further state updates from occurring.

Note that this denial of service similarly applies if the owner of the LendingPoolAddressProvider contract were to set the VMEX treasury address to address(0). This would DOS the whole protocol, not just one tranche. Per conversations with the team, a trusted multisig will be the contract owner and their ability to undermine the system poses less risk than untrusted tranche admins.

### Technical Details

Tranche admins can update their fee address via LendingPoolConfigurator.updateTreasuryAddress(), and there is no check to ensure that the address they set is not address(0).

The function ReserveLogic.updateState() is called in nearly all of the protocol’s critical functions, including deposit(), withdraw(), repay(), borrowHelper(), and liquidationCall(). The following (abridged) call sequence occurs whenever updateState() is performed: ReserveLogic.updateState() -> ReserveLogic.\_mintToTreasury() -> AToken.mintToTreasury() -> AToken.\_mint():

The AToken functions are below:

```solidity
function mintToTreasury(uint256 amount, uint256 index) external override onlyLendingPool {
        if (amount == 0) {
            return;
        }
        // get tranche admin's fee address from configurator
        address treasury = ILendingPoolConfigurator(_lendingPoolConfigurator).trancheAdminTreasuryAddresses(_tranche);
        _mint(treasury, amount.rayDiv(index));
        ...
}
function _mint(address account, uint256 amount) internal virtual {
    require(account != address(0), "ERC20: mint to the zero address");
    ...
}
```

The require statement in \_mint() will cause the state update to revert if the recipient address is 0x0, preventing the protocol from functioning.

Note that a global admin can recover the tranche from this state by setting the tranche’s treasury address to a valid address (and likely removing the tranche admin). However, this is a manual process and requires the global admin to be aware of the issue and take action.

To reproduce this issue, modify helpers/contract-deployment.ts to set treasuryAddress to "0x0000000000000000000000000000000000000000" at L237 and L623. Observe that many tests now revert with “ERC20: mint to the zero address”.

### Impact

Critical. Tranche admins are considered untrusted actors and are able to cause the protocol to stop functioning at key moments (e.g. as a large position is nearing liquidation).

### Recommendation

Enforce an address(0) check on untrusted input.

## [Medium - Blacklist/Whitelist does not behave as expected and tranche admins can block all transfers](https://reports.yaudit.dev/reports/06-2023-VMEX/#6-medium---blacklistwhitelist-does-not-behave-as-expected-and-tranche-admins-can-block-all-transfers)

Tranche admins are considered semi-privileged but still untrusted actors in the system. They are responsible for tranche management, and can control a whitelist/blacklist that governs who can interact with the protocol’s tranches that they manage. Blacklisted users are generally still allowed to repay debts and withdraw their funds from the system, but are not allowed to deposit or borrow.

The current implementation of the whitelist/blacklist does not correctly prevent a blacklisted user from transferring their tokens to a new account and continuing to use the protocol. The current design is also prone to tranche admin abuse, as they are able block all aToken transfers for their tranche at any time by either enabling whitelist mode for their tranche or blacklisting a reserve’s aToken contract.

### Technical Details

AToken.\_transfer() calls LendingPool.finalizeTransfer() which internally calls checkWhitelistBlacklist() to check if both the msg.sender and the token receiver are whitelisted/blacklisted for the respective tranche. In the context of this call however, msg.sender is the aToken contract, rather than the transfer’s from address. Accordingly, even if a token sender is blacklisted (or non-whitelisted), they will still be able to transfer their tokens to a new address as the from address is never checked. Afterwards, the receiving address will be able to freely interact with the protocol (in the blacklist case; if the new address is not on the whitelist they will still be blocked from deposit/borrowing).

Similarly, a tranche admin can block all aToken transfers for their tranche by either:

Adding the aToken’s address to the blacklist
Enabling the whitelist
Note that even if a tranche admin blocks transfers, users will still be able to withdraw their funds directly from the system. However, if they are using their aTokens with a different protocol (e.g. depositing them in yield farm or using them as collateral for a loan elsewhere), they will not be able to remove their tokens from the outside protocol to withdraw from VMEX.

LendingPool.sol ([\_checkWhitelistBlacklist()](https://github.com/VMEX-finance/vmex/blob/e1c910c6cda1988524841684ed1f37fd649450b3/packages/contracts/contracts/protocol/lendingpool/LendingPool.sol#L79), [finalizeTransfer()](https://github.com/VMEX-finance/vmex/blob/e1c910c6cda1988524841684ed1f37fd649450b3/packages/contracts/contracts/protocol/lendingpool/LendingPool.sol#L605)):

```
// @audit "user" is always either msg.sender or "to" address; never the token transfer's "from".
function _checkWhitelistBlacklist(uint64 trancheId, address user) internal view {
    if (trancheParams[trancheId].isUsingWhitelist) {
        require(
            _usersConfig[user][trancheId].configuration.getWhitelist(),
            Errors.LP_NOT_WHITELISTED_TRANCHE_PARTICIPANT
        );
    }
    require(!_usersConfig[user][trancheId].configuration.getBlacklist(), Errors.LP_BLACKLISTED_TRANCHE_PARTICIPANT);
}

function checkWhitelistBlacklist(uint64 trancheId, address onBehalfOf) internal view {
    _checkWhitelistBlacklist(trancheId, msg.sender);
    if (onBehalfOf != msg.sender) {
        _checkWhitelistBlacklist(trancheId, onBehalfOf);
    }
}
...
// @audit this is called from the aToken contract in AToken._transfer()
function finalizeTransfer(
        address asset,
        uint64 trancheId,
        address from,
        address to,
        uint256 amount,
        uint256 balanceFromBefore,
        uint256 balanceToBefore
    ) external override whenTrancheNotPausedAndExists(trancheId) {
        require(msg.sender == _reserves[asset][trancheId].aTokenAddress, Errors.LP_CALLER_MUST_BE_AN_ATOKEN);
        // @audit The "from" address is not passed to this check. By blacklisted (or not whitelisting)
        //        the aToken address, a tranche admin can cause this to always revert.
        checkWhitelistBlacklist(trancheId, to);
        ...
}
```

[AToken.\_transfer():](https://github.com/VMEX-finance/vmex/blob/e1c910c6cda1988524841684ed1f37fd649450b3/packages/contracts/contracts/protocol/tokenization/AToken.sol#L481)

function \_transfer(address from, address to, uint256 amount, bool validate) internal {
address underlyingAsset = \_underlyingAsset;
ILendingPool pool = \_pool;

    ...
    // @audit "validate" is true for standard transfer() and transferFrom() calls; not on liquidations
    if (validate) {
        pool.finalizeTransfer(
            underlyingAsset,
            _tranche,
            from,
            to,
            amount,
            fromBalanceBefore,
            toBalanceBefore
        );
    }

### Impact

Medium. Sender (from) will never be impacted by the whitelist/blacklist. AToken transfers can be broken by untrusted tranche admins.

### Recommendation

To correctly enforce whitelist/blacklist, recommend using the function \_checkWhitelistBlacklist(uint64 trancheId, address user) twice to check the passed addresses (the to and the from), rather than checkWhitelistBlacklist() (no underscore), which checks msg.sender and to.

- checkWhitelistBlacklist(trancheId, to);

* \_checkWhitelistBlacklist(trancheId, from);
* \_checkWhitelistBlacklist(trancheId, to);
  To prevent tranche admins from blocking all transfers, consider only allowing the whitelist to be enabled before the tranche is truly “active” and allowing deposits (e.g. can only be enabled before batchInitReserve()). If that is not desirable, consider adding a ~24h timelock before the whitelist becomes active so that a global/emergency admin has an opportunity to prevent the change - and also so users can react.

## [Low/Info - Blacklisted users are considered by the system to have active borrows](https://reports.yaudit.dev/reports/06-2023-VMEX/#23-informational---blacklisted-users-are-considered-by-the-system-to-have-active-borrows)

Due to the bitmap modifications made to the UserConfiguration library, a blacklisted user will be reported as having an active borrow even if they have none. Note that even if a user is blacklisted, the intended functionality of the system is to still allow them to repay their debt + withdraw their funds. That ability is not impacted by this issue.

### Technical Details

As a part of the changes to the AAVE v2 protocol, VMEX added whitelist/blacklist functionality. A user’s inclusion in these lists is determined by the most significants bits in their UserConfiguration.data bitmap. Consider the most significant bits below, as well the way that isBorrowingAny() is performed:

```solidity
UserConfiguration.data
whitelisted user
0b10000000000000000000000...
blacklisted user
0b01000000000000000000000...
BORROWING_MASK
0b01010101010101010101010...
    function isBorrowingAny(...) internal pure returns (bool):
        return self.data & BORROWING_MASK != 0;
```

This will return true for a blacklisted user. Most of the other functions in UserConfiguration account for the added whitelist/blacklist most significant bits, but isBorrowingAny() does not.

There is little impact to the system however, as isBorrowingAny() is only called at the beginning of GenericLogic.balanceDecreaseAllowed() to short circuit and return early to save gas. There is no risk from this that a user is borrowing and isBorrowingAny() returns they are not, just a false positive (i.e. they are blacklisted + not borrowing). This will then then be caught by either the !userConfig.isUsingAsCollateral(...) check, which will return accurately for the specific collateral, or the later check for if (vars.totalDebtInETH == 0) {return true;}.

```solidity
function balanceDecreaseAllowed(...) external returns (bool) {
    if (
        !userConfig.isBorrowingAny() ||
        !userConfig.isUsingAsCollateral(
            reservesData[params.asset][params.trancheId].id
        )
    ) {
        return true;
    }
    ...
    (...,vars.totalDebtInETH,,...,,...) = calculateUserAccountData(...);
    if (vars.totalDebtInETH == 0) {
      return true;
    }
```

### Impact

Informational. There is minimal impact on the system, but might cause errors on frontends.

### Recommendation

Modify UserConfiguration.BORROWING_MASK to account for the new blacklist bit:

```
// 0b010101...
-BORROWING_MASK = 0x5555555555555555555555555555555555555555555555555555555555555555
// 0b000101...
+BORROWING_MASK = 0x1555555555555555555555555555555555555555555555555555555555555555
```

##

```

```
