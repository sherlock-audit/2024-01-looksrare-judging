Great Carmine Fox

medium

# Reservoir "bid-ask midpoint" oracle can be used instead of the "floor" oracle

## Summary
Users can use the reservoir `Collection bid-ask midpoint` instead of the `Collection floor` pricing.

## Vulnerability Detail
Reservoir NFT oracle offers three main different pricing:
- Collection floor
- Collection bid-ask midpoint
- Collection top bid oracle

The price returned by the oracle determines the amount of entries a user should get when depositing an NFT, the protocol is meant to use the `Collection floor` but a user might decide to use the `Collection bid-ask midpoint` instead because they both return the same [`messageHash`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1613-L1626).

Let's take as an example the pudge penguins collection. 

Querying reservoir for the [`Collection floor`](https://docs.reservoir.tools/reference/getoraclecollectionsflooraskv6):
```bash
curl --request GET \
     --url 'https://api.reservoir.tools/oracle/collections/floor-ask/v6?kind=twap&twapSeconds=3600&collection=0xbd3531da5cf5857e7cfaa92426877b022e612cf8' \
     --header 'accept: */*' \
     --header 'x-api-key: demo-api-key'
```

returns:
```bash
{
  "price": 18.18581,
  "message": {
    "id": "0x15530ce04dad67f6b6e8213c7e5552bc046bb3c404fe4b338dca757a684c684c",
    "payload": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000fc60fb68a730bb8e",
    "timestamp": 1706006459,
    "chainId": "1",
    "signature": "0x27d3c605f06b0627263971f73f687a49a9b395de26975c2643c3eb0236af8096523a22911f327c100ca3578cf7ae599c5c0c711dca043718f549ea1dc51f69ed1c"
   }
}
```

Querying reservoir for the [`Collection bid-ask midpoint`](https://docs.reservoir.tools/reference/getoraclecollectionsbidaskmidpointv1):
```bash
curl --request GET \
     --url 'https://api.reservoir.tools/oracle/collections/bid-ask-midpoint/v1?kind=twap&twapSeconds=3600&collection=0xbd3531da5cf5857e7cfaa92426877b022e612cf8' \
     --header 'accept: */*' \
     --header 'x-api-key: demo-api-key'
```
returns:
```bash
{
  "price": 18.0121,
  "message": {
    "id": "0x15530ce04dad67f6b6e8213c7e5552bc046bb3c404fe4b338dca757a684c684c",
    "payload": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f9f7d4c8fff5f31d",
    "timestamp": 1706006939,
    "chainId": "1",
    "signature": "0x7adee5fad3928460773af76aaaa1a969fb1d196de8c879353d21b36d6f30fb091fd177a0b398ee7ed05af8c79e7c702231b4f1cc7f72d38fb17e37b50d96d4f41b"
  }
}
```

As shown the `message.id` is the same, but the price is different. Because the `message.id` is the same it's possible for an user to choose which pricing to use at its discretion.

Since the `Collection bid-ask midpoint` price is always lower than the `Collection floor` this can be used by an attacker to lower the amount of entries a depositor will get and increase his chances of winning as a consequence. 

Let's suppose Alice is participating in round where `valuePerEntry = 0.1ETH` and she already owns 500 entry tickets because of some ETH she deposited.

Bob also wants to join and sends a [`deposit()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L305) transaction to deposit 10 pudgy penguins. He gets the reservoir oracle signature for the pudgy penguins `Collection floor` for a price of `18.18581ETH` per NFT and submit his transaction. Bob is expecting `18.18/0.1 = 181*10 = 1810` entry tickets. 

Alice notices this and frontruns Bob deposit by depositing a pudgy penguin herself, but she uses the reservoir pricing for the `Collection bid-ask midpoint` instead, this sets the price of a pudgy penguin to `18.0121ETH` for the whole round. Alice receives `18.01/0.1 = 180` entry tickets for this.

At this point Bob transaction goes through, but he receives `18.01/0.1 = 180*10 = 1800` entry tickets instead of `1810`. 

The odds of winning are:
- Alice: (500 + 180)/(500+180+1800) = `27.41%`
- Bob: (1800)/(500+180+1800) = `72.58%`

If Alice deposited a pudgy penguin at the price intended by the protocol instead of the lowered one the odds of winning would have been:
- Alice: (500+181)/(500+180+1810) = `27.34%`
- Bob: (1810)/(500+180+1810) = `72.69%`

## Impact
An attacker is able to increase his chances of winning by lowering the entry tickets received by other participants.

## Code Snippet
 - [`messageHash`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1613-L1626)
## Tool used

Manual Review

## Recommendation

I'm not sure why Reservoir allows this to happen. Given that the protocol admins are trusted a fix to this could be to add an extra signature from a protocol-controlled private key on top of the reservoir oracle signature to ensure only the `Collection floor` can be used.
