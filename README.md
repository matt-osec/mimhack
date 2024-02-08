# MIM Hack
## Introduction

Terminology:

- *base* / *part*: borrow shares

- *elastic* / *amount*: borrow assets

When a user borrows a certain amount of money, he will receive shares based on the current ratio of assets (`elastic`) and shares (`base`).  As interest is owed by the user, `elastic` will increase without increasing `base`, and this in turn will increase the proportional amount that the user has to pay back.

Anyone can repay anyone debt through `repayForAll` function, or repay just for a single user calling `repay`.

### Root Cause Analysis

The root cause is a share deflation attack, made possible because the `repayForAll` function doesn't properly adjust the ratio between shares and assets. This leads to a specific situation where the attacker could borrow an arbitrary amount of tokens from the vault and then withdraw them, bypassing the health check.

Here is `repayForAll` function and the important part:

```
    function repayForAll(uint128 amount, bool skim) public returns(uint128) {
        
        ...

        uint128 previousElastic = totalBorrow.elastic;

        require(previousElastic - amount > 1000 * 1e18, "Total Elastic too small");

        totalBorrow.elastic = previousElastic - amount;

        ...

    }
```

As we can see it just subtracts the amount paid to `total.elastic` without adjusting the shares. In itself it is not a problem, it just depletes the value of users' shares.

The following part of the attack is done thanks to this code:
```
    function toBase(
        Rebase memory total,
        uint256 elastic,
        bool roundUp
    ) internal pure returns (uint256 base) {
        
        ...
        
        } else {
            base = (elastic * total.base) / total.elastic;
            if (roundUp && (base * total.elastic) / total.base < elastic) {
                base++;
            }
        }

        ...
```

The attacker was able to first have `total.base = 98` and `total.elastic = 1`, in this way they can first inflate `total.base` value to make shares even more insignificant and have `total.elastic = 0`.

This creates an edge case where, with some collateral, they could borrow and mint an arbitrary amount of tokens.
```
    function toBase(
        Rebase memory total,
        uint256 elastic,
        bool roundUp
    ) internal pure returns (uint256 base) {
        if (total.elastic == 0) {
            base = elastic;
        } else {

        ...

        }
    }
```

As we can see, the branch should only be hit when there aren't borrowed tokens, however the function doesn't check if `total.base` is also zero, making this attack possible.


## Exploitation

The exploitation happens in six steps:

1. Flashloan MIM to perform the attack

2. Deposit on Cauldron and repay most of the debt using `repayForAll`, this would break the share ratio, then complete the share deflation manually repaying users' debt. The latter is also required because there's a check on `repayForAll` that prevents the elastic value from going below 1000 ether. At the end of this step the count of `total.base` count is 97 and `total.elastic` is 0.

3. Deposit 3CRV LP tokens (collateral tokens for the vault), this would help following phases of the attack.

4. Attacker puts up 100 wei of collateral and repeatedly borrows 1 and repays 1. Due to the initial desync of the ratio, after many iterations there will be 1 `elastic` (assets) and very high amount of `base` (shares). This because borrowing 1 wei of asset will double the amount of shares every round. Since the protocol round in its favour, even if the repay amount is zero it will round to 1 letting the user repay 1 share. At the end they pay back the last 1 wei and the final situation `total.elastic` is zero and `total.base` is a very high number.

5. Now the attacker can borrow funds, since the value of `elastic` is zero the system lets them mint the same amount of shares as assets. They can now withdraw the funds bypassing the health check because their shares are worth almost nothing. In particular, since `totalBorrow.base` is a big number and it is in the denominator it will shrink the borrowPart enough to pass the health check.

6. Once they have withdrawn the MIM tokens, they can repay the flashloan and get profit.

## Fix

In order to fix the issue, the edge case explained should be handled. It shouldn't be possible to mint arbitraty amount of token when the vault is not empty.

## References

[initial analysis from Kankodu](https://twitter.com/kankodu/status/1752581744803680680)

[rekt.news analysis](https://rekt.news/abra-rekt/)

[attack tx 1](https://phalcon.blocksec.com/explorer/tx/eth/0x26a83db7e28838dd9fee6fb7314ae58dcc6aee9a20bf224c386ff5e80f7e4cf2)

[attack tx 2](https://phalcon.blocksec.com/explorer/tx/eth/0xdb4616b89ad82062787a4e924d520639791302476484b9a6eca5126f79b6d877)
