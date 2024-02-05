# MIM Hack
## background
 
Cauldrons are implemented BentoBox contracts.
BentoBox is a vault for yield, it offers loan service, also flash-loans and it repays users with earned fees.

The Vault has a collateral token and a liquidity token that can be deposited
In the codebase, borrow shares are referred to as `base`/part and borrow assets amounts are referred to as `elastic`/amount.

The root cause of this exploit is a *precision loss bug* in the function responsible to calculate shares. The attacker drained MIM liquidity from the `yvCrv3Crypto` and `magicAPE` cauldrons, taking advantage of the incorrect debt calculation. Exploited CauldronV4 contracts:

- `yvCrv3Crypto` 0x7259e152103756e1616A77Ae982353c3751A6a90

- `magicAPE` 0x692887E8877C6Dd31593cda44c382DB5b289B684

## Root Cause

Consider this function:
```
    function toBase(
        Rebase memory total,
        uint256 elastic,
        bool roundUp
    ) internal pure returns (uint256 base) {
        if (total.elastic == 0) {
            base = elastic;
        } else {
            base = (elastic * total.base) / total.elastic;
            if (roundUp && (base * total.elastic) / total.base < elastic) {
                base++;
            }
        }
    }
```
when we can control the parameters of the formula that calculates `base`, we can trick the function to to perform calculations that results in a result that is not an integer, making solidity to discard the part after the comma.

This function is called by DegenBox `toShare` function, that is responsible to calculate share of users based on token amount.
```
    function toShare(
        IERC20 token,
        uint256 amount,
        bool roundUp
    ) external view returns (uint256 share) {
        share = totals[token].toBase(amount, roundUp);
    }
```
This function is both called from Cauldron in both `borrow` and `repay` functions.

In order to exploit the issue and make it profitable, the attacker first creates the situation where they can control such expression, it lower the base and elastic amount to 100 through repayForAll and repay.

After the first stage the totalBorrow amounts are:
- base: 3
- elastic: 100

at this point they lower base to zero repaying 1 wei a time. First round calculation:
```
1 * 3 / 100 = 0.03
```
Second round:
```
1 * 2 / 99 = 0.03030303
```
Third round:
```
1 * 1 / 98 = 0.01020408
```
Since roundUp is true, the function will increase the result (base) by one, each time. Since we're repaying the base amount is lowered by one.
At this point totalBorrow values are:
- base: 0
- elastic: 97

After adding little amount of collateral (100), they can repeatly borrow 1 and repay 1, this would cause the precision loss to inflate a lot elastic value keeping base to 1.

Using the same formula, we can see that each round the base rest the same (1) and elastic will roughly double each round:
1. base 1, elastic 195
2. base 1, elastic 389
3. base 1, elastic 777
4. base 1, elastic 1553

and so on..

## Exploitation

The exploitation strategy is divived in five steps:

1. Flashloan MIM token with Degenbox

2. deposit on Cauldron and repay most of the debt using repayForAll and repay to decrease base (shares) to zero and elastic (assets) lower than 100.

In order to empty the pool, the attacker uses repayForAll and repay functions, this creates the ideal situation to inflate the shares (base: 0, elastic: 97).
It wasn't possible to do in a single call because there is a 'require' statement in `repayForAll` that prevents the elastic amount from being lowered to less than 1000 * 1e18.

3. Repeatedly borrow(1) and repay(1) to inflate the share price

4. Add collateral and borrow a large amount of MIM token

5. Repay flashloan and take profit

## References

[initial analysis from Kankodu](https://twitter.com/kankodu/status/1752581744803680680)

[rekt.news analysis](https://rekt.news/abra-rekt/)

[attack tx 1](https://phalcon.blocksec.com/explorer/tx/eth/0x26a83db7e28838dd9fee6fb7314ae58dcc6aee9a20bf224c386ff5e80f7e4cf2)

[attack tx 2](https://phalcon.blocksec.com/explorer/tx/eth/0xdb4616b89ad82062787a4e924d520639791302476484b9a6eca5126f79b6d877)