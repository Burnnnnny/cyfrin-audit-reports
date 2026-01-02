**수석 감사 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)

**보조 감사 (Assisting Auditors)**



---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 크리에이터 지급액 계산의 수학적 오류

**설명:** Tunnl 프로토콜은 3가지 종류의 수수료를 적용합니다.
- **고정 수수료 (Flat Fee)**: 광고주가 지불하는 고정 수수료로, 제안이 수락될 때 청구됩니다.
- **광고주 비율 수수료 (Advertiser Percentage Fee)**: 광고주에게 청구되는 비율 수수료입니다. 제안 금액에 추가로 부과되는 금액입니다.
- **크리에이터 비율 수수료 (Creator Percentage Fee)**: 크리에이터에게 청구되는 비율 수수료입니다. 크리에이터가 받는 금액을 기준으로 청구됩니다.

광고주는 최대 제안 금액 $maxOfferAmount$부터 시작하여 제안을 생성하며, 가능한 모든 수수료를 포함한 총 금액은 $maxOfferAmount * (1 + advertiserFee)+flatFee$로 계산됩니다.
이 금액은 `Offer::maxValueUsdc`에 저장되어 회계 처리의 여러 곳에서 사용됩니다.

검증이 성공하면 `TunnlTwitterOffers::sendFunctionsRequest()`가 호출되어 크리에이터에게 지급될 실제 금액을 가져옵니다.
이 함수는 요청과 함께 `FunctionClient::_sendRequest()` 함수를 호출하며, 최대 제안 금액이 요청 인수에 인코딩됩니다.
하지만 `maxCreatorPayment` 계산 시 `s_config.advertiserFeePercentageBP` 대신 `s_config.creatorFeePercentageBP`가 사용됩니다.
```solidity
        uint256 maxCreatorPayment = uint256((s_offers[offerId].maxValueUsdc - s_offers[offerId].flatFeeUsdc) * 10000)
            / uint256(10000 + s_config.creatorFeePercentageBP);//@audit-issue this should be 10000+s_config.advertiserFeePercentageBP

        console.log("maxCreatorPayment: %s", maxCreatorPayment);//@audit-info

        bytes[] memory bytesArgs = new bytes[](4);
        bytesArgs[0] = abi.encode(offerId);
        bytesArgs[1] = abi.encodePacked(s_offers[offerId].creationDate);
        bytesArgs[2] = abi.encodePacked(maxCreatorPayment);//@audit-info maximum offer amount relayed
        bytesArgs[3] = abi.encodePacked(s_offers[offerId].offerDurationSeconds);
        req.setBytesArgs(bytesArgs);

        bytes32 requestId = _sendRequest(
            req.encodeCBOR(),
            s_config.functionsSubscriptionId,
            s_config.functionsCallbackGasLimit,
            s_config.functionsDonId
        );
```
현재 구현은 콘텐츠 성과(예: 좋아요, 조회수 등)에 따라 지급액을 변경하지 않으며, 전달된 최대 금액이 크리에이터에게 전액 지급됩니다. 따라서 크리에이터에 대한 최종 지급액이 잘못됩니다.

이 오류는 두 가지 문제를 일으킬 수 있습니다.
- `creatorFeePercentageBP < advertiserFeePercentageBP`인 경우, `maxCreatorPayment`가 실제 최대 제안 금액보다 커져서 `TunnlTwitterOffers::fulfillRequest` 함수의 지급 분배가 `Exceeds max` 오류와 함께 되돌려집니다(revert).
- `creatorFeePercentageBP > advertiserFeePercentageBP`인 경우, `maxCreatorPayment`가 실제 최대 제안 금액보다 작아져서 크리에이터가 더 적은 금액을 받게 됩니다.

현재 테스트 스위트는 첫 번째 경우에 해당하지만, 요청 이행이 `calculatepayment.js` 스크립트를 사용하여 노드가 설정하는 실제 값이 아닌 인위적인 값으로 시뮬레이션되기 때문에 오류가 포착되지 않습니다.
```solidity
    function test_PayOut() public {
        // Define the amount to be paid in USDC
        uint256 amountPaidUsdc = uint256(uint256(100e6));
        test_Verification_Success();
        // Warp time for payout and perform Chainlink automation
        vm.warp(block.timestamp + (1 weeks - 100));
        performUpkeep();
        // Fulfill request for payout  with PayOutamount
        mockFunctionsRouter.fulfill(
            address(tunnlTwitterOffers),
            functionsRequestIds[offerId],
            abi.encode(amountPaidUsdc),//@audit-info should be same to the maxCreatorPayout in the current implementation
            ""
        );

        // Assert balances after payout
        assertEq(mockUsdcToken.balanceOf(advertiser), 0);
        assertEq(mockUsdcToken.balanceOf(address(tunnlTwitterOffers)), (0));
        assertEq(
            mockUsdcToken.balanceOf(contentCreator),
            amountPaidUsdc - (amountPaidUsdc * tunnlTwitterOffers.getConfig().creatorFeePercentageBP) / 10000
        );
    }
```

**영향:** 프로토콜이 전혀 작동하지 않거나 크리에이터가 제안된 금액보다 적은 금액을 체계적으로 받게 되므로 이 영향을 치명적(CRITICAL)으로 평가합니다.

**개념 증명 (Proof Of Concept):**
요청 인수로 설정된 실제 `maxCreatorPayment` 값을 확인하기 쉽지 않으므로 `TunnlTwitterOffers::sendFunctionsRequest()` 함수에 값을 로깅하는 라인을 삽입했습니다.

```solidity
console.log("maxCreatorPayment: %s", maxCreatorPayment);
```
`test_PayOut()` 테스트 케이스를 실행하면 다음과 같이 출력됩니다.

```bash
[PASS] test_PayOut() (gas: 531524)
Logs:
  100000000 110000000 <- Max offered amount and maxValueUsdc
  maxCreatorPayment: 102439024 <- This must be the same to the offered amount 100e6
  maxCreatorPayment: 102439024
```
**권장 완화 방법:** `sendFunctionsRequest()` 함수를 아래와 같이 수정하십시오. 팀은 `s_offers[offerId]` 대신 `s_config`의 수수료 비율 값을 사용하는 또 다른 문제를 보고했으며, 아래의 완화 조치에는 해당 내용도 반영되어 있습니다.
```diff
        uint256 maxCreatorPayment = uint256((s_offers[offerId].maxValueUsdc - s_offers[offerId].flatFeeUsdc) * 10000)//@audit-info why not use the percentage directly - (s_offers[offerId].maxValueUsdc - s_offers[offerId].flatFeeUsdc) * (10000 - s_config.creatorFeePercentageBP) / 10000
--            / uint256(10000 + s_config.creatorFeePercentageBP);
++           / uint256(10000 + s_offers[offerId].advertiserFeePercentageBP);
```
**Tunnl:** [PR 37](https://github.com/tunnl-io/Tunnl-Contracts/pull/37/files)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## 중간 위험 (Medium Risk)


### 공격자가 거짓 제안으로 프로토콜을 플러딩(flood)하여 DoS를 유발할 수 있음

**참고:** 이 발견 사항은 프로토콜이 보류 중(Pending)에서 만료됨(Expired)으로의 상태 전환 구현이 불완전한 다른 문제를 수정했다고 가정합니다.

**설명:** 프로토콜은 모든 제안 집합을 유지하며 Chainlink 자동화 함수인 `checkUpkeep()` 및 `performUpkeep()`에 의해 관리됩니다.
`checkUpkeep()` 함수는 `s_offersToUpkeep`의 모든 제안을 반복하고 유지 관리의 필요성을 확인합니다.
```solidity
TunnlTwitterOffers.sol
299:     function checkUpkeep(bytes calldata) external view override returns (bool upkeepNeeded, bytes memory performData) {
300:         bytes32[] memory offersToUpkeep = new bytes32[](0);
301:         for (uint256 i = 0; i < s_offersToUpkeep.length(); i++) {
302:             bytes32 offerId = s_offersToUpkeep.at(i);
303:             if (
304:                 _needsVerificationRequest(s_offers[offerId].status, s_offers[offerId].dateToAttemptVerification) ||
305:                 _needsExpiration(s_offers[offerId].status, s_offers[offerId].payoutDate, s_offers[offerId].acceptanceExpirationDate) ||
306:                 _needsPayoutRequest(s_offers[offerId].status, s_offers[offerId].payoutDate)
307:             ) {
308:                 offersToUpkeep = appendOfferToUpkeep(offersToUpkeep, offerId);
309:                 if (offersToUpkeep.length == s_config.automationUpkeepBatchSize) {
310:                     return (true, abi.encode(offersToUpkeep));
311:                 }
312:             }
313:         }
314:         return (offersToUpkeep.length > 0, abi.encode(offersToUpkeep));
315:     }
```
유지 관리가 필요한 제안은 로컬 변수 `offersToUpkeep`에 추가되며, 유지 관리할 제안 수가 `s_config.automationUpkeepBatchSize`와 같아지면 반환됩니다.

검증/지급 기능 요청이 필요하거나 만료된 경우 제안을 유지 관리해야 합니다.
```solidity
TunnlTwitterOffers.sol
479:     function _needsExpiration(Status offerStatus, uint32 payoutDate, uint32 acceptanceExpirationDate) internal view returns (bool) {
480:         return (
481:             (acceptanceExpirationDate < block.timestamp && offerStatus == Status.Pending) ||
482:             (payoutDate < block.timestamp &&
483:             (offerStatus == Status.Accepted ||
484:             offerStatus == Status.AwaitingVerification ||
485:             offerStatus == Status.VerificationInFlight ||
486:             offerStatus == Status.VerificationFailed))
487:         );
488:     }
```
위에서 볼 수 있듯이 제안이 여전히 보류 중이고 현재 시간이 `acceptanceExpirationDate`보다 늦은 경우 만료가 필요합니다.

문제는 누구나 충분한 승인(allowance)이 있고 `maxPaymentUsdc`(최대 제안 금액)에 하한선이 없으면 어떤 제안이든 생성할 수 있다는 것입니다.
따라서 기술적으로 `flatFeeUSDC`에 대한 단일 승인만 있으면 누구나 제안 금액이 0인 제안을 원하는 만큼 생성할 수 있습니다.
이러한 거짓 제안은 크리에이터에게 경제적 이익을 가져다주지 않기 때문에 누구에게도 수락되지 않을 것입니다.
결국 제안은 `offersToUpkeep`에서 보류 상태로 유지됩니다.
공격자는 수많은 거짓 제안을 생성하여 Chainlink 자동화 기능이 처리할 올바른 제안(검증 또는 지급)을 선택하지 못하게 하고 서비스 거부(DoS)를 유발함으로써 이를 악용할 수 있습니다.

**영향:** 공격자에게 직접적인 경제적 이익을 가져다주지 않으므로 영향을 중간(Medium)으로 평가합니다.

**개념 증명 (Proof Of Concept):**
아래의 PoC는 누구나 상당한 금액을 승인하지 않고도 원하는 수의 제안을 생성할 수 있음을 보여줍니다.
```solidity
    function test_CreateFalseOffers() public {
        uint32 acceptanceDurationSeconds = tunnlTwitterOffers.getConfig().minAcceptanceDurationSeconds;
        uint offerAmount = 0;
        uint maxValueUsdc = tunnlTwitterOffers.getConfig().flatFeeUsdc;

        vm.startPrank(advertiser);
        mockUsdcToken.mint(advertiser, maxValueUsdc);
        mockUsdcToken.approve(address(tunnlTwitterOffers), maxValueUsdc);//@audit-info single allowance of 5 USDC
        for(uint i = 100; i < 200; i++) {
            offerId = bytes32(i);
            tunnlTwitterOffers.createOffer(offerId, offerAmount, acceptanceDurationSeconds, 4 days);

            //@audit-info verifies the offer is created with the correct values
            assertEq(uint(getOffer(offerId).status), 0);
            assertEq(getOffer(offerId).flatFeeUsdc, tunnlTwitterOffers.getConfig().flatFeeUsdc);
            assertEq(getOffer(offerId).advertiserFeePercentageBP, tunnlTwitterOffers.getConfig().advertiserFeePercentageBP);
            assertEq(getOffer(offerId).creationDate, block.timestamp);
            assertEq(getOffer(offerId).acceptanceExpirationDate, block.timestamp + acceptanceDurationSeconds);
            assertEq(getOffer(offerId).advertiser, advertiser);
        }
        vm.stopPrank();
    }
```
**권장 완화 방법:** 몇 가지 가능한 완화 방법이 있습니다.
- 한 주소가 동시에 보유할 수 있는 제안 수를 제한합니다. 이렇게 하면 사용자가 적은 양의 승인으로 많은 제안을 유지하는 것을 방지할 수 있습니다.
- 광고주가 제안 수락 시점이 아닌 제안 생성 시점에 flatFee를 지불하도록 요구하고 제안이 취소되거나 만료되면 환불하도록 합니다.


**Tunnl:** 먼저 보류 중인 제안이 `s_offersToUpkeep`에서 올바르게 추가 및 제거되도록 수정해야 합니다.
출시 후 첫 몇 주 동안 경쟁자가 우리 제품을 공격할 위험이 매우 낮다고 생각하므로 메인넷 베타 출시 전에 DoS 문제를 해결하지 않기로 결정했습니다.
그러나 악의적인 경쟁자의 공격을 방지하기 위해 다음 반복에서 이 문제를 해결하기 위한 백로그 작업을 만들 것입니다.

**Cyfrin:** 인지함.


### 악의적인 크리에이터가 수락된 제안을 취소하여 광고주가 고정 수수료를 지불하도록 강요할 수 있음

**설명:** 프로토콜은 모든 제안이 수락될 때 `flatFee`를 청구합니다. 일단 수락되면 `flatFee`는 프로토콜 소유자의 주소로 전송되며 취소나 만료를 포함한 어떠한 상황에서도 환불되지 않습니다.

이것이 의도된 것일 수 있지만 악의적인 크리에이터와 관련된 또 다른 문제가 있습니다. 수락 후 콘텐츠 크리에이터가 아무것도 하지 않으면 제안은 `Offer.payoutDate` 이후에 만료되고 고정 수수료를 제외한 자금은 광고주에게 환불됩니다. 또는 콘텐츠 크리에이터가 `cancelOffer()`를 호출하여 제안을 취소할 수 있습니다.

크리에이터를 관리하기 위한 오프체인 백엔드 메커니즘이 존재한다는 것은 이해하지만, 크리에이터를 위한 온체인 에스크로 기능도 갖추는 것이 바람직합니다. 이를 위해서는 크리에이터가 제안을 수락할 때 일정 담보를 제공해야 하며, 크리에이터가 거래를 위반할 경우 광고주의 손실을 배상하는 데 사용됩니다.

현재 구현은 환불되지 않는 고정 수수료와 콘텐츠 크리에이터의 잠재적인 악의적 행동으로 인해 광고주가 손실을 입는 상황으로 이어질 수 있습니다. 이는 신뢰 부족과 프로토콜 참여 감소로 이어질 수 있습니다.

**영향:** 제안 수락이 관리자에 의해 처리되므로 전반적인 영향을 중간(Medium)으로 평가합니다.

**권장 완화 방법:** 제안을 수락할 때 크리에이터가 담보를 제공하도록 요구하는 온체인 에스크로 기능을 구현하십시오. 이 담보는 크리에이터가 의무를 이행하지 못할 경우 광고주에게 배상하는 데 사용됩니다.

**Tunnl:** Tunnl 베타 출시의 경우 이를 "설계된 대로" 수락하고 있습니다. Tunnl은 항상 이 고정 수수료에 대해 광고주에게 수동으로 환불할 수 있는 옵션이 있으며, 초기 사용자가 수십 명에 불과할 것이므로 베타 버전에서는 실행 가능합니다.
크리에이터에게 요금을 청구하는 것은 허용 가능한 해결책이 아닙니다. 이제 크리에이터가 트랜잭션을 보내거나 다른 결제 수단을 입력해야 하므로 크리에이터의 사용자 경험이 악화되기 때문입니다. 현재 크리에이터는 UI에 지갑 주소를 붙여넣기만 하면 되고 나머지는 우리가 처리합니다. 크리에이터가 다른 작업을 수행할 필요 없이 이 매우 간단한 UX를 유지하고 싶습니다.

그러나: 향후에는 혼란을 피하기 위해 환불 메커니즘 없이 제안을 생성할 때 항상 광고주에게 고정 수수료를 청구하도록 업데이트할 것입니다. 광고주가 더 쉽게 받아들일 수 있도록 더 낮은 고정 수수료(예: $1 미만)를 청구할 수 있습니다. 광고주가 보내는 각 제안에 대해 항상 비용을 지불해야 하므로 "스팸성" 제안도 완화할 수 있습니다. 베타 출시 기간 동안 Base 블록체인의 가스 가격을 기준으로 타당성과 최종 수수료 값을 결정할 것입니다.


**Cyfrin:** 인지함.


### 잘못된 수수료 비율 사용 [Self-Reported]

**설명:** 감사 도중 Tunnl 팀이 `sendFunctionsRequest()` 함수에서 버그를 발견했습니다.
함수가 `maxCreatorPayment`를 계산하는 동안 `s_offers[offerId].creatorFeePercentageBP` 대신 `s_config.creatorFeePercentageBP`를 사용하고 있었습니다.
이는 구성 변경이 있을 때 콘텐츠 크리에이터에게 지급되는 금액에 잠재적으로 영향을 미칠 수 있습니다.

**영향:** 가능성이 낮기 때문에 영향을 중간(Medium)으로 평가합니다.

**Tunnl:** [PR 37](https://github.com/tunnl-io/Tunnl-Contracts/pull/37/files)에서 수정됨.

**Cyfrin:** 확인됨.




### 보류 중(Pending)에서 만료됨(Expired)으로의 상태 전환 구현 불완전

**설명:** 모든 제안은 프로토콜에 의해 정의된 엄격한 상태 전환을 따릅니다. 우리는 새로운 상태로 변경하기 전 가능한 상태를 정의하는 조건부를 기반으로 가능한 모든 상태 전환을 추출했습니다.

`performUpkeep()` 함수의 구현을 보면 개발자가 `Pending`에서 `Expired`로의 상태 전환 가능성을 가정했음을 알 수 있습니다.

```solidity
TunnlTwitterOffers.sol
322:     function performUpkeep(bytes calldata performData) external override {
323:         bytes32[] memory offersToUpkeep = abi.decode(performData, (bytes32[]));
324:
325:         for (uint256 i = 0; i < offersToUpkeep.length; i++) {//@audit-info checks only the offersToUpkeep
326:             bytes32 offerId = offersToUpkeep[i];
327:
328:             if (_needsVerificationRequest(s_offers[offerId].status, s_offers[offerId].dateToAttemptVerification)) {
329:                 s_offers[offerId].status = Status.VerificationInFlight;
330:                 sendFunctionsRequest(offerId, RequestType.Verification);
331:             } else if (_needsExpiration(s_offers[offerId].status, s_offers[offerId].payoutDate, s_offers[offerId].acceptanceExpirationDate)) {
332:                 // Update status before making the external call
333:                 Status previousStatus = s_offers[offerId].status;
334:                 s_offers[offerId].status = Status.Expired;
335:                 s_offersToUpkeep.remove(offerId);
336:                 if (previousStatus == Status.Pending) {//@audit-info Pending -> Expired
337:                     // If expired BEFORE offer is accepted & funds have been locked in escrow, "revoke" the allowance by sending the advertiser's funds back to themselves
338:                     _revokeAllowanceForPendingOffer(offerId);
339:                 } else {
340:                     // If expired AFTER offer is accepted & funds have been locked in escrow, release the funds in escrow back to the advertiser
341:                     uint256 amountToTransfer = s_offers[offerId].maxValueUsdc - s_offers[offerId].flatFeeUsdc;
342:                     usdcToken.safeTransfer(s_offers[offerId].advertiser, amountToTransfer);
343:                 }
344:                 emit OfferStatus(offerId, previousStatus, Status.Expired);
345:             } else if (_needsPayoutRequest(s_offers[offerId].status, s_offers[offerId].payoutDate)) {
346:                 s_offers[offerId].status = Status.PayoutInFlight;
347:                 sendFunctionsRequest(offerId, RequestType.Payout);
348:             }
349:         }
350:     }
```
외부 for 루프는 `checkUpkeep()` 함수를 사용하여 검색되어야 하는 `offersToUpkeep`에 대한 것입니다. `checkUpkeep()` 함수는 `s_offersToUpkeep` 세트의 제안을 확인하며, 제안은 수락될 때만 이 세트에 추가됩니다. 따라서 보류 중인 제안은 `s_offersToUpkeep`에 전혀 추가되지 않으며 보류 중에서 만료됨으로의 상태 전환은 발생하지 않습니다.

Tunnl 팀과의 커뮤니케이션을 통해 그들이 `s_offersToUpkeep` 세트에 보류 중인 제안을 추가하려 했던 것으로 확인되었습니다.

**영향:** 이 발견이 금전적 손실이나 심각한 시스템 오작동을 직접적으로 초래하지 않으므로 영향을 중간(Medium)으로 평가합니다.

**권장 완화 방법:** 유지 관리 목록에 보류 중인 제안을 추가하고 다른 상태 전환도 검토할 것을 권장합니다.


**Tunnl:** [PR 38](https://github.com/tunnl-io/Tunnl-Contracts/pull/38/files)에서 수정됨.

**Cyfrin:** 확인됨.


### 공격자가 선행 매매(frontrunning)하여 사용자가 제안을 생성하는 것을 막을 수 있음

**설명:** 프로토콜은 누구나 사용자가 제공한 제안 ID로 제안을 생성할 수 있도록 허용합니다. 이 디자인은 사용자에게 제안 ID를 선택할 수 있는 유연성을 제공하지만 문제도 발생시킵니다.

`createOffer` 함수는 제공된 ID를 가진 제안이 이미 존재하는지 확인하고 존재하면 되돌립니다(revert).
```solidity
TunnlTwitterOffers.sol
152:     function createOffer(bytes32 offerId, uint256 maxPaymentUsdc, uint32 acceptanceDurationSeconds, uint32 _offerDurationSeconds) external {
153:
154:         require(offerId != bytes32(0), "Invalid offerId");
155:         require(s_offers[offerId].creationDate == 0, "Offer already exists"); //@audit Can be abused to prevent offer creation
156:         require(acceptanceDurationSeconds >= s_config.minAcceptanceDurationSeconds, "AcceptanceDuration is too short");
157:         require(_offerDurationSeconds >= s_config.minOfferDurationSeconds, "OfferDuration is too short");
```

악의적인 행위자는 이를 악용하여 일반 사용자가 제안을 생성하는 것을 막을 수 있습니다.

예:
- 공격자인 Alice는 프로토콜의 평판을 해치기로 결정합니다.
- Alice는 프로토콜에 `flatFee` 금액의 USDC를 승인하고 멤풀을 모니터링합니다.
- 합법적인 사용자 Bob이 ID="BOB"인 제안을 생성하려고 합니다.
- Alice는 동일한 ID로 가짜 제안을 생성하기 위해 선행 매매 트랜잭션을 보냅니다.
- Alice의 가짜 제안이 생성되고 동일한 ID를 가진 제안이 이미 존재하므로 Bob의 트랜잭션이 되돌려집니다.

이 영향은 다음과 같은 이유로 증폭됩니다.
- 프로토콜은 실제 자금 조달이 필요하지 않고 제안 생성을 위한 승인만 필요합니다. `flatFee` 금액의 USDC에 대한 단일 승인만 있으면 누구나 원하는 수의 제안을 생성할 수 있습니다.
- Alice는 평판이 좋은 광고주의 "큰" 제안만 방해하도록 선택할 수 있습니다.

공격은 공격자에게 경제적으로 이익이 되지 않지만 경쟁자가 이러한 종류의 악의적인 활동을 실행하는 것은 상당히 실현 가능합니다. 또한 프로토콜은 가스 비용이 적은 Base(L2)에 배포될 예정입니다.

**영향:** 공격이 사용자의 자금에 직접적인 영향을 미치지 않고 공격자에게 경제적으로 이익이 되지 않으므로 영향을 중간(Medium)으로 평가합니다.

**개념 증명 (Proof Of Concept):**
```solidity
    function test_FrontrunCreateOffer() public {
        address alice = address(0x10); // malicious actor
        address bob = address(0x20); // legitimate actor

        uint32 acceptanceDurationSeconds = tunnlTwitterOffers.getConfig().minAcceptanceDurationSeconds;
        uint offerAmount = 1e6;
        uint maxValueUsdc = tunnlTwitterOffers.getConfig().flatFeeUsdc + offerAmount;
        uint minAllowance = tunnlTwitterOffers.getConfig().flatFeeUsdc;

        // alice allows minimal amount of USDC to the contract
        vm.startPrank(alice);
        mockUsdcToken.mint(alice, minAllowance);
        mockUsdcToken.approve(address(tunnlTwitterOffers), minAllowance);
        vm.stopPrank();

        // bob intends to create an offer with id = 101010 (this is normally calculated offchain but here we just use an imaginary id)
        vm.startPrank(bob);
        mockUsdcToken.mint(bob, maxValueUsdc);
        mockUsdcToken.approve(address(tunnlTwitterOffers), maxValueUsdc);
        vm.stopPrank();

        offerId = bytes32(uint(0x101010));

        // alice front runs bob and creates an offer with the same id
        vm.startPrank(alice);
        tunnlTwitterOffers.createOffer(offerId, 0, acceptanceDurationSeconds, 4 days);
        vm.stopPrank();

        // now bob's offer creation reverts
        vm.startPrank(bob);
        vm.expectRevert("Offer already exists");
        tunnlTwitterOffers.createOffer(offerId, offerAmount, acceptanceDurationSeconds, 4 days);
        vm.stopPrank();
    }
```
**권장 완화 방법:** `createOffer()`에서 `msg.sender`와 `offerId`를 확인하는 메커니즘을 추가할 것을 권장합니다.

**Tunnl:** 출시 후 첫 몇 주 동안 경쟁자가 우리 제품을 공격할 위험이 매우 낮다고 생각하므로 메인넷 베타 출시 전에 이 선행 매매 문제를 해결하지 않기로 결정했습니다.
그러나: 악의적인 경쟁자의 공격을 방지하기 위해 다음 반복에서 이 문제를 해결하기 위한 백로그 작업을 만들 것입니다.
잠재적인 해결책은 백엔드에서 사용자에게 제공되는 승인 서명을 생성하고 이를 `createOffer` 트랜잭션에 포함해야 하는 것입니다.

**Cyfrin:** 인지함. (잠재적인 해결 접근 방식이 문제를 해결할 수 있습니다.)


\clearpage
## 정보성 (Informational)


### 일부 변수 및 인수 이름이 오해의 소지가 있음

**설명:** 오해의 소지가 있는 변수 및 인수 이름을 사용하면 잘못된 구현이 발생할 수 있습니다. 명확성을 높이고 코드 무결성을 유지하기 위해 이러한 이름을 수정하는 것이 좋습니다.

- `maxPaymentUsdc` -> `maxOfferedAmountUSDC`
```solidity
L152: function createOffer(bytes32 offerId, uint256 maxPaymentUsdc, uint32 acceptanceDurationSeconds, uint32 _offerDurationSeconds)
```

- `Offer.maxValueUsdc` -> `Offer.totalPaymentFromAdvertiser`
```solidity
L72: uint256 maxValueUsdc; // Percentage fee + flat fee + maximum payment the content creator can receive in USDC
```

- `Offer.amountPaidUsdc` -> `Offer.netAmountPaidUsdc`
```solidity
L73: uint256 amountPaidUsdc; // The final amount paid to the content creator in USDC
```

**Tunnl:** [PR 38](https://github.com/tunnl-io/Tunnl-Contracts/pull/38/files)에서 수정됨.

**Cyfrin:** 확인됨.


### 불필요한 중복 상태 로직

**설명:** `performUpkeep` 및 `sendFunctionsRequest` 함수에서 제안 상태가 동일한 값으로 두 번 설정됩니다.
이러한 종류의 중복 로직은 나중에 보안 문제를 일으킬 수 있습니다. 예를 들어 팀이 두 곳 중 하나를 변경하는 것을 놓칠 수 있습니다.

```solidity
TunnlTwitterOffers.sol - performUpkeep
328:             if (_needsVerificationRequest(s_offers[offerId].status, s_offers[offerId].dateToAttemptVerification)) {
329:                 s_offers[offerId].status = Status.VerificationInFlight;
330:                 sendFunctionsRequest(offerId, RequestType.Verification);
331:
```

```solidity
TunnlTwitterOffers.sol - sendFunctionsRequest
395:         s_offers[offerId].status = requestType == RequestType.Verification
396:             ? Status.VerificationInFlight
397:             : Status.PayoutInFlight;
398:         emit RequestSent(offerId, requestId, s_offers[offerId].status);
```

**Tunnl:** [PR 38](https://github.com/tunnl-io/Tunnl-Contracts/pull/38/files)에서 수정됨.

**Cyfrin:** 확인됨.


### OpenZeppelin의 nonReentrant의 불필요한 사용

**설명:** `TunnlTwitterOffers` 계약은 `retryRequests()` 함수에서 OpenZeppelin의 `nonReentrant` 수정자를 사용합니다.
그러나 구현에는 이 수정자가 필요하지 않습니다.
`retryRequests()` 함수는 내부 함수 `sendFunctionsRequest()`에 대한 호출만 수행하며 프로세스에 외부 호출이 없습니다.
또한 `retryRequests()` 함수는 `onlyAdmins` 수정자에 의해 제한됩니다.
우리는 추가 분석을 수행했으며 지불 토큰이 USDC로 고정되어 있는 한 프로토콜에 `nonReentrant` 수정자가 필요하지 않다는 결론을 내렸습니다.

**Tunnl:** [PR 38](https://github.com/tunnl-io/Tunnl-Contracts/pull/38/files)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage

