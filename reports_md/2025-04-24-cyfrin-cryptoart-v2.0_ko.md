**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Hans](https://x.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Medium Risk


### 수집가가 `CreatorStory`를 추가하여 예술 작품의 출처를 손상시킬 수 있음 (Collector can add `CreatorStory`, corrupting the provenance of an artwork)

**설명:** `IStory` 인터페이스의 목적은 3개의 서로 다른 주체(관리자, 창작자, 수집가)가 주어진 예술 작품(NFT)에 대해 [예술 작품의 출처를 설명하는](https://docs.transientlabs.xyz/tl-creator-contracts/common-features/story-inscriptions) "Stories"를 추가할 수 있도록 하는 것입니다. 예술계에서 아이템의 "출처(provenance)"는 그 지위와 가격에 영향을 미칠 수 있으므로, `IStory` 인터페이스는 예술 작품의 "출처"에 대한 온체인 기록을 용이하게 하는 것을 목표로 합니다.

`IStory`는 다음과 같이 작동하도록 설계되었습니다:
* 창작자/관리자는 컬렉션이 계약에 추가될 때 `CollectionsStory`를 추가할 수 있습니다.
* 예술 작품의 창작자는 `CreatorStory`를 추가할 수 있습니다.
* 예술 작품의 수집가는 예술 작품을 보유하는 동안의 경험에 대해 하나 이상의 `Story`를 추가할 수 있습니다.

`IStory` 인터페이스 사양은 `addCreatorStory`가 창작자에 의해서만 호출될 것을 요구합니다:
```solidity
/// @notice Function to let the creator add a story to any token they have created
/// @dev This function MUST implement logic to restrict access to only the creator
function addCreatorStory(uint256 tokenId, string calldata creatorName, string calldata story) external;
```

그러나 `IStory` 인터페이스의 CryptoArt 구현에서는 현재 토큰 소유자가 항상 `CreatorStory` 이벤트를 방출할 수 있습니다:
```solidity
function addCreatorStory(uint256 tokenId, string calldata, /*creatorName*/ string calldata story)
    external
    onlyTokenOwner(tokenId)
{
    emit CreatorStory(tokenId, msg.sender, msg.sender.toHexString(), story);
}
```

**영향:** NFT가 새로운 소유자에게 판매되거나 양도됨에 따라, 각 후속 소유자는 자신이 예술 작품의 창작자가 아님에도 불구하고 새로운 `CreatorStory` 이벤트를 계속 추가할 수 있습니다. 이는 수집가가 마치 창작자인 것처럼 `CreatorStory`에 추가할 수 있게 함으로써 예술 작품의 출처를 훼손합니다.

**권장 완화 방법:** 예술 작품의 창작자만이 `CreatorStory` 이벤트를 방출할 수 있어야 합니다. 현재 온체인 프로토콜은 창작자의 주소를 기록하지 않습니다. 이를 추가하거나 계약 소유자가 창작자의 대리인 역할을 하는 경우 `onlyOwner`를 사용할 수 있습니다.

**CryptoArt:**
커밋 [94bfc1b](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/94bfc1b1454e783ef1fb9627cfaf0328ebe17b47#diff-1c61f2d0e364fa26a4245d1033cdf73f09117fbee360a672a3cb98bc0eef02adL439-R439)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Low Risk


### `IStory` 이벤트에서 사용자 지정 창작자 및 수집가 이름을 허용하여 예술 작품 출처 구축 (Allow custom Creator and Collector names to be emitted in `IStory` events to build artwork provenance)

**설명:** `IStory` 인터페이스는 창작자 및 수집가 이벤트에 대해 사용자 지정 이름을 방출할 수 있도록 설계되었습니다. 다음은 창작자가 `lytke`라는 사용자 지정 이름을 사용한 [예시](https://www.transient.xyz/nfts/base/0x6c81306129b3cc63b0a6c7cec3dd50721ac378fe/9)입니다.

그러나 CryptoArt의 `IStory` 인터페이스 구현에서는 사용자 지정 이름이 허용되지 않으며 항상 호출자의 16진수 문자열이 설정됩니다:
```solidity
function addCollectionStory(string calldata, /*creatorName*/ string calldata story) external onlyOwner {
    emit CollectionStory(msg.sender, msg.sender.toHexString(), story);
}

/// @inheritdoc IStory
function addCreatorStory(uint256 tokenId, string calldata, /*creatorName*/ string calldata story)
    external
    onlyTokenOwner(tokenId)
{
    emit CreatorStory(tokenId, msg.sender, msg.sender.toHexString(), story);
}

/// @inheritdoc IStory
function addStory(uint256 tokenId, string calldata, /*collectorName*/ string calldata story)
    external
    onlyTokenOwner(tokenId)
{
    emit Story(tokenId, msg.sender, msg.sender.toHexString(), story);
}
```

**영향:** 사용자 지정 이름은 예술 작품의 "출처"의 일부를 형성하므로 허용되어야 합니다. 예술 작품의 가치는 종종 누가 창작했는지, 그리고 과거에 중요한 수집가가 보유했는지에 따라 결정됩니다. 적절한 사용자 지정 이름은 0x1343335...보다 훨씬 기억하기 쉽고 이야기를 전달하기 쉽습니다. 사용자 지정 이름이 있는 예술 작품은 더 나은 이야기를 구축하여 "출처"를 향상시킬 수 있습니다.

**CryptoArt:**
커밋 [77f34a4](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/77f34a49cbc27589f3179b35b58a86696696bf83)에서 수정됨.

**Cyfrin:** 검증됨.


### `IStory` 이벤트 방출 또는 `tokenURI` 필드 설정 시 `string` 필드 내 코드 주입 방지 (Prevent code-injection inside `string` fields when emitting `IStory` events or setting `tokenURI` fields)

**설명:** 웹3와 웹2의 교차점에 걸쳐 있는 공격 벡터는 사용자가 [임의의 메타데이터 문자열을 NFT와 연관](https://medium.com/zokyo-io/under-the-hackers-hood-json-injection-in-nft-metadata-3be78d0f93a7)시킬 수 있고 해당 문자열이 나중에 웹사이트에서 처리되거나 표시될 때 발생합니다.

특히 `IStory::Story` 이벤트는 다음과 같습니다:
* 신뢰할 수 없는 주체인 예술 작품의 현재 보유자에 의해 방출됨
* 두 개의 임의 문자열 매개변수 `collectorName`과 `story`를 방출함
* 이러한 문자열 매개변수는 사용자에게 표시되도록 설계되었으며 웹2 앱에서 처리될 수 있음

**권장 완화 방법:** 가장 중요한 유효성 검사는 신뢰할 수 없는 사용자가 시작한 함수에 대한 것입니다. 예:
* 창작자가 `CreatorStory`를 방출하거나 수집가가 `Story`를 방출할 때, `name` 및 `story` 문자열에 예상치 못한 특수 문자가 포함되어 있으면 revert합니다.
* 토큰을 발행(minting)할 때 `TokenURISet::uriWhenRedeemable` 및 `uriWhenNotRedeemable`에 예상치 못한 특수 문자가 포함되어 있으면 revert합니다. 단, 이는 프로토콜이 제어하는 오프체인 구성 요소를 사용하여 수행되어야 하므로 여기서는 위험이 최소화됩니다.
* 오프체인 코드에서는 사용자가 제공한 문자열을 신뢰하지 말고 삭제하거나 예상치 못한 특수 문자가 있는지 확인하세요.

**CryptoArt:**
인지함; 사전 서명 URI 유효성 검사 및 이벤트 후 Story 문자열 삭제/인코딩을 통해 오프체인에서 완화 처리됨.


### 사용자가 nonce를 증가시켜 서명을 무효화할 수 있도록 허용 (Allow users to increment their nonce to void their signatures)

**설명:** 현재 사용자는 `NoncesUpgradeable::_useNonce`가 `internal`이며 사용자 서명을 검증하는 작업 중에만 호출되므로 nonce를 증가시켜 서명을 무효화할 수 없습니다.

사용자는 이전 서명이 사용되는 것을 방지하기 위해 무효화하고 싶을 수 있지만 불가능합니다.

**영향:** 사용자는 서명이 사용되기 전에 이전 서명을 무효화할 수 없습니다.

**권장 완화 방법:** 사용자가 자신의 nonce를 증가시킬 수 있도록 허용하는 `public` 함수를 통해 `NoncesUpgradeable::_useNonce`를 노출하세요.

**CryptoArt:**
커밋 [cf82aeb](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/cf82aeb30d6a262cde51897f52c302be995d0202)에서 수정됨.

**Cyfrin:** 검증됨.


### `IERC7160` 사양은 존재하지 않는 `tokenId`에 대해 `hasPinnedTokenURI`가 revert되도록 요구함 (`IERC7160` specification requires `hasPinnedTokenURI` to revert for non-existent `tokenId`)

**설명:** `IERC7160` 사양에 따르면:
```solidity
/// @notice Check on-chain if a token id has a pinned uri or not
/// @dev This call MUST revert if the token does not exist
function hasPinnedTokenURI(uint256 tokenId) external view returns (bool pinned);
```

그러나 `hasPinnedTokenURI`의 구현은 존재하지 않는 토큰에 대해 revert하지 않으며, 단순히 `false`를 반환하거나 값이 true일 때 토큰이 소각된 경우 `true`를 반환합니다. 소각이 `_hasPinnedTokenURI`를 삭제하지 않기 때문입니다(이를 추적하기 위해 다른 이슈가 생성됨):
```solidity
function hasPinnedTokenURI(uint256 tokenId) external view returns (bool) {
    return _hasPinnedTokenURI[tokenId];
}
```

**권장 완화 방법:** `onlyIfTokenExists` modifier를 사용하세요:
```diff
-    function hasPinnedTokenURI(uint256 tokenId) external view returns (bool) {
+    function hasPinnedTokenURI(uint256 tokenId) external view onlyIfTokenExists(tokenId) returns (bool) {
```

**CryptoArt:**
커밋 [56d0e22](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/56d0e222cdf25a971cd6466fd4757185a4362069)에서 수정됨.

**Cyfrin:** 검증됨.


### `IERC7160` 사양은 존재하지 않는 `tokenId`에 대해 `pinTokenURI`가 revert되도록 요구함 (`IERC7160` specification requires `pinTokenURI` to revert for non-existent `tokenId`)

**설명:** `IERC7160` 사양에 따르면:
```solidity
/// @notice Pin a specific token uri for a particular token
/// @dev This call MUST revert if the token does not exist
function pinTokenURI(uint256 tokenId, uint256 index) external;
```

그러나 `pinTokenURI`의 구현은 존재하지 않는 토큰에 대해 revert하지 않습니다. 왜냐하면 `tokenId`가 존재하지 않더라도 `_tokenURIs[tokenId].length`는 항상 2와 같기 때문입니다:
```solidity
// 매핑 값은 항상 고정 배열 크기 2를 가짐
mapping(uint256 tokenId => string[2] tokenURIs) private _tokenURIs;

function pinTokenURI(uint256 tokenId, uint256 index) external onlyOwner {
    if (index >= _tokenURIs[tokenId].length) {
        revert Error.Token_IndexOutOfBounds(tokenId, index, _tokenURIs[tokenId].length - 1);
    }

    _pinnedURIIndex[tokenId] = index;

    emit TokenUriPinned(tokenId, index);
    emit MetadataUpdate(tokenId);
}
```

**권장 완화 방법:** `onlyIfTokenExists` modifier를 사용하세요:
```diff
-    function pinTokenURI(uint256 tokenId, uint256 index) external onlyOwner {
+    function pinTokenURI(uint256 tokenId, uint256 index) external onlyIfTokenExists(tokenId) onlyOwner {
```

**CryptoArt:**
커밋 [0409ae4](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/0409ae4d81225a351c4d42620502843242f2604f)에서 수정됨.

**Cyfrin:** 검증됨.


### 일관되지 않은 일시 중지 기능으로 인해 계약이 일시 중지되었을 때 특정 상태 변경 작업이 허용됨 (Inconsistent pause functionality allows certain state-changing operations when contract is paused)

**설명:** `CryptoartNFT` 계약은 OpenZeppelin의 `PausableUpgradeable` 계약을 사용하여 일시 중지 메커니즘을 구현합니다. 그러나 일시 중지 기능은 계약의 함수 전반에 걸쳐 일관되지 않게 적용됩니다. 발행(minting) 및 소각(burning) 작업은 `whenNotPaused` modifier로 적절하게 보호되지만, 토큰 전송, 메타데이터 관리 및 스토리 관련 기능을 포함한 여러 다른 상태 변경 함수는 계약이 일시 중지된 경우에도 여전히 액세스할 수 있습니다.

다음 상태 변경 함수에는 `whenNotPaused` modifier가 없습니다:

1. 토큰 전송 및 승인 (ERC721에서 상속됨)
2. 메타데이터 관리 함수:
   - `updateMetadata`
   - `pinTokenURI`
   - `markAsRedeemable`
3. 스토리 관련 함수:
   - `addCollectionStory`
   - `addCreatorStory`
   - `addStory`
   - `toggleStoryVisibility`

**영향:** 계약이 일시 중지된 경우(일반적으로 비상 상황 또는 업그레이드 중), 사용자는 여전히 일시 중지 기간 동안 바람직하지 않을 수 있는 다양한 상태 변경 작업을 수행할 수 있습니다. 이는 계약 업그레이드 또는 비상 상황 중에 예상치 못한 상태 변경으로 이어질 수 있습니다.

**권장 완화 방법:** 모든 상태 변경 함수에 `whenNotPaused` modifier를 추가하여 계약이 일시 중지될 때 일관된 동작을 보장하세요.

**Cryptoart:**
커밋 [e7d7e5b](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/e7d7e5b3b1c8976a11d49f889b4168ce649be2ee)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Informational


### 프로토콜이 크로스 체인 서명 재생에 취약함 (Protocol vulnerable to cross-chain signature replay)

**설명:** 서명에 `chainId`가 포함되지 않으므로 서명 검증이 [크로스 체인 재생](https://dacian.me/signature-replay-attacks#heading-cross-chain-replay)에 취약합니다.

**영향:** 프로토콜은 향후 크로스 체인 배포를 계획하고 있지만, 이 감사의 사양은 하나의 체인에만 배포하는 것을 고려합니다. 따라서 이 공격 경로는 프로토콜이 하나의 체인에만 배포될 때는 불가능하므로 이 발견 사항은 정보성일 뿐입니다.

**권장 완화 방법:** `block.chainid`를 서명 매개변수로 포함하세요.

**CryptoArt:**
커밋 [1e25f8c](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/1e25f8cd172a32e3e35ccf8a86e7af9fe1ed47fe)에서 수정됨.

**Cyfrin:** 검증됨.


### 서명에 만료 기한이 없음 (Signatures have no expiration deadline)

**설명:** [만료 매개변수가 없는](https://dacian.me/signature-replay-attacks#heading-no-expiration) 서명은 사실상 평생 라이선스를 부여합니다. 서명에 만료 매개변수를 추가하여 해당 시간 이후에 사용하면 서명이 무효화되도록 하는 것을 고려하세요.

**CryptoArt:**
커밋 [a93977d](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/a93977d2ef0b54319c7668d9fc6abda688b355c1)에서 수정됨.

**Cyfrin:** 검증됨.


### 판매 수수료의 상당 부분 또는 전부가 로열티로 부과되는 것을 방지하기 위해 최대 로열티 제한 고려 (Consider limiting max royalty to prevent large amount or all of the sale fee being taken as royalty)

**설명:** 현재 `updateRoyalties` 및 `setTokenRoyalty`는 계약 소유자가 최대 `10_000`까지 로열티를 설정할 수 있도록 허용하며, 이는 판매 수수료 전체를 로열티로 가져가게 됩니다. 이러한 함수를 1000(10%)과 같이 더 합리적인 수준으로 최대 로열티를 설정하도록 제한하는 것을 고려하세요.

**CryptoArt:**
커밋 [1d1125e](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/1d1125e5a021f2926dc2a2e39e05c065e3bd207c)에서 수정됨.

**Cyfrin:** 검증됨.


### `MintType`이 거의 강제되지 않음 (`MintType` is almost never enforced)

**설명:** 계약에는 여러 유형의 민트(mint)를 정의하는 `MintType` 열거형이 있습니다:
```solidity
enum MintType {
    OpenMint,
    Whitelist,
    Claim,
    Burn
}
```

그러나 이러한 민트 유형에 대한 확인은 거의 없습니다. 예를 들어:
* `MintType.Whitelist`에 대한 확인이 없으며 그에 상응하는 화이트리스트 강제도 없습니다.
* `claim` 함수는 입력 `data.mintType == MintType.Claim`을 강제하지 않습니다.
* 마찬가지로 `burnAndMint`는 입력 `data.mintType == MintType.Burn`을 강제하지 않습니다.

입력 `data.mintType`이 사용되는 유일한 곳은 `_validateSignature`에서 입력 매개변수가 서명된 내용과 일치하는지 확인하는 것뿐이며, 올바른 작업에 올바른 민트 유형이 사용되고 있는지에 대한 다른 유효성 검사는 없습니다.

**CryptoArt:**
커밋 [deaf964](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/deaf96420b3176be09c1522ea8c79a211f77ef82)에서 수정됨.

**Cyfrin:** 검증됨.


### 사용되지 않는 상수 `CryptoartNFT::ROYALTY_BASE` 제거 (Remove unused constant `CryptoartNFT::ROYALTY_BASE`)

**설명:** `CryptoartNFT` 계약은 계약에서 사용되지 않는 값이 10,000인 상수 `ROYALTY_BASE`를 정의합니다. 이 상수는 로열티 비율 계산의 분모(10,000 = 100%)를 나타내기 위한 것이지만 계약 구현의 어디에서도 참조되지 않습니다.

**권장 완화 방법:** 코드 명확성을 개선하고 배포 가스 비용을 줄이기 위해 사용되지 않는 상수를 제거하세요.

**Cryptoart:**
커밋 [0c0dd8c](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/0c0dd8c8d01e1b5b396852d38faceee007b37891)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Gas Optimization


### 특히 `memory` 반환의 경우 명명된 반환 매개변수 선호 (Prefer named return parameters, especially for `memory` returns)

**설명:** 특히 메모리 반환의 경우 명명된 반환 매개변수를 선호하세요. 예를 들어 `tokenURIs`는 지역 변수와 명시적 반환을 제거하도록 리팩토링할 수 있습니다:
```solidity
function tokenURIs(uint256 tokenId)
    external
    view
    override
    onlyIfTokenExists(tokenId)
    returns (uint256 index, string[2] memory uris, bool isPinned)
{
    index = _getTokenURIIndex(tokenId);
    uris = _tokenURIs[tokenId];
    isPinned = _hasPinnedTokenURI[tokenId];
}
```

**CryptoArt:**
커밋 [bdd28fa](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/bdd28fa71f8d445fb3306a1fdc16b49fa5b5d1e4)에서 수정됨.

**Cyfrin:** 검증됨.


### 매직 넘버의 목적을 나타내기 위해 명명된 상수 사용 (Use named constants to indicate purpose of magic numbers)

**설명:** 매직 넘버의 목적을 나타내기 위해 명명된 상수를 사용하세요. 예를 들어 `_tokenURIs` 매핑의 값과 관련하여:
* 리터럴 `2`를 사용하는 대신 기존의 명명된 상수 `URIS_PER_TOKEN`을 사용하세요:
```solidity
CryptoartNFT.sol
72:    mapping(uint256 tokenId => string[2] tokenURIs) private _tokenURIs;
358:        returns (uint256, string[2] memory, bool)
361:        string[2] memory uris = _tokenURIs[tokenId];
698:        string[2] memory uris = _tokenURIs[tokenId];
```

* `updateMetadata` 및 `_setTokenURIs`에서 uri를 설정할 때 인덱스에 대해 명명된 상수를 사용하세요:
```solidity
function updateMetadata(uint256 tokenId, string calldata newRedeemableURI, string calldata newNotRedeemableURI)
    external
    onlyOwner
    onlyIfTokenExists(tokenId)
{
    _tokenURIs[tokenId][URI_REDEEMABLE_INDEX] = newRedeemableURI;
    _tokenURIs[tokenId][URI_NOT_REDEEMABLE_INDEX] = newNotRedeemableURI;
    emit MetadataUpdate(tokenId); // ERC4906
}
```

이는 가스도 절약할 수 있습니다. 예를 들어 `pinTokenURI`에서 `_tokenURIs[tokenId].length`를 사용하는 대신 `URIS_PER_TOKEN` 상수는 절대 변경되지 않으므로 이를 사용하세요:
```solidity
function pinTokenURI(uint256 tokenId, uint256 index) external onlyOwner {
    if (index >= URIS_PER_TOKEN) {
        revert Error.Token_IndexOutOfBounds(tokenId, index, URIS_PER_TOKEN - 1);
    }
```

**CryptoArt:**
커밋 [97ef0ad](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/97ef0add6848540e158927092d0a1af820e840fe)에서 수정됨.

**Cyfrin:** 검증됨.


### `_transferToNftReceiver`에서 구식 `onlyTokenOwner` 제거 (Remove obsolete `onlyTokenOwner` from `_transferToNftReceiver`)

**설명:** `_transferToNftReceiver`가 `ERC721Upgradeable::safeTransferFrom`을 호출하므로 `onlyTokenOwner` modifier는 구식이며 비효율적입니다:
* `safeTransferFrom`은 `transferFrom`을 [호출](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v5.1/contracts/token/ERC721/ERC721Upgradeable.sol#L183)합니다.
* `transferFrom`은 `_msgSender()`를 마지막 `auth` 매개변수로 전달하여 `_update`를 [호출](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v5.1/contracts/token/ERC721/ERC721Upgradeable.sol#L166)합니다.
* `auth` 매개변수가 유효하므로 `_update`는 `_checkAuthorized`를 [호출](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v5.1/contracts/token/ERC721/ERC721Upgradeable.sol#L274)합니다.
* `_checkAuthorized`는 호출자가 토큰의 소유자이거나 토큰 소유자가 승인한 사람인지 확인하는 `_isAuthorized`를 [호출](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v5.1/contracts/token/ERC721/ERC721Upgradeable.sol#L215-L238)합니다.

**CryptoArt:**
커밋 [75e179b](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/75e179b3cea8855977a391ace169313053bc2de5)에서 수정됨.

**Cyfrin:** 검증됨.


### `tokenURI`에서 `_tokenURIs[tokenId]` 전체를 `storage`에서 `memory`로 복사하는 것을 방지 (In `tokenURI` avoid copying entire `_tokenURIs[tokenId]` from `storage` into `memory`)

**설명:** `tokenURI`는 "고정된(pinned)" URI 인덱스만 사용하므로 두 토큰 URI를 모두 `storage`에서 `memory`로 복사할 이유가 없습니다. 다음과 같이 단순히 `storage` 참조를 사용하세요:
```diff
    function tokenURI(uint256 tokenId)
        public
        view
        override(ERC721Upgradeable)
        onlyIfTokenExists(tokenId)
        returns (string memory)
    {
-       string[2] memory uris = _tokenURIs[tokenId];
+       string[2] storage uris = _tokenURIs[tokenId];
        string memory uri = uris[_getTokenURIIndex(tokenId)];

        if (bytes(uri).length == 0) {
            revert Error.Token_NoURIFound(tokenId);
        }

        return string.concat(_baseURI(), uri);
    }
```

**CryptoArt:**
커밋 [591fed0](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/591fed0798ab0cd61fe965c9a4d0b3e8461e0f12)에서 수정됨.

**Cyfrin:** 검증됨.


### `burn`은 `tokenURI` 관련 데이터를 삭제하고 `TokenUriUnpinned` 이벤트를 방출해야 함 (`burn` should delete `tokenURI` related data and emit `TokenUriUnpinned` event)

**설명:** `burn` 함수는 `tokenURI` 관련 데이터를 삭제하고 `TokenUriUnpinned` 이벤트를 방출해야 합니다:
```diff
    function burn(uint256 tokenId) public override whenNotPaused {
        // require sender is owner or approved has been removed as the internal burn function already checks this
        ERC2981Upgradeable._resetTokenRoyalty(tokenId);
        ERC721BurnableUpgradeable.burn(tokenId);
        emit Burned(tokenId);
+       emit TokenUriUnpinned(tokenId);
+       delete _tokenURIs[tokenId];
+       delete _pinnedURIIndex[tokenId];
+       delete _hasPinnedTokenURI[tokenId];
    }
```

이것은 소각의 일부로 가스 환불을 제공하고 더 이상 존재하지 않아야 할 토큰 데이터를 제거합니다. 또한 `hasPinnedTokenURI`가 유효한 토큰 ID를 확인하지 않기 때문에 소각된 토큰에 대해 `true`를 반환하는 것을 방지합니다(이를 추적하기 위해 다른 이슈가 생성됨).

**CryptoArt:**
커밋 [b554763](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/b5547630515c8da112db6754a3e25dda1e69b4a7)에서 수정됨.

**Cyfrin:** 검증됨.


### `_batchBurn`에서 중복 ID를 방지하기 위해 중첩된 `for` 루프 대신 오름차순 순서 강제 (To prevent duplicate ids in `_batchBurn`, enforce ascending order instead of nested `for` loops)

**설명:** `_batchBurn`에서 중복 ID를 방지하기 위해, 중첩된 `for` 루프를 사용하는 대신 1개의 `for` 루프만 사용하여 [ID의 오름차순 순서를 강제](https://x.com/DevDacian/status/1734885772829045205)하는 것이 더 효율적입니다.

또한 중복 ID가 있으면 두 번째 `burn` 호출이 revert되므로 중복 ID 확인을 완전히 제거할 수 있습니다. 예를 들어 `test/unit/BurnOperationsTest.t.sol`에 추가된 다음 테스트는:
```solidity
    function test_DoubleBurn() public {
        // Mint a token to user1
        mintNFT(user1, TOKEN_ID, TOKEN_PRICE, TOKEN_PRICE);
        testAssertions.assertTokenOwnership(nft, TOKEN_ID, user1);

        // First burn should succeed
        vm.prank(user1);
        nft.burn(TOKEN_ID);

        // Second burn should revert since token no longer exists
        vm.prank(user1);
        // vm.expectRevert();
        nft.burn(TOKEN_ID);
    }
```

다음과 같은 결과를 초래합니다:
```solidity
    ├─ [4294] TransparentUpgradeableProxy::fallback(1)
    │   ├─ [3940] CryptoartNFT::burn(1) [delegatecall]
    │   │   └─ ← [Revert] ERC721NonexistentToken(1)
    │   └─ ← [Revert] ERC721NonexistentToken(1)
    └─ ← [Revert] ERC721NonexistentToken(1)
```

**CryptoArt:**
커밋 [3c39fb8](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/3c39fb86db6b92424a0cf55c315d0d6284c267bf)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
