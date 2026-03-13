**Lead Auditors**

[Immeas](https://x.com/0ximmeas)

[BengalCatBalu](https://x.com/BengalCatBalu)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### `BasisTradeTailor`의 출금 요청 덮어쓰기로 race condition이 발생할 수 있음

**설명:** `BasisTradeTailor::requestWithdrawal` 함수는 기존 출금 요청이 있어도 무조건 덮어씁니다.

```solidity
// BasisTradeTailor.sol:556-560
function requestWithdrawal(address pocket, uint256 amount) external onlyPocketUser(pocket) {
    withdrawalRequests[pocket] = amount;  // Overwrites existing request
    emit WithdrawalRequested(pocket, amount);
}
```

기존 요청 덮어쓰기를 막는 검증도 없고, 명시적인 취소 메커니즘도 없습니다. 사용자가 출금 금액을 수정하려고 하면 agent의 `processWithdrawal()` 호출과 경쟁 상태가 생깁니다.

**영향:** 사용자의 `requestWithdrawal()`과 agent의 `processWithdrawal()` 중 어느 트랜잭션이 먼저 실행되는지에 따라 요청이 교체될 수도 있고 누적될 수도 있어, 사용자가 의도한 것보다 더 많이 출금될 수 있습니다.

사용자는 기존 요청을 수정하기 위해 `requestWithdrawal()`을 다시 호출할 때, 새 금액이 이전 금액을 **대체**할 것이라고 기대합니다. 하지만 agent가 먼저 기존 요청을 처리해버리면, 사용자의 두 번째 호출은 기존 요청을 바꾸는 것이 아니라 **새 요청**을 생성하게 됩니다. 그 결과 두 금액이 모두 출금됩니다.

**개념 증명 (Proof of Concept):**
```
Block N:
  User: requestWithdrawal(pocket, 100 baseAsset)
  withdrawalRequests[pocket] = 100

User realizes they want only 50 total, submits modification:

Block N+1 (both transactions in same block):
  User: requestWithdrawal(pocket, 50)    // User wants to REPLACE 100 with 50
  Agent: processWithdrawal(pocket, 100)  // Agent processes original request

Outcome depends on transaction order within block:

Case 1 - User tx executes first:
  1. withdrawalRequests[pocket] = 50 (replaced)
  2. Agent processes 50 baseAsset
  3. withdrawalRequests[pocket] = 0
  4. Result: User withdraws 50 total

Case 2 - Agent tx executes first:
  1. Agent processes 100 baseAsset, sets withdrawalRequests[pocket] = 0
  2. User sets withdrawalRequests[pocket] = 50 (creates NEW request)
  3. Agent later processes this 50 baseAsset request
  4. Result: User withdraws 150 total (100 + 50)

In Case 2, the user wanted to reduce their total withdrawal to 50 but received 150 due to transaction ordering.
```

사용자가 `requestWithdrawal(0)`으로 취소하려는 경우도 같은 문제가 발생합니다. agent가 먼저 처리해버리면 취소가 반영되기 전에 전체 금액이 출금될 수 있습니다.

**권장 완화 조치:** 먼저 명시적으로 취소하지 않으면 non-zero에서 non-zero로 바꿀 수 없게 하십시오. 요청을 0으로 되돌리는 별도의 `cancelWithdrawal()` 함수를 추가하는 방식입니다.

```solidity
function requestWithdrawal(address pocket, uint256 amount) external onlyPocketUser(pocket) {
    require(amount > 0, "Amount must be positive");
    require(withdrawalRequests[pocket] == 0, "Cancel existing request first");

    withdrawalRequests[pocket] = amount;
    emit WithdrawalRequested(pocket, amount);
}

function cancelWithdrawal(address pocket) external onlyPocketUser(pocket) {
    uint256 currentRequest = withdrawalRequests[pocket];
    require(currentRequest > 0, "No pending request");

    withdrawalRequests[pocket] = 0;
    emit WithdrawalCancelled(pocket, currentRequest);
}
```

이렇게 하면 덮어쓰기를 막을 수 있고(100 → 50 직접 변경 불가), 취소도 명시적인 동작이 됩니다(`requestWithdrawal(0)` 대신 `cancelWithdrawal()` 사용).

**Button:** 커밋 [`2aa92eb`](https://github.com/buttonxyz/button-protocol/commit/2aa92ebd0912eac61451767364cc31fd2671d8fc)에서 수정됨

**Cyfrin:** 확인함. 권고안이 구현되었습니다.


\clearpage
## 낮은 위험 (Low Risk)


### `PocketFactory`는 ERC-165를 준수하지 않음

**설명:** `PocketFactory`는 `IPocketFactory`를 구현하지만 `supportsInterface()`는 이를 검사하지 않습니다.

```solidity
// PocketFactory.sol:93-100
function supportsInterface(bytes4 interfaceId)
    public view override(AccessControlEnumerable) returns (bool)
{
    return super.supportsInterface(interfaceId);  // doesn't check IPocketFactory
}
```

이는 ERC-165 표준 위반입니다. `pocketFactory.supportsInterface(type(IPocketFactory).interfaceId)`를 호출해도, 실제로 인터페이스를 구현하고 있음에도 `false`가 반환됩니다.

**권장 완화 조치:** `IPocketFactory` 인터페이스를 명시적으로 검사하십시오.

```diff
function supportsInterface(bytes4 interfaceId)
    public view override(AccessControlEnumerable) returns (bool)
{
-   return super.supportsInterface(interfaceId);
+   return
+       interfaceId == type(IPocketFactory).interfaceId ||
+       super.supportsInterface(interfaceId);
}
```

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.


### 활성화되지 않은 HyperCore 계정으로의 첫 USDC 전송은 활성화 수수료로 1 USDC를 잃음

**설명:** HyperCore 계정은 활성화되어야 수수료 없이 spot transfer를 받을 수 있습니다. [HyperLiquid 문서](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/activation-gas-fee)에 따르면, 활성화되지 않은 계정으로 들어오는 첫 전송은 자동으로 계정 활성화를 유발하며, 전송 금액에서 약 1 USDC가 활성화 수수료로 차감됩니다.

그런데 `BasisTradeTailor::transferAssetToCore` 함수는 pocket의 Core 계정이 활성화되어 있는지 확인하지 않고 USDC를 CCTP를 통해 HyperCore로 전송합니다.

```solidity
// BasisTradeTailor.sol:295-312
function transferAssetToCore(address pocket, uint64 tokenIndex, uint256 amount) external onlyAgent {
    require(pocketUser[pocket] != address(0), "Pocket does not exist");
    require(amount > 0, "Amount must be positive");

    AssetConfig memory assetConfig = supportedAssets[tokenIndex];
    require(assetConfig.tokenAddress != address(0), "Asset not supported");

    if (tokenIndex == USDC_TOKEN_INDEX) {
        _transferUsdcToCore(pocket, amount);  // No activation check
    } else {
        address systemAddress = CoreWriterEncoder.getTokenSystemAddress(tokenIndex);
        IPocket(pocket).transfer(assetConfig.tokenAddress, systemAddress, amount);
    }
    // ...
}
```

`_transferUsdcToCore()` 내부에서는:

```solidity
// BasisTradeTailor.sol:319-335
function _transferUsdcToCore(address pocket, uint256 amount) internal {
    // Approve CoreDepositWallet
    bytes memory approveData = abi.encodeWithSelector(
        IERC20.approve.selector,
        coreDepositWallet,
        amount
    );
    IPocket(pocket).exec(usdcAddress, approveData);

    // Deposit to HyperCore via CCTP - no activation check!
    bytes memory depositData = abi.encodeWithSelector(
        ICoreDepositWallet.depositFor.selector,
        pocket,
        amount,
        uint32(type(uint32).max)
    );
    IPocket(pocket).exec(coreDepositWallet, depositData);
}
```

[`L1Read`](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore)는 활성화 여부를 검사하는 `coreUserExists(address user)` 메서드를 제공하지만 여기서는 사용되지 않습니다. 또한 배포 스크립트(`script/TransferAssetToCore.s.sol`)도 전송 전에 활성화 여부를 검증하지 않습니다.

**영향:** 새로 생성된 pocket의 Core 계정이 첫 USDC 전송 전에 활성화되지 않았다면:

1. agent가 `transferAssetToCore(pocket, USDC_TOKEN_INDEX, 100e6)`을 호출해 100 USDC가 Core에 도착하길 기대함
2. `coreDepositWallet.depositFor()`를 통한 CCTP 전송이 시작됨
3. HyperLiquid가 미활성 계정을 감지하고 약 1 USDC의 활성화 수수료를 차감함
4. pocket의 Core spot 계정에는 약 99 USDC만 도착함
5. 온체인에는 이 수수료가 부과되었다는 이벤트나 에러가 남지 않음

즉, 각 새 pocket의 첫 Core 전송마다 예상치 못한 1 USDC 손실이 발생할 수 있고, 온체인상으로도 이를 확인할 단서가 없습니다. 오직 신뢰된 `AGENT_ROLE` 주소만 전송을 일으킬 수 있고 수수료도 pocket당 1 USDC로 작지만, 운영자가 오프체인에서 미리 계정을 활성화해 두지 않으면 손실이 발생합니다. 현재는 온체인 강제 수단이나 스크립트 기반 검사가 없고, 운영자의 절차 준수에만 의존하고 있습니다.

**권장 완화 조치:** `transferAssetToCore()` 또는 `_transferUsdcToCore()`에 `coreUserExists` precompile 체크를 추가해 첫 전송 전에 계정 활성화 여부를 확인하십시오. 활성화되지 않았다면, 운영자에게 먼저 오프체인 활성화를 수행하라는 명확한 에러 메시지와 함께 revert시키거나, 본예치 전 별도 스크립트로 활성화(수수료를 덮을 1.1 USDC 전송)를 처리하도록 하십시오.

추가로 `script/TransferAssetToCore.s.sol`도 활성화 상태를 검사해, 미활성 계정으로 전송하려는 경우 운영자에게 경고를 출력하도록 갱신하는 것이 좋습니다.

**Button:** [`8ca2df0`](https://github.com/buttonxyz/button-protocol/commit/8ca2df0164d8aea99bc34dd8a5e0e3ca5ec2234c)에서 수정됨

**Cyfrin:** 확인함. `BasisTradeTailor`에 `activateCoreAccount` 호출이 추가되어 `> 1 USDC`를 전송하도록 했습니다.


### `BasisTradeTailor:transferHypeToCore`의 decimal 불일치로 정밀도가 손실됨

**설명:** `BasisTradeTailor::transferHypeToCore` 함수는 18 decimals(HyperEVM 표준)의 `uint256 amount`를 받지만, HyperCore로 브리징될 때는 이 금액이 8 decimals로 잘립니다. 즉 8 decimals를 초과하는 정밀도는 영구적으로 사라집니다.

```solidity
// BasisTradeTailor.sol:400-412
function transferHypeToCore(address pocket, uint256 amount) external onlyOperator {
    require(pocketUser[pocket] != address(0), "Pocket does not exist");
    require(amount > 0, "Amount must be positive");

    IPocket(pocket).transferNative(
        CoreWriterEncoder.HYPE_SYSTEM_ADDRESS,  // Bridges to Core with 8 decimals
        amount  // uint256 with 18 decimals - precision beyond 8 decimals lost
    );

    lastCoreActionBlock[pocket] = block.number;
    emit LastCoreActionBlockUpdated(pocket, block.number);
    emit HypeTransferredToCore(pocket, amount);  // Emits full 18-decimal amount
}
```

[HyperLiquid 문서](https://hyperliquid.gitbook.io/hyperliquid-docs)에 따르면 HYPE는 HyperEVM에서는 18 decimals를 사용하지만 HyperCore에서는 8 decimals만 사용합니다. 브리지는 8 decimals를 초과하는 부분을 자동으로 잘라냅니다.

**영향:** 각 브리지 트랜잭션은 8 decimals를 넘는 소수 부분을 잃게 됩니다. 개별 손실은 매우 작을 수 있지만(최악의 경우 트랜잭션당 약 ~9e-10 HYPE), 시간이 지나며 누적되고 영구적 자금 손실이 됩니다.

**권장 완화 조치:** 금액이 `1e10`의 배수(8-decimal precision)에 맞도록 검증을 추가하십시오.

```solidity
function transferHypeToCore(address pocket, uint256 amount) external onlyOperator {
    require(pocketUser[pocket] != address(0), "Pocket does not exist");
    require(amount > 0, "Amount must be positive");
    require(amount % 1e10 == 0, "Amount must be multiple of 1e10 for 8-decimal precision");

    IPocket(pocket).transferNative(CoreWriterEncoder.HYPE_SYSTEM_ADDRESS, amount);

    lastCoreActionBlock[pocket] = block.number;
    emit LastCoreActionBlockUpdated(pocket, block.number);
    emit HypeTransferredToCore(pocket, amount);
}
```

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함. 이제 금액은 `1e10`의 배수인지 검증됩니다.


### Morpho Blue Market 보상은 Pocket 소유자가 청구할 수 없음

**설명:** Morpho Blue는 [Merkl](https://docs.merkl.xyz/)이라는 서드파티 보상 배포 서비스를 통해 외부 보상을 분배합니다. Merkl 문서에 따르면, 보상은 수동으로 청구해야 하며 사용자가 직접 claim 함수를 호출해야만 수령할 수 있습니다.

하지만 pocket 소유자는 자신의 pocket이 벌어들인 Morpho 보상을 청구할 메커니즘이 없습니다.

```solidity
// Pocket.sol:92-110
function exec(address target, bytes calldata data)
    external
    onlyOwner
    returns (bytes memory result)
{
// Restricted only to interaction with CoreWriter
}
```

MorphoBlueAdapter도 supply, withdraw, borrow, repay 같은 핵심 Morpho 동작만 지원하며, 보상 청구 기능은 포함하지 않습니다.

**영향:** Morpho Blue 시장 참여를 통해 벌어들인 보상은 Merkl 분배 컨트랙트에 누적되지만, pocket 사용자는 이 보상에 접근할 수 없습니다. [Merkl 문서](https://docs.merkl.xyz/earn-with-merkl/earning-with-merkl)에 따르면 일부 보상 캠페인은 청구 마감 기한이 있어, 기한 내에 청구하지 않으면 보상이 사라질 수 있습니다.

Merkl은 [address remapping](https://docs.merkl.xyz/earn-with-merkl/earning-with-merkl#address-remapping)을 우회책으로 제공하긴 합니다. 즉 claim이 불가능한 스마트 컨트랙트 주소에서 EOA로 보상을 전달하도록 재매핑하는 방식입니다. 하지만 이 방식은 수동 개입이 필요합니다. pocket 소유자가 Merkl 팀에 연락해 소유권 증명을 제출하고, pocket별로 remapping을 요청해야 합니다.

또한 프로토콜이 `BasisTradeTailor`를 업그레이드해 보상 청구 기능을 추가할 수도 있지만, 그 업그레이드가 배포되고 실행되기 전까지 기존 보상은 계속 청구되지 못한 채 남게 됩니다.

**권장 완화 조치:** 이를 해결하는 방법은 크게 두 가지입니다.

Approach 1: Operator-Controlled Reward Claiming 추가(컨트랙트 변경 필요)

OPERATOR 또는 AGENT가 pocket을 대신해 보상을 청구하고, 그 보상을 pocket owner에게 전달할 수 있는 제한된 함수를 구현합니다.

Approach 2: Merkl Address Remapping 모니터링(컨트랙트 변경 없음)

보다 단순한 대안으로, 프로토콜 팀은 다음을 수행할 수 있습니다.

1. 모든 Morpho 캠페인에 대한 Merkl 보상 분배를 모니터링합니다.
2. Merkl의 [address remapping feature](https://docs.merkl.xyz/earn-with-merkl/earning-with-merkl#address-remapping)를 선제적으로 사용해, 스마트 컨트랙트인 pocket 주소에 쌓인 보상을 해당 pocket owner의 EOA로 리디렉션합니다.
3. 이 모니터링 및 remapping 과정을 프로토콜 운영의 일부로 자동화합니다.

**Button:** 인지함. 현재 이 코드베이스는 외부 vault 컨트랙트가 tailor를 통해 내부 pocket을 소유하는 방향을 계획하고 있어, 당장 다수의 리테일 pocket을 지원할 가능성은 낮다고 봅니다. vault 관련 보상을 정리해야 한다면 보상 팀과 직접 조율할 수 있습니다.



### 이중 목적 매핑으로 인해 권한이 교집합으로 강제됨

**설명:** `BasisTradeTailor::addAsset` 함수는 `registeredAssets`(`executeAdapter()`에서 어댑터 승인용)와 `supportedAssets`(`transferAssetToCore/FromCore`에서 Core 전송용)를 동시에 설정합니다.

```solidity
// BasisTradeTailor.sol:652-672
function addAsset(address asset, uint64 tokenIndex) external onlyOperator {
    registeredAssets[asset] = true;  // Used for executeAdapter approvals (line 796)
    supportedAssets[tokenIndex] = AssetConfig({
        asset: asset,
        tokenIndex: tokenIndex
    });  // Used for Core transfers
}
```

MorphoBlueAdapter를 사용할 때는 loanToken과 collateralToken 모두 `executeAdapter()`에서 승인되어야 하므로 `registeredAssets`에 등록돼 있어야 합니다. 그런데 `addAsset()`가 두 매핑을 함께 설정하기 때문에, MorphoBlueAdapter를 활성화하면 자동으로 해당 토큰에 대한 `transferAssetToCore()`와 `transferAssetFromCore()`도 함께 가능해집니다.

**영향:** 프로토콜은 MorphoBlueAdapter를 사용하면서 동시에 해당 자산의 Core 전송 기능을 비활성화하는 구성을 만들 수 없습니다. 즉 OPERATOR는 Morpho 대출만 허용하고 직접 브리지 전송은 막는 식의 세밀한 설정을 할 수 없습니다.

예를 들어 프로토콜이 Morpho에서 USDC 대출은 지원하되, USDC를 Core로 직접 보내는 기능은 막고 싶어도 불가능합니다. `addAsset()`가 두 권한을 묶어버리기 때문입니다. 결과적으로 프로토콜은 자산별로 어떤 작업을 허용할지 세밀하게 통제할 수 없습니다.

**권장 완화 조치:** 브리지 기능과 어댑터 기능을 독립적으로 제어할 수 있도록 매핑을 분리하십시오.

**Button:** [`9a1ad0a`](https://github.com/buttonxyz/button-protocol/commit/9a1ad0a07694d1366cbf300db140e8d785fd2e7d)에서 수정됨

**Cyfrin:** 확인함. 이제 매핑이 분리되었습니다.

\clearpage
## 정보 (Informational)


### `Pocket::execWithValue`는 native transfer 이벤트를 발생시키지 않음

**설명:** `Pocket::execWithValue` 함수는 `value` 파라미터를 통해 native token을 보내지만, `NativeTransferred(to, amount)` 대신 `Executed(target, selector)`만 발생시킵니다.

```solidity
// Pocket.sol:107
result = target.functionCallWithValue(data, value);  // sends native tokens

emit Executed(target, selector);  // doesn't include value amount
```

이는 `transferNative()`가 `NativeTransferred(to, amount)`를 올바르게 발생시키는 것(line 130)과 다릅니다. 이벤트만 추적하는 오프체인 시스템은 `execWithValue()`로 이뤄진 native token 이동을 놓치게 됩니다. `Executed` 이벤트에 `value` 정보가 없기 때문입니다.

**권장 완화 조치:** 일관성을 위해 value가 전송될 때 `NativeTransferred`도 함께 발생시키십시오.

```diff
function execWithValue(...) external onlyOwner returns (bytes memory result) {
    require(target != address(0), "Invalid target");
    result = target.functionCallWithValue(data, value);

    bytes4 selector;
    if (data.length >= 4) {
        selector = bytes4(data[:4]);
    }

+   if (value > 0) {
+       emit NativeTransferred(target, value);
+   }
    emit Executed(target, selector);
}
```

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.


### `BasisTradeTailor::transferPerp` 주석 불일치

**설명:** `BasisTradeTailor::transferPerp`의 NatSpec 주석은 "(agent only)"라고 되어 있지만, 실제 함수는 `onlyEngine` 수정자를 사용합니다.

```solidity
// BasisTradeTailor.sol:373-378
/**
 * @notice Transfer funds between spot and perp accounts on HyperCore (agent only)
 */
function transferPerp(address pocket, uint64 amount, bool toPerp) external onlyEngine {
    // Comment says "agent only" but modifier is onlyEngine
```

구현 자체는 아마 올바르며, `ENGINE_ROLE`만 이 함수를 호출할 수 있으므로 주석이 오해를 부르는 상태입니다.

**권장 완화 조치:** 주석을 실제 구현과 맞게 수정하십시오.

```diff
/**
- * @notice Transfer funds between spot and perp accounts on HyperCore (agent only)
+ * @notice Transfer funds between spot and perp accounts on HyperCore (engine only)
 * @param pocket Address of the pocket
 * @param amount Amount to transfer
 * @param toPerp True to transfer to perp, false to transfer to spot
 */
function transferPerp(address pocket, uint64 amount, bool toPerp) external onlyEngine {
```

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.



### `BasisTradeTailor::coreDepositWallet`는 adapter 호출 대상에서 차단되지 않음

**설명:** `BasisTradeTailor::executeAdapter`는 `coreWriter`는 차단하지만 `coreDepositWallet`은 차단하지 않습니다.

```solidity
// BasisTradeTailor.sol:781
require(target != coreWriter, "Adapter cannot call coreWriter");
// No check for coreDepositWallet
```

`CoreDepositWallet.depositFor(address user, ...)`는 pocket의 USDC를 임의의 Core 주소로 보낼 수 있습니다. adapter 자체는 신뢰된다고 하더라도, defense-in-depth 차원에서 `coreWriter`를 막아 두었다면 Core와 상호작용하는 또 다른 핵심 계약인 `coreDepositWallet`도 막는 것이 일관적입니다.

**권장 완화 조치:**
```diff
-require(target != coreWriter, "Adapter cannot call coreWriter");
+require(target != coreWriter && target != coreDepositWallet, "Adapter cannot call Core contracts");
```

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.



### `MorphoBlueAdapter::validateCalls`는 `receiver == pocket`을 강제하지 않음

**설명**
`MorphoBlueAdapter`는 Morpho 호출을 만들 때 `receiver = pocket`으로 구성합니다(예: `borrow(..., receiver)`, `withdrawCollateral(..., receiver)`). 그러나 `validateCalls`는 `onBehalf == pocket`만 검증하고, 디코딩된 `receiver` 파라미터가 실제로도 `pocket`인지 확인하지 않습니다. 이로 인해 "호출이 Pocket 내부에서만 안전하게 닫혀 있다"는 adapter의 의도된 보장 강도가 약해집니다.

적용 가능한 호출에 대해서는 `onBehalf == pocket`뿐 아니라 `receiver == pocket`도 요구하는 것을 고려하십시오.

**Button:** [`877074a`](https://github.com/buttonxyz/button-protocol/commit/877074a243524a6856c39a1d1fda803cdf927f3d)에서 수정됨.

**Cyfrin:** 확인함. 추가적인 market params 검증도 함께 추가되었습니다.


### `PocketFactory::approveTailor`는 `ITailor` 인터페이스 구현 여부를 검증하지 않음

**설명:** `PocketFactory.approveTailor()` 함수는 tailor 주소가 0이 아니고 코드가 존재하는지만 검증할 뿐, pocket 관리 작업에 필요한 `ITailor` 인터페이스를 실제로 구현하는지는 확인하지 않습니다. `ApproveTailorInFactory.s.sol`에도 이런 검사가 없습니다.

```solidity
// PocketFactory.sol:56-62
function approveTailor(address tailor) external onlyRole(OPERATOR_ROLE) {
    require(tailor != address(0), "Invalid tailor address");
    require(tailor.code.length > 0, "Tailor must be a contract");  // Only checks has code

    approvedTailors[tailor] = true;
    emit TailorApproved(tailor);
}
```

**권장 완화 조치:** `approveTailor()`에 ERC-165 인터페이스 검사를 추가하십시오. 추가 안전장치로 `ApproveTailorInFactory.s.sol` 배포 스크립트에도 동일 검사를 넣는 것을 고려하십시오.

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.



### 업그레이드 스크립트는 구현체만 배포하고 실제 업그레이드는 수행하지 않음

**설명:** `UpgradeBasisTradeTailor.s.sol` 스크립트는 새 구현체를 배포하지만 실제 업그레이드 실행은 하지 않습니다. 44번째 줄이 주석 처리되어 있기 때문입니다.

```solidity
// UpgradeBasisTradeTailor.s.sol:38-48
// Deploy new implementation
BasisTradeTailor newImpl = new BasisTradeTailor(hypeTokenIndex, usdcAddress, coreDepositWallet);
console.log("New implementation deployed at:", address(newImpl));
console.log("HYPE Token Index:", hypeTokenIndex);
console.log("USDC Address:", usdcAddress);
console.log("CoreDepositWallet:", coreDepositWallet);

//tailor.upgradeToAndCall(address(newImpl), "");  // Commented out

vm.stopBroadcast();

console.log("\n=== Tailor Upgraded ===");  // Misleading - upgrade didn't happen
```

스크립트는 "Tailor Upgraded"를 출력하지만, 실제 프록시는 여전히 이전 구현체를 가리킵니다. `runSafe()` 함수(lines 60-83)를 보면 업그레이드는 Gnosis Safe를 통한 실행을 염두에 둔 것으로 보이지만, `run()` 함수의 현재 동작은 완전한 업그레이드를 기대하는 운영자를 혼란스럽게 만들 수 있습니다.

**권장 완화 조치:** 44번째 줄의 주석을 해제해 실제 업그레이드를 실행하거나, 문서와 콘솔 로그를 "deploy-only" 모드임을 명확히 설명하도록 수정하고, 업그레이드는 Safe 또는 수동 `upgradeToAndCall()`로 따로 실행해야 한다고 명시하십시오.

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.


### adapter 제거 스크립트에는 Safe-mode calldata 출력이 없음

**설명**
`RemoveAdapterFromBasisTradeTailor.s.sol`는 다른 운영 스크립트들에서 사용하는 `SAFE_MODE` 패턴, 즉 multisig 실행용 `to/value/data` calldata 출력 경로를 따르지 않고 직접 실행 흐름에 의존합니다.

`removeAdapter(adapter)` 호출에 대해 인코딩된 calldata(`to`, `value`, `data`)를 출력하는 `SAFE_MODE` 경로를 추가하는 것을 고려하십시오.

**Button:** 커밋 [`b74a07d`](https://github.com/buttonxyz/button-protocol/commit/b74a07db825a07098bf83e06f9467308d9f0a211)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `BasisTradeTailor::_transferUsdcToCore`에서 pocket 호출을 배치 처리

**설명**
현재 `BasisTradeTailor::_transferUsdcToCore`는 `IPocket.exec(...)`를 세 번 따로 호출합니다. USDC 승인, `depositFor` 호출, 그리고 approval을 0으로 되돌리는 순서입니다. 각 `exec` 호출은 외부 호출 오버헤드를 반복적으로 발생시킵니다.

`Pocket`에 배치 호출 함수(예: `call(Call[] calldata calls)`)를 추가하고, `_transferUsdcToCore`가 approve, depositFor, reset sequence를 한 번의 Pocket 호출로 수행하도록 바꾸는 것을 고려하십시오.

* `IPocket.sol`:
  ```solidity
  struct Call {
      address target;
      bytes data;
      uint256 value;
  }

  function exec(Call[] calldata calls) external returns (bytes[] memory results);
  ```

* `Pocket.sol`:

  ```solidity
  function exec(Call[] calldata calls) external onlyOwner returns (bytes[] memory results) {
      uint256 len = calls.length;
      results = new bytes[](len);

      for (uint256 i = 0; i < len; ++i) {
          Call memory call = calls[i];
          if (call.value == 0) {
              results[i] = exec(call.target, call.data);
          } else {
              results[i] = execWithValue(call.target, call.data, call.value);
          }
      }
  }
  ```
* `BasisTradeTailor._transferUsdcToCore`:
  ```solidity
  IPocket.Call calls = new IPocket.Call[3];
  calls[0] = IPocket.Call({ target: usdcAddress, data: approveData, value: 0 });
  calls[1] = IPocket.Call({ target: coreDepositWallet, data: depositData, value: 0 });
  calls[2] = IPocket.Call({ target: usdcAddress, data: resetApproveData, value: 0 });

  IPocket(pocket).call(calls);
  ```

**Button:** [`4122fe7`](https://github.com/buttonxyz/button-protocol/commit/4122fe757d1db5cf0575379157357d65e23d1eab)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
