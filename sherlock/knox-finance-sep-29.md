# Sherlock / knox-finance-29
***************************
***************************

# [H-01] Wrong implementation of orderbook can make user can't get their fund back

## Lines of code

[https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/OrderBook.sol#L240](https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/OrderBook.sol#L240)[https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/OrderBook.sol#L182-L190](https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/OrderBook.sol#L182-L190)

## Summary

When a user remove an order, next user call `addLimitOrder` can override the latest order with his/her order. It will make one who is owner of that latest order lose their fund.

## Vulnerability Detail

Function `_remove` will decrease value of `index.length` by 1 when an order is removed

```
// url = <https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/OrderBook.sol#L240>
function _remove(Index storage index, uint256 id) internal returns (bool) {
    index.length = index.length > 0 ? index.length - 1 : 1;
    ...
}

```

Instead of reserving `id` of removed order to reuse for next created order, function `_insert` use the id of new order is `index.length + 1`

```
// url = <https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/OrderBook.sol#L165-L172>
function _insert(
    Index storage index,
    int128 price64x64,
    uint256 size,
    address buyer
) internal returns (uint256) {
    index.length = index.length > 0 ? index.length + 1 : 1;
    uint256 id = index.length;
    ...
}

```

It will override the latest order with new order's data.

For example

- Alice create an order with price = 10 --> `id = 1, index.length = 1`
- Bob create an order with price = 20 --> `id = 2, index.length = 2`
- Alice cancel order `id = 1` --> `index.length = 1`
- Candice create new order with price = 30
    - At this time, new order will have `id = index.length + 1 = 1 + 1 = 2`. It will override the state of Bob's order: price from 20 -> 30

## Impact

User whose order is overrided can't withdraw their refund `ERC20` and their exercised tokens.

## Code Snippet

```
it.only("bug", async() => {
    const totalContracts = await auction.getTotalContracts(epoch);

    const buyer1OrderSize = totalContracts.div(5);
    const buyer2OrderSize = totalContracts.div(5);
    const buyer3OrderSize = totalContracts.div(5);

    // buyer1 create order with price = 10
    await asset
      .connect(signers.buyer1)
      .approve(addresses.auction, ethers.constants.MaxUint256);
    await auction.addLimitOrder(epoch, 10, buyer1OrderSize);

    // buyer2 create order with price = 20
    await asset
      .connect(signers.buyer2)
      .approve(addresses.auction, ethers.constants.MaxUint256);
    await auction
      .connect(signers.buyer2)
      .addLimitOrder(epoch, 20, buyer2OrderSize);

    // order with id = 2 have price = 20
    expect( (await auction.getOrderById(epoch, 2)).price64x64 ).to.equal(20);

    // buyer1 cancel order with id = 1
    await auction.connect(signers.buyer1).cancelLimitOrder(epoch, 1);

    // buyer3 create order with price = 30
    await asset
      .connect(signers.buyer3)
      .approve(addresses.auction, ethers.constants.MaxUint256);
    await auction
      .connect(signers.buyer3)
      .addLimitOrder(epoch, 30, buyer3OrderSize);

    // order with id = 2 have price = 30 --> nervous
    expect( (await auction.getOrderById(epoch, 2)).price64x64 ).to.equal(30);
});

```

To check with test, u can use this file
[https://gist.github.com/Trumpero/adbcd84c33f71856dbf379f581e8abbb](https://gist.github.com/Trumpero/adbcd84c33f71856dbf379f581e8abbb)
I write one more describe `::Bug` beside your original describe `::Auction` in file `Auction.behavior.ts` (just too lazy to write a new one).

## Tool used

Hardhat

## Recommendation

Use an array to store unused (removed) id, then assign each id to the new limit order created instead of using `index.length`.

***************************
***************************
# [H-02] User can't withdraw from auction when number of sold contracts bigger than totalContracts

## Lines of code

[https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/AuctionInternal.sol#L312-L335](https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/AuctionInternal.sol#L312-L335)

## Summary

## Vulnerability Detail

Function `_previewWithdraw` check if `totalContractsSold + data.size` is bigger than `auction.totalContracts` or not. If true it will increase `fill` amount by the difference between `auction.totalContracts` and `totalContractSolds` and `refund` amount will be increased by the remaining.

```
// url = <https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/AuctionInternal.sol#L312-L322>
if (
    totalContractsSold + data.size >= auction.totalContracts
) {
    // if part of the current order exceeds the total contracts available, partially
    // fill the order, and refund the remainder
    uint256 remainder =
        auction.totalContracts - totalContractsSold;

    cost = lastPrice64x64.mulu(remainder);
    fill += remainder;
}

```

But this updatation is just true when `auction.totalContracts >= totalContractsSold`. For the case when `auction.totalContracts < totalContracsSold`, this function will revert because of underflow issue.

**For example**

- Auction has `totalContracts = 10`
- Alice buy 3 contracts with price = `maxPrice`
- Bob buy 8 contracts with price = `maxPrice`
- Candice buy 1 contracts with price = `maxPrice`

When the auction end, Candice calls `withdraw(0)` to get her refund (cause Alice and Bob will win all the contracts). But when function `withdraw()` call to function`_previewWithdraw` it will reverted. Here is the detail of for loops in function `_previewWithdraw`

- i = 0
    - `data.buyer = Alice`
    - `data.size = 3`
    - `totalContractsSold = 0 + 3 = 3`
- i = 1
    - `data.buyer = Bob`
    - `data.size = 8`
    - `totalContractsSold = 3 + 8 = 11`
- i = 2
    - `data.buyer = Candice`
    - `data.size = 1`
    - `remainder = totalContracts - totalContractSolds = 10 - 11 = -1 < 0` --> revert here

## Impact

User can't get their tokens back

## Code Snippet

```
const buyer1OrderSize = totalContracts.add(1); //.sub(totalContracts.div(10));
const buyer2OrderSize = totalContracts.sub(10);
const buyer3OrderSize = totalContracts.div(10).mul(2);

// buyer1 buy totalContracts + 1 contracts
await asset
  .connect(signers.buyer1)
  .approve(addresses.auction, ethers.constants.MaxUint256);
await auction.addLimitOrder(epoch, maxPrice64x64, buyer1OrderSize);

// buyer2 buy totalContracts - 10
await asset
  .connect(signers.buyer2)
  .approve(addresses.auction, ethers.constants.MaxUint256);
await auction
  .connect(signers.buyer2)
  .addLimitOrder(epoch, maxPrice64x64, buyer2OrderSize);

// buyer2 call withdraw to get the refund --> revert
await expect(auction.connect(signers.buyer2).withdraw(0)).to.reverted;

```

To check with test, u can use this file
[https://gist.github.com/Trumpero/575d1b3b035e2f8a4e104d1e6062c648](https://gist.github.com/Trumpero/575d1b3b035e2f8a4e104d1e6062c648)
I write one more describe `::Bug` beside your original describe `::Auction` in file `Auction.behavior.ts` (just too lazy to write a new one).

## Tool used

Hardhat

## Recommendation

- Add more one more `if` condition when `totalContractsSold > auction.totalContracts`. For this case we will increase `refund` amount by `data.size`.

***************************
***************************

# [M-01] Oracle data can be outdated

## Lines of code

[https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/pricer/PricerInternal.sol#L50-L52](https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/pricer/PricerInternal.sol#L50-L52)

## Summary

The `lastRoundData()`'s parameters according to [Chainlink](https://docs.chain.link/docs/data-feeds/price-feeds/api-reference/) are the following:

```
function latestRoundData() external view
    returns (
        uint80 roundId,             //  The round ID.
        int256 answer,              //  The price.
        uint256 startedAt,          //  Timestamp of when the round started.
        uint256 updatedAt,          //  Timestamp of when the round was updated.
        uint80 answeredInRound      //  The round ID of the round in which the answer was computed.
    )

```

But function `_lastestAnswer64x64` just get the parameter `price`, and hasn't checked the others.

```
function _latestAnswer64x64() internal view returns (int128) {
    (, int256 basePrice, , , ) = BaseSpotOracle.latestRoundData();
    (, int256 underlyingPrice, , , ) =
        UnderlyingSpotOracle.latestRoundData();

    return ABDKMath64x64.divi(underlyingPrice, basePrice);
}

```

It can lead to a potential risk for protocol

## Vulnerability Detail

A strong reliance on the price feeds has to be also monitored as recommended on the [Risk Mitigation section](https://docs.chain.link/docs/data-feeds/selecting-data-feeds/#risk-mitigation). There are several reasons why a data feed may fail such as unforeseen market events, volatile market conditions, degraded performance of infrastructure, chains, or networks, upstream data providers outage, malicious activities from third parties among others.

Chainlink recommends using their data feeds along with some controls to prevent mismatches with the retrieved data. Along some recommendations, the feed can include circuit breakers (for extreme price events), contract update delays (to ensure that the injected data into the protocol is fresh enough), manual kill-switches (to cease connection in case of found bug or vulnerability in an upstream contract), monitoring (control the deviation of the data) and soak testing (of the price feeds).

The retrieved price of the `BaseSpotOracle` and `UnderlyingSpotOracle` can be outdated and used anyways as a valid data because no timestamp tolerance of the update source time is checked while storing the return parameters of `latestRoundData()` inside `_latestAnswer64x64` as recommended by Chainlink. The usage of outdated data can impact on how the Payment terminals work regarding pricing calculation and value measurement.

## Impact

Function `latestAnswer64x64` is used to get the spot price to initialize the `maxPrice64x64` and `minPrice64x64` of a new epoch

```
// url = <https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/vault/VaultInternal.sol#L579-L581>

// fetches the spot price of the underlying
int128 spot64x64 = l.Pricer.latestAnswer64x64();

...
// calculates the auction max price using the strike price further (ITM)
int128 maxPrice64x64 =
    l.Pricer.getBlackScholesPrice64x64(
        spot64x64,
        lastOption.strike64x64,
        timeToMaturity64x64,
        l.isCall
    );

// calculates the auction min price using the offset strike price further (OTM)
int128 minPrice64x64 =
    l.Pricer.getBlackScholesPrice64x64(
        spot64x64,
        offsetStrike64x64,
        timeToMaturity64x64,
        l.isCall
    );

```

If the retrieved value of the oracle is outdated, `minPrice` and `maxPrice` will suffer from this mismatch. It will affect the `_priceCurve64x64` function and make the [validation](https://github.com/sherlock-audit/2022-09-knox-Trumpero/blob/925d7ab97869e77bad6bce8bde2f39bfd5abb32e/knox-contracts/contracts/auction/AuctionInternal.sol#L500-L513) of market order wrong

## Tool used

Manual Review

## Recommendation

Follow the Chainlink's recommendation:
[https://docs.chain.link/docs/data-feeds/#check-the-timestamp-of-the-latest-answer](https://docs.chain.link/docs/data-feeds/#check-the-timestamp-of-the-latest-answer)
It is recommended both to add also a tolerance that compares the updatedAt return timestamp from latestRoundData() with the current block timestamp and ensure that the priceFeed is being updated with the required frequency.