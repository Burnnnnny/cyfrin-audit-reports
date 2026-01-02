**수석 감사 (Lead Auditors)**

[Dacian](https://twitter.com/DevDacian)


**보조 감사 (Assisting Auditors)**

  


---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### `L1MessageServiceV1::__messageSender`의 사용되지 않는 변수가 여전히 사용되어 이전 메시지에 대해 `TokenBridge::completeBridging`이 되돌아가(revert) 사용자가 브리지된 토큰을 받지 못하게 함

**설명:** `L1MessageServiceV1`에서 이 [주석](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/l1/v1/L1MessageServiceV1.sol#L27)은 `_messageSender` 저장 변수가 임시 저장소(transient storage)를 위해 더 이상 사용되지 않음을 나타냅니다.

그러나 사용되지 않는 `_messageSender`는 `L1MessageServiceV1::claimMessage`에서 [업데이트](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/l1/v1/L1MessageServiceV1.sol#L122)되고 `L1MessageService::__MessageService_init`에서도 [설정](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/l1/L1MessageService.sol#L68)됩니다. 하지만 `L1MessageService::claimMessageWithProof`는 임시 저장소 방식을 [사용](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/l1/L1MessageService.sol#L142)합니다.

`MessageServiceBase::onlyAuthorizedRemoteSender`는 메시지 발신자를 검색하기 위해 `L1MessagingService::sender`를 [호출](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/MessageServiceBase.sol#L52-L57)하며, 이 함수는 항상 사용되지 않는 `_messageSender`가 아닌 임시 저장소에서 [가져옵니다](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/l1/L1MessageService.sol#L166-L168).

`MessageServiceBase::onlyAuthorizedRemoteSender`는 차례로 `TokenBridge`::[setDeployed](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/tokenBridge/TokenBridge.sol#L302) 및 [`completeBridging`](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/tokenBridge/TokenBridge.sol#L241)에서 사용됩니다.

**영향:** `claimMessage`를 사용하여 `TokenBridge::setDeployed` 및 `completeBridging`을 호출하는 오래된 L2->L1 메시지는 다음과 같은 이유로 항상 되돌려집니다(revert).
* 메시지 발신자가 임시 저장소에서 가져와집니다.
* `claimMessage`는 여전히 사용되지 않는 `_messageSender` 저장 변수를 업데이트하므로 임시 저장소에 메시지 발신자를 설정하지 않습니다.

`completeBridging`이 이러한 트랜잭션에 대해 항상 되돌려지므로 사용자는 브리지된 토큰을 받을 수 없습니다.

**권장 완화 방법:** `L1MessageServiceV1::claimMessage`를 변경하여 메시지 발신자를 임시 저장소에 저장하고 사용되지 않는 `_messageSender`의 모든 사용을 제거하십시오.

**Linea:**
[PR3191](https://github.com/Consensys/zkevm-monorepo/pull/3191/files#diff-e9f6a0c3577321e5aa88a9d7e12499c2c4819146062e33a99576971682396bb5R119-R141)에 병합된 커밋 [55d936a](https://github.com/Consensys/zkevm-monorepo/pull/3191/commits/55d936a47ca31952cc65cc9ee1c7629b919c6aa2#diff-e9f6a0c3577321e5aa88a9d7e12499c2c4819146062e33a99576971682396bb5R122-R144)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TokenBridge::bridgeToken`은 단방향 `ERC721` 브리징을 허용하여 사용자가 영구적으로 NFT를 잃게 함

**설명:** `TokenBridge::bridgeToken`은 `ERC20` 토큰만 지원하도록 의도되었지만 `ERC721` 토큰을 기꺼이 수락하고 L2로 성공적으로 브리지할 수 있습니다. 그러나 사용자가 L1으로 다시 브리지하려고 시도하면 항상 되돌려져(revert) 사용자 NFT가 `TokenBridge` 계약 내에 영구적으로 갇히게 됩니다.

이는 최신 변경 사항에서 도입된 것은 아니지만 현재 메인넷 `TokenBridge` [코드](https://github.com/Consensys/linea-contracts/blob/main/contracts/tokenBridge/TokenBridge.sol)에 존재합니다.

**개념 증명 (Proof of Concept):** `contracts/tokenBridge/mocks/MockERC721.sol`에 새 파일 추가:
```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.0;

import { ERC721 } from "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract MockERC721 is ERC721 {
  constructor(string memory _tokenName, string memory _tokenSymbol) ERC721(_tokenName, _tokenSymbol) {}

  function mint(address _account, uint256 _tokenId) public returns (bool) {
    _mint(_account, _tokenId);
    return true;
  }
}
```

`Bridging` 섹션 아래 `test/tokenBridge/E2E.ts`에 이 새 테스트를 추가하십시오:
```typescript
it("Can erroneously bridge ERC721 and can't bridge it back", async function () {
  const {
    user,
    l1TokenBridge,
    l2TokenBridge,
    tokens: { L1DAI },
    chainIds,
  } = await loadFixture(deployContractsFixture);

  // deploy ERC721 contract
  const ERC721 = await ethers.getContractFactory("MockERC721");
  let mockERC721 = await ERC721.deploy("MOCKERC721", "M721");
  await mockERC721.waitForDeployment();

  // mint user some NFTs
  const bridgedNft = 5;
  await mockERC721.mint(user.address, 1);
  await mockERC721.mint(user.address, 2);
  await mockERC721.mint(user.address, 3);
  await mockERC721.mint(user.address, 4);
  await mockERC721.mint(user.address, bridgedNft);

  const mockERC721Address = await mockERC721.getAddress();
  const l1TokenBridgeAddress = await l1TokenBridge.getAddress();
  const l2TokenBridgeAddress = await l2TokenBridge.getAddress();

  // user approves L1 token bridge to spend nft to be bridged
  await mockERC721.connect(user).approve(l1TokenBridgeAddress, bridgedNft);

  // user bridges their nft
  await l1TokenBridge.connect(user).bridgeToken(mockERC721Address, bridgedNft, user.address);

  // verify user has lost their nft which is now owned by L1 token bridge
  expect((await mockERC721.ownerOf(bridgedNft))).to.be.equal(l1TokenBridgeAddress);

  // verify user has received 1 token on the bridged erc20 version which
  // the token bridge created
  const l2TokenAddress = await l2TokenBridge.nativeToBridgedToken(chainIds[0], mockERC721Address);
  const bridgedToken = await ethers.getContractFactory("BridgedToken");
  const l2Token = bridgedToken.attach(l2TokenAddress) as BridgedToken;

  const l2ReceivedTokenAmount = 1;

  expect(await l2Token.balanceOf(user.address)).to.be.equal(l2ReceivedTokenAmount);
  expect(await l2Token.totalSupply()).to.be.equal(l2ReceivedTokenAmount);

  // now user attempts to bridge back which fails
  await l2Token.connect(user).approve(l2TokenBridgeAddress, l2ReceivedTokenAmount);
  await expectRevertWithReason(
    l2TokenBridge.connect(user).bridgeToken(l2TokenAddress, l2ReceivedTokenAmount, user.address),
    "SafeERC20: low-level call failed",
  );
});
```

다음으로 테스트 실행: `npx hardhat test --grep "Can erroneously bridge ERC721"`

**권장 완화 방법:** `TokenBridge`는 사용자가 `ERC721` 토큰을 브리지하도록 허용해서는 안 됩니다. 현재 허용하는 이유는 `ERC721` 인터페이스가 `ERC20` 인터페이스와 매우 유사하기 때문입니다. 이를 방지하는 한 가지 옵션은 다음과 같은 경우 `TokenBridge::_safeDecimals`가 되돌려지도록(revert) 변경하는 것입니다.
* `ERC20` 토큰은 항상 소수점 자릿수(decimals)를 제공해야 합니다.
* `ERC721` 토큰에는 소수점 자릿수가 없습니다.

또 다른 옵션은 `ERC165` 인터페이스 감지를 사용하는 것이지만 모든 NFT가 이를 지원하는 것은 아닙니다.

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L435-R449)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L435-R446)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 중간 위험 (Medium Risk)


### Linea는 중앙 집중식 시퀀서를 통해 사용자 트랜잭션을 검열하여 L2에서 사용자 ETH를 영구적으로 잠글 수 있음

**설명:** Linea는 중앙 집중식 시퀀서를 통해 사용자 트랜잭션을 검열하여 L2에서 사용자 ETH를 영구적으로 잠글 수 있습니다. Linea 팀이 시퀀서를 제어하므로 L2 블록을 생성할 때 L2 트랜잭션을 포함하지 않음으로써 사용자(또는 정부 기관에 의해 강제됨)를 검열하도록 선택할 수 있습니다.

또한 사용자가 [L2 검열을 피할 수 있는 L1 트랜잭션을 시작하여](https://solodit.xyz/issues/m-09-lack-of-access-to-eth-on-l2-through-l1-l2-transactions-code4rena-zksync-zksync-git) L2 자산을 인출할 수 있는 방법이 없습니다.

이러한 요소의 조합은 자산을 Linea로 브리지하는 사용자가 계속해서 자유롭게 거래하기 위해 전적으로 Linea 팀의 자비에 의존하고 있음을 의미합니다.

이는 지난 30일 동안 OFAC 검열 체제조차 [38%의 집행률](https://www.mevwatch.info)만 보인 이더리움의 검열 저항성보다 상당히 후퇴한 것입니다.

그러나 이것은 1세대 zkEVM의 일반적인 결함[[1](https://youtu.be/Y4n38NPQhrY?si=88NeDOAsyA_kgVpo&t=167), [2](https://youtu.be/e8pwv9DIojI?si=Ru4oTdy0VM7S7Hmj&t=183)]이며 분산형 zk L2 시퀀서는 활발한 연구 분야[[1](https://www.youtube.com/watch?v=nF4jTEcQbkA), [2](https://www.youtube.com/watch?v=DP84VWng2i8)]이므로 이와 관련하여 Linea가 예외적인 경우는 아님을 주목해야 합니다.

**권장 완화 방법:** 분산형 시퀀서로 마이그레이션하거나 사용자가 L2 검열을 완전히 우회하여 L1 트랜잭션을 시작하는 것만으로 ETH(및 이상적으로는 브리지된 토큰)를 검색할 수 있는 메커니즘을 구현하십시오.

**Linea:**
인지함; 이것은 우리가 분산화할 때 해결될 것입니다.


### 대상 체인 트랜잭션이 지속적으로 실패하면 사용자가 소스 체인 트랜잭션을 취소하거나 환불받을 수 없어 브리지된 자금이 손실됨

**설명:** Linea의 메시징 서비스를 통해 사용자는 이더리움 L1 메인넷과 Linea L2 간에 ETH를 브리지할 수 있습니다.

브리징 프로세스는 첫 번째가 성공하고 두 번째가 서로 완전히 독립적으로 실패할 수 있는 2개의 별도 트랜잭션으로 작동합니다.

소스 체인의 브리징 프로세스 첫 번째 단계에서 사용자는 일부 매개변수와 함께 해당 체인의 메시징 서비스 계약에 ETH를 제공합니다.

대상 체인의 브리징 프로세스 마지막 단계에서 배달 서비스("우편 배달부") 또는 사용자 자신이 `claimMessage` 또는 `claimMessageWithProof`를 호출하여 브리징 프로세스를 완료합니다.

메시지 청구의 마지막 단계는 사용자가 선택한 주소로 임의의 calldata를 사용하여 임의의 [호출](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/messageService/l1/L1MessageService.sol#L144-L154)을 수행하는 것입니다. 어떤 이유로든 이 호출이 실패하면 사용자는 메시지 청구를 무한히 다시 시도할 수 있습니다. 그러나 이 호출이 계속 되돌려지거나(revert) `claim` 트랜잭션이 다른 이유로 계속 되돌려지는 경우 사용자는:
* 대상 체인에서 브리지된 ETH를 절대 받지 못합니다.
* 소스 체인에서 취소 또는 환불 트랜잭션을 시작할 방법이 없습니다.

**영향:** 이 시나리오는 사용자에게 브리지된 자금의 손실을 초래합니다.

**권장 완화 방법:** 청구 트랜잭션이 대상 체인에서 계속 되돌려지는 경우 사용자가 자금을 회수하기 위해 소스 체인에서 시작할 수 있는 취소 또는 환불 메커니즘을 구현하십시오.

**Linea:**
인지함. 현재 취소 또는 환불 기능은 없지만 사용자는 원하는 만큼 대상 체인 배달을 다시 시도할 수 있습니다. 향후 구현을 고려할 수 있습니다.

\clearpage
## 낮은 위험 (Low Risk)


### `TokenBridge::setCustomContract`를 사용하여 여러 L2 `BridgeToken`을 동일한 L1 `NativeToken`과 연결할 수 있음

**설명:** `TokenBridge::setCustomContract`를 사용하여 여러 L2 `BridgeToken`을 동일한 L1 `NativeToken`과 연결할 수 있습니다.

이는 최신 변경 사항에서 도입된 것은 아니지만 현재 메인넷 `TokenBridge` [코드](https://github.com/Consensys/linea-contracts/blob/main/contracts/tokenBridge/TokenBridge.sol)에 존재합니다.

**영향:** 이는 하나의 L1 `NativeToken`이 하나의 L2 `BridgeToken`과만 연결되어야 한다는 `TokenBridge` 불변성을 깨뜨립니다.

**개념 증명 (Proof of Concept):** `setCustomsContract` 섹션 아래 `test/tokenBridge/E2E.ts`에 이 PoC를 추가하십시오:
```typescript
    it("Multiple L2 BridgeTokens can be associated with a single L1 NativeToken by using setCustomContract", async function () {
      const {
        user,
        l1TokenBridge,
        l2TokenBridge,
        tokens: { L1USDT },
      } = await loadFixture(deployContractsFixture);

      // @audit the setup of this PoC has been copied from the test
      // "Should mint and burn correctly from a target contract"
      const amountBridged = 100;

      // Deploy custom contract
      const CustomContract = await ethers.getContractFactory("MockERC20MintBurn");
      const customContract = await CustomContract.deploy("CustomContract", "CC");

      // Set custom contract
      await l2TokenBridge.setCustomContract(await L1USDT.getAddress(), await customContract.getAddress());

      // @audit save user balance before both bridging attempts
      const USDTBalanceBefore = await L1USDT.balanceOf(user.address);

      // Bridge token (allowance has been set in the fixture)
      await l1TokenBridge.connect(user).bridgeToken(await L1USDT.getAddress(), amountBridged, user.address);

      expect(await L1USDT.balanceOf(await l1TokenBridge.getAddress())).to.be.equal(amountBridged);
      expect(await customContract.balanceOf(user.address)).to.be.equal(amountBridged);
      expect(await customContract.totalSupply()).to.be.equal(amountBridged);

      // @audit deploy a new second custom contract
      const customContract2 = await CustomContract.deploy("CustomContract2", "CC2");

      // @audit using setCustomContract, this second L2 BridgeToken can be associated
      // with L1USDT Token even though it should be impossible to do this as L1USDT has
      // already been bridged and already has a paired L2 BridgeToken
      //
      // @audit Once the suggested fix is applied, this should revert
      await l2TokenBridge.setCustomContract(await L1USDT.getAddress(), await customContract2.getAddress());

      // @audit same user now bridges L1->L2 again
      await l1TokenBridge.connect(user).bridgeToken(await L1USDT.getAddress(), amountBridged, user.address);

      // @audit L1 TokenBridge has twice the balance
      expect(await L1USDT.balanceOf(await l1TokenBridge.getAddress())).to.be.equal(amountBridged*2);

      // @audit first L2 BridgeToken still has the same balance and supply
      expect(await customContract.balanceOf(user.address)).to.be.equal(amountBridged);
      expect(await customContract.totalSupply()).to.be.equal(amountBridged);

      // @audit second L2 BridgeToken received new balance and supply from second bridging
      expect(await customContract2.balanceOf(user.address)).to.be.equal(amountBridged);
      expect(await customContract2.totalSupply()).to.be.equal(amountBridged);

      // @audit user L1 USDT balance deducted due to both bridging attempts
      const USDTBalanceAfterTwoL1ToL2Bridges = await L1USDT.balanceOf(user.address);
      expect(USDTBalanceBefore - USDTBalanceAfterTwoL1ToL2Bridges).to.be.equal(amountBridged*2);

      // @audit user now bridges back from first L2 BridgeToken
      await l2TokenBridge.connect(user).bridgeToken(await customContract.getAddress(), amountBridged, user.address);

      // @audit first L2 BridgeToken has 0 balance and supply
      expect(await customContract.balanceOf(user.address)).to.be.equal(0);
      expect(await customContract.totalSupply()).to.be.equal(0);

      // @audit user then bridges back from second L2 BridgeToken
      await l2TokenBridge.connect(user).bridgeToken(await customContract2.getAddress(), amountBridged, user.address);

      // @audit second L2 BridgeToken has 0 balance and supply
      expect(await customContract2.balanceOf(user.address)).to.be.equal(0);
      expect(await customContract2.totalSupply()).to.be.equal(0);

      // @audit user should have received back both bridged amounts
      const USDTBalanceAfter = await L1USDT.balanceOf(user.address);
      expect(USDTBalanceAfter - USDTBalanceAfterTwoL1ToL2Bridges).to.be.equal(amountBridged*2);

      // @audit user ends up with initial token amount
      expect(USDTBalanceAfter).to.be.equal(USDTBalanceBefore);

      // @audit user was able to bridge L1->L2 to two different L2 BridgeTokens,
      // and was able to bridge back L2->L1 from both different L2 BridgeTokens back
      // into the same L1 NativeToken. This breaks the invariant that one L1 NativeToken
      // should only be associated with one L2 BridgeToken
    });
```

다음으로 실행: `npx hardhat test --grep "Multiple L2 BridgeTokens can be associated with"`

**권장 완화 방법:** `TokenBridge::setCustomContract`는 여러 L2 `BridgeToken`이 동일한 L1 `NativeToken`과 연결되는 것을 방지해야 합니다. 가능한 구현 중 하나는 다음 확인을 추가하는 것입니다.
```solidity
  function setCustomContract(
    address _nativeToken,
    address _targetContract
  ) external nonZeroAddress(_nativeToken) nonZeroAddress(_targetContract) onlyOwner isNewToken(_nativeToken) {
    if (bridgedToNativeToken[_targetContract] != EMPTY) {
      revert AlreadyBrigedToNativeTokenSet(_targetContract);
    }
    if (_targetContract == NATIVE_STATUS || _targetContract == DEPLOYED_STATUS || _targetContract == RESERVED_STATUS) {
      revert StatusAddressNotAllowed(_targetContract);
    }

    // @audit proposed fix
    uint256 targetChainIdCache = targetChainId;

    if (nativeToBridgedToken[targetChainIdCache][_nativeToken] != EMPTY) {
      revert("Already set a custom contract for this native token");
    }

    nativeToBridgedToken[targetChainIdCache][_nativeToken] = _targetContract;
    bridgedToNativeToken[_targetContract] = _nativeToken;
    emit CustomContractSet(_nativeToken, _targetContract, msg.sender);
  }
```

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L388-R397)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L388-R394)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### Linea는 L2->L1 블록 확정을 위해 유효성 증명 확인을 우회할 수 있음

**설명:** Optimistic 롤업보다 zk 롤업을 사용할 때의 주요 이점 중 하나는 zk 롤업이 수학 및 암호화를 사용하여 제출하고 확인해야 하는 유효성 증명을 통해 L2->L1 블록 확정을 확인한다는 것입니다.

현재 코드베이스에서 Linea 팀은 블록 확정 중 유효성 증명 확인을 우회할 수 있는 두 가지 옵션이 있습니다.

1) `LineaRollup::finalizeBlocksWithoutProof`를 명시적으로 호출하여, 이는 증명 매개변수를 입력으로 받지 않으며 `_verifyProof` 함수를 호출하지 않습니다.

2) 더 미묘하게 `LineaRollup::setVerifierAddress`를 호출하여 `IPlonkVerifier::Verify` 인터페이스를 구현하지만 단순히 `true`를 반환하고 실제 확인을 수행하지 않는 `_newVerifierAddress`와 `_proofType`을 연결합니다. 그런 다음 항상 true를 반환하는 `_newVerifierAddress`와 관련된 `_proofType`을 전달하여 `LineaRollup::finalizeBlocksWithProof`를 호출합니다.

**영향:** 두 옵션 중 하나를 사용하면 Linea는 감시자(Watchers)와 이의 제기 기능이 없는 Optimistic 롤업과 거의 동일해집니다. 그렇긴 하지만 두 옵션 중 하나가 사용되면 `LineaRollup::_finalizeBlocks`가 항상 실행되어 이러한 옵션이 악용될 수 있는 방법을 상당히 제한하는 많은 "건전성 검사(sanity checks)"를 시행합니다.

"증명 없는" 기능은 거짓 L2 머클 루트를 추가하는 데 사용될 수도 있으며, 이는 L1에서 ETH를 빼내기 위해 `L1MessageService::claimMessageWithProof`를 호출하는 데 사용될 수 있습니다.

**권장 완화 방법:** Linea는 아직 알파 단계에 있으므로 이러한 기능은 최후의 수단으로 필요할 수 있습니다. 이상적으로 Linea가 더 성숙해지면 이러한 기능은 제거될 것입니다.

**Linea:**
인지함; 증명 없이 블록을 확정하는 기능은 주로 보안 위원회를 통해 통제되고 보안 파트너가 분석하며, 상태 관리자를 다른 해싱 알고리즘으로 업그레이드했을 때(마지막으로 사용된 경우)와 같은 특정 사례를 위해 예약된 "보조 바퀴(training wheels)"의 일부입니다.


### `LineaRollup::submitDataAsCalldata`를 사용하여 데이터를 제출할 때 기존 shnarf에 대한 연결된 `finalBlockNumber`를 덮어쓸 수 있음

**설명:** `LineaRollup::submitDataAsCalldata`를 사용하여 데이터를 제출할 때 현재 제출에 대한 `computedShnarf`가 이전에 이미 제출된 shnarf로 판명되면 이 shnarf에 연결된 `finalBlockNumber`가 새 `submissionData.finalBlockInData`에 의해 덮어쓰여집니다.

문제는 기존 확인이 shnarf가 이미 다른 `finalBlockNumber`와 연결되어 있는지 확인하는 대신 shnarf와 연결된 `finalBlockInNumber`가 새 `submissionData.finalBlockInData`와 동일한지 비교하기 때문에 발생합니다.

**영향:** 기존 shnarf와 연결된 기록 데이터는 아직 확정되지 않은 블록에 대한 최신 shnarf의 제출 데이터로 덮어쓸 수 있습니다.

**개념 증명 (Proof of Concept):** `describe("Data submission tests", () => {` 섹션 내부의 `test/LineaRollup.ts`에 PoC를 추가하십시오:
```typescript
    it.only("Submitting data on a range of blocks where data had already been submitted", async () => {
      let [firstSubmissionData, secondSubmissionData] = generateCallDataSubmission(0, 2);
      const firstParentShnarfData = generateParentShnarfData(0);

      // @audit Assume this is the lastFinalizedBlock, in the sense that any of the
      //        next submissions can't submit data to a block prior to this one
      // @audit The `firstBlockInData` of the next submissions must start at block 550
      firstSubmissionData.finalBlockInData = 549n;
      await expect(
        lineaRollup
          .connect(operator)
          .submitDataAsCalldata(firstParentShnarfData, firstSubmissionData, { gasLimit: 30_000_000 }),
      ).to.not.be.reverted;

      parentSubmissionData = generateParentSubmissionDataForIndex(1);
      const secondParentShnarfData = generateParentShnarfData(1);

      // @audit Submission for block range 550 - 599
      secondSubmissionData.firstBlockInData = 550n;
      secondSubmissionData.finalBlockInData = 599n;
      await expect(
        lineaRollup.connect(operator).submitDataAsCalldata(secondParentShnarfData, secondSubmissionData, {
          gasLimit: 30_000_000,
        }),
      ).to.not.be.reverted;

      let finalBlockNumber = await lineaRollup.shnarfFinalBlockNumbers(secondExpectedShnarf);
      expect(finalBlockNumber == 599n).to.be.true;
      console.log("finalBlockNumber: ", finalBlockNumber);

      // @audit Submission for block range 550 - 649
      //        Instead of continuing from the "lastSubmitedBlock (599)"
      let secondSubmissionOnSameInitialBlockData = Object.assign({}, secondSubmissionData);
      secondSubmissionOnSameInitialBlockData.firstBlockInData = 550n;
      secondSubmissionOnSameInitialBlockData.finalBlockInData = 649n;
      await expect(
        lineaRollup.connect(operator).submitDataAsCalldata(secondParentShnarfData, secondSubmissionOnSameInitialBlockData, {
          gasLimit: 30_000_000,
        }),
      ).to.not.be.reverted;

      finalBlockNumber = await lineaRollup.shnarfFinalBlockNumbers(secondExpectedShnarf);
      expect(finalBlockNumber == 649n).to.be.true;
      console.log("finalBlockNumber: ", finalBlockNumber);

      // @audit Submission for block range 550 - 575
      let thirdSubmissionOnSameInitialBlockData = Object.assign({}, secondSubmissionData);;
      thirdSubmissionOnSameInitialBlockData.firstBlockInData = 550n;
      thirdSubmissionOnSameInitialBlockData.finalBlockInData = 575n;
      await expect(
        lineaRollup.connect(operator).submitDataAsCalldata(secondParentShnarfData, thirdSubmissionOnSameInitialBlockData, {
          gasLimit: 30_000_000,
        }),
      ).to.not.be.reverted;

      finalBlockNumber = await lineaRollup.shnarfFinalBlockNumbers(secondExpectedShnarf);
      expect(finalBlockNumber == 575n).to.be.true;
      console.log("finalBlockNumber: ", finalBlockNumber);

      // @audit Each time a duplicated shnarf was computed the associated
      // `finalBlockNumber` was updated for the new `finalBlockNumber` of the newest submission.
    });
```

다음으로 실행: `npx hardhat test --grep "Submitting data on a range of blocks where data had"`

테스트가 실행될 때 동일한 shnarf와 연결된 `finalBlockNumber`가 이전에 이미 제출된 기존 shnarf를 생성하는 각 제출에서 어떻게 업데이트되는지 콘솔에서 관찰하십시오.

**권장 완화 방법:** 현재 제출에 대한 `computedShnarf`가 이전 `finalBlockInNumber`와 이미 연결되어 있는지 확인하십시오:

```solidity
function submitDataAsCalldata(
  ...
) external whenTypeAndGeneralNotPaused(PROVING_SYSTEM_PAUSE_TYPE) onlyRole(OPERATOR_ROLE) {
  ...

- if (shnarfFinalBlockNumbers[shnarf] == _submissionData.finalBlockInData) {
-   revert DataAlreadySubmitted(shnarf);
- }

+ if (shnarfFinalBlockNumbers[shnarf] != 0) {
+   revert DataAlreadySubmitted(shnarf);
+ }

  ...
}

```

**Linea:**
[PR3191](https://github.com/Consensys/zkevm-monorepo/pull/3191/files#diff-408c766d202f05efdde69ebcd815ba16ad10f68249dd353f6359121ab312a9fbL311)에 병합된 커밋 [1816c80](https://github.com/Consensys/zkevm-monorepo/pull/3191/commits/1816c80f8e11528aea48f265bf557b1934023721#diff-408c766d202f05efdde69ebcd815ba16ad10f68249dd353f6359121ab312a9fbR311)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 잠재적인 L1->L2 `ERC20` 소수점 불일치로 인해 사용자 브리지 토큰이 손실될 수 있음

**설명:** 새 ERC20 토큰이 처음 브리지될 때 `TokenBridge::bridgeToken`은 `IERC20MetadataUpgradeable` 인터페이스를 사용하여 토큰의 이름, 심볼 및 소수점을 가져와 브리지된 버전의 토큰을 생성합니다.

`IERC20MetadataUpgradeable::decimals`에 대한 `staticcall`이 유효한 값을 반환하지 못하면 `TokenBridge::_safeDecimals`는 기본값인 18 소수점으로 설정됩니다.

**영향:** 이는 다음과 같은 이유로 잠재적으로 위험합니다.
* 비표준 ERC20 토큰은 내부적으로 18이 아닌 소수점 값을 사용하지만 `decimals` 함수가 없을 수 있습니다.
* 새로 생성된 브리지된 버전은 기본 토큰과 다른 18 소수점을 갖게 됩니다.

사용자가 다시 브리지하려고 시도하면 사용자의 브리지된 토큰이 소각되고 기본 체인에서 다음과 같이 됩니다.
* 기본 토큰의 소수점이 18 미만인 경우 전송 호출이 되돌려져(revert) 사용자가 원래 브리지한 토큰이 토큰 브리지에 갇히게 됩니다.
* 기본 토큰의 소수점이 18보다 큰 경우 전송 호출이 성공하여 사용자가 처음에 브리지한 것보다 적은 토큰을 받게 됩니다.

**권장 완화 방법:** `TokenBridge::_safeDecimals`는 브리지되는 토큰이 `IERC20MetadataUpgradeable::decimals` 호출에서 소수점을 반환하지 않으면 되돌려져야(revert) 합니다. 흥미롭게도 Consensys 자체는 [2021년 11월 Arbitrum 감사](https://github.com/OffchainLabs/nitro/blob/master/audits/ConsenSys_Diligence_Arbitrum_Contracts_11_2021.pdf)에서 문제 5.4로 동일한 발견 사항을 보고했습니다.

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L435-R449)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L435-R446)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### `L2MessageManager::anchorL1L2MessageHashes`에서 `messageHashesLength == 0`인 경우 되돌리기(revert) 위한 건전성 검사 추가

**설명:** `L2MessageManager::anchorL1L2MessageHashes`에서 `messageHashesLength == 0`인 경우 되돌리기 위한 건전성 검사를 추가합니다.

**Linea:**
인지함; 향후 릴리스에 포함될 예정입니다.


### `L1MessageManager::_addL2MerkleRoots`에서 `_treeDepth == 0`인 경우 되돌리기(revert) 위한 건전성 검사 추가

**설명:** `treeDepth == 0`인 L2 머클 루트가 추가되어 `L1MessagingService::claimMessageWithProof`에 대한 후속 호출이 `L2MerkleRootDoesNotExist` 오류와 함께 되돌려지는 상태를 생성하는 것을 방지하기 위해 `L1MessageManager::_addL2MerkleRoots`에서 `_treeDepth == 0`인 경우 되돌리기 위한 건전성 검사를 추가합니다.

**Linea:**
깊이(depth)는 항상 회로(circuit)에서 시행되며 공개 입력의 일부를 형성하므로 0으로 설정될 수 없습니다. 회로에는 고정된 크기가 있습니다. [PR3214](https://github.com/Consensys/zkevm-monorepo/pull/3214)에 병합된 커밋 [54c4670](https://github.com/Consensys/zkevm-monorepo/commit/54c467056b0a95d411f1060f77a9940652b1554c)에 이를 설명하는 주석을 추가했습니다.


### `SparseMerkleTreeVerifier`는 동일한 함수를 복제하는 대신 `Utils::_efficientKeccak`을 사용해야 함

**설명:** `SparseMerkleTreeVerifier`는 동일한 함수를 복제하는 대신 `Utils::_efficientKeccak`을 사용해야 합니다.

**Linea:**
인지함; 향후 릴리스에 포함될 예정입니다.


### 저장소 업데이트 시 `TokenBridge::removeReserved`에서 이벤트 방출

**설명:** `TokenBridge::setReserved`는 이벤트를 방출하므로 반대 함수인 `removeReserved`도 저장소를 업데이트할 때 이벤트를 방출해야 합니다.

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4R370)에 병합된 커밋 [796f2d9](https://github.com/Consensys/zkevm-monorepo/commit/796f2d9bbacf8c2403b8a4473f9c711172067940#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4R366)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `remoteSender` 저장소 업데이트 시 `MessageServiceBase::_setRemoteSender`에서 이벤트 방출

**설명:** `remoteSender` 저장소를 업데이트할 때 `MessageServiceBase::_setRemoteSender`에서 이벤트를 방출합니다.

**Linea:**
인지함; 향후 릴리스에 포함될 수 있습니다. `MessageServiceBase::_setRemoteSender`는 새 테스트넷 체인에서만 다시 사용되며 기존 Sepolia 및 메인넷 배포에는 사용할 수 없습니다.


### 모든 blob이 검증되었는지 확인하기 위해 `LineaRollup::submitBlobs`에 건전성 검사 추가

**설명:** `LineaRollup::submitBlobs`에서 `_blobSubmissionData.length`가 실제 첨부된 blob 수보다 작으면 `for` 루프가 모든 blob을 반복하지 않았기 때문에 코드가 성공적으로 실행되고 중간 값을 `shnarfFinalBlockNumbers[computedShnarf]`에 저장합니다.

이 엣지 케이스가 발생하는 것을 방지하려면 `for` 루프 앞에 조기 실패 건전성 검사를 추가하십시오. 예:
```solidity
if(blobhash(blobSubmissionLength) != EMPTY_HASH) {
      revert MoreBlobsThanData();
}
```

**Linea:**
인지함; 분산화에 중점을 둔 향후 릴리스에서 해결될 것입니다. 현재 Operator Service는 이러한 일이 발생할 수 없도록 작성되었으며 항상 데이터:blob의 1:1 매핑이 있습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### `TokenBridge::deployBridgedToken`은 명명된 반환 변수를 사용하여 추가 로컬 변수를 피할 수 있음

**설명:** `TokenBridge::deployBridgedToken`은 명명된 반환 변수를 사용하여 추가 로컬 변수를 피할 수 있습니다. 예:

```solidity
  function deployBridgedToken(
    address _nativeToken,
    bytes calldata _tokenMetadata,
    uint256 _chainId
  ) internal returns (address bridgedTokenAddress) { // @audit named return
    bytes32 _salt = keccak256(abi.encode(_chainId, _nativeToken));
    BeaconProxy bridgedToken = new BeaconProxy{ salt: _salt }(tokenBeacon, "");
    bridgedTokenAddress = address(bridgedToken); // @audit assign to named return

    (string memory name, string memory symbol, uint8 decimals) = abi.decode(_tokenMetadata, (string, string, uint8));
    BridgedToken(bridgedTokenAddress).initialize(name, symbol, decimals);
    emit NewTokenDeployed(bridgedTokenAddress, _nativeToken);
    // @audit removed explicit return statement
  }
```

**Linea:**
인지함; 향후 릴리스의 일부가 될 수 있습니다.


### 2개의 저장소 읽기를 절약하기 위해 `TokenBridge::isNewToken`에서 확인 순서 반전

**설명:** `TokenBridge::isNewToken`은 확인 순서를 반전시켜 2개의 저장소 읽기를 종종 절약할 수 있습니다. 예:

```solidity
  modifier isNewToken(address _token) {
    // @audit swapped order of checks to save 2 storage reads by often failing on first check
    if (bridgedToNativeToken[_token] != EMPTY || nativeToBridgedToken[sourceChainId][_token] != EMPTY)
      revert AlreadyBridgedToken(_token);
    _;
  }
```

이런 식으로 수정자(modifier)는 첫 번째 확인에서 실패할 가능성이 높으므로 1개의 저장소 읽기만 수행하고 두 번째 확인에서 발생하는 2개의 저장소 읽기를 트리거하지 않습니다.

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L68-R68)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L68-R69)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TokenBridge::bridgeToken`에서 `bridgedMappingValue` 변수 최적화 제거

**설명:** 예를 들어 다음과 같이 `TokenBridge::bridgeToken`에서 `bridgedMappingValue` 변수를 최적화하여 제거합니다.

```solidity
  function bridgeToken(
    address _token,
    uint256 _amount,
    address _recipient
  ) public payable nonZeroAddress(_token) nonZeroAddress(_recipient) nonZeroAmount(_amount) whenNotPaused nonReentrant {
    address nativeMappingValue = nativeToBridgedToken[sourceChainId][_token];

    if (nativeMappingValue == RESERVED_STATUS) {
      // Token is reserved
      revert ReservedToken(_token);
    }

    // @audit remove `bridgedMappingValue` and
    // instead initialize `nativeToken`
    address nativeToken = bridgedToNativeToken[_token];
    uint256 chainId;
    bytes memory tokenMetadata;

    // @audit use `nativeToken` in the comparison
    if (nativeToken != EMPTY) {
      // Token is bridged
      BridgedToken(_token).burn(msg.sender, _amount);
      // @audit remove assignment `nativeToken = bridgedMappingValue`
      chainId = targetChainId;
    } else {
   // @audit remaining code continues unchanged
```

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L153-R190)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L153-R188)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `sourceChainId`를 캐싱하여 `TokenBridge::bridgeToken`에서 2개의 저장소 읽기 절약

**설명:** `sourceChainId`는 항상 `TokenBridge::bridgeToken`의 시작 부분인 [L153](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/tokenBridge/TokenBridge.sol#L153)에서 한 번 읽힙니다.

그런 다음 `TokenBridge::bridgeToken`의 `else` 분기에서 `sourceChainId`는 항상 최소한 한 번 [L191](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/tokenBridge/TokenBridge.sol#L191)에서 읽히지만 두 번째로 읽힐 수도 있습니다 [L183](https://github.com/Consensys/zkevm-monorepo/blob/364b64d2a2704567adac00e3fa428a1901717666/contracts/contracts/tokenBridge/TokenBridge.sol#L183).

따라서 2개의 저장소 읽기를 절약하기 위해 `TokenBridge::bridgeToken` 시작 부분에서 `sourceChainId`를 캐시하는 것이 더 효율적입니다. 아래에 있는 `bridgeToken`의 최적화된 버전에는 이전 문제 _"TokenBridge::bridgeToken에서 bridgedMappingValue 변수 최적화 제거"_의 최적화도 포함되어 있습니다. 이는 이 최적화로 인해 생성될 수 있는 "스택이 너무 깊음(stack too deep)" 오류를 피하기 위해 필요합니다.
```solidity
  function bridgeToken(
    address _token,
    uint256 _amount,
    address _recipient
  ) public payable nonZeroAddress(_token) nonZeroAddress(_recipient) nonZeroAmount(_amount) whenNotPaused nonReentrant {
    // @audit cache `sourceChainId` and use it when setting `nativeMappingValue`
    uint256 sourceChainIdCache = sourceChainId;
    address nativeMappingValue = nativeToBridgedToken[sourceChainIdCache][_token];

    if (nativeMappingValue == RESERVED_STATUS) {
      // Token is reserved
      revert ReservedToken(_token);
    }

    // @audit remove `bridgedMappingValue` and
    // instead initialize `nativeToken`
    address nativeToken = bridgedToNativeToken[_token];
    uint256 chainId;
    bytes memory tokenMetadata;

    // @audit use `nativeToken` in the comparison
    if (nativeToken != EMPTY) {
      // Token is bridged
      BridgedToken(_token).burn(msg.sender, _amount);
      // @audit remove assignment `nativeToken = bridgedMappingValue`
      chainId = targetChainId;
    } else {
      // Token is native

      // For tokens with special fee logic, ensure that only the amount received
      // by the bridge will be minted on the target chain.
      uint256 balanceBefore = IERC20Upgradeable(_token).balanceOf(address(this));
      IERC20Upgradeable(_token).safeTransferFrom(msg.sender, address(this), _amount);
      _amount = IERC20Upgradeable(_token).balanceOf(address(this)) - balanceBefore;

      nativeToken = _token;

      if (nativeMappingValue == EMPTY) {
        // New token
        // @audit using cached `sourceChainIdCache`
        nativeToBridgedToken[sourceChainIdCache][_token] = NATIVE_STATUS;
        emit NewToken(_token);
      }

      // Send Metadata only when the token has not been deployed on the other chain yet
      if (nativeMappingValue != DEPLOYED_STATUS) {
        tokenMetadata = abi.encode(_safeName(_token), _safeSymbol(_token), _safeDecimals(_token));
      }
      // @audit using cached `sourceChainIdCache`
      chainId = sourceChainIdCache;
    }

    messageService.sendMessage{ value: msg.value }(
      remoteSender,
      msg.value, // fees
      abi.encodeCall(ITokenBridge.completeBridging, (nativeToken, _amount, _recipient, chainId, tokenMetadata))
    );
    emit BridgingInitiated(msg.sender, _recipient, _token, _amount);
  }
```

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L153-R190)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L153-R188)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TokenBridge::bridgeTokenWithPermit`에서 중복 수정자(modifiers) 제거

**설명:** `TokenBridge::bridgeTokenWithPermit`은 `bridgeToken`을 호출하므로 `bridgeToken`에 포함된 것과 동일한 수정자를 반복할 필요가 없습니다. 이는 가스 비용만 추가할 뿐입니다.

중복 수정자의 유일한 이점은 트랜잭션을 더 빠르게 되돌린다는 것이지만, 0이 아닌 입력이 전달되는 일반적인 대부분의 경우 이러한 추가 중복 수정자는 일반 사용자의 모든 성공적인 브리징 트랜잭션의 가스 비용을 증가시킬 뿐입니다.

`bridgeTokenWithPermit`에서 `nonZeroAddress`, `nonZeroAmount` 및 `whenNotPaused` 수정자를 제거하십시오. 이는 이미 `bridgeToken`에 의해 시행되기 때문입니다.

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L217-R217)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L217-R212)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TokenBridge::completeBridging`에서 `nativeMappingValue` 변수 최적화 제거

**설명:** 예를 들어 다음과 같이 `TokenBridge::completeBridging`에서 `nativeMappingValue` 변수를 최적화하여 제거합니다.
```solidity
  function completeBridging(
    address _nativeToken,
    uint256 _amount,
    address _recipient,
    uint256 _chainId,
    bytes calldata _tokenMetadata
  ) external nonReentrant onlyMessagingService onlyAuthorizedRemoteSender whenNotPaused {
    // @audit remove `nativeMappingValue` and instead
    // initialize `bridgedToken`
    address bridgedToken = nativeToBridgedToken[_chainId][_nativeToken];

    // @audit use `bridgedToken` for the check
    if (bridgedToken == NATIVE_STATUS || bridgedToken == DEPLOYED_STATUS) {
      // Token is native on the local chain
      IERC20Upgradeable(_nativeToken).safeTransfer(_recipient, _amount);
    } else {
       // @audit remove assignment `bridgedToken = nativeMappingValue;`
       //
       // @audit use `bridgedToken` for the check
      if (bridgedToken == EMPTY) {
        // New token
        bridgedToken = deployBridgedToken(_nativeToken, _tokenMetadata, sourceChainId);
        bridgedToNativeToken[bridgedToken] = _nativeToken;
        nativeToBridgedToken[targetChainId][_nativeToken] = bridgedToken;
      }
      BridgedToken(bridgedToken).mint(_recipient, _amount);
    }

    emit BridgingFinalized(
      _nativeToken,
      // @audit return 0 address as before if token was native on local chain
      bridgedToken == NATIVE_STATUS || bridgedToken == DEPLOYED_STATUS ? address(0x0) : bridgedToken,
      _amount,
      _recipient);
  }
```

**Linea:**
인지함; 향후 릴리스에 포함될 예정입니다.


### `TokenBridge::setDeployed`에서 `nativeToken` 변수 최적화 제거

**설명:** `TokenBridge::setDeployed`에서 쓰기만 하고 읽지 않는 `nativeToken` 변수를 최적화하여 제거합니다. 예:
```solidity
  function setDeployed(address[] calldata _nativeTokens) external onlyMessagingService onlyAuthorizedRemoteSender {
    // @audit remove `nativeToken`
    unchecked {
      for (uint256 i; i < _nativeTokens.length; ) {
        // @audit remove assignment nativeToken = _nativeTokens[i]
        nativeToBridgedToken[sourceChainId][_nativeTokens[i]] = DEPLOYED_STATUS;
        emit TokenDeployed(_nativeTokens[i]);
        ++i;
      }
    }
  }
```

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L303-L306)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L303-L306)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `sourceChainId`를 캐싱하여 `TokenBridge::removeReserved`에서 1개의 저장소 읽기 절약

**설명:** `sourceChainId`는 `TokenBridge::removeReserved`에서 저장소에서 두 번 읽히므로 캐싱하면 1개의 저장소 읽기를 절약할 수 있습니다. 예:
```solidity
  function removeReserved(address _token) external nonZeroAddress(_token) onlyOwner {
    // @audit cache `sourceChainId`
    uint256 sourceChainIdCache = sourceChainId;

    // @audit used cached version
    if (nativeToBridgedToken[sourceChainIdCache][_token] != RESERVED_STATUS) revert NotReserved(_token);
    nativeToBridgedToken[sourceChainIdCache][_token] = EMPTY;
  }
```

**Linea:**
[PR3249](https://github.com/Consensys/zkevm-monorepo/pull/3249/files#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L367-R370)에 병합된 커밋 [35c7807](https://github.com/Consensys/zkevm-monorepo/commit/35c7807756ccac2756be7da472f5e6ce0fd9b16a#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L367-R367)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `_feeInWei` 매개변수를 사용하여 `MinimumFeeChanged` 이벤트를 방출함으로써 `L2MessageServiceV1::setMinimumFee`에서 1개의 저장소 읽기 절약

**설명:** `_feeInWei` 매개변수를 사용하여 `MinimumFeeChanged` 이벤트를 방출함으로써 `L2MessageServiceV1::setMinimumFee`에서 1개의 저장소 읽기를 절약합니다. 예:
```solidity
  function setMinimumFee(uint256 _feeInWei) external onlyRole(MINIMUM_FEE_SETTER_ROLE) {
    uint256 previousMinimumFee = minimumFeeInWei;
    minimumFeeInWei = _feeInWei;

    emit MinimumFeeChanged(previousMinimumFee, _feeInWei, msg.sender);
  }
```

**Linea:**
인지함; 향후 릴리스에 포함될 예정입니다.


### `RateLimiter::_addUsedAmount`에서 `currentPeriodAmountTemp` 변수 최적화 제거

**설명:** 예를 들어 다음과 같이 `RateLimiter::_addUsedAmount`에서 `currentPeriodAmountTemp`를 최적화하여 제거합니다.
```solidity
  function _addUsedAmount(uint256 _usedAmount) internal {
    // @audit removed `currentPeriodAmountTemp`

    if (currentPeriodEnd < block.timestamp) {
      currentPeriodEnd = block.timestamp + periodInSeconds;
      // @audit removed assignment `currentPeriodAmountTemp = _usedAmount`
    } else {
      // @audit modifying `_usedAmount` instead of `currentPeriodAmountTemp`
      _usedAmount += currentPeriodAmountInWei;
    }

    // @audit use `_usedAmount` in check
    if (_usedAmount > limitInWei) {
      revert RateLimitExceeded();
    }

    // @audit use `_usedAmount` in assignment
    currentPeriodAmountInWei = _usedAmount;
  }
```

**Linea:**
인지함; 향후 릴리스의 일부가 될 수 있습니다.


### `RateLimiter::resetRateLimitAmount`에서 `usedLimitAmountToSet`, `amountUsedLoweredToLimit` 및 `usedAmountResetToZero` 변수 최적화 제거

**설명:** 예를 들어 다음과 같이 `RateLimiter::resetRateLimitAmount`에서 `usedLimitAmountToSet`, `amountUsedLoweredToLimit` 및 `usedAmountResetToZero` 변수를 최적화하여 제거합니다.
```solidity
  function resetRateLimitAmount(uint256 _amount) external onlyRole(RATE_LIMIT_SETTER_ROLE) {
    // @audit remove `usedLimitAmountToSet`, `amountUsedLoweredToLimit` and `usedAmountResetToZero`

    // @audit update this first as this always happens
    limitInWei = _amount;

    if (currentPeriodEnd < block.timestamp) {
      currentPeriodEnd = block.timestamp + periodInSeconds;
      // @audit remove assignment to `usedAmountResetToZero` and instead directly set
      // `currentPeriodAmountInWei` to zero instead of doing it later
      currentPeriodAmountInWei = 0;

      // @audit emitting event here as boolean variables removed
      emit LimitAmountChanged(_msgSender(), _amount, false, true);
    }
    // @audit refactor `else/if` statement to simplify
    else if(_amount < currentPeriodAmountInWei) {
        // @audit remove assignment usedLimitAmountToSet = _amount
        // @audit remove assignemnt to `amountUsedLoweredToLimit` and instead directly
        // set `currentPeriodAmountInWei` to `_amount` instead of doing it later
        currentPeriodAmountInWei = _amount;

        // @audit emitting event here as boolean variables removed
        emit LimitAmountChanged(_msgSender(), _amount, true, false);
    }
    // @audit new `else` condition to emit event for when
    // neither boolean was true
    else{
      emit LimitAmountChanged(_msgSender(), _amount, false, false);
    }
  }
```

또는 `usedLimitAmountToSet` 변수만 최적화하여 제거합니다.
```solidity
  function resetRateLimitAmount(uint256 _amount) external onlyRole(RATE_LIMIT_SETTER_ROLE) {
    // @audit remove `usedLimitAmountToSet`
    bool amountUsedLoweredToLimit;
    bool usedAmountResetToZero;

    if (currentPeriodEnd < block.timestamp) {
      currentPeriodEnd = block.timestamp + periodInSeconds;
      usedAmountResetToZero = true;
    } else {
      if (_amount < currentPeriodAmountInWei) {
        // @audit remove assignment usedLimitAmountToSet = _amount
        amountUsedLoweredToLimit = true;
      }
    }

    limitInWei = _amount;

    // @audit refactor if statement to handle zero case
    if(usedAmountResetToZero) currentPeriodAmountInWei = 0;
    // @audit use `_amount` for assignment instead of `usedLimitAmountToSet`
    else if(amountUsedLoweredToLimit) currentPeriodAmountInWei = _amount;

    emit LimitAmountChanged(_msgSender(), _amount, amountUsedLoweredToLimit, usedAmountResetToZero);
  }
```

**Linea:**
인지함; 향후 릴리스에 포함될 예정입니다.


### `ILineaRollup.sol`에서 사용되지 않는 `SubmissionDataV2::dataParentHash` 및 `SupportingSubmissionDataV2::dataParentHash` 제거

**설명:** `ILineaRollup.sol`에서 사용되지 않는 `SubmissionDataV2::dataParentHash` 및 `SupportingSubmissionDataV2::dataParentHash`를 제거합니다.

이들은 이전 구현의 잔재이며 제거해야 합니다. 현재 `LineaRollup.sol`에서 `dataParentHash`는 `submitDataAsCalldata`에서 `calldata`에서 `memory`로 복사되기만 할 뿐 실제로는 아무 데도 사용되지 않습니다.

**Linea:**
[PR3191](https://github.com/Consensys/zkevm-monorepo/pull/3191/files#diff-00fe7f4d2727f66fbd6b2e24a070b5c3783c70f0c8e838bc33a9c2f809eabd27L21)에 병합된 커밋 [2261c78](https://github.com/Consensys/zkevm-monorepo/pull/3191/commits/2261c7841c24f17b4174ec234166912f6c47a1e1#diff-00fe7f4d2727f66fbd6b2e24a070b5c3783c70f0c8e838bc33a9c2f809eabd27L21)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `LineaRollup::submitDataAsCalldata`에서 사용되지 않는 확인 제거

**설명:** 커밋 9840265eeb7d8ed9a4a5107b7fad056093b224e0에서 `WrongParentShnarfFinalBlockNumber` 오류와 함께 되돌려지는(revert) 확인이 `submitBlobs`에서 [제거](https://github.com/Consensys/zkevm-monorepo/commit/9840265eeb7d8ed9a4a5107b7fad056093b224e0#diff-408c766d202f05efdde69ebcd815ba16ad10f68249dd353f6359121ab312a9fbL229-L234)되었으며, 해당 오류를 포착하는 테스트도 다른 오류 `DataStartingBlockDoesNotMatch`를 포착하도록 [업데이트](https://github.com/Consensys/zkevm-monorepo/commit/9840265eeb7d8ed9a4a5107b7fad056093b224e0#diff-e503dae0dab65006b7464f1028b4605cf972c99ff7a41345d050b350c3568096L426)되었습니다.

그러나 동일한 확인이 `submitDataAsCalldata`에는 여전히 [존재](https://github.com/Consensys/zkevm-monorepo/blob/oz-2024-04-22-audit-code/contracts/contracts/LineaRollup.sol#L295-L300)하지만 이제 `LineaRollup.ts`에는 `WrongParentShnarfFinalBlockNumber` 오류를 예상하는 테스트가 없습니다.

이 오류는 이제 `_validateSubmissionData` 내에서 감지되어 새 오류 `DataStartingBlockDoesNotMatch`와 함께 되돌려집니다. 따라서 `WrongParentShnarfFinalBlockNumber`와 함께 되돌려지는 `submitDataAsCalldata`의 사용되지 않는 확인은 쓸모가 없으므로 삭제할 수 있습니다.

**권장 완화 방법:** `submitDataAsCalldata`에서 사용되지 않는 확인을 제거하십시오.

**Linea:**
[PR3191](https://github.com/Consensys/zkevm-monorepo/pull/3191/files#diff-408c766d202f05efdde69ebcd815ba16ad10f68249dd353f6359121ab312a9fbL293-L300)에 병합된 커밋 [2eba80b](https://github.com/Consensys/zkevm-monorepo/commit/2eba80b7b2fed5e7959d02eae8c451298d8fb7ec#diff-408c766d202f05efdde69ebcd815ba16ad10f68249dd353f6359121ab312a9fbL293-L300)에서 수정되었습니다.

**Cyfrin:** 확인됨.

