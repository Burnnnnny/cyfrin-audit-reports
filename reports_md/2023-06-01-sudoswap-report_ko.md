# 발견 사항 (Findings)

## 높은 중요도 (High Risk)
### `LSSVMRouter::swapNFTsForSpecificNFTsThroughETH`에서 지정된 `minOutput`이 잠긴 상태로 남음
**설명:**
Cyfrin 팀은 `LSSVMRouter`가 향후 `VeryFastRouter`로 대체되어 더 이상 사용되지 않을 예정이므로 이번 감사 범위에서 약간 벗어난다는 점을 이해하고 있습니다. 그러나 이 컨트랙트의 약간 수정된 버전이 현재 배포되어 [메인넷에서 라이브](https://etherscan.io/address/0x2b2e8cda09bba9660dca5cb6233787738ad68329#code) 상태입니다. 우리는 [`LSSVMRouter::swapNFTsForSpecificNFTsThroughETH`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L88)와 `LSSVMRouter::swapNFTsForAnyNFTsThroughETH`에서 버그를 발견했으며, 메인넷 포크에 대해 검증한 결과 [`minOutput`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L90) 매개변수로 지정된 대로 함수 호출과 함께 전송된 사용자 자금이 잠기는 것을 확인했습니다. 즉, 슬리피지로터 자신을 보호하려는 사용자가 오히려 자금이 잠기는 상황을 겪게 되며, 지정된 최소 기대 출력값이 높을수록 잠기는 자금의 가치도 높아집니다.

0이 아닌 `minOutput` 값을 지정한 사용자는 이 금액이 ETH에서 NFT로의 스왑 후반부에 전송된 [`inputAmount`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L105)에서 [차감](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L560)됩니다. 이는 내부 함수 [`LSSVMRouter::_swapETHForSpecificNFTs`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L560) 및 `LSSVMRouter::_swapETHForAnyNFTs`에 의해 처리됩니다. 이 [`inputAmount`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L562) 매개변수를 기반으로 사용되지 않은 ETH를 환불하는 것이 이러한 내부 함수의 책임이므로, `minOutput`으로 표시되는 초과 가치는 [`remainingValue`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L566) [계산](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L582)에 포함되지 않으며, 따라서 후속 [ETH 전송](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L591-L595)에도 포함되지 않습니다. 중간에 언더플로우가 발생하지 않는다면(충분히 큰 `minOutput` 값으로 인해), `minOutput`으로 지정된 초과 ETH는 라우터에 영원히 잠기게 됩니다.

다행히도 메인넷 배포에서 이 함수들은 Sudoswap 프런트엔드에 연결되지 않았기 때문에 실제로 호출된 적은 없는 것으로 보입니다. Sudoswap이 클라이언트에서 이 함수들을 사용하지 않지만, 컨트랙트 수준의 통합자들은 잠재적으로 자금 손실을 겪을 수 있으므로 Sudorandom Labs 팀은 잠재적인 피해자들에게 연락을 취했습니다.

**개념 증명 (Proof of Concept):**
다음 git diff를 적용하세요:

```diff
diff --git a/src/test/interfaces/ILSSVMPairFactoryMainnet.sol b/src/test/interfaces/ILSSVMPairFactoryMainnet.sol
new file mode 100644
index 0000000..3cdea5b
--- /dev/null
+++ b/src/test/interfaces/ILSSVMPairFactoryMainnet.sol
@@ -0,0 +1,20 @@
+// SPDX-License-Identifier: MIT
+pragma solidity ^0.8.0;
+
+import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
+import {ICurve} from "../../bonding-curves/ICurve.sol";
+import {LSSVMPair} from "../../LSSVMPair.sol";
+import {LSSVMPairETH} from "../../LSSVMPairETH.sol";
+
+interface ILSSVMPairFactoryMainnet {
+    function createPairETH(
+        IERC721 _nft,
+        ICurve _bondingCurve,
+        address payable _assetRecipient,
+        LSSVMPair.PoolType _poolType,
+        uint128 _delta,
+        uint96 _fee,
+        uint128 _spotPrice,
+        uint256[] calldata _initialNFTIDs
+    ) external payable returns (LSSVMPairETH pair);
+}
diff --git a/src/test/mixins/UsingETH.sol b/src/test/mixins/UsingETH.sol
index 0e5cb40..8fecb1e 100644
--- a/src/test/mixins/UsingETH.sol
+++ b/src/test/mixins/UsingETH.sol
@@ -14,6 +14,8 @@ import {LSSVMPairFactory} from "../../LSSVMPairFactory.sol";
 import {LSSVMPairERC721} from "../../erc721/LSSVMPairERC721.sol";
 import {LSSVMPairERC1155} from "../../erc1155/LSSVMPairERC1155.sol";
 
+import {ILSSVMPairFactoryMainnet} from "../interfaces/ILSSVMPairFactoryMainnet.sol";
+
 abstract contract UsingETH is Configurable, RouterCaller {
     function modifyInputAmount(uint256 inputAmount) public pure override returns (uint256) {
         return inputAmount;
@@ -46,6 +48,25 @@ abstract contract UsingETH is Configurable, RouterCaller {
         return pair;
     }
 
+    function setupPairERC721Mainnet(
+        ILSSVMPairFactoryMainnet factory,
+        IERC721 nft,
+        ICurve bondingCurve,
+        address payable assetRecipient,
+        LSSVMPair.PoolType poolType,
+        uint128 delta,
+        uint96 fee,
+        uint128 spotPrice,
+        uint256[] memory _idList,
+        uint256,
+        address
+    ) public payable returns (LSSVMPair) {
+        LSSVMPairETH pair = factory.createPairETH{value: msg.value}(
+            nft, bondingCurve, assetRecipient, poolType, delta, fee, spotPrice, _idList
+        );
+        return pair;
+    }
+
     function setupPairERC1155(CreateERC1155PairParams memory params) public payable override returns (LSSVMPair) {
         LSSVMPairETH pair = params.factory.createPairERC1155ETH{value: msg.value}(
             params.nft,
diff --git a/src/test/single-test-cases/CyfrinLSSVMRouterPoC.t.sol b/src/test/single-test-cases/CyfrinLSSVMRouterPoC.t.sol
new file mode 100644
index 0000000..596da45
--- /dev/null
+++ b/src/test/single-test-cases/CyfrinLSSVMRouterPoC.t.sol
@@ -0,0 +1,114 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity ^0.8.0;
+
+import "forge-std/Test.sol";
+
+import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
+import {Test721} from "../../mocks/Test721.sol";
+
+import {ICurve} from "../../bonding-curves/ICurve.sol";
+import {ILSSVMPairFactoryMainnet} from "../interfaces/ILSSVMPairFactoryMainnet.sol";
+
+import {UsingETH} from "../mixins/UsingETH.sol";
+import {ConfigurableWithRoyalties} from "../mixins/ConfigurableWithRoyalties.sol";
+import {LinearCurve, UsingLinearCurve} from "../../test/mixins/UsingLinearCurve.sol";
+
+import {LSSVMPair} from "../../LSSVMPair.sol";
+import {LSSVMPairETH} from "../../LSSVMPairETH.sol";
+import {LSSVMRouter} from "../../LSSVMRouter.sol";
+import {RoyaltyEngine} from "../../RoyaltyEngine.sol";
+import {LSSVMPairFactory} from "../../LSSVMPairFactory.sol";
+
+
+contract CyfrinLSSVMRouterPoC is Test, ConfigurableWithRoyalties, UsingLinearCurve, UsingETH {
+    IERC721 test721;
+    address payable alice;
+
+    LSSVMRouter constant LSSVM_ROUTER = LSSVMRouter(payable(address(0x2B2e8cDA09bBA9660dCA5cB6233787738Ad68329)));
+    LSSVMPairFactory constant LSSVM_PAIR_FACTORY = LSSVMPairFactory(payable(address(0xb16c1342E617A5B6E4b631EB114483FDB289c0A4)));
+    LinearCurve constant LINEAR_CURVE = LinearCurve(payable(address(0x5B6aC51d9B1CeDE0068a1B26533CAce807f883Ee)));
+
+    function setUp() public {
+        vm.createSelectFork(vm.envOr("MAINNET_RPC_URL", string.concat("https://rpc.ankr.com/eth")));
+
+        test721 = setup721();
+        alice = payable(makeAddr("alice"));
+        deal(alice, 1 ether);
+    }
+
+    function test_minOutputIsLockedInRouterWhenCallingswapNFTsForSpecificNFTsThroughETH() public {
+        Test721(address(test721)).mint(alice, 1);
+        uint256[] memory nftToTokenTradesIds = new uint256[](1);
+        nftToTokenTradesIds[0] = 1;
+        Test721(address(test721)).mint(address(this), 2);
+        Test721(address(test721)).mint(address(this), 3);
+        Test721(address(test721)).mint(address(this), 4);
+        uint256[] memory ids = new uint256[](3);
+        ids[0] = 2;
+        ids[1] = 3;
+        ids[2] = 4;
+        uint256[] memory tokenToNFTTradesIds = new uint256[](1);
+        tokenToNFTTradesIds[0] = ids[ids.length - 1];
+
+        test721.setApprovalForAll(address(LSSVM_PAIR_FACTORY), true);
+        LSSVMPair pair721 = this.setupPairERC721Mainnet{value: 10 ether}(
+            ILSSVMPairFactoryMainnet(address(LSSVM_PAIR_FACTORY)),
+            test721,
+            LINEAR_CURVE,
+            payable(address(0)),
+            LSSVMPair.PoolType.TRADE,
+            0.1 ether, // delta
+            0.1 ether, // 10% for trade fee
+            1 ether, // spot price
+            ids,
+            10 ether,
+            address(0)
+        );
+
+        uint256 pairETHBalanceBefore = address(pair721).balance;
+        uint256 aliceETHBalanceBefore = address(alice).balance;
+        uint256 routerETHBalanceBefore = address(LSSVM_ROUTER).balance;
+
+        emit log_named_uint("pairETHBalanceBefore", pairETHBalanceBefore);
+        emit log_named_uint("aliceETHBalanceBefore", aliceETHBalanceBefore);
+        emit log_named_uint("routerETHBalanceBefore", routerETHBalanceBefore);
+
+        uint256 minOutput;
+        {
+            LSSVMRouter.PairSwapSpecific[] memory nftToTokenTrades = new LSSVMRouter.PairSwapSpecific[](1);
+            nftToTokenTrades[0] = LSSVMRouter.PairSwapSpecific({
+                pair: pair721,
+                nftIds: nftToTokenTradesIds
+            });
+
+            LSSVMRouter.PairSwapSpecific[] memory tokenToNFTTrades = new LSSVMRouter.PairSwapSpecific[](1);
+            tokenToNFTTrades[0] = LSSVMRouter.PairSwapSpecific({
+                pair: pair721,
+                nftIds: tokenToNFTTradesIds
+            });
+
+            LSSVMRouter.NFTsForSpecificNFTsTrade memory trade = LSSVMRouter.NFTsForSpecificNFTsTrade({
+                nftToTokenTrades: nftToTokenTrades,
+                tokenToNFTTrades: tokenToNFTTrades
+            });
+
+
+            vm.startPrank(alice);
+            test721.setApprovalForAll(address(LSSVM_ROUTER), true);
+            minOutput = 0.79 ether;
+            LSSVM_ROUTER.swapNFTsForSpecificNFTsThroughETH{value: 1 ether}(trade, minOutput, alice, alice, block.timestamp + 10);
+        }
+
+        uint256 pairETHBalanceAfter = address(pair721).balance;
+        uint256 aliceETHBalanceAfter = address(alice).balance;
+        uint256 routerETHBalanceAfter = address(LSSVM_ROUTER).balance;
+
+        assertTrue(test721.ownerOf(tokenToNFTTradesIds[0]) == alice);
+        assertGt(pairETHBalanceAfter, pairETHBalanceBefore);
+        assertEq(routerETHBalanceAfter, minOutput);
+
+        emit log_named_uint("pairETHBalanceAfter", pairETHBalanceAfter);
+        emit log_named_uint("aliceETHBalanceAfter", aliceETHBalanceAfter);
+        emit log_named_uint("routerETHBalanceAfter", routerETHBalanceAfter);
+    }
+}
+```

**파급력 (Impact):**
이 취약점은 높은 영향력과 가능성으로 사용자 자금의 잠금을 초래합니다. 문제가 되는 함수가 UI에 통합되었다면 이는 '치명적(CRITICAL)'으로 평가되었겠지만, 현재 통합 상황이 가능성을 크게 줄여주므로 심각도를 '높음(HIGH)'으로 평가합니다.

**권장 완화 조치 (Recommended Mitigation):**
환불 계산에 사용하고 실제 계약 잔액을 올바르게 반영하며 이 금액이 초과되지 않았는지 확인하기 위해 `minOutput`을 내부 함수로 전달하세요. 이렇게 하면 [`outputAmount`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L94) [반환 값](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMRouter.sol#L104-L106)이 호출자에게 전송된 초과 ETH를 올바르게 반영하게 됩니다.

**Sudoswap:**
확인했습니다. 이 문제는 라우터의 현재 구현에 존재하지만, 현재 이 특정 함수와 상호 작용하도록 통합된 UI는 없습니다. 이 계약은 곧 VeryFastRouter로 대체되어 더 이상 사용되지 않을 예정입니다.

**Cyfrin:**
확인했습니다.

### 악의적인 쌍(pair)이 `VeryFastRouter`에 재진입하여 원래 호출자의 자금을 탈취할 수 있음
**설명:**
[`VeryFastRouter::swap`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L266)은 사용자가 부분 채결 조건을 지정하여 새로운 Sudoswap 라우터에서 매도 및 매수 주문 일괄 처리를 수행할 수 있는 주요 진입점입니다. 매도 주문이 먼저 실행된 다음 매수 주문이 실행됩니다. `LSSVMPair` 계약 자체는 재진입이 불가능하도록 구현되어 있지만, `VeryFastRouter`는 그렇지 않습니다. 사용자가 `VeryFastRouter::swap`을 호출하여 NFT 일부를 매도하고 후속 매수 주문을 위해 추가 ETH 가치를 전달한다고 가정할 때, 공격자는 특정 조건에서 이 함수에 재진입하여 원래 호출자의 자금을 훔칠 수 있습니다. 이 함수는 사용자 입력에 유효한 쌍이 포함되어 있는지 확인하지 않으므로, 공격자는 이를 사용하여 `LSSVMPair::swapNFTsForToken` 및 `LSSVMPair::swapTokenForSpecificNFTs`의 반환 값을 조작하여 내부 회계를 방해할 수 있습니다. 이러한 방식으로 공격자는 매수/매도 주문이 예상보다 더 많거나 적은 가치를 입력/출력하는 것처럼 보이게 만들 수 있습니다.

공격자가 악의적인 로열티 수령인이고, 공격자의 재진입 스왑 주문에 단일 매도 주문과 빈 매수 주문 배열이 포함된 경우를 생각해 보십시오. 악의적인 쌍을 호출하면 [`outputAmount`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L296) 값에 대한 제어권이 부여되며, 이 값은 모든 주문이 실행된 후(부분 체결 여부와 상관없이) [남은 ETH를 전송](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L482-L486)하는 데 사용되는 [가상 잔액](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L301-L302) `ethAmount`에 추가 할당(addition assignment)하는 데 사용됩니다. 현재 계약 잔액은 원래 호출자의 남은 ETH 가치이므로, 공격자는 악의적인 쌍이 이 금액을 반환하여 자금을 빼내도록 의도할 것입니다. 그러나 공격자의 재진입 주문과 원래 호출자의 주문 모두에 악의적인 쌍 계약이 도입되지 않으면, `ethAmount`의 안전한 ETH 전송으로 인해 공격자는 남은 중간 자금을 훔칠 수 없습니다. 왜냐하면 이는 원래 호출자의 트랜잭션이 동일한 라인에서 되돌려지게(revert) 하기 때문입니다. 즉, 계약이 더 이상 가지고 있지 않은 잔액을 전송하려고 시도하게 됩니다. 만약 이것이 가상 잔액이 아니라 계약 잔액을 직접 전송하는 것이었다면, 공격자는 사용자를 속여 악의적인 쌍을 호출하게 하지 않고도 자금을 훔칠 수 있었을 것입니다. 물론 악의적인 쌍을 호출하면 호출과 함께 전송된 모든 자금을 훔칠 수 있지만, 앞서 설명한 대로 잘못된 반환 값을 통해 내부 회계를 조작할 수 있다는 점을 감안할 때, 이 쌍을 호출하면 다른 스왑 주문/부분 체결에 영향을 미쳐 계약이 원래 호출자의 트랜잭션 수명 동안 실제보다 더 적은 자금을 가지고 있다고 생각하게 만들어 공격자가 재진입하여 ETH를 가지고 도망칠 수 있습니다. 그렇지 않으면 이 취약점의 범위는 라우터 호출에 대한 DoS 공격입니다.

이 악용을 수행하는 단계는 다음과 같습니다:

* 호출자를 속여 공격자의 악의적인 쌍에 대한 주문을 포함시킵니다.
* 공격자가 재진입하여 매도 주문을 전달하고 검증되지 않은 사용자 입력으로 인해 악의적인 쌍 계약을 다시 호출합니다. 이는 `outputAmount`를 부풀리고, 결과적으로 통화에 대한 `ethAmount`도 부풀립니다.
* 초과 ETH가 공격자에게 전송됩니다.
* 악의적인 쌍은 큰 `inputAmount`를 반환하여 `ethAmount`를 조작합니다.
* 원래 호출자의 추가 부분 매수 주문이 체결에 실패하고 NFT 매도에 대한 대가로 ETH를 받지 못합니다.

두 번째 악용 사례는 호출자가 라우터 계약을 토큰 수령인으로 지정하여 후속 매수 주문을 위해 일종의 DIY ETH 재활용 기능을 수행하는 경우입니다(아마도 0 입력 `msg.value` 사용). 이를 통해 공격자는 매수 주문에 자금이 소비되기 전에 최종 매도 주문을 재진입하여 중간 잔액을 훔칠 수 있습니다. 이 자금은 `ethAmount`에 의해 추적되지 않으므로 최종 전송이 되돌려지지 않기 때문입니다. 악의적인 로열티 수령인과 관계없이, 이는 호출자가 라우터 계약을 토큰 수령인으로 지정하면 후속 매수 주문에 의해 소비되지 않은 초과 ETH가 계약에 잠겨 있게 됨을 의미합니다. 라우터로의 전송을 담당하는 쌍 스왑 함수에 대한 호출을 금지하는 팩토리 재진입 가드 사용으로 인해 풀 자금은 안전합니다. 악의적인 로열티 수령인의 경우 사용자 잘못된 구성으로 인한 ERC-20 기반 스왑과 함께 전송된 ETH 값도 취약합니다.

**개념 증명 (Proof of Concept):**
다음 diff는 스왑에 재진입하여 원래 호출자의 ETH를 빼내는 허니팟(honeypot) 쌍을 보여줍니다: (코드 생략) [원문 참고]

**파급력 (Impact):**
이 취약점은 사용자 자금 손실을 초래하며, 파급력이 높고 가능성이 중간이므로 심각도를 '높음(HIGH)'으로 평가합니다.

**권장 완화 조치 (Recommended Mitigation):**
`VeryFastRouter::swap`에 대한 사용자 입력, 특히 쌍(pair)을 검증하고, 이 함수를 재진입 불가능(non-reentrant)으로 만드는 것을 고려하십시오.

**Sudoswap:**
확인했습니다. 위험 표면이 부적절한 인수를 전달하는 호출자에게 설정되어 있으므로 당분간 변경 사항은 없습니다. 쌍 검증은 클라이언트 측에서 수행되므로 우려가 적습니다.

**Cyfrin:**
확인했습니다.

### 로열티에 대한 선형성 가정으로 인해 서비스 거부 발생 가능
**설명:**
`VeryFastRouter::swap`은 내부 함수 [`VeryFastRouter::_findMaxFillableAmtForSell`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L556) 및 [`VeryFastRouter::_findMaxFillableAmtForBuy`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L503)에 의존하여 이진 검색을 통해 스왑 가능한 최대 토큰 양을 찾습니다.

이 프로토콜은 다양한 로열티 정보 제공자를 통합하도록 설계되었습니다. 라인 598은 로열티 금액이 선형적이라고 가정하지만, 악의적일 수 있는 외부 로열티 정보 제공자가 비선형 로열티 금액을 반환하는 경우 이 가정이 위반될 수 있습니다.
예를 들어, 로열티 금액은 스왑할 토큰 수의 함수일 수 있습니다 (예: 판매 금액이 크거나 작을수록 로열티가 더 크거나 적음).
이 경우 라인 598이 위반되고 최대 채결 가능 함수는 잘못된 `priceToFillAt` 및 `numItemsToFill`을 반환합니다.

예를 들어, `KODAV2` 로열티 계산은 반올림으로 인해 입력 금액에 정확하게 선형적이지 않습니다.

로열티 정보 제공자가 더 큰 판매 금액에 대해 더 높은 로열티를 반환하면 `priceToFillAt`은 실제 판매보다 높을 것입니다.
선형성 가정으로 계산된 `priceToFillAt` 값은 스왑 매도 로직 내에서 `ILSSVMPairERC721::swapNFTsForToken` 함수의 [최소 기대 출력 매개변수](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/VeryFastRouter.sol#L351)로 사용된다는 점에 유의하십시오. 스왑 매수 로직에 대해서도 유사한 추론이 적용됩니다.

따라서 `priceToFillAt`이 실제 판매보다 큰 것으로 계산되면 스왑이 실패합니다.

Cyfrin 팀은 Sudoswap이 모든 컬렉션이 ERC-2981을 준수할 것으로 예상하며, EIP-2981은 로열티 금액이 금액에 선형적이어야 한다고 명시하고 있음을 인정합니다.
그러나 토큰은 EIP-2981을 준수하지 않는 로열티 조회를 사용할 수 있으며 정직한 사용자의 유효한 트랜잭션을 방지하기 위해 악용될 수 있으므로 프로토콜은 로열티 금액이 선형적이라는 가정에 의존해서는 안 됩니다.

**파급력 (Impact):**
선형성 가정은 특히 외부 로열티 정보 제공자(악의적일 가능성 있음)의 경우 위반될 수 있으며, 이는 합법적인 스왑이 실패하므로 프로토콜이 예상대로 작동하지 못하게 할 수 있습니다.
이러한 잘못된 가정이 핵심 함수에 영향을 미치므로 심각도를 '높음(HIGH)'으로 평가합니다.

**권장 완화 조치 (Recommended Mitigation):**
프로토콜 팀이 선형성 가정을 사용하여 가스 비용을 줄이려고 했다는 것을 이해하지만, `priceToFillAt` 및 `numItemsToFill`을 계산할 때 실제 로열티 금액을 사용하는 것을 권장합니다.

**Sudoswap:**
확인했습니다. 대부분의 NFT가 ERC-2981을 준수할 것으로 예상됩니다.

**Cyfrin:**
확인했습니다.

## 중간 중요도 (Medium Risk)
### 내부 스왑에서 더 엄격한 요구 사항을 사용하여 되돌려질(revert) 가능성 있음
**설명:**
`VeryFastRouter::swap`은 내부 함수 `VeryFastRouter::_findMaxFillableAmtForSell` 및 `VeryFastRouter::_findMaxFillableAmtForBuy`에 의존하여 스왑 가능한 최대 토큰 양을 찾습니다.
출력은 스왑의 *실제* 비용이어야 하며, 매도 로직의 경우 [`minExpectedTokenOutput`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/erc721/LSSVMPairERC721.sol#L92) 매개변수로, 매수 로직의 경우 [`maxExpectedTokenInput`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/erc721/LSSVMPairERC721.sol#L32) 매개변수로 사용됩니다. 그러나 이는 문제가 있으며 스왑의 실제 비용이 이러한 함수의 출력과 다를 수 있기 때문에 의도하지 않은 프로토콜 동작으로 이어질 수 있습니다. 다른 발견 사항에서 선형성 가정 문제를 지적했지만, 실제 쌍의 스왑 함수가 더 엄격한 요구 사항으로 호출되고 있기 때문에 이 문제를 별도로 제기합니다.

`VeryFastRouter::_findMaxFillableAmtForSell` 및 `VeryFastRouter::_findMaxFillableAmtForBuy`의 출력보다 스왑의 실제 판매가 낮은 경우 스왑은 실패하지만, 원래 `minExpectedTokenOutputPerNumNFTs` 및 `maxCostPerNumNFTs`가 대신 사용되었다면 통과했을 수 있습니다.
`VeryFastRouter::_findMaxFillableAmtForSell` 및 `VeryFastRouter::_findMaxFillableAmtForBuy`의 출력이 항상 정확한 판매/비용을 나타냄을 보장할 수 있다면 괜찮을 수 있지만, 원래의 `minExpectedTokenOutputPerNumNFTs` 및 `maxCostPerNumNFTs`가 사용되지 않는 이유는 명확하지 않습니다.

**파급력 (Impact):**
직접적인 자금 손실로 이어지지는 않지만 의도하지 않은 프로토콜 동작을 유발할 수 있으므로 심각도를 '중간(MEDIUM)'으로 평가합니다.

**권장 완화 조치 (Recommended Mitigation):**
실제 스왑 함수의 인수로 `VeryFastRouter::_findMaxFillableAmtForSell` 및 `VeryFastRouter::_findMaxFillableAmtForBuy`의 출력 대신 `minExpectedTokenOutputPerNumNFTs` 및 `maxCostPerNumNFTs`를 사용하는 것을 권장합니다.

**Sudoswap:**
확인했습니다. 입력 값이 Bonding Curve에서 반환될 것으로 예상되므로 이는 매우 드물게 발생할 가능성이 높습니다.

**Cyfrin:**
확인했습니다.

### 매수/매도 정보를 얻기 위해 다른 반올림 방향 권장
**설명:**
이 문제는 AMM 풀에서 매수 및 매도 작업에 대해 서로 다른 반올림 방향을 구현해야 할 필요성과 관련이 있습니다.
여러 `ICurve` 구현(`XykCurve`, `GDACurve`)에서 `ICurve::getBuyInfo` 및 `ICurve::getSellInfo` 함수는 동일한 반올림 방향을 사용하여 구현됩니다.
이는 잠재적인 문제를 방지하기 위해 매수 및 매도 작업에 대해 서로 다른 반올림 방향을 적용해야 한다는 AMM 풀의 모범 사례와 일치하지 않습니다. 이 문제는 소수점 자릿수가 적은 토큰의 경우 더욱 심각해져 가격 불일치가 커집니다.

`ExponentialCurve`는 매수 및 매도 작업에 대해 명시적으로 다른 반올림 방향을 사용하며, 이는 모범 사례와 일치합니다.

또한 모든 곡선에서 프로토콜 및 거래 수수료 계산이 현재 프로토콜 및 수수료 수령인에게 유리한 방향으로 반올림되지 않으므로 가치가 트레이더에게 유리하게 시스템에서 유출될 수 있습니다.

**파급력 (Impact):**
이 문제는 쌍 생성자에게 금전적 손실을 초래할 수 있으며, 특히 소수점 자릿수가 적은 토큰의 경우 플랫폼의 전반적인 안정성에 부정적인 영향을 미칠 수 있습니다. 따라서 심각도를 '중간(MEDIUM)'으로 평가합니다.

**권장 완화 조치 (Recommended Mitigation):**
원하는 가격보다 낮은 가격으로 품목을 판매하고 시스템에서 가치가 유출되는 것을 방지하기 위해 매수 가격과 프로토콜/거래 수수료를 올림(round up)하도록 하십시오.

**Sudoswap:**
[커밋 902eee](https://github.com/sudoswap/lssvm2/commit/902eee37890af3953a55472d885bf6265b329434)에서 수정되었습니다.

**Cyfrin:**
검증되었습니다.


### GDACurve가 새로운 현물 가격(spot price)을 검증하지 않음
**설명:**
`GDACurve::getBuyInfo` 및 `GDACurve::getSellInfo`에서 계산된 새로운 현물 가격은 현재 `MIN_PRICE`에 대해 검증되지 않았습니다. 즉, 가격이 이 값 아래로 떨어질 수 있습니다.

`GDACurve::validateSpotPrice`에서 최소 가격 확인이 명시적으로 수행되지만, 가격이 업데이트될 때 동일한 검증이 누락되었습니다.

감쇠 계수(decay factor)의 최대값은 상당히 큰 값(2^20)으로 제한되므로, `lambda`가 높고 초기 가격이 낮으며 수요가 낮은(즉, 연속 구매 간의 시간 간격이 긴) 시나리오에서는 현물 가격이 `MIN_PRICE` 수준(현재 상수 값인 `1 gwei`로 설정됨) 아래로 떨어질 가능성이 있습니다.

**파급력 (Impact)**
대부분의 경우 더치 경매(Dutch auction)는 가격이 `MIN_PRICE` 수준에 도달하기 훨씬 전에 구매자를 찾는 경향이 있습니다. 또한 GDA 본딩 커브는 단면 풀(single-sided pools)에만 사용되도록 되어 있으므로 풀이 극도로 낮은 가격에 대량 거래될 즉각적인 위험은 없어 보입니다.
그러나 예약 가격(reserve price)이 없다는 것은 일부 풀에 대한 마켓 메이킹이 영구적으로 극도로 낮은 가격에서 발생할 수 있음을 의미할 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):**
지수 본딩 커브(exponential bonding curve)와 마찬가지로 현물 가격이 업데이트될 때 `GDACurve::getBuyInfo` 및 `GDACurve::getSellInfo`에 최소 가격 검증을 도입하는 것을 권장합니다.

**Sudoswap:**
[커밋 c4dc61](https://github.com/sudoswap/lssvm2/commit/c4dc6159b8e3a3252f82ef4afea1f62417994425)에서 수정되었습니다.

**Cyfrin:**
검증되었습니다.


### 이진 검색 구현이 항상 최적의 솔루션을 찾지는 않을 수 있음
**설명:**
`VeryFastRouter::_findMaxFillableAmtForBuy` 및 `VeryFastRouter::_findMaxFillableAmtForSell`은 이진 검색을 활용하여 최대 거래 금액을 결정합니다. 이진 검색은 테스트 함수(다음 검색 영역을 결정하는 데 사용되는 기준 함수)를 기반으로 선형 정렬된 공간에서 솔루션을 찾습니다. 그러나 현재 구현은 이를 보장하지 못할 가능성이 높으며 솔루션을 찾지 못하거나 차선의 솔루션을 식별할 수 있습니다.

먼저, `VeryFastRouter::_findMaxFillableAmtForSell`은 `[1, …, minOutputPerNumNFTs.length]` 배열에서 솔루션을 찾으려고 시도하며 사용된 테스트 부울 함수는
`error != CurveErrorCodes.Error.OK || currentOutput < minOutputPerNumNFTs[(start + end) / 2 - 1] || currentOutput > pairTokenBalance`입니다. 여기서 `currentOutput`은 사용자가 `(start+end)/2-1`개 아이템 거래에 대해 받게 될 토큰 수입니다.
테스트 기준을 `f(x)`로 나타내고(`x`는 거래할 아이템 수), `true`를 `1`, `false`를 `0`으로 처리하면 `f([1, …, minOutputPerNumNFTs.length])`는 `[1, 1, .... 1, 0, ..., 0]`과 같아야 하며 이진 검색은 마지막 `1`을 찾아야 합니다.
이를 달성하려면 테스트 함수의 각 조건이 `[1, 1, .... 1, 0, ..., 0]`과 같은 출력을 생성해야 합니다. 그렇지 않으면 이진 검색이 솔루션을 찾지 못하거나 차선의 솔루션을 찾을 수 있습니다.
그러나 `GDACurve`는 `error != CurveErrorCodes.Error.OK` 조건을 만족하지 않습니다. `x` (`numItems`)가 증가함에 따라 `alphaPowN`이 증가하고 `newSpotPrice_`가 감소합니다. 따라서 `x`가 특정 임계값보다 작으면 `ERROR.SPOT_PRICE_OVERFLOW`를 반환하고 `x`가 임계값보다 크거나 같으면 `ERROR.OK`를 반환합니다. 결과적으로 이 조건의 출력은 `[1, 1, .... 1, 0, ..., 0]`이 아니라 `[0, 0, .... 0, 1, ..., 1]`과 같게 됩니다.

다음으로, `VeryFastRouter::_findMaxFillableAmtForBuy`에서 이진 검색의 핵심 기준 함수는 `i`개의 NFT 구매 비용(이를 `C[i]`라고 함)이 사용자가 `i`개의 NFT 구매에 대해 입찰한 최대 매수 가격(`maxCostPerNumNFTs` 배열의 `(i-1)`번째 요소, `maxCostPerNumNFTs[i-1]`)보다 작거나 같아야 한다는 것입니다. 여기서 암시적인 가정은 `C[i] > maxCostPerNumNFTs[i-1]`이면 모든 `j > i`에 대해 `C[j] > maxCostPerNumNFTs[j-1]`이라는 것입니다.
이는 풀에서 `i`개의 NFT를 구매하는 비용이 사용자가 `i`개의 NFT를 획득하기 위해 입찰한 최대 매수를 초과하는 경우, 더 많은 수의 NFT를 구매하려고 할 때에도 구매 비용이 최대 입찰가를 지속적으로 초과해야 함을 의미합니다. 이 조건이 항상 참인 것은 아닙니다. 예를 들어, 사용자가 풀에서 더 큰 NFT 컬렉션을 구매하는 데만 관심이 있는 경우, 더 적은 수량의 아이템을 구매할 때는 낮은 입찰가를, 더 많은 수의 아이템을 획득할 때는 더 공격적인 입찰가를 제시할 수 있습니다.

**개념 증명 (Proof of Concept):**
다음 조건의 상황을 고려하십시오:

1. 지수 본딩 커브(현물 가격: 1 eth, 델타: 1.05)가 있는 11개의 NFT가 포함된 풀.
2. Alice는 `VeryFastRouter::getNFTQuoteForBuyOrderWithPartialFill`을 사용하여 NFT 컬렉션 구매를 위한 최대 입찰가를 계산합니다.
3. Alice는 풀에 있는 NFT의 대부분을 획득하기를 원하지만 NFT의 일부만 획득하는 경우에는 구매에 관심이 없습니다. 이러한 목표를 달성하기 위해 그녀는 다음과 같이 최대 입찰가 배열을 조정합니다.
4. Alice는 풀의 총 아이템 중 처음 50%까지 구매하기 위한 최대 입찰가(2단계에서 계산됨)를 25% 줄입니다.
5. Alice는 풀의 총 아이템 중 50% 이상을 구매하기 위한 최대 입찰가(2단계에서 계산됨)를 10% 늘립니다.

Alice의 입찰가가 전체 컬렉션을 구매하기에 충분함에도 불구하고 그녀의 주문은 부분적으로만 체결되어 총 11개 중 5개의 NFT만 받게 됩니다.

아래 코드 스니펫은 이 시나리오를 재현합니다: (코드 생략) [원문 참고]

전체 11개 아이템을 획득하기 위한 최대 입찰가가 11개 아이템의 실제 비용보다 상당히 높음에도 불구하고 최종 결과는 사용자가 5개 아이템만 획득한다는 것을 보여줍니다. 따라서 사용자가 더 많은 양의 NFT를 구매하기 위해 입찰가를 제출하더라도 기존 구현은 부분 주문을 수행하고 사용자에게 더 적은 NFT를 판매합니다.

**파급력 (Impact):**
이 문제는 즉각적인 금전적 손실을 초래하지는 않습니다. 그럼에도 불구하고 사용자는 원래 구매하려고 계획했던 것보다 적은 NFT를 획득합니다. 핵심 로직의 중요성을 감안할 때 이 문제의 영향은 '중간(MEDIUM)'으로 간주됩니다.

**권장 사항 (Recommendation):**
이진 검색의 대안으로 무차별 대입(brute-force) 접근 방식을 사용하는 것은 상당한 양의 가스를 소비할 수 있음을 인정하지만, 모든 합리적인 사용자 입력에 대해 이진 검색이 최상의 솔루션을 산출하도록 보장하기 위해 가능한 엣지 케이스(모든 커브 구현에 걸쳐)에 대한 포괄적인 검사를 수행할 것을 제안합니다.

**Sudoswap:**
확인했습니다.

**Cyfrin:**
확인했습니다.

## 낮은 중요도 (Low Risk)
### 소유자가 `LSSVMPair::changeSpotPrice`를 호출하면 나중에 산술 오버/언더플로우가 발생할 수 있음
**설명:**
현물 가격을 쌍의 현재 ERC20 (또는 ETH) 잔액보다 높은 값으로 변경하면 나중에 유효한 스왑 호출에서 의도하지 않은 되돌리기(revert)가 발생할 수 있습니다.

**개념 증명 (Proof of Concept):**
전체 개념 증명은 [여기](https://github.com/sudoswap/lssvm2/pull/1/files#diff-ccfdcc468a169eaf116e657d2cd2406530c2a693627167e375d78dcf8a73d87d)에 있습니다.

**파급력 (Impact):**
사용자가 이전 호출에서 `ILSSVMPair::getSellNFTQuote`를 사용하여 견적을 얻었음에도 불구하고 `ILSSVMPair::swapNFTsForToken`이 `TRANSFER_FAILED` 오류 메시지와 함께 되돌려집니다.

**권장 완화 조치 (Recommended Mitigation):**
예상 출력이 쌍의 잔액을 초과하는 경우 `ILSSVMPair::getSellNFTQuote`는 합법적인 오류 코드를 반환해야 합니다.
이렇게 하면 스왑을 수행하려는 최종 사용자에 대한 영향을 완화할 수 있습니다. 실제 시도에서 산술 오류로 실패하는 대신 스왑 시도 전 뷰(view) 함수에서 유용한 오류를 반환하게 됩니다.

**Sudoswap:**
확인했습니다. 되돌리기로 이어지지만 사용자 자금에는 영향을 미칠 수 없으므로 위험을 수용할 수 있습니다.
소유자가 스왑 전에 가격을 조정하는 경우 사용자가 과도한 슬리피지로터 자신을 보호하기 위해 minOutput/maxInput 금액을 사용하는 것이 의도입니다.

**Cyfrin:**
확인했습니다.

### 부분적으로 체결 가능한 주문이 되돌려질 수 있음
**설명:**
`VeryFastRouter::swap`의 매도 로직에서 프로토콜은 체결 가능 금액을 확인하지 않고 [`pairSpotPrice == order.expectedSpotPrice`](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/VeryFastRouter.sol#L281)인 경우 스왑을 실행합니다. 그러나 쌍의 잔액이 부족하면 되돌려집니다. 반면, 이러한 기준이 충족되지 않으면 프로토콜은 최대 체결 가능 금액을 다시 확인하고 이진 검색에서 잔액 확인을 사용합니다.

**파급력 (Impact):**
부분적으로 체결 가능한 주문이 불필요하게 되돌려집니다.
자금에는 영향을 미치지 않지만 사용자는 유효한 트랜잭션에 대해 되돌리기를 경험하게 됩니다.

**권장 완화 조치 (Recommended Mitigation):**
부분적으로 체결 가능한 주문에 대해 되돌리기를 방지하기 위해 쌍의 잔액이 부족한 경우를 처리하는 것을 고려하십시오.

**Sudoswap:**
확인했습니다. 본딩 커브 가격이 증가하고 (즉, X번째 아이템 구매 가격이 X번째-1 아이템 구매 가격보다 비쌈) 부분 체결 주문이 역순으로 생성되는 한 이 문제는 방지됩니다.

**Cyfrin:**
확인했습니다.

### 외부 호출과 함께 값을 전송할 수 있도록 `LSSVMPair::call`을 payable로 만들기
**설명:**
현재 [`LSSVMPair::call`](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMPair.sol#L640)은 단순히 [0의 값](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/LSSVMPair.sol#L661)을 전달합니다. 그러나 0이 아닌 `msg.value`가 필요하거나 원하는 경우가 있을 수 있으므로 소유자가 외부 호출에 대해 ETH를 공급할 수 있도록 함수를 payable로 표시해야 합니다.

**파급력 (Impact):**
소유자가 외부 호출과 함께 ETH를 보내고자 하는 경우 그렇게 할 수 없으며 이는 외부 호출의 기능에 영향을 미칠 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):**
`LSSVMPair::call`을 payable로 만들어 외부 호출에 값을 전달할 수 있도록 하는 것을 고려하십시오.

**Sudoswap:**
확인했습니다.

**Cyfrin:**
확인했습니다.


## 정보성 발견 사항 (Informational)
### 부정확/불완전한 NatSpec 및 주석
`LSSVMPair::getAssetRecipient`에 대한 NatSpec은 현재 "이 쌍으로 스왑이 완료되었을 때 자산을 받는 자산 주소를 반환합니다(Returns the address that assets that receives assets when a swap is done with this pair)"라고 [되어 있지만](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/LSSVMPair.sol#L319) "이 쌍으로 스왑이 완료되었을 때 자산을 받는 주소를 반환합니다(Returns the address that receives assets when a swap is done with this pair)"여야 합니다. 또한 여러 인스턴스에서 NatSpec은 매개변수와 반환 값을 더 잘 설명하기 위해 더 상세할 수 있습니다. 예: [`LSSVMPair::getSellNFTQuote`](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/LSSVMPair.sol#L245-L249).

`GDACurve::getSellInfo`에는 현재 "인덱스 n에서 경매에 대한 예상 출력...(The expected output at for an auction at index n...)"이라고 [읽히는 주석](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/bonding-curves/GDACurve.sol#L180)이 있지만 "인덱스 n에서 경매에 대한 예상 출력...(The expected output for an auction at index n...)"이어야 합니다.

`LSSVMPairCloner::cloneERC1155ETHPair`에는 "RUNTIME (53 bytes of code + 61 bytes of extra data = 114 bytes)"라는 [주석](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/lib/LSSVMPairCloner.sol#L299)이 있지만 이는 93바이트의 추가 데이터여야 하므로 총 146바이트이며 이는 위의 0x92 런타임 크기에 해당합니다. [주석](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/lib/LSSVMPairCloner.sol#L386) "RUNTIME (53 bytes of code + 81 bytes of extra data = 134 bytes)"가 있는 `LSSVMPairCloner::cloneERC1155ERC20Pair`의 경우도 마찬가지입니다. 이는 113바이트의 추가 데이터여야 하므로 총 166바이트이며 이는 위의 0xa6 런타임 크기에 해당합니다.

`VeryFastRouter::_findMaxFillableAmtForSell`의 [주석](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/VeryFastRouter.sol#L604) "이것은 우리가 지불할 의사가 있는 최대 비용, 0부터 시작하는 인덱스(this is the max cost we are willing to pay, zero-indexed)"는 "이것은 우리가 판매에서 기대하는 최소 출력, 0부터 시작하는 인덱스(this is the minimum output we are expecting from the sale, zero-indexed)"여야 합니다.

**Sudoswap:**
[cb98b6](https://github.com/sudoswap/lssvm2/commit/cb98b66513a5dd149fcf169caafe6be370f5295b) 및 [9bf4be](https://github.com/sudoswap/lssvm2/commit/9bf4be10366b0339dbae08d77bbf7ff81d4f51ff) 커밋에서 수정되었습니다.

**Cyfrin:**
검증되었습니다.


### `RoyaltyEngine::_getRoyaltyAndSpec`의 도달할 수 없는 코드 경로
`RoyaltyEngine` 내에서 `int16 private constant NONE = -1;` 및 `int16 private constant NOT_CONFIGURED = 0;`과 같이 다양한 로열티 사양과 관련된 열거형 값으로 사용하기 위해 `int16` 값이 매니폴드(manifold) 계약에서 복사되었습니다. `RoyaltyEngine::_getRoyaltyAndSpec`의 [if 케이스](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/RoyaltyEngine.sol#L152)는 `spec <= NOT_CONFIGURED`를 포착하므로 else 블록의 다음 [코드](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/RoyaltyEngine.sol#L237-L238)는 도달할 수 없으며 제거할 수 있습니다.
```solidity
if (spec == NONE) {
    return (recipients, amounts, spec, royaltyAddress, addToCache);
}
```

**Sudoswap:**
[9bf4be](https://github.com/sudoswap/lssvm2/commit/9bf4be10366b0339dbae08d77bbf7ff81d4f51ff) 커밋에서 수정되었습니다.

**Cyfrin:**
확인했습니다.

### 테스트 내 중첩 함수 호출 전 `vm.prank()`가 의도한 대로 작동하지 않음
Foundry의 prank 치트 코드는 다음 외부 호출에만 적용됩니다. [중첩된](https://github.com/sudoswap/lssvm2/blob/78d38753b2042d7813132f26e5573c6699b605ef/src/test/base/VeryFastRouterAllSwapTypes.sol#L1090-L1092) [호출](https://github.com/sudoswap/lssvm2/blob/0967f16fb0f32b0b76f37c09e6acf35ac007225f/src/test/base/VeryFastRouterWithRoyalties.sol#L1108-L1110)은 오른쪽에서 왼쪽으로 평가되므로 prank가 의도한 대로 적용되지 않습니다. 필요한 함수 호출을 지역 변수로 캐시하거나 `vm.startPrank(addr)` 및 `vm.stopPrank()`를 사용하십시오. 예상되는 이벤트 방출을 주장(assert)하도록 테스트를 확장하는 것이 좋으며 이와 같은 경우를 포착하는 데 도움이 될 것입니다.

**Sudoswap:**
확인했습니다.

**Cyfrin:**
확인했습니다.

### 이벤트 매개변수 이름이 부정확함
다음 `LSSVMPair` 이벤트 각각의 첫 번째 매개변수 이름이 잘못 지정되었습니다:

```solidity
event SwapNFTInPair(uint256 amountIn, uint256[] ids);
event SwapNFTInPair(uint256 amountIn, uint256 numNFTs);
event SwapNFTOutPair(uint256 amountOut, uint256[] ids);
event SwapNFTOutPair(uint256 amountOut, uint256 numNFTs);
```

대신 다음과 같아야 합니다:

```solidity
event SwapNFTInPair(uint256 amountOut, uint256[] ids);
event SwapNFTInPair(uint256 amountOut, uint256 numNFTs);
event SwapNFTOutPair(uint256 amountIn, uint256[] ids);
event SwapNFTOutPair(uint256 amountIn, uint256 numNFTs);
```

**Sudoswap:**
[29449e](https://github.com/sudoswap/lssvm2/commit/29449e45fd7ebaf1b932c135df65b08e998d1071) 커밋에서 수정되었습니다.

**Cyfrin:**
검증되었습니다.

### `LSSVMPair`가 `ILSSVMPair`를 상속하지 않음
인터페이스 `ILSSVMPair`는 여러 곳에서 정의되고 사용되지만 추상 `LSSVMPair`는 이를 상속하지 않습니다.
`LSSVMPair` 계약 선언에 `is ILSSVMPair`를 추가하는 것을 고려하십시오.

**Sudoswap:**
확인했습니다.

**Cyfrin:**
확인했습니다.


### `MerklePropertyChecker::hasProperties`가 `ids.length`와 `proofList.length`가 같은지 검증하지 않음
이 함수는 제공된 ID를 반복하고 각 ID가 유효한지 확인하기 위해 증명 검증을 수행합니다.
추가 증명이 제공되면 루프는 이것들에 도달하기 전에 종료되므로 검증에 사용되지 않지만, 배열 길이가 동일한지 검증하는 것이 일반적으로 모범 사례입니다.

**Sudoswap:**
[0f8f94](https://github.com/sudoswap/lssvm2/pull/113/commits/0f8f94872743d67c304b310d22c653c8d75d80ea) 커밋에서 수정되었습니다.

**Cyfrin:**
검증되었습니다.


## 가스 최적화 (Gas Optimizations)
### `LSSVMPair::initialize`의 중복된 0 주소 확인
초기화 인수 `_assetRecipient`는 0 주소가 아닌 경우에만 상태 변수에 할당됩니다. 그러나 `LSSVMPair::getAssetRecipient`에서는 `assetRecipient`가 0 주소인 경우 소유자 주소가 반환되므로 이 확인은 중복됩니다. 기본값은 `address(0)`이며 `LSSVMPair::getAssetRecipient` 함수에서만 사용됩니다. `LSSVMPair::getFeeRecipient`의 경우 0 주소 자산 수령인의 경우 쌍 계약 자체에 수수료가 누적됩니다.

### `LSSVMPair::getBuyNFTQuote` 및 `LSSVMPair::getSellNFTQuote`의 중복된 0 값 확인
`LSSVMPair::getBuyNFTQuote`에 대한 `numNFTs` 인수는 0이 아닌 값으로 검증됩니다. 그러나 `ICurve::getBuyNFTQuote`에서 모든 현재 구현은 `numItems` 매개변수가 0이면 되돌려지므로 쌍에 대한 이 확인은 중복됩니다. 동일한 추론이 `LSSVMPair::getSellNFTQuote`에도 적용됩니다.

### `LSSVMPair::_calculateBuyInfoAndUpdatePoolParams` 및 `LSSVMPair::_calculateSellInfoAndUpdatePoolParams` 조건문 단순화
`LSSVMPair::_calculateBuyInfoAndUpdatePoolParams`의 "가스를 절약하기 위해 쓰기 통합(Consolidate writes to save gas)" 주석에 따라 로직을 두 개의 if 문으로 줄이고 값을 함께 저장/이벤트 방출하여 추가 최적화를 수행할 수 있습니다:
```solidity
// 가스를 절약하기 위해 쓰기 통합
// 현물 가격이 업데이트된 경우 현물 가격 업데이트 방출
if (currentSpotPrice != newSpotPrice) {
    spotPrice = newSpotPrice;
    emit SpotPriceUpdate(newSpotPrice);
}

// 델타가 업데이트된 경우 델타 업데이트 방출
if (currentDelta != newDelta) {
    delta = newDelta;
    emit DeltaUpdate(newDelta);
}
```
동일한 권장 사항이 `LSSVMPair::_calculateSellInfoAndUpdatePoolParams`에도 적용됩니다.

### `LSSVMPairERC20::_pullTokenInputs`에서 지역 변수 조기 캐시
`token()`이 직접 호출되는 인스턴스가 여러 개 있으므로 저장소 읽기 횟수를 줄이기 위해 함수 시작 부분에서 `_assetRecipient`와 같은 방식으로 지역 변수 `ERC20 token_ = token()`을 캐시해야 합니다.

또한 `LSSVMPairERC1155::swapTokenForSpecificNFTs` 및 `LSSVMPairERC721::swapTokenForSpecificNFTs`에서 `factory()`를 캐시하는 것이 좋습니다. `LSSVMPairERC1155::swapNFTsForToken`, `LSSVMPairERC721::swapTokenForSpecificNFTs` 및 `LSSVMPairERC721::_swapNFTsForToken`에서는 한 번만 사용되므로 `poolType()`을 캐시할 필요가 없습니다.

# 추가 의견 (Additional Comments)
Spearbit의 Sudoswap LSSVM2 보안 검토에서도 강조된 몇 가지 발견 사항을 확인했습니다. Sudorandom Labs 팀에서 이미 확인했지만 여전히 해결할 가치가 있는 문제라고 생각되어 여기에 다시 강조합니다.
특히 다음이 포함됩니다:
* `VeryFastRouter::swap`이 토큰을 ETH와 혼합할 수 있음 (Spearbit 5.2.4) - 이는 우리의 6.1.2와 연결될 수 있음
* 사용자가 `LinearCurve`가 있는 풀에 대해 `ILSSVMPair::swapNFTsForToken`을 호출할 때 초과 NFT를 잃을 수 있음 (Spearbit 5.2.9)
* `ICurve::getBuyInfo` 및 `ICurve::getSellInfo`의 나눗셈이 0으로 내림(round down)될 수 있음 (Spearbit 5.3.19)

# 부록 (Appendix)

## 4nalyz3r 출력
아래는 이 리포지토리의 계약에 대해 4nalyz3r 정적 분석 도구를 실행한 결과입니다:

### 가스 최적화 (Gas Optimizations)


| |문제|인스턴스|
|-|:-|:-:|
| [GAS-1](#GAS-1) | `address(this).balance` 대신 `selfbalance()` 사용 | 4 |
| [GAS-2](#GAS-2) | `address(0)` 확인을 위해 어셈블리 사용 | 10 |
| [GAS-3](#GAS-3) | `array[index] += amount`가 `array[index] = array[index] + amount` (또는 관련 변형)보다 저렴함 | 2 |
| [GAS-4](#GAS-4) | 저장소에 bool 사용 시 오버헤드 발생 | 3 |
| [GAS-5](#GAS-5) | 루프 외부에서 배열 길이 캐시 | 22 |
| [GAS-6](#GAS-6) | 변경되지 않는 함수 인수에 대해 메모리 대신 calldata 사용 | 3 |
| [GAS-7](#GAS-7) | 사용자 정의 오류(Custom Errors) 사용 | 21 |
| [GAS-8](#GAS-8) | 기본값으로 변수 초기화하지 않기 | 10 |
| [GAS-9](#GAS-9) | 상수(constants)에 `public` 대신 `private` 사용 시 가스 절약 | 2 |
| [GAS-10](#GAS-10) | 가능한 경우 나눗셈/곱셈 대신 시프트 오른쪽/왼쪽 사용 | 13 |
| [GAS-11](#GAS-11) | 구조체/배열에 `memory` 대신 `storage` 사용 | 29 |
| [GAS-12](#GAS-12) | 부호 없는 정수 비교 시 > 0 대신 != 0 사용 | 3 |
| [GAS-13](#GAS-13) | 계약에서 호출하지 않는 `internal` 함수 제거 | 8 |


### 비치명적 문제 (Non Critical Issues)

| |문제|인스턴스|
|-|:-|:-:|
| [NC-1](#NC-1) | 주소 상태 변수에 값을 할당할 때 `address(0)`에 대한 확인 누락 | 4 |
| [NC-2](#NC-2) | `require()` / `revert()` 문에 설명적인 이유 문자열이 있어야 함 | 1 |
| [NC-3](#NC-3) | 이벤트에 `indexed` 필드 누락 | 18 |
| [NC-4](#NC-4) | 매직 넘버 대신 상수 정의 사용 | 3 |
| [NC-5](#NC-5) | 내부적으로 사용되지 않는 함수는 external로 표시 가능 | 28 |
| [NC-6](#NC-6) | 오타 | 84 |

(상세 인스턴스는 코드이므로 유지합니다.)
#### <a name="GAS-1"></a>[GAS-1] `address(this).balance` 대신 `selfbalance()` 사용
ETH 계약 잔액을 가져올 때 어셈블리를 사용하십시오.

ETH 계약 잔액을 가져올 때 `address(this).balance` 대신 `selfbalance()`를 사용하여 가스를 절약할 수 있습니다.
또한 외부 계약의 ETH 잔액을 가져올 때 `address.balance()` 대신 `balance(address)`를 사용할 수 있습니다.

*내부 잔액 확인 시 15 가스, 외부 잔액 확인 시 6 가스 절약*

*인스턴스 (4)*:
```solidity
File: LSSVMPairETH.sol

91:         withdrawETH(address(this).balance);

```

```solidity
File: LSSVMPairFactory.sol

421:         protocolFeeRecipient.safeTransferETH(address(this).balance);

```

```solidity
File: VeryFastRouter.sol

172:             balance = address(pair).balance;

```

```solidity
File: settings/Splitter.sol

27:         uint256 ethBalance = address(this).balance;

```

#### <a name="GAS-2"></a>[GAS-2] `address(0)` 확인을 위해 어셈블리 사용
*인스턴스당 6 가스 절약*

*Instances (10)*:
```solidity
File: LSSVMPair.sol

139:         if (owner() != address(0)) revert LSSVMPair__AlreadyInitialized();

152:         if (_assetRecipient != address(0)) {

333:         if (_assetRecipient == address(0)) {

346:         if (_feeRecipient == address(0)) {

```

```solidity
File: LSSVMPairFactory.sol

438:         if (_protocolFeeRecipient == address(0)) revert LSSVMPairFactory__ZeroAddress();

501:         if (settingsAddress == address(0)) {

```

```solidity
File: erc721/LSSVMPairERC721.sol

98:             if (propertyChecker() != address(0)) revert LSSVMPairERC721__NeedPropertyChecking();

255:                 if ((numNFTs > 1) && (propertyChecker() == address(0))) {

```

```solidity
File: lib/OwnableWithTransferCallback.sol

46:         if (newOwner == address(0)) revert Ownable_NewOwnerZeroAddress();

```

```solidity
File: settings/StandardSettings.sol

45:         require(owner() == address(0), "Initialized");

```

#### <a name="GAS-3"></a>[GAS-3] `array[index] += amount`가 `array[index] = array[index] + amount` (또는 관련 변형)보다 저렴함
산술 연산으로 배열의 값을 업데이트할 때 `array[index] = array[index] + amount`보다 `array[index] += amount`를 사용하는 것이 더 저렴합니다.
이는 배열이 메모리에 저장될 때 추가 `mload`를 피하고 저장소에 저장될 때 `sload`를 피하기 때문입니다.
이는 `+=`, `-=`,`/=`,`*=`,`^=`,`&=`, `%=`, `<<=`,`>>=`, `>>>=`를 포함한 모든 산술 연산에 적용될 수 있습니다.
이 패턴이 루프 중에 발생하는 경우 최적화가 특히 중요할 수 있습니다.

*저장소 배열의 경우 28 가스, 메모리 배열의 경우 38 가스 절약*

*Instances (2)*:
```solidity
File: VeryFastRouter.sol

126:                 prices[i] = prices[i] + (prices[i] * slippageScaling / 1e18);

218:                 outputAmounts[i] = outputAmounts[i] - (outputAmounts[i] * slippageScaling / 1e18);

```

#### <a name="GAS-4"></a>[GAS-4] 저장소에 bool 사용 시 오버헤드 발생
Gwarmaccess(100 가스)를 피하고 과거에 ‘true’였다가 ‘false’에서 ‘true’로 변경할 때 Gsset(20000 가스)을 피하기 위해 true/false 대신 uint256(1) 및 uint256(2)를 사용하십시오. [소스](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27) 참조.

*Instances (3)*:
```solidity
File: LSSVMPairFactory.sol

56:     mapping(ICurve => bool) public bondingCurveAllowed;

57:     mapping(address => bool) public override callAllowed;

60:     mapping(address => mapping(address => bool)) public settingsForCollection;

```

#### <a name="GAS-5"></a>[GAS-5] 루프 외부에서 배열 길이 캐시
캐시되지 않은 경우 solidty 컴파일러는 각 반복 중에 항상 배열 길이를 읽습니다. 즉, 저장소 배열인 경우 추가 sload 작업(첫 번째 반복을 제외한 각 반복에 대해 100 추가 가스)이고 메모리 배열인 경우 추가 mload 작업(첫 번째 반복을 제외한 각 반복에 대해 3 추가 가스)입니다.

*Instances (22)*:
```solidity
File: LSSVMPair.sol

673:         for (uint256 i; i < calls.length;) {

```

```solidity
File: LSSVMPairERC20.sol

68:             for (uint256 i; i < royaltyRecipients.length;) {

93:             for (uint256 i; i < royaltyRecipients.length;) {

```

```solidity
File: LSSVMPairETH.sol

57:         for (uint256 i; i < royaltyRecipients.length;) {

```

```solidity
File: LSSVMRouter.sol

215:                 swapList[i].swapInfo.nftIds[0], swapList[i].swapInfo.nftIds.length

262:                 swapList[i].swapInfo.nftIds[0], swapList[i].swapInfo.nftIds.length

305:                 (error,,, pairOutput,,) = swapList[i].swapInfo.pair.getSellNFTQuote(nftIds[0], nftIds.length);

354:                     params.tokenToNFTTrades[i].swapInfo.nftIds[0], params.tokenToNFTTrades[i].swapInfo.nftIds.length

387:                         assetId, params.nftToTokenTrades[i].swapInfo.nftIds.length

437:                     params.tokenToNFTTrades[i].swapInfo.nftIds[0], params.tokenToNFTTrades[i].swapInfo.nftIds.length

463:                         assetId, params.nftToTokenTrades[i].swapInfo.nftIds.length

```

```solidity
File: RoyaltyEngine.sol

182:                 for (uint256 i = 0; i < royalties.length;) {

264:                 for (uint256 i = 0; i < royalties.length;) {

```

```solidity
File: VeryFastRouter.sol

125:             for (uint256 i = 0; i < prices.length;) {

217:             for (uint256 i = 0; i < outputAmounts.length;) {

275:         for (uint256 i; i < swapOrder.sellOrders.length;) {

387:         for (uint256 i; i < swapOrder.buyOrders.length;) {

```

```solidity
File: erc1155/LSSVMPairERC1155.sol

143:         for (uint256 i; i < royaltyRecipients.length;) {

```

```solidity
File: erc721/LSSVMPairERC721.sol

52:             _calculateBuyInfoAndUpdatePoolParams(nftIds.length, bondingCurve(), factory());

172:         (protocolFee, outputAmount) = _calculateSellInfoAndUpdatePoolParams(nftIds.length, bondingCurve(), _factory);

189:         for (uint256 i; i < royaltyRecipients.length;) {

```

```solidity
File: settings/StandardSettings.sol

318:         for (uint256 i; i < splitterAddresses.length;) {

```

#### <a name="GAS-6"></a>[GAS-6] 변경되지 않는 함수 인수에 대해 메모리 대신 calldata 사용
가능한 경우 데이터 유형을 `memory` 대신 `calldata`로 표시하십시오. 이렇게 하면 데이터가 자동으로 메모리에 로드되지 않습니다. 함수에 전달된 데이터를 변경할 필요가 없는 경우(예: 배열 값 업데이트) `calldata`로 전달할 수 있습니다. 한 가지 예외는 인수가 나중에 `memory` 저장소를 지정하는 인수를 취하는 다른 함수로 전달되어야 하는 경우입니다.

*Instances (3)*:
```solidity
File: lib/IOwnershipTransferReceiver.sol

6:     function onOwnershipTransferred(address oldOwner, bytes memory data) external payable;

```

```solidity
File: lib/OwnableWithTransferCallback.sol

45:     function transferOwnership(address newOwner, bytes memory data) public payable virtual onlyOwner {

```

```solidity
File: settings/StandardSettings.sol

124:     function onOwnershipTransferred(address prevOwner, bytes memory) public payable {

```

#### <a name="GAS-7"></a>[GAS-7] 사용자 정의 오류(Custom Errors) 사용
[소스](https://blog.soliditylang.org/2021/04/21/custom-errors/)
오류 문자열을 사용하는 대신 배포 및 런타임 비용을 줄이려면 사용자 정의 오류(Custom Errors)를 사용해야 합니다. 이렇게 하면 배포 및 런타임 비용이 모두 절약됩니다.

*Instances (21)*:
```solidity
File: LSSVMRouter.sol

504:         require(factory.isValidPair(msg.sender), "Not pair");

506:         require(factory.getPairTokenType(msg.sender) == ILSSVMPairFactoryLike.PairTokenType.ERC20, "Not ERC20 pair");

522:         require(factory.isValidPair(msg.sender), "Not pair");

536:         require(factory.isValidPair(msg.sender), "Not pair");

549:         require(block.timestamp <= deadline, "Deadline passed");

578:             require(error == CurveErrorCodes.Error.OK, "Bonding curve error");

658:         require(outputAmount >= minOutput, "outputAmount too low");

```

```solidity
File: settings/StandardSettings.sol

45:         require(owner() == address(0), "Initialized");

128:         require(pair.poolType() == ILSSVMPair.PoolType.TRADE, "Only TRADE pairs");

131:         require(pair.fee() <= MAX_SETTABLE_FEE, "Fee too high");

134:         require(msg.value == getSettingsCost(), "Insufficient payment");

144:             revert("Pair verification failed");

178:             require(block.timestamp > pairInfo.unlockTime, "Lockup not over");

180:             revert("Not prev owner or authed");

214:         require(msg.sender == pairInfo.prevOwner, "Not prev owner");

215:         require(newFee <= MAX_SETTABLE_FEE, "Fee too high");

231:         require(msg.sender == pairInfo.prevOwner, "Not prev owner");

309:         revert("Pricing and liquidity mismatch");

```

```solidity
File: settings/StandardSettingsFactory.sol

28:         require(royaltyBps <= (BASE / 10), "Max 10% for modified royalty bps");

29:         require(feeSplitBps <= BASE, "Max 100% for trade fee bps split");

30:         require(secDuration <= ONE_YEAR_SECS, "Max lock duration 1 year");

```

#### <a name="GAS-8"></a>[GAS-8] 기본값으로 변수 초기화하지 않기

*Instances (10)*:
```solidity
File: RoyaltyEngine.sol

35:     int16 private constant NOT_CONFIGURED = 0;

81:         for (uint256 i = 0; i < numTokens;) {

182:                 for (uint256 i = 0; i < royalties.length;) {

264:                 for (uint256 i = 0; i < royalties.length;) {

336:         for (uint256 i = 0; i < numBps;) {

349:         for (uint256 i = 0; i < numRoyalties;) {

```

```solidity
File: VeryFastRouter.sol

125:             for (uint256 i = 0; i < prices.length;) {

217:             for (uint256 i = 0; i < outputAmounts.length;) {

632:         uint256 numIdsFound = 0;

```

```solidity
File: erc721/LSSVMPairERC721.sol

257:                     for (uint256 i = 0; i < numNFTs;) {

```

#### <a name="GAS-9"></a>[GAS-9] 상수(constants)에 `public` 대신 `private` 사용 시 가스 절약
필요한 경우 확인된 계약 소스 코드에서 값을 읽을 수 있거나, 여러 값이 있는 경우 현재 공개된 모든 상수의 값을 [튜플로 반환](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178)하는 단일 getter 함수를 사용할 수 있습니다. 컴파일러가 배포 calldata에 대한 비지불(non-payable) getter 함수를 생성할 필요가 없고, 사용되는 곳 외부에 값의 바이트를 저장할 필요가 없으며, 메서드 ID 테이블에 다른 항목을 추가할 필요가 없으므로 배포 가스에서 **3406-3606 가스**를 절약합니다.

*Instances (2)*:
```solidity
File: bonding-curves/ExponentialCurve.sol

15:     uint256 public constant MIN_PRICE = 1000000 wei;

```

```solidity
File: bonding-curves/GDACurve.sol

21:     uint256 public constant MIN_PRICE = 1 gwei;

```

#### <a name="GAS-10"></a>[GAS-10] 가능한 경우 나눗셈/곱셈 대신 시프트 오른쪽/왼쪽 사용
N만큼 왼쪽으로 시프트하는 것은 2^N을 곱하는 것과 같고 N만큼 오른쪽으로 시프트하는 것은 2^N으로 나누는 것과 같습니다.

*Instances (13)*:
```solidity
File: LSSVMPairFactory.sol

335:     }

```

```solidity
File: VeryFastRouter.sol

535:             );

544:             ) {

546:             }

550:                 start = (start + end) / 2 + 1;

551:                 priceToFillAt = currentCost;

594:                 // get feeMultiplier from deltaAndFeeMultiplier

605:                     || currentOutput > pairTokenBalance

608:             }

612:                 start = (start + end) / 2 + 1;

613:                 priceToFillAt = currentOutput;

```

```solidity
File: bonding-curves/LinearCurve.sol

77:

145:

```

#### <a name="GAS-11"></a>[GAS-11] 구조체/배열에 `memory` 대신 `storage` 사용
`memory`를 사용하면 구조체나 배열이 메모리에 복사됩니다. `storage`를 사용하여 저장소에 위치를 저장하고 더 저렴하게 읽으십시오:

*Instances (29)*:
```solidity
File: LSSVMPair.sol

492:             ROYALTY_ENGINE.getRoyalty(nft(), assetId, saleAmount);

505:             ROYALTY_ENGINE.getRoyaltyView(nft(), assetId, saleAmount);

679:             if (!success && revertOnFail) {

```

```solidity
File: LSSVMRouter.sol

299:                 if (nftIds.length == 0) {

```

```solidity
File: RoyaltyEngine.sol

87:                 _getRoyaltyAndSpec(tokenAddresses[i], tokenIds[i], values[i]);

126:             _getRoyaltyAndSpec(tokenAddress, tokenId, value);

318:             abi.encodeWithSelector(IRoyaltyRegistry.getRoyaltyLookupAddress.selector, tokenAddress)

```

```solidity
File: VeryFastRouter.sol

100:

180:         arr[0] = valueToWrap;

194:

437:

632:         uint256 numIdsFound = 0;

653:             for (uint256 i; i < numIdsFound;) {

663:         return emptyArr;

```

```solidity
File: erc1155/LSSVMPairERC1155.sol

60:             _calculateRoyalties(nftId(), inputAmountExcludingRoyalty - protocolFee - tradeFee);

129:             _calculateRoyalties(nftId(), outputAmount);

213:             ids[0] = _nftId;

215:             amounts[0] = numNFTs;

```

```solidity
File: erc721/LSSVMPairERC721.sol

56:             _calculateRoyalties(nftIds[0], inputAmountExcludingRoyalty - protocolFee - tradeFee);

176:             _calculateRoyalties(nftIds[0], outputAmount);

```

```solidity
File: property-checking/MerklePropertyChecker.sol

22:         uint256 numIds = ids.length;

25:             if (!MerkleProof.verify(proof, root, keccak256(abi.encodePacked(ids[i])))) {

```

```solidity
File: property-checking/PropertyCheckerFactory.sol

27:         MerklePropertyChecker checker = MerklePropertyChecker(address(merklePropertyCheckerImplementation).clone(data));

37:         RangePropertyChecker checker = RangePropertyChecker(address(rangePropertyCheckerImplementation).clone(data));

```

```solidity
File: settings/StandardSettings.sol

158:         address splitterAddress = address(splitterImplementation).clone(data);

172:

213:         // Verify that the caller is the previous owner of the pair

229:

```

```solidity
File: settings/StandardSettingsFactory.sol

32:         settings = StandardSettings(address(standardSettingsImplementation).clone(data));

```

#### <a name="GAS-12"></a>[GAS-12] 부호 없는 정수 비교 시 > 0 대신 != 0 사용

*Instances (3)*:
```solidity
File: LSSVMRouter.sol

233:         if (remainingValue > 0) {

372:             if (remainingValue > 0) {

592:         if (remainingValue > 0) {

```

#### <a name="GAS-13"></a>[GAS-13] 계약에서 호출하지 않는 `internal` 함수 제거
함수가 인터페이스에 필요한 경우 계약은 해당 인터페이스를 상속하고 `override` 키워드를 사용해야 합니다.

*Instances (8)*:
```solidity
File: lib/LSSVMPairCloner.sol

22:     function cloneERC721ETHPair(

112:     function cloneERC721ERC20Pair(

204:     function isERC721ETHPairClone(address factory, address implementation, address query)

238:     function isERC721ERC20PairClone(address factory, address implementation, address query)

274:         address implementation,

361:         ILSSVMPairFactoryLike factory,

450:         returns (bool result)

484:         returns (bool result)

```


### Non Critical Issues


| |Issue|Instances|
|-|:-|:-:|
| [NC-1](#NC-1) | Missing checks for `address(0)` when assigning values to address state variables | 4 |
| [NC-2](#NC-2) |  `require()` / `revert()` statements should have descriptive reason strings | 1 |
| [NC-3](#NC-3) | Event is missing `indexed` fields | 18 |
| [NC-4](#NC-4) | Constants should be defined rather than using magic numbers | 3 |
| [NC-5](#NC-5) | Functions not used internally could be marked external | 28 |
| [NC-6](#NC-6) | Typos | 84 |
#### <a name="NC-1"></a>[NC-1] 주소 상태 변수에 값을 할당할 때 `address(0)`에 대한 확인 누락

*Instances (4)*:
```solidity
File: LSSVMPairFactory.sol

110:         _caller = _NOT_ENTERED;

349:         _caller = _NOT_ENTERED;

```

```solidity
File: lib/OwnableWithTransferCallback.sol

25:         _owner = initialOwner;

71:         _owner = newOwner;

```

#### <a name="NC-2"></a>[NC-2] `require()` / `revert()` 문에 설명적인 이유 문자열이 있어야 함

*Instances (1)*:
```solidity
File: LSSVMPairETH.sol

126:         require(msg.data.length == _immutableParamsLength());

```

#### <a name="NC-3"></a>[NC-3] 이벤트에 `indexed` 필드 누락
이벤트 필드를 인덱싱하면 이벤트를 파싱하는 오프체인 도구에서 필드에 더 빠르게 액세스할 수 있습니다. 그러나 각 인덱스 필드는 방출 중에 추가 가스가 들므로 이벤트당 허용되는 최대값(3개 필드)을 인덱싱하는 것이 반드시 최선은 아닙니다. 3개 이상의 필드가 있고 해당 이벤트에 대해 가스 사용량이 특별히 우려되지 않는 경우 각 이벤트는 3개의 인덱스 필드를 사용해야 합니다. 필드가 3개 미만인 경우 모든 필드를 인덱싱해야 합니다.

*Instances (18)*:
```solidity
File: LSSVMPair.sol

82:     event SwapNFTInPair(uint256 amountIn, uint256[] ids);

83:     event SwapNFTInPair(uint256 amountIn, uint256 numNFTs);

84:     event SwapNFTOutPair(uint256 amountOut, uint256[] ids);

85:     event SwapNFTOutPair(uint256 amountOut, uint256 numNFTs);

86:     event SpotPriceUpdate(uint128 newSpotPrice);

87:     event TokenDeposit(uint256 amount);

88:     event TokenWithdrawal(uint256 amount);

89:     event NFTWithdrawal(uint256[] ids);

90:     event NFTWithdrawal(uint256 numNFTs);

91:     event DeltaUpdate(uint128 newDelta);

92:     event FeeUpdate(uint96 newFee);

```

```solidity
File: LSSVMPairFactory.sol

75:     event ERC20Deposit(address indexed poolAddress, uint256 amount);

76:     event NFTDeposit(address indexed poolAddress, uint256[] ids);

77:     event ERC1155Deposit(address indexed poolAddress, uint256 indexed id, uint256 amount);

79:     event ProtocolFeeMultiplierUpdate(uint256 newMultiplier);

80:     event BondingCurveStatusUpdate(ICurve indexed bondingCurve, bool isAllowed);

81:     event CallTargetStatusUpdate(address indexed target, bool isAllowed);

82:     event RouterStatusUpdate(LSSVMRouter indexed router, bool isAllowed);

```

#### <a name="NC-4"></a>[NC-4] 매직 넘버 대신 상수 정의 사용

*Instances (3)*:
```solidity
File: settings/Splitter.sol

23:         return _getArgAddress(20);

```

```solidity
File: settings/StandardSettings.sol

70:         return _getArgUint64(40);

77:         return _getArgUint64(48);

```

#### <a name="NC-5"></a>[NC-5] 내부적으로 사용되지 않는 함수는 external로 표시 가능

*Instances (28)*:
```solidity
File: LSSVMPair.sol

323:     function getAssetRecipient() public view returns (address payable) {

344:     function getFeeRecipient() public view returns (address payable _feeRecipient) {

```

```solidity
File: LSSVMPairFactory.sol

342:     function openLock() public {

347:     function closeLock() public {

499:     function getSettingsForPair(address pairAddress) public view returns (bool settingsEnabled, uint96 bps) {

513:     function toggleSettingsForCollection(address settings, address collectionAddress, bool enable) public {

529:     function enableSettingsForPair(address settings, address pairAddress) public {

546:     function disableSettingsForPair(address settings, address pairAddress) public {

```

```solidity
File: RoyaltyEngine.sol

66:     function getCachedRoyaltySpec(address tokenAddress) public view returns (int16) {

77:     function bulkCacheSpecs(address[] calldata tokenAddresses, uint256[] calldata tokenIds, uint256[] calldata values)

100:     function getRoyalty(address tokenAddress, uint256 tokenId, uint256 value)

119:     function getRoyaltyView(address tokenAddress, uint256 tokenId, uint256 value)

```

```solidity
File: erc721/LSSVMPairERC721ERC20.sol

27:     function pairVariant() public pure override returns (ILSSVMPairFactoryLike.PairVariant) {

```

```solidity
File: erc721/LSSVMPairERC721ETH.sol

27:     function pairVariant() public pure override returns (ILSSVMPairFactoryLike.PairVariant) {

```

```solidity
File: property-checking/PropertyCheckerFactory.sol

25:     function createMerklePropertyChecker(bytes32 root) public returns (MerklePropertyChecker) {

32:     function createRangePropertyChecker(uint256 startInclusive, uint256 endInclusive)

```

```solidity
File: settings/Splitter.sol

26:     function withdrawAllETHInSplitter() public {

41:     function withdrawAllBaseQuoteTokens() public {

47:     function withdrawAllTokens(ERC20 token) public {

```

```solidity
File: settings/StandardSettings.sol

44:     function initialize(address _owner, address payable _settingsFeeRecipient) public {

69:     function getFeeSplitBps() public pure returns (uint64) {

85:     function setSettingsFeeRecipient(address payable newFeeRecipient) public onlyOwner {

95:     function getPrevFeeRecipientForPair(address pairAddress) public view returns (address) {

124:     function onOwnershipTransferred(address prevOwner, bytes memory) public payable {

170:     function reclaimPair(address pairAddress) public {

211:     function changeFee(address pairAddress, uint96 newFee) public {

225:     function changeSpotPriceAndDelta(address pairAddress, uint128 newSpotPrice, uint128 newDelta, uint256 assetId)

```

```solidity
File: settings/StandardSettingsFactory.sol

21:     function createSettings(

```

#### <a name="NC-6"></a>[NC-6] 오타

*Instances (84)*:
```diff
File: LSSVMPair.sol

- 52:     // Sudoswap Royalty Engine
+ 52:     // @notice Sudoswap Royalty Engine

- 60:     // However, this should NOT be assumed, as bonding curves may use spotPrice in different ways.
+ 60:     // @notice However, this should NOT be assumed, as bonding curves may use spotPrice in different ways.

- 61:     // Use getBuyNFTQuote and getSellNFTQuote for accurate pricing info.
+ 61:     // @notice Use getBuyNFTQuote and getSellNFTQuote for accurate pricing info.

- 65:     // Units and meaning are bonding curve dependent.
+ 65:     // @notice Units and meaning are bonding curve dependent.

- 68:     // The spread between buy and sell prices, set to be a multiplier we apply to the buy price
+ 68:     // @notice The spread between buy and sell prices, set to be a multiplier we apply to the buy price

- 69:     // Fee is only relevant for TRADE pools
+ 69:     // @notice Fee is only relevant for TRADE pools

- 70:     // Units are in base 1e18
+ 70:     // @notice Units are in base 1e18

- 73:     // The address that swapped assets are sent to
+ 73:     // @notice The address that swapped assets are sent to

- 74:     // For TRADE pools, assets are always sent to the pool, so this is used to track trade fee
+ 74:     // @notice For TRADE pools, assets are always sent to the pool, so this is used to track trade fee

- 75:     // If set to address(0), will default to owner() for NFT and TOKEN pools
+ 75:     // @notice If set to address(0), will default to owner() for NFT and TOKEN pools

- 235:             // Calculate the inputAmount minus tradeFee and protocolFee
+ 235:             // @notice Calculate the inputAmount minus tradeFee and protocolFee

- 238:             // Compute royalties
+ 238:             // @notice Compute royalties

- 265:             // Compute royalties
+ 265:             // @notice Compute royalties

- 268:             // Deduct royalties from outputAmount
+ 268:             // @notice Deduct royalties from outputAmount

- 270:                 // Safe because we already require outputAmount >= royaltyAmount in _calculateRoyalties()
+ 270:                 // @notice Safe because we already require outputAmount >= royaltyAmount in _calculateRoyalties()

- 324:         // TRADE pools will always receive the asset themselves
+ 324:         // @notice TRADE pools will always receive the asset themselves

- 331:         // Otherwise, we return the recipient if it's been set
+ 331:         // @notice Otherwise, we return the recipient if it's been set

- 332:         // Or, we replace it with owner() if it's address(0)
+ 332:         // @notice Or, we replace it with owner() if it's address(0)

- 369:         // Save on 2 SLOADs by caching
+ 369:         // @notice Save on 2 SLOADs by caching

- 377:         // Revert if bonding curve had an error
+ 377:         // @notice Revert if bonding curve had an error

- 382:         // Consolidate writes to save gas
+ 382:         // @notice Consolidate writes to save gas

- 388:         // Emit spot price update if it has been updated
+ 388:         // @notice Emit spot price update if it has been updated

- 393:         // Emit delta update if it has been updated
+ 393:         // @notice Emit delta update if it has been updated

- 413:         // Save on 2 SLOADs by caching
+ 413:         // @notice Save on 2 SLOADs by caching

- 421:         // Revert if bonding curve had an error
+ 421:         // @notice Revert if bonding curve had an error

- 426:         // Consolidate writes to save gas
+ 426:         // @notice Consolidate writes to save gas

- 432:         // Emit spot price update if it has been updated
+ 432:         // @notice Emit spot price update if it has been updated

- 437:         // Emit delta update if it has been updated
+ 437:         // @notice Emit delta update if it has been updated

- 517:         // cache to save gas
+ 517:         // @notice cache to save gas

- 521:             // If a pair has custom Settings, use the overridden royalty amount and only use the first receiver
+ 521:             // @notice If a pair has custom Settings, use the overridden royalty amount and only use the first receiver

- 529:                 // update numRecipients to match new recipients list
+ 529:                 // @notice update numRecipients to match new recipients list

- 544:         // Ensure royalty total is at most 25% of the sale amount
+ 544:         // @notice Ensure royalty total is at most 25% of the sale amount

- 545:         // This defends against a rogue Manifold registry that charges extremely
+ 545:         // @notice This defends against a rogue Manifold registry that charges extremely

- 546:         // high royalties
+ 546:         // @notice high royalties

- 644:         // Ensure the call isn't calling a banned function
+ 644:         // @notice Ensure the call isn't calling a banned function

- 655:         // Prevent calling the pair's underlying nft
+ 655:         // @notice Prevent calling the pair's underlying nft

- 656:         // (We ban calling the underlying NFT/ERC20 to avoid maliciously transferring assets approved for the pair to spend)
+ 656:         // @notice (We ban calling the underlying NFT/ERC20 to avoid maliciously transferring assets approved for the pair to spend)

- 675:             // We ban calling transferOwnership when ownership
+ 675:             // @notice We ban calling transferOwnership when ownership

```

```diff
File: LSSVMPairERC20.sol

- 92:             // Transfer royalties (if they exists)
+ 92:             // Transfer royalties (if they exist)

- 105:         // Send trade fee if it exists, is TRADE pool, and fee recipient != pool address
+ 105:         // Send trade fee if it exists, is TRADE pool, and fee recipient is not pool address

```

```diff
File: LSSVMPairETH.sol

- 31:         // Require that the input amount is sufficient to pay for the sale amount and royalties
+ 31:         // Require that the input amount is sufficient to pay for the sale amount, royalties, and fees

- 34:         // Transfer inputAmountExcludingRoyalty ETH to assetRecipient if it's been set
+ 34:         // Transfer inputAmountExcludingRoyalty ETH to assetRecipient if it has been set

```

```diff
File: LSSVMPairFactory.sol

- 470:         // ensure target is not / was not ever a router
+ 470:         // Ensure target is not / was not ever a router

- 485:         // ensure target is not arbitrarily callable by pairs
+ 485:         // Ensure target is not arbitrarily callable by pairs

- 570:         // transfer initial ETH to pair
+ 570:         // Transfer initial ETH to pair

- 573:         // transfer initial NFTs from sender to pair
+ 573:         // Transfer initial NFTs from sender to pair

- 598:         // transfer initial tokens to pair (if != 0)
+ 598:         // Transfer initial tokens to pair (if != 0)

- 603:         // transfer initial NFTs from sender to pair
+ 603:         // Transfer initial NFTs from sender to pair

- 627:         // transfer initial ETH to pair
+ 627:         // Transfer initial ETH to pair

- 630:         // transfer initial NFTs from sender to pair
+ 630:         // Transfer initial NFTs from sender to pair

- 651:         // transfer initial tokens to pair
+ 651:         // Transfer initial tokens to pair

- 656:         // transfer initial NFTs from sender to pair
+ 656:         // Transfer initial NFTs from sender to pair

- 668:         // early return for trivial transfers
+ 668:         // Early return for trivial transfers

- 671:         // transfer NFTs from caller to recipient
+ 671:         // Transfer NFTs from caller to recipient

- 688:         // early return for trivial transfers
+ 688:         // Early return for trivial transfers

```

```diff
File: VeryFastRouter.sol

- 116:             // Set the price to buy numNFT - i items
+ 116:             // Set the price to buy numNFTs - i items

- 204:             // Calculate output to sell the remaining numNFTs - i items, factoring in royalties
+ 204:             // Calculate output to sell the remaining numNFTs - i items, factoring in royalties and fees

```

```diff
File: bonding-curves/CurveErrorCodes.sol

- 7:         INVALID_NUMITEMS, // The numItem value is 0
+ 7:         INVALID_NUMITEMS, // The numItems value is 0

- 10:         SPOT_PRICE_UNDERFLOW // The updated spot price goes too low
+ 10:         SPOT_PRICE_UNDERFLOW // The updated spot price is too low

```

```diff
File: bonding-curves/ExponentialCurve.sol

- 53:         // NOTE: we assume delta is > 1, as checked by validateDelta()
+ 53:         // NOTE: we assume delta is > 1, as checked by validateDelta

- 123:         // NOTE: we assume delta is > 1, as checked by validateDelta()
+ 123:         // NOTE: we assume delta is > 1, as checked by validateDelta

- 153:         // Account for the trade fee, only for Trade pools
+ 153:         // Account for the trade fee, only for TRADE pools

```

```diff
File: bonding-curves/GDACurve.sol

- 218:         // however, because our alpha value needs to be 18 decimals of precision, we multiple by a scaling factor
+ 218:         // however, because our alpha value needs to be 18 decimals of precision, we multiply by a scaling factor

- 224:         // lambda also needs to be 18 decimals of precision so we multiple by a scaling factor
+ 224:         // lambda also needs to be 18 decimals of precision so we multiply by a scaling factor

- 228:         // this works because solidity cuts off higher bits when converting
+ 228:         // this works because solidity cuts off higher bits when converting from a larger type to a smaller type

- 229:         // from a larger type to a smaller type
+ 229:         // see https://docs.soliditylang.org/en/latest/types.html#explicit-conversions

- 230:         // see https://docs.soliditylang.org/en/latest/types.html#explicit-conversions
+ 230:         // Clear lower 48 bits

- 235:         // Clear lower 48 bits
+ 235:         // Set lower 48 bits to be the current timestamp

237:         // Set lower 48 bits to be the current timestamp

```

```diff
File: bonding-curves/LinearCurve.sol

- 69:         // The new spot price would become (S+delta), so selling would also yield (S+delta) ETH.
+ 69:         // The new spot price would become (S+delta), so selling would also yield (S+delta) ETH, netting them delta ETH profit.

- 75:         // because we have n instances of buy spot price, and then we sum up from delta to (n-1)*delta
+ 75:         // because we have n instances of buy spot price, and then we sum up from 1*delta to (n-1)*delta

```

```diff
File: bonding-curves/XykCurve.sol

- 63:         // If numItems is too large, we will get divide by zero error
+ 63:         // If numItems is too large, we will get a divide by zero error

- 84:         // If we got all the way here, no math error happened
+ 84:         // If we got all the way here, no math errors happened

- 134:         // If we got all the way here, no math error happened
+ 134:         // If we got all the way here, no math errors happened

```

```diff
File: erc1155/LSSVMPairERC1155.sol

- 258:             // check if we need to emit an event for withdrawing the NFT this pool is trading
+ 258: // Check if we need to emit an event for withdrawing the NFT this pool is trading

- 272:                 // only emit for the pair's NFT
+ 272: // Only emit for the pair's NFT

```

```diff
File: erc721/LSSVMPairERC721.sol

- 254:                 // If more than 1 NFT is being transfered, and there is no property checker, we can do a balance check instead of an ownership check, as pools are indifferent between NFTs from the same collection
+ 254: // If more than 1 NFT is being transferred, and there is no property checker, we can do a balance check instead of an ownership check, as pools are indifferent between NFTs from the same collection

```

```diff
File: lib/LSSVMPairCloner.sol

- 170:             // 60 0x33     | PUSH1 0x33            | 0x33 sucess 0 rds       | [0, rds) = return data
+ 170:             // 60 0x33     | PUSH1 0x33            | 0x33 success 0 rds      | [0, rds) = return data

- 417:             // 60 0x33     | PUSH1 0x33            | 0x33 sucess 0 rds       | [0, rds) = return data
+ 417:             // 60 0x33     | PUSH1 0x33            | 0x33 success 0 rds      | [0, rds) = return data

```

```diff
File: lib/OwnableWithTransferCallback.sol

- 51:             // If revert...
+ 51: /// If revert...

- 53:                 // If we just transferred to a contract w/ no callback, this is fine
+ 53: /// If we just transferred to a contract w/ no callback, this is fine

- 55:                     // i.e., no need to revert
+ 55: /// i.e., no need to revert

- 57:                 // Otherwise, the callback had an error, and we should revert
+ 57: /// Otherwise, the callback had an error, and we should revert

```

```diff
File: settings/StandardSettings.sol

- 26:     uint96 constant MAX_SETTABLE_FEE = 0.2e18; // Max fee of 20%
+ 26:     uint96 constant MAX_SETTABLE_FEE = 0.2e18; // Max fee of 20% (0.2)

```


### Low Issues


| |Issue|Instances|
|-|:-|:-:|
| [L-1](#L-1) |  `abi.encodePacked()` should not be used with dynamic types when passing the result to a hash function such as `keccak256()` | 1 |
| [L-2](#L-2) | Empty Function Body - Consider commenting why | 32 |
| [L-3](#L-3) | Initializers could be front-run | 10 |
| [L-4](#L-4) | Unsafe ERC20 operation(s) | 7 |
#### <a name="L-1"></a>[L-1] `keccak256()`과 같은 해시 함수에 결과를 전달할 때 동적 유형과 함께 `abi.encodePacked()`를 사용해서는 안 됨
항목을 32바이트로 패딩하여 [해시 충돌을 방지](https://docs.soliditylang.org/en/v0.8.13/abi-spec.html#non-standard-packed-mode)하는 `abi.encode()`를 대신 사용하십시오(예: `abi.encodePacked(0x123,0x456)` => `0x123456` => `abi.encodePacked(0x1,0x23456)`, 하지만 `abi.encode(0x123,0x456)` => `0x0...1230...456`). "설득력 있는 이유가 없다면 `abi.encode`를 선호해야 합니다". `abi.encodePacked()`에 대한 인수가 하나만 있는 경우 종종 `bytes()` 또는 `bytes32()`로 [대신](https://ethereum.stackexchange.com/questions/30912/how-to-compare-strings-in-solidity#answer-82739) 캐스팅할 수 있습니다.
모든 인수가 문자열 또는 바이트인 경우 `bytes.concat()`을 대신 사용해야 합니다.

*Instances (1)*:
```solidity
File: property-checking/MerklePropertyChecker.sol

25:             if (!MerkleProof.verify(proof, root, keccak256(abi.encodePacked(ids[i])))) {

```

#### <a name="L-2"></a>[L-2] 빈 함수 본문 - 이유 주석 처리 고려

*Instances (32)*:
```solidity
File: LSSVMPairETH.sol

130:     function _preCallCheck(address) internal pure override {}

```

```solidity
File: LSSVMPairFactory.sol

373:                 } catch {}

375:         } catch {}

379:         } catch {}

384:             } catch {}

385:         } catch {}

390:             } catch {}

391:         } catch {}

398:             } catch {}

399:         } catch {}

403:         } catch {}

410:     receive() external payable {}

```

```solidity
File: LSSVMRouter.sol

488:     receive() external payable {}

```

```solidity
File: RoyaltyEngine.sol

164:             } catch {}

170:             } catch {}

176:             } catch {}

191:             } catch {}

197:                 } catch {}

198:             } catch {}

211:                     } catch {}

212:                 } catch {}

219:             } catch {}

225:             } catch {}

231:             } catch {}

```

```solidity
File: VeryFastRouter.sol

488:     receive() external payable {}

```

```solidity
File: erc1155/LSSVMPairERC1155ERC20.sol

18:     constructor(IRoyaltyEngineV1 royaltyEngine) LSSVMPair(royaltyEngine) {}

```

```solidity
File: erc1155/LSSVMPairERC1155ETH.sol

18:     constructor(IRoyaltyEngineV1 royaltyEngine) LSSVMPair(royaltyEngine) {}

```

```solidity
File: erc721/LSSVMPairERC721ERC20.sol

18:     constructor(IRoyaltyEngineV1 royaltyEngine) LSSVMPair(royaltyEngine) {}

```

```solidity
File: erc721/LSSVMPairERC721ETH.sol

18:     constructor(IRoyaltyEngineV1 royaltyEngine) LSSVMPair(royaltyEngine) {}

```

```solidity
File: lib/OwnableWithTransferCallback.sol

50:             try IOwnershipTransferReceiver(newOwner).onOwnershipTransferred{value: msg.value}(msg.sender, data) {}

```

```solidity
File: settings/Splitter.sol

62:     fallback() external payable {}

```

```solidity
File: settings/StandardSettings.sol

142:         try pairFactory.enableSettingsForPair(address(this), msg.sender) {}

```

#### <a name="L-3"></a>[L-3] 초기화자가 프론트러닝(front-run)될 수 있음
초기화자가 프론트러닝되어 공격자가 자신의 값을 설정하거나 계약의 소유권을 가져갈 수 있으며, 최선의 경우에도 재배포를 강제할 수 있습니다.

*Instances (10)*:
```solidity
File: LSSVMPair.sol

132:     function initialize(

140:         __Ownable_init(_owner);

```

```solidity
File: LSSVMPairFactory.sol

568:         _pair.initialize(msg.sender, _assetRecipient, _delta, _fee, _spotPrice);

596:         _pair.initialize(msg.sender, _assetRecipient, _delta, _fee, _spotPrice);

625:         _pair.initialize(msg.sender, _assetRecipient, _delta, _fee, _spotPrice);

649:         _pair.initialize(msg.sender, _assetRecipient, _delta, _fee, _spotPrice);

```

```solidity
File: lib/OwnableWithTransferCallback.sol

24:     function __Ownable_init(address initialOwner) internal {

```

```solidity
File: settings/StandardSettings.sol

44:     function initialize(address _owner, address payable _settingsFeeRecipient) public {

46:         __Ownable_init(_owner);

```

```solidity
File: settings/StandardSettingsFactory.sol

33:         settings.initialize(msg.sender, settingsFeeRecipient);

```

#### <a name="L-4"></a>[L-4] 안전하지 않은 ERC20 작업

*Instances (7)*:
```solidity
File: LSSVMPairFactory.sol

576:             _nft.transferFrom(msg.sender, address(_pair), _initialNFTIDs[i]);

606:             _nft.transferFrom(msg.sender, address(_pair), _initialNFTIDs[i]);

673:             _nft.transferFrom(msg.sender, recipient, ids[i]);

```

```solidity
File: LSSVMRouter.sol

525:         nft.transferFrom(from, to, id);

```

```solidity
File: VeryFastRouter.sol

713:         nft.transferFrom(from, to, id);

```

```solidity
File: erc721/LSSVMPairERC721.sol

218:             _nft.transferFrom(address(this), nftRecipient, nftIds[i]);

281:                     _nft.transferFrom(msg.sender, _assetRecipient, nftIds[i]);

```


### Medium Issues


| |Issue|Instances|
|-|:-|:-:|
| [M-1](#M-1) | Centralization Risk for trusted owners | 23 |
| [M-2](#M-2) |  Solmate's SafeTransferLib does not check for token contract's existence | 18 |
#### <a name="M-1"></a>[M-1] 신뢰할 수 있는 소유자에 대한 중앙화 위험

**파급력 (Impact):**
계약에는 관리 작업을 수행할 수 있는 특권이 있는 소유자가 있으며 악의적인 업데이트를 수행하거나 자금을 유출하지 않도록 신뢰해야 합니다.

*Instances (23)*:
```solidity
File: LSSVMPair.sol

582:     function changeSpotPrice(uint128 newSpotPrice) external onlyOwner {

595:     function changeDelta(uint128 newDelta) external onlyOwner {

610:     function changeFee(uint96 newFee) external onlyOwner {

625:     function changeAssetRecipient(address payable newRecipient) external onlyOwner {

640:     function call(address payable target, bytes calldata data) external onlyOwner {

672:     function multicall(bytes[] calldata calls, bool revertOnFail) external onlyOwner {

```

```solidity
File: LSSVMPairERC20.sol

129:     function withdrawERC20(ERC20 a, uint256 amount) external override onlyOwner {

```

```solidity
File: LSSVMPairETH.sol

90:     function withdrawAllETH() external onlyOwner {

100:     function withdrawETH(uint256 amount) public onlyOwner {

108:     function withdrawERC20(ERC20 a, uint256 amount) external override onlyOwner {

```

```solidity
File: LSSVMPairFactory.sol

39: contract LSSVMPairFactory is Owned, ILSSVMPairFactoryLike {

102:     ) Owned(_owner) {

420:     function withdrawETHProtocolFees() external onlyOwner {

429:     function withdrawERC20ProtocolFees(ERC20 token, uint256 amount) external onlyOwner {

437:     function changeProtocolFeeRecipient(address payable _protocolFeeRecipient) external onlyOwner {

447:     function changeProtocolFeeMultiplier(uint256 _protocolFeeMultiplier) external onlyOwner {

458:     function setBondingCurveAllowed(ICurve bondingCurve, bool isAllowed) external onlyOwner {

469:     function setCallAllowed(address payable target, bool isAllowed) external onlyOwner {

484:     function setRouterAllowed(LSSVMRouter _router, bool isAllowed) external onlyOwner {

```

```solidity
File: erc1155/LSSVMPairERC1155.sol

235:     function withdrawERC721(IERC721 a, uint256[] calldata nftIds) external virtual override onlyOwner {

```

```solidity
File: erc721/LSSVMPairERC721.sol

299:     function withdrawERC721(IERC721 a, uint256[] calldata nftIds) external virtual override onlyOwner {

```

```solidity
File: lib/OwnableWithTransferCallback.sol

45:     function transferOwnership(address newOwner, bytes memory data) public payable virtual onlyOwner {

```

```solidity
File: settings/StandardSettings.sol

85:     function setSettingsFeeRecipient(address payable newFeeRecipient) public onlyOwner {

```

#### <a name="M-2"></a>[M-2] Solmate의 SafeTransferLib는 토큰 계약의 존재를 확인하지 않음
Solmate의 SafeTransferLib와 OZ의 SafeERC20 구현 사이에는 미묘한 차이가 있습니다. OZ의 SafeERC20은 토큰이 계약인지 아닌지 확인하지만 Solmate의 SafeTransferLib는 확인하지 않습니다.
https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol#L9
`@dev 이 라이브러리의 어떤 함수도 토큰에 코드가 있는지 확인하지 않는다는 점에 유의하십시오! 그 책임은 호출자에게 위임됩니다`


*Instances (18)*:
```solidity
File: LSSVMPairERC20.sol

90:             token_.safeTransferFrom(msg.sender, _assetRecipient, inputAmountExcludingRoyalty - protocolFee);

94:                 token_.safeTransferFrom(msg.sender, royaltyRecipients[i], royaltyAmounts[i]);

102:                 token_.safeTransferFrom(msg.sender, address(factory()), protocolFee);

110:                 token().safeTransfer(_feeRecipient, tradeFeeAmount);

124:             token().safeTransfer(tokenRecipient, outputAmount);

130:         a.safeTransfer(msg.sender, amount);

```

```solidity
File: LSSVMPairETH.sol

109:         a.safeTransfer(msg.sender, amount);

```

```solidity
File: LSSVMPairFactory.sol

430:         token.safeTransfer(protocolFeeRecipient, amount);

600:             _token.safeTransferFrom(msg.sender, address(_pair), _initialTokenBalance);

632:             _nft.safeTransferFrom(msg.sender, address(_pair), _nftId, _initialNFTBalance, bytes(""));

653:             _token.safeTransferFrom(msg.sender, address(_pair), _initialTokenBalance);

658:             _nft.safeTransferFrom(msg.sender, address(_pair), _nftId, _initialNFTBalance, bytes(""));

691:         token.safeTransferFrom(msg.sender, recipient, amount);

706:         nft.safeTransferFrom(msg.sender, recipient, id, amount, bytes(""));

```

```solidity
File: LSSVMRouter.sol

509:         token.safeTransferFrom(from, to, amount);

```

```solidity
File: VeryFastRouter.sol

690:         token.safeTransferFrom(from, to, amount);

```

```solidity
File: settings/Splitter.sol

55:         token.safeTransfer(parentSettings.settingsFeeRecipient(), amtToSendToSettingsFeeRecipient);

57:         token.safeTransfer(

```
