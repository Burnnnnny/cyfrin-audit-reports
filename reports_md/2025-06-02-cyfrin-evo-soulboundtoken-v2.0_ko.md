**수석 감사관**

[Dacian](https://x.com/DevDacian)

[Giovanni Di Siena](https://x.com/giovannidisiena)

**보조 감사관**

 

---

# 발견 사항 (Findings)
## 저위험 (Low Risk)


### 일반적인 Chainlink 오라클 유효성 검사 누락

**설명:** 프로토콜에 일반적인 [Chainlink 오라클 유효성 검사](https://medium.com/contractlevel/chainlink-oracle-defi-attacks-93b6cb6541bf)가 누락되어 있습니다. 결과에 대한 유효성 검사 없이 `AggregatorV3Interface::latestRoundData`를 호출합니다:
```solidity
function _getLatestPrice() internal view returns (uint256) {
    //slither-disable-next-line unused-return
    (, int256 price,,,) = i_nativeUsdFeed.latestRoundData();
    return uint256(price);
}
```

**권장 완화 방안:** 다음과 같은 일반적인 Chainlink 오라클 유효성 검사를 구현하십시오:
* 특정 오라클에 대한 [올바른 하트비트(heartbeat)](https://medium.com/contractlevel/chainlink-oracle-defi-attacks-93b6cb6541bf#fb78)를 사용하여 [오래된 가격(stale prices)](https://medium.com/contractlevel/chainlink-oracle-defi-attacks-93b6cb6541bf#99af) 확인
* [다운된 L2 시퀀서(down L2 sequencer)](https://medium.com/contractlevel/chainlink-oracle-defi-attacks-93b6cb6541bf#0faf) 확인, [`startedAt == 0`인 경우 되돌리기](https://solodit.contractlevel.io/issues/insufficient-checks-to-confirm-the-correct-status-of-the-sequenceruptimefeed-codehawks-zaros-git), 그리고 가격 데이터 가져오기를 재개하기 전 복구 후 약 2분의 작은 [유예 기간(grace period)](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code) 적용 가능성
* [반환된 가격이 최소 또는 최대 경계에 있지 않은지 확인](https://medium.com/contractlevel/chainlink-oracle-defi-attacks-93b6cb6541bf#00ac)

이 프로토콜의 경우 이러한 검사를 생략하는 것의 영향은 매우 미미합니다. 최악의 시나리오에서 사용자는 NFT를 더 저렴하거나 더 비싼 가격에 구매할 수 있지만, 다른 프로토콜에서 위협이 될 수 있는 프로토콜 지급 능력/사용자 청산 등에 대한 위협은 없습니다. 그리고 사용자는 1개의 NFT만 구매할 수 있고 판매/전송할 수 없으므로 큰 문제가 아닙니다. 가스 비용을 낮추기 위해 이러한 검사가 제외된 경우 이를 알리는 주석을 달아두는 것이 좋습니다.

**Evo:**
커밋 [6af531e](https://github.com/contractlevel/sbt/commit/6af531e49f7d7dd525da449bcdbdacb171e0c70d), [7a06688](https://github.com/contractlevel/sbt/commit/7a0668860f9b4f43798988349a966147bf94f33f), [93021e4](https://github.com/contractlevel/sbt/commit/93021e4f9c2afb40f64f9f9de69661a134702313)에서 수정됨.

**Cyfrin:** 확인함.


### 블랙리스트에서 제거된 사용자가 NFT에 대해 다시 지불해야 함

**설명:** 사용자가 블랙리스트에 추가되면 이미 지불한 NFT가 소각됩니다:
```solidity
function _addToBlacklist(address account) internal {
    // *snip: code not relevant *//

    if (balanceOf(account) > 0) {
        uint256 tokenId = tokenOfOwnerByIndex(account, 0); // Get first token
        _burn(tokenId); // Burn the token
    }
}
```

그러나 블랙리스트에서 제거되면 이전에 소각된 NFT를 보상하기 위한 무료 NFT를 받지 못하며, 수수료를 지불하지 않고 다시 NFT를 민팅할 수 있게 하는 플래그도 설정되지 않습니다:
```solidity
function _removeFromBlacklist(address account) internal {
    if (!s_blacklist[account]) revert SoulBoundToken__NotBlacklisted(account);

    s_blacklist[account] = false;
    emit RemovedFromBlacklist(account);
}
```

**파급력:** NFT를 구매한 후 블랙리스트에 올랐다가 제거된 사용자는 NFT를 얻기 위해 두 번 지불해야 합니다.

**권장 완화 방안:** 이것은 공정해 보이지 않습니다. 사용자가 블랙리스트에 올랐을 때 NFT가 소각되었다면, 나중에 블랙리스트에서 제거될 때 무료 NFT를 다시 받아야 합니다.

**Evo:**
인지함; 관리자 오류로 인해 사용자가 블랙리스트에 올랐다가 이후 블랙리스트에서 제거되는 드문 경우, DAO는 커뮤니티 투표를 통해 사용자에게 보상할 것입니다.


### 사용자에게 불리하게 수수료 올림(round up)

**설명:** Solidity는 기본적으로 내림(round down)을 수행하지만, 일반적으로 수수료는 사용자에게 불리하게 올림(round up)해야 합니다. Solady의 [라이브러리](https://github.com/Vectorized/solady/blob/main/src/utils/FixedPointMathLib.sol)를 사용하는 것이 OpenZeppelin보다 훨씬 효율적입니다:
```solidity
import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";

function _getFee() internal view returns (uint256 fee) {
    // read fee factor directly to output variable
    fee = s_feeFactor;

    // only do extra work if non-zero
    if(fee != 0) fee = FixedPointMathLib.fullMulDivUp(fee, PRICE_FEED_PRECISION, _getLatestPrice());
}
```

위의 방법을 사용할 때의 부차적인 이점은 [중간 곱셈 오버플로](https://x.com/DevDacian/status/1892529633104396479)로 인한 되돌리기 가능성을 제거하는 것이지만, 이 코드에서는 실제 가능성이 아닙니다.

사용자에게 불리하게 올림하고 싶지 않지만 기본값보다 약간 더 빠른 구현을 원한다면:
```solidity
function _getFee() internal view returns (uint256 fee) {
    // read fee factor directly to output variable
    fee = s_feeFactor;

    // only do extra work if non-zero
    if(fee != 0) fee = (fee * PRICE_FEED_PRECISION) / _getLatestPrice();
}
```

**Evo:**
커밋 [52c5384](https://github.com/contractlevel/sbt/commit/52c538448fbceb09f27ae657bcecb0c1483eb933)에서 수정됨.

**Cyfrin:** 확인함.


### 고래(Whale)가 `mintWithTerms`를 통해 거의 무한한 투표권을 구매할 수 있음

**설명:** 고래는 거의 무한한 주소를 제어할 수 있으므로 자금이 있는 한 `mintWithTerms`를 통해 거의 무한한 투표권을 구매할 수 있습니다.

**파급력:** 제안이 만료되기 몇 초 전에 제안을 결정하는 데 사용될 수 있습니다. 그러나 관리자가 NFT를 소각하는 주소를 블랙리스트에 올릴 수 있으므로 장기적인 영향은 제한적입니다.

**권장 완화 방안:** 제안 전에 전체 및 개별 사용자 투표권을 캡처하는 스냅샷 메커니즘을 구현하십시오. `mintWithTerms`가 "무제한 민트"를 허용하는 유일한 함수이므로 이를 통해 사용자가 NFT를 민팅하는 것을 방지하기 위해 일시 중지 기능을 구현하는 것을 고려하십시오.

**Evo:**
커밋 [2fbd2c5](https://github.com/contractlevel/sbt/commit/2fbd2c5379a5f07a8166ec4041f392d867f5bedd)에서 관리자가 `mintWithTerms`를 일시 중지할 수 있도록 하여 수정됨.

**Cyfrin:** 확인함.


### `_verifySignature`는 스마트 컨트랙트 지갑 또는 기타 스마트 계정과 호환되지 않음

**설명:** `_verifySignature`에 구현된 서명 검증은 EOA에 의해 생성된 서명만 처리할 수 있으므로 스마트 계정(예: [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)) 및 기타 스마트 컨트랙트 지갑(예: Safe{Wallet})의 토큰 민팅 지원은 현재 불가능합니다. 여기서 스마트 계정뿐만 아니라 다중 서명 지갑 또는 기타 사용 사례를 포함할 수 있는 다른 스마트 컨트랙트에 대한 서명 검증을 지원하는 것이 유익할 수 있습니다. 예를 들어 다른 조직이 회원으로 참여할 수 있도록 자체 스마트 컨트랙트 인프라를 갖춘 DAO가 있습니다.

[ERC-7702](https://eips.ethereum.org/EIPS/eip-7702) 계정으로 업그레이드된 EOA는 영향을 받지 않지만, [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271)을 구현하지 않으면 다른 스마트 컨트랙트 서명을 검증할 수 없습니다. 그러나 이는 EIP-7702 계정의 경우 코드 길이가 0이 아니라는 추가적인 고려 사항을 추가하므로, 이러한 계정은 EIP-1271을 사용하여 서명을 검증할 수 있지만 개인 키는 여전히 트랜잭션 서명에 대한 완전한 권한을 보유합니다. 즉, [OpenZeppelin SignatureChecker 라이브러리](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/SignatureChecker.sol)와 같은 코드 길이 확인 구현은 `eth/personal_sign`을 사용하여 생성된 이러한 계정의 서명 검증을 계속 허용하도록 약간 수정되어야 합니다. 추가 논의는 [여기](https://blog.rhinestone.wtf/unlocking-chain-abstracted-eoas-with-eip-7702-and-irrevocable-signatures-adc820a150ef)에서 찾을 수 있습니다.

이러한 스마트 컨트랙트 서명을 지원하려면 다음과 같이 OpenZeppelin SignatureChecker 라이브러리 함수 `isValidERC1271SignatureNow`로 대체하는 것을 고려하십시오:

```diff
    function _verifySignature(bytes memory signature) internal view returns (bool) {
        /// @dev compute the message hash: keccak256(termsHash, msg.sender)
        bytes32 messageHash = keccak256(abi.encodePacked(s_termsHash, msg.sender));

        /// @dev apply Ethereum signed message prefix
        bytes32 ethSignedMessageHash = MessageHashUtils.toEthSignedMessageHash(messageHash);

        /// @dev attempt to recover the signer
        //slither-disable-next-line unused-return
        (address recovered, ECDSA.RecoverError error,) = ECDSA.tryRecover(ethSignedMessageHash, signature);

        /// @dev return false if errors or incorrect signer
        if (error == ECDSA.RecoverError.NoError && recovered == msg.sender) return true;
        else return SignatureChecker.isValidERC1271SignatureNow(msg.sender, ethSignedMessageHash, signature);
    }
```

**Evo:**
커밋 [6d4f41c](https://github.com/contractlevel/sbt/commit/6d4f41ce160e19713bb4a6cafdf1b739df98e027#diff-39790f8feee6ea105eee119137d7c3d881007ed50d9a62590b54ff559f45b27aL511-R514)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보성 (Informational)


### `Ownable` 대신 `Ownable2Step` 선호

**설명:** [더 안전한 소유권 이전](https://www.rareskills.io/post/openzeppelin-ownable2step)을 위해 `Ownable` 대신 [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)을 선호하십시오.

**Evo:**
커밋 [620120e](https://github.com/contractlevel/sbt/commit/620120ec5a85c3a2dbd1be52af4ee34fe946efbe)에서 수정됨.

**Cyfrin:** 확인함.


### Chainlink 가격 피드 소수점을 가정하면 의도하지 않은 오류가 발생할 수 있음

**설명:** 일반적으로 Chainlink x/USD 가격 피드는 8자리 소수점 정밀도를 사용하지만 이것이 보편적으로 사실인 것은 아닙니다. 예를 들어 [AMPL/USD](https://etherscan.io/address/0xe20CA8D7546932360e37E9D72c1a47334af57706#readContract#F3)는 18자리 소수점 정밀도를 사용합니다.

[Chainlink 오라클 가격 정밀도를 가정](https://medium.com/contractlevel/chainlink-oracle-defi-attacks-93b6cb6541bf#87fc)하는 대신 정밀도 변수를 `immutable`로 선언하고 [`AggregatorV3Interface::decimals`](https://docs.chain.link/data-feeds/api-reference#decimals)를 통해 생성자에서 초기화할 수 있습니다.

그러나 실제로는 가격 오라클이 `script/HelperConfig.s.sol`에 하드 코딩되어 있고 Optimism에서 8자리 소수를 사용하므로 현재 구성은 잘 작동할 것입니다.

**Evo:**
커밋 [f594ae0](https://github.com/contractlevel/sbt/commit/f594ae004d4afc80f19e17c0f61d50caa00a4811)에서 수정됨.

**Cyfrin:** 확인함.


### 기본값으로 초기화하지 말 것

**설명:** Solidity가 이미 수행하므로 기본값으로 초기화하지 마십시오:
```solidity
SoulBoundToken.sol
125:        for (uint256 i = 0; i < admins.length; ++i) {
162:        for (uint256 i = 0; i < accounts.length; ++i) {
226:        for (uint256 i = 0; i < accounts.length; ++i) {
245:        for (uint256 i = 0; i < accounts.length; ++i) {
270:        for (uint256 i = 0; i < accounts.length; ++i) {
289:        for (uint256 i = 0; i < accounts.length; ++i) {
312:        for (uint256 i = 0; i < accounts.length; ++i) {
```

**Evo:**
커밋 [f594ae0](https://github.com/contractlevel/sbt/commit/f594ae004d4afc80f19e17c0f61d50caa00a4811)에서 수정됨.

**Cyfrin:** 확인함.


### 명명된 반환(named returns)을 이미 사용하는 경우 불필요한 `return` 문 제거

**설명:** 명명된 반환을 이미 사용하는 경우 불필요한 `return` 문을 제거하십시오:
```diff
    function _mintSoulBoundToken(address account) internal returns (uint256 tokenId) {
        tokenId = _incrementTokenIdCounter(1);
        _safeMint(account, tokenId);
-       return tokenId;
    }

    function _incrementTokenIdCounter(uint256 count) internal returns (uint256 startId) {
        startId = s_tokenIdCounter;
        s_tokenIdCounter += count;
-       return startId;
    }
```

**Evo:**
커밋 [f594ae0](https://github.com/contractlevel/sbt/commit/f594ae004d4afc80f19e17c0f61d50caa00a4811)에서 수정됨.

**Cyfrin:** 확인함.


### 사용자가 초과 지불하는 것을 방지하는 것 고려

**설명:** 현재 프로토콜은 사용자가 초과 지불하는 것을 허용합니다:
```solidity
function _revertIfInsufficientFee() internal view {
    if (msg.value < _getFee()) revert SoulBoundToken__InsufficientFee();
}
```

사용자가 실수로 초과 지불하는 것을 방지하기 위해 정확한 수수료를 요구하도록 변경하는 것을 고려하십시오:
```solidity
function _revertIfIncorrectFee() internal view {
    if (msg.value != _getFee()) revert SoulBoundToken__IncorrectFee();
}
```

[팻 핑거(Fat Finger)](https://en.wikipedia.org/wiki/Fat-finger_error) 오류는 이전에 금융 시장에서 악명 높은 의도하지 않은 오류를 초래했습니다. 프로토콜은 방어적인 태도를 취하고 사용자 자신을 보호하도록 선택할 수 있습니다.

**Evo:**
커밋 [e3b2f74](https://github.com/contractlevel/sbt/commit/e3b2f74239601b2721118e11aaa92b42dbb502e9)에서 수정됨.

**Cyfrin:** 확인함.


### `mintWithTerms`에서 `else` 생략 가능

**설명:** 서명 유효성 검사가 실패하면 되돌리기가 발생하므로 `mintWithTerms`에서 `else`를 생략할 수 있습니다:
```diff
        if (!_verifySignature(signature)) revert SoulBoundToken__InvalidSignature();
-       else emit SignatureVerified(msg.sender, signature);
+       emit SignatureVerified(msg.sender, signature);
        tokenId = _mintSoulBoundToken(msg.sender);
```

**Evo:**
커밋 [e3b2f74](https://github.com/contractlevel/sbt/commit/e3b2f74239601b2721118e11aaa92b42dbb502e9)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 로컬 함수 변수를 제거할 수 있는 곳과 `memory` 반환에 명명된 반환 사용

**설명:** 명명된 반환을 사용하는 것은 로컬 함수 변수를 제거할 수 있는 곳과 `memory` 반환에 더 가스 효율적입니다:
```diff
-   function batchMintAsAdmin(address[] calldata accounts) external onlyAdmin returns (uint256[] memory) {
+   function batchMintAsAdmin(address[] calldata accounts) external onlyAdmin returns (uint256[] memory tokenIds) {
        _revertIfEmptyArray(accounts);
        uint256 startId = _incrementTokenIdCounter(accounts.length);

-      uint256[] memory tokenIds = new uint256[](accounts.length);
+      tokenIds = new uint256[](accounts.length);
        for (uint256 i = 0; i < accounts.length; ++i) {
            _mintAsAdminChecks(accounts[i]);
            tokenIds[i] = startId + i;
            _safeMint(accounts[i], tokenIds[i]);
        }
-      return tokenIds;
    }
```

가스 결과:
```diff
{
-  "batchMintAsAdmin": "252114"
+  "batchMintAsAdmin": "252102"
}
```

**Evo:**
커밋 [b4fcadb](https://github.com/contractlevel/sbt/commit/b4fcadbd9c5684cc4e3b1ee3c39f72c406aaf658)에서 수정됨.

**Cyfrin:** 확인함.


### 최적화 도구 활성화

**설명:** `foundry.toml`에서 [최적화 도구를 활성화](https://dacian.me/the-yieldoor-gas-optimizoor#heading-enabling-the-optimizer)하십시오.

가스 결과:
```diff
{
-  "addToBlacklist": "31090"
+  "addToBlacklist": "30691"

-  "addToWhitelist": "28754"
+  "addToWhitelist": "28392"

-  "batchAddToBlacklist": "60282"
+  "batchAddToBlacklist": "59482"

-  "batchAddToWhitelist": "55790"
+  "batchAddToWhitelist": "54997"

-  "batchMintAsAdmin": "252102"
+  "batchMintAsAdmin": "248867"

-  "batchRemoveFromBlacklist": "5289"
+  "batchRemoveFromBlacklist": "4594"

-  "batchRemoveFromWhitelist": "5305"
+  "batchRemoveFromWhitelist": "4677"

-  "batchSetAdmin": "28090"
+  "batchSetAdmin": "27412"

-  "mintAsAdmin": "130754"
+  "mintAsAdmin": "129447"

-  "mintAsWhitelisted": "135623"
+  "mintAsWhitelisted": "132292"

-  "mintWithTerms": "142281"
+  "mintWithTerms": "137638"

-  "removeFromBlacklist": "2516"
+  "removeFromBlacklist": "2203"

-  "removeFromWhitelist": "2634"
+  "removeFromWhitelist": "2254"

-  "setAdmin": "27187"
+  "setAdmin": "26677"

-  "setContractURI": "29118"
+  "setContractURI": "26842"

-  "setFeeFactor": "26075"
+  "setFeeFactor": "25666"

-  "setWhitelistEnabled": "7175"
+  "setWhitelistEnabled": "6902"

-  "withdrawFees": "14114"
+  "withdrawFees": "13462"
}

```

**Evo:**
커밋 [b4fcadb](https://github.com/contractlevel/sbt/commit/b4fcadbd9c5684cc4e3b1ee3c39f72c406aaf658)에서 수정됨.

**Cyfrin:** 확인함.


### 외부 읽기 전용 입력에 `memory` 대신 `calldata` 선호

**설명:** 외부 읽기 전용 입력에 `memory` 대신 `calldata`를 선호하십시오:
```diff
-   function mintWithTerms(bytes memory signature) external payable returns (uint256 tokenId) {
+   function mintWithTerms(bytes calldata signature) external payable returns (uint256 tokenId) {

-   function _verifySignature(bytes memory signature) internal view returns (bool) {
+   function _verifySignature(bytes calldata signature) internal view returns (bool) {
```

가스 결과:
```diff
{
-  "mintWithTerms": "137638"
+  "mintWithTerms": "137299"
}
```

**Evo:**
커밋 [b4fcadb](https://github.com/contractlevel/sbt/commit/b4fcadbd9c5684cc4e3b1ee3c39f72c406aaf658)에서 수정됨.

**Cyfrin:** 확인함.


### eth 전송을 위해 solady `safeTransferETH` 사용

**설명:** solady [`safeTransferETH`](https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol#L90-L98)를 사용하는 것은 eth를 보내는 [더 효율적인](https://github.com/devdacian/solidity-gas-optimization?tab=readme-ov-file#10-use-safetransferlibsafetransfereth-instead-of-solidity-call-effective-035-cheaper) 방법입니다. 또한 컨트랙트 내부에 eth를 남겨둘 이유가 없으므로 `amountToWithdraw` 입력 매개변수와 관련된 검사를 제거하는 것을 고려하십시오. 대신 전체 컨트랙트 잔액을 보내십시오:
```solidity
function withdrawFees() external onlyOwner {
    uint256 amountToWithdraw = address(this).balance;
    if(amountToWithdraw > 0) {
        // from https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol#L90-L98
        /// @solidity memory-safe-assembly
        assembly {
            if iszero(call(gas(), caller(), amountToWithdraw, codesize(), 0x00, codesize(), 0x00)) {
                mstore(0x00, 0xefde920d) // `SoulBoundToken__WithdrawFailed()`.
                revert(0x1c, 0x04)
            }
        }

        emit FeesWithdrawn(amountToWithdraw);
    }
}
```

가스 결과:
```diff
{
- "withdrawFees": "13462"
+ "withdrawFees": "13353"
}
```

**Evo:**
커밋 [b4fcadb](https://github.com/contractlevel/sbt/commit/b4fcadbd9c5684cc4e3b1ee3c39f72c406aaf658)에서 수정됨.

**Cyfrin:** 확인함.

