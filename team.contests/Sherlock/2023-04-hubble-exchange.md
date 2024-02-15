# [M-1] InsuranceFund#syncDeps - Governance can change vUSD address at any time and deposits can get lost
https://github.com/sherlock-audit/2023-04-hubble-exchange-judging/issues/155

## Summary
The smart contract InsuranceFund.sol contains a potentially harmful function, syncDeps(), which allows for the contract address of vusd to be changed at any time by the Governance address. In some edge cases, this may cause users' funds to be lost.

Vulnerability Detail
The syncDeps() function can change the contract address of vusd without any restrictions. This change can interfere with the transactions of depositing and withdrawing vusd by users, possibly causing a loss of funds. If a new address is set for vusd between a user's deposit and withdraw transactions, the user could end up withdrawing a different vusd variant, such as VUSDv2, while having initially deposited VUSD.

## Impact
The users' funds are at risk of being lost due to this vulnerability. If the syncDeps() function is called and vusd is set to a new address in the middle of deposit and withdraw transactions, users could end up withdrawing nothing, hence suffering a fund loss.

## Code Snippet
https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/InsuranceFund.sol#L321

```javascript
function syncDeps(IRegistry _registry) public onlyGovernance {
    vusd = IERC20(_registry.vusd());
    marginAccount = _registry.marginAccount();
}
```
## Tool used
Manual Review

## Recommendation
A recommended solution is to consider making vusd unchangeable. However, if migration of vusd must be considered for future upgrades, you should change the syncDeps() function to ensure that the balance after the change is not less than the balance before the change. Here is a recommended change to the function:

```diff
function syncDeps(IRegistry _registry) public onlyGovernance {
+   uint _balance = balance();
    vusd = IERC20(_registry.vusd());
+   require(balance() >= _balance);
    marginAccount = _registry.marginAccount();
}
```
This will ensure that the balance of vusd does not decrease, preventing potential losses to the users' funds.

# [M-2] Oracle#getUnderlyingPrice - No stale price checks could lead to price manipulation by the user
https://github.com/sherlock-audit/2023-04-hubble-exchange-judging/issues/157 
## Summary
The smart contract Oracle.sol does not implement stale price checks by sanitizing the return values potentially leading to outdated and inaccurate oracle data.

## Vulnerability Detail
The getUnderlyingPrice() function in the Oracle.sol contract fetches the price of an asset from the Chainlink Oracle but doesn’t check if the price data is stale. This oversight could result in outdated and potentially inaccurate Oracle data if there are problems reaching consensus (e.g., Chainlink nodes abandon the Oracle, chain congestion, vulnerability/attacks on the Chainlink system).

## Impact
Given the current market price, users could exploit this to execute transactions at stale prices, which can be exploited to borrow more assets than they should be able to.

## Code Snippet
https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/Oracle.sol#L24

```javascript
function getUnderlyingPrice(address underlying)
    virtual
    external
    view
    returns(int256 answer)
{
    if (stablePrice[underlying] != 0) {
        return stablePrice[underlying];
    }
    (,answer,,,) = AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData();
    require(answer > 0, "Oracle.getUnderlyingPrice.non_positive");
    answer /= 100;
}
```
## Tool used
Manual Review

## Recommendation
getUnderlyingPrice() should be updated to do additional checks to ensure the Oracle prices are not stale. The below variables should be returned and used: roundId, timestamp, and answeredInRound.

```diff
function getUnderlyingPrice(address underlying)
    virtual
    external
    view
    returns(int256 answer)
{
    if (stablePrice[underlying] != 0) {
        return stablePrice[underlying];
    }
- (,answer,,,) = AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData();
+ (uint80 roundId, int256 answer,uint256 timestamp,, uint80 answeredInRound) = AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData();+ require(answeredInRound >= roundId, "Stale price") 
+ require(timestamp != 0, "Round not complete")
   require(answer > 0, "Oracle.getUnderlyingPrice.non_positive");
   answer /= 100;
}
```

# [M-3] Oracle#getUnderlyingPrice - ChainLinkAdapterOracle will return the wrong price for asset if underlying aggregator hits minAnswer
https://github.com/sherlock-audit/2023-04-hubble-exchange-judging/issues/227 
## Summary
Chainlink Oracles have a built-in circuit breaker in case prices go outside predetermined minPrice and maxPrice price bands. Therefore, if an asset suffers a huge loss in value, such as the LUNA crash, the chainlink oracle will return the wrong prices, and the protocol can go into debt.

## Vulnerability Detail
The `Oracle.sol` contract uses a chainlink aggregator oracle to get the latest price for setting the index price in the protocol. However, if an asset listed on the exchange suffers a huge change in value, like that of the LUNA crash, the Chainlink oracle will return the wrong prices. The protocol will keep getting the set minPrice or maxPrice as the answer, while the real price might differ. Since the index price will be set wrong because of this, the funding rates will be wrong and users will suffer losses.

The referred code snippet where prices are fetched is as follows:

```javascript
function getUnderlyingPrice(address underlying)
        virtual
        external
        view
        returns(int256 answer)
    {
        if (stablePrice[underlying] != 0) {
            return stablePrice[underlying];
        }
        (,answer,,,) = AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData();
        require(answer > 0, "Oracle.getUnderlyingPrice.non_positive");
        answer /= 100;
    }
```
## Impact
The Oracle contract does not check if minPrice or maxPrice circuit breakers are hit by the chainlink aggregator. This might result in a loss for users of the protocol.

## Code Snippet
https://github.com/sherlock-audit/2023-04-hubble-exchange/blob/main/hubble-protocol/contracts/Oracle.sol#L24

## Tool used
Manual Review

## Recommendation
Check if minPrice/maxPrice circuit breakers are hit, and apply appropriate procedures if they are hit.

Reference
Venus on BSC was exploited similarly when LUNA crashed: https://rekt.news/venus-blizz-rekt/.