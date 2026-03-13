**수석 감사자**

[Dacian](https://x.com/DevDacian)

**보조 감사자**

 


---

# 발견 사항 (Findings)
## 중대 위험 (Critical Risk)


### `remainingBoxesAmount == 0`이 될 만큼 충분한 코드가 연결되면 코드에 연결된 드롭 박스를 공개(reveal)하는 것이 불가능함

**설명:** `DropBox::associateOneTimeCodeToAddress`에서 할당된 코드에 연결된 `boxAmount`가 `remainingBoxesAmount` 저장소에서 차감됩니다:
```solidity
// Decrease the amount of boxes remaining to mint, since they are now associated to be minted
remainingBoxesAmount -= boxAmount;
```

그 후 사용자가 이전에 코드와 연결된 상자를 공개하려고 시도할 때 `DropBox::revealDropBoxes`에서, 모든 코드가 이전에 연결되어 `remainingBoxesAmount == 0`이 된 경우 다음 안전 검사로 인해 트랜잭션이 `NoMoreBoxesToMint` 에러와 함께 잘못 revert 됩니다:
```solidity
// @audit read current `remainingBoxesAmount` from storage
// the box amount linked with this code has already been
// previously deducted in `associateOneTimeCodeToAddress`
uint256 remainingBoxesAmountCache = remainingBoxesAmount;

// @audit since the box amount linked with this code has
// already been deducted from `remainingBoxesAmount`, if
// all codes have been associated, even though not all codes have been
// revealed attempting to reveal any codes will erroneously revert since
// remainingBoxesAmountCache == 0
if (remainingBoxesAmountCache == 0) revert NoMoreBoxesToMint();

// @audit even if the previous check was removed, the transaction would
// still revert here since remainingBoxesAmountCache == 0 but
// oneTimeCodeData.boxAmount > 0
if (remainingBoxesAmountCache < oneTimeCodeData.boxAmount) revert NoMoreBoxesToMint();
```

**파급력:** 모든 코드가 연결되고 나면 코드를 사용하여 상자를 공개하는 것이 불가능합니다. 이로 인해 연결되었지만 아직 공개되지 않은 코드에 연결된 나머지 드롭 박스를 발행(mint)할 수 없게 되므로, 그와 연관된 `EARNM` 토큰을 청구하는 것이 불가능해집니다.

**개념 증명 (PoC):** 먼저 `test/DropBox/DropBox.t.sol`에서 상자 수를 변경하여 상자가 1개만 있도록 합니다:
```diff
    dropBox = new DropBoxMock(
      address(users.operatorOwner),
      "https://api.example.com/",
      "https://api.example.com/contract",
      address(earnmERC20Token),
      address(users.apiAddress),
      [10_000_000, 1_000_000, 100_000, 10_000, 2500, 750],
-     [1, 10, 100, 1000, 2000, 6889],
+     [uint16(1), 0, 0, 0, 0, 0],
      ["Mythical Box", "Legendary Box", "Epic Box", "Rare Box", "Uncommon Box", "Common Box"],
      address(vrfHandler)
    );
```

그런 다음 `test/DropBox/behaviors/revealDropBoxes.t.sol`에 PoC 함수를 추가합니다:
```solidity
  function test_revealDropBoxes_ImpossibleToMintLastBox() public {
    string memory code = "validCode";
    uint32 boxAmount = 1;

    _setupAndAllowReveal(code, users.stranger, boxAmount);
    _mockVrfFulfillment(code, users.stranger, boxAmount);
    _validateMinting(dropBox.mock_getMintedTierAmounts(), boxAmount, code);
  }
```

`forge test --match-test test_revealDropBoxes_ImpossibleToMintLastBox -vvv`로 PoC를 실행합니다.

PoC 스택 추적을 보면 `DropBoxMock::revealDropBoxes`의 `[Revert] NoMoreBoxesToMint()`로 인해 마지막이자 유일한 상자를 공개하는 데 실패했음을 알 수 있습니다.

`DropBox::revealDropBoxes`에서 `remainingBoxesAmountCache` 확인을 주석 처리한 후 PoC를 다시 실행하면 마지막 상자를 공개할 수 있음을 알 수 있습니다.

**권장 완화 조치:** 코드와 연관된 상자는 이미 `DropBox::associateOneTimeCodeToAddress` 내부에서 차감되었으므로 `DropBox::revealDropBoxes`에서 `remainingBoxesAmountCache` 확인을 제거하십시오.

**모드 (Mode):**
커밋 [974fd2c](https://github.com/Earnft/dropbox-smart-contracts/commit/974fd2ce72b9e2d3c0355fdddb303c7c0146a692)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 중급 위험 (Medium Risk)


### `DropBox::tokenURI`가 존재하지 않는 `tokenId`에 대해 데이터를 반환하여 `EIP721`을 위반함

**설명:** `DropBox::tokenURI`는 OpenZeppelin의 기본 구현을 다음 코드로 재정의합니다:
```solidity
function tokenURI(uint256 tokenId) public view override returns (string memory) {
  return string(abi.encodePacked(bytes(_baseTokenURI), Strings.toString(tokenId), ".json"));
}
```

이 구현은 `tokenId`가 존재하는지 확인하지 않으므로 존재하지 않는 `tokenId`에 대한 데이터를 반환합니다. 이는 OpenZeppelin의 [구현](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L88-L93)과 대조적이며 다음과 같이 명시한 [EIP721](https://eips.ethereum.org/EIPS/eip-721)과도 배치됩니다:

```solidity
/// @notice A distinct Uniform Resource Identifier (URI) for a given asset.
/// @dev Throws if `_tokenId` is not a valid NFT. URIs are defined in RFC
///  3986. The URI may point to a JSON file that conforms to the "ERC721
///  Metadata JSON Schema".
function tokenURI(uint256 _tokenId) external view returns (string);
```

**파급력:** 단순히 잘못된 데이터를 반환하는 것 외에도 가장 가능성 있는 부정적인 영향은 NFT 마켓플레이스와의 통합 문제입니다.

**권장 완화 조치:** `tokenId`가 존재하지 않는 경우 revert 하십시오:
```diff
function tokenURI(uint256 tokenId) public view override returns (string memory) {
+ _requireOwned(tokenId);
  return string(abi.encodePacked(bytes(_baseTokenURI), Strings.toString(tokenId), ".json"));
}
```

**모드 (Mode):**
커밋 [2d8c3c0](https://github.com/Earnft/dropbox-smart-contracts/commit/2d8c3c0142f70fae183f1185ea1987a0707711ef)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### Chainlink VRF와 상호 작용할 때 업그레이드 가능하거나 교체 가능한 `VRFHandler` 계약 사용

**설명:** Cyfrin은 개인 감사 클라이언트로부터 Chainlink가 2024년 11월 말에 VRF 2.0을 중단하여 기존 구독에 대해서도 _"VRF 2.0이 작동을 멈출 것"_이라는 사실을 알게 되었습니다.

**파급력:** VRF 2.0에 의존하는 모든 불변(immutable) 계약은 Chainlink가 VRF 2.0을 중단할 때 작동이 중지됩니다(bricked). 향후 Chainlink가 VRF 2.5에 대해서도 같은 조치를 취할 경우 VRF 2.5에 의존하는 불변 계약에도 동일하게 적용됩니다.

**권장 완화 조치:** 불변 계약은 Chainlink VRF와 직접 상호 작용하지 않도록 격리되어야 합니다. 이를 달성하는 한 가지 방법은 불변 계약과 Chainlink VRF 사이의 다리 역할을 하는 별도의 `VRFHandler` 계약을 생성하는 것입니다. 이 계약은 다음을 수행해야 합니다:
* 새로운 `VRFHandler`를 배포할 수 있도록 [VRFConsumerBaseV2Plus](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol)를 사용하는 교체 가능한 불변 계약이거나, [VRFConsumerBaseV2Upgradeable](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Upgradeable.sol)를 사용하는 업그레이드 가능한 계약이어야 합니다.
* 계약 소유자가 불변 계약의 주소를 설정할 수 있도록 허용해야 합니다 (반대로 불변 계약에서도 `VRF Handler`의 주소를 설정할 수 있어야 함).
* 변경할 필요가 없는 API를 불변 계약에 제공해야 합니다.
* 모든 Chainlink VRF API 호출 및 콜백을 처리하고 필요에 따라 무작위성을 불변 계약으로 다시 전달해야 합니다.

**모드 (Mode):**
커밋 [dc3412f](https://github.com/Earnft/dropbox-smart-contracts/commit/dc3412fda8bf988bac579d215c1b7f8f58b973a1)에서 `DropBox`와 Chainlink VRF 사이의 다리 역할을 하는 교체 가능한 불변 `VRFHandler` 계약을 구현하여 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 저위험 (Low Risk)


### 청구되는 상자의 수와 관계없이 트랜잭션당 일정한 `claimFee`로 인해 프로토콜이 예상보다 적거나 많은 수수료를 징수할 수 있음

**설명:** `DropBox::claimDropBoxes`는 사용자가 트랜잭션당 청구할 `boxId` 배열을 전달할 수 있도록 허용하면서 트랜잭션당 일정한 `claimFee`를 부과합니다:
```solidity
// Require that the user sends a fee of "claimFee" amount
if (msg.value != claimFee) revert InvalidClaimFee();
```

이는 다음을 의미합니다 (가스비 제외):
* 1개의 상자를 청구하는 비용은 100개의 상자를 1개의 트랜잭션으로 청구하는 비용과 같습니다.
* 100개의 상자를 하나씩 청구하는 비용은 100개의 상자를 1개의 트랜잭션으로 청구하는 것보다 100배 더 비쌉니다.

**파급력:** 프로토콜은 다음과 같이 수수료를 징수하게 됩니다:
* 사용자가 상자를 일괄(batch) 청구하는 경우 더 적은 수수료
* 사용자가 상자를 하나씩 청구하는 경우 더 많은 수수료

**권장 완화 조치:** 청구되는 상자의 수를 고려하여 트랜잭션당 `claimFee`를 부과하십시오. 예:
```solidity
// Require that the user sends a fee of "claimFee" amount per box
if (msg.value != claimFee * _boxIds.length) revert InvalidClaimFee();
```

**모드 (Mode):**
인지함(Acknowledged); 이는 제품 사양에서 요구된 사항이었습니다.


### `DropBox::supportsInterface`에서 `super`가 올바른 부모 함수를 호출하도록 "가장 기본(base-like)"에서 "가장 파생된(most-derived)" 순서로 상속 순서 사용

**설명:** "가장 기본(base-like)"에서 "가장 파생된(most-derived)" 순서로 상속 순서를 사용하십시오. 이는 좋은 관행으로 간주되며 Solidity가 부모 함수를 오른쪽에서 왼쪽으로 [검색](https://solidity-by-example.org/inheritance/)하기 때문에 중요할 수 있습니다.

`DropBox`는 두 개의 서로 다른 부모 계약에서 다음 함수를 상속하고 재정의하며 `super`를 호출합니다:
```solidity
  function supportsInterface(bytes4 interfaceId) public view virtual override(ERC721C, ERC2981) returns (bool) {
    return super.supportsInterface(interfaceId);
  }
```

현재 순서는 다음과 같습니다:
```solidity
contract DropBox is OwnableBasic, ERC721C, BasicRoyalties, ReentrancyGuard, DropBoxFractalProtocol, IVRFHandlerReceiver {
```

Solidity가 오른쪽에서 왼쪽으로 검색하므로, 상속 순서에서 `BasicRoyalties`가 `ERC721C` 뒤에 나타나기 때문에 `ERC721C::supportsInterface`를 우회하고 대신 `ERC2981::supportsInterface`를 실행합니다.

`ERC2981::supportsInterface`는 결국 `ERC165::supportsInterface`를 [실행](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/common/ERC2981.sol#L22-L55)하지만, `ERC721C::supportsInterface`는 명시적으로 `ERC165::supportsInterface`를 재정의하려고 합니다:
```solidity
    /**
     * @notice Indicates whether the contract implements the specified interface.
     * @dev Overrides supportsInterface in ERC165.
     * @param interfaceId The interface id
     * @return true if the contract implements the specified interface, false otherwise
     */
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return
        interfaceId == type(ICreatorToken).interfaceId ||
        interfaceId == type(ICreatorTokenLegacy).interfaceId ||
        super.supportsInterface(interfaceId);
    }
```

따라서 현재 상속 순서로 인해 `DropBox::supportsInterface`는 `ERC721C::supportsInterface`를 우회하여 `ERC2981::supportsInterface` 및 `ERC165::supportsInterface`를 실행하게 되며, 이는 잘못된 것으로 보입니다.

**권장 완화 조치:** 파일: `DropBox.sol`:
```diff
- contract DropBox is OwnableBasic, ERC721C, BasicRoyalties, ReentrancyGuard, DropBoxFractalProtocol, IVRFHandlerReceiver {
+ contract DropBox is IVRFHandlerReceiver, DropBoxFractalProtocol, ReentrancyGuard, OwnableBasic, BasicRoyalties, ERC721C {

  function supportsInterface(bytes4 interfaceId) public view virtual override(ERC721C, ERC2981) returns (bool) {
-    return super.supportsInterface(interfaceId);
// @audit make it explicit which parent function to call
+   return ERC721C.supportsInterface(interfaceId);
  }
```

파일: `VRFHandler.sol`:
```diff
- contract VRFHandler is VRFConsumerBaseV2Plus, IVRFHandler {
+ contract VRFHandler is IVRFHandler, VRFConsumerBaseV2Plus {
```

**모드 (Mode):**
커밋 [c66de99](https://github.com/Earnft/dropbox-smart-contracts/commit/c66de99a6776bfb5198bee90d7a9c139ec08fc5f)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### `DropBox::_assignTierAndMint`와 `DropBoxFractalProtocol::_determineTier`의 예상 `randomNumber` 불일치 해결

**설명:** `DropBox::_assignTierAndMint`는 0에서 9999 사이의 `randomNumber`를 생성한 다음 이를 입력으로 전달하여 `DropBoxFractalProtocol::_determineTier`를 호출합니다:
```solidity
// Generate a random number between 0 and 9999
uint256 randomNumber = boxSeed % 10_000;

// Determine the tier of the box based on the random number
uint128 tierId = _determineTier(randomNumber, mintedTierAmountCache);
```

하지만 `DropBoxFractalProtocol::_determineTier`에는 `randomNumber`가 0에서 999_999 사이일 것으로 예상하는 주석이 있습니다:
```solidity
/// @notice Function to determine the tier of the box based on the random number.
/// @param randomNumber The random number to use to determine the tier. 0 to 999_999
/// @param mintedTierAmountCache Total cached amount minted for each tier.
/// @return determinedTierId The tier id of the box.
function _determineTier(uint256 randomNumber, ...)
```

이 불일치를 해결하십시오; 후자의 주석이 잘못된 것으로 보입니다.

**모드 (Mode):**
커밋 [4e48b15](https://github.com/Earnft/dropbox-smart-contracts/commit/4e48b15237e0e9f21fa5666595a05fb645d722c6) & [5a3cbf9](https://github.com/Earnft/dropbox-smart-contracts/commit/5a3cbf9b68d1f426bce08984b06b1c92d7675385)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 라이브러리를 생성하지 않는 한 부동(floating) 프라그마 피하기

**설명:** [SWC-103](https://swcregistry.io/docs/SWC-103/)에 따라 라이브러리를 생성하지 않는 한 프라그마의 컴파일러 버전은 고정되어야 합니다. 개발, 테스트 및 배포에 사용할 특정 컴파일러 버전을 선택하십시오. 예:
```diff
- pragma solidity ^0.8.19;
+ pragma solidity 0.8.19;
```

**모드 (Mode):**
커밋 [668011e](https://github.com/Earnft/dropbox-smart-contracts/commit/668011ebd393dd1c224b2236e70fc9d0bf043e77)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 중복된 수수료 전송 코드를 하나의 비공개(private) 함수로 리팩터링

**설명:** `DropBox::claimDropBoxes`와 `revealDropBoxes`는 둘 다 중복된 수수료 전송 코드를 구현하는 `payable` 외부 함수입니다. 이를 1개의 비공개 함수로 리팩터링하십시오:
```solidity
  function _sendFee(uint256 amount) private {
    bool sent;
    address feesReceiver = feesReceiverAddress;
    // Use low level call to send fee to the fee receiver address
    assembly {
      sent := call(gas(), feesReceiver, amount, 0, 0, 0, 0)
    }
    if (!sent) revert Unauthorized();
  }
```

그런 다음 두 외부 함수에서 호출하십시오:
```solidity
_sendFee(msg.value);
```

**모드 (Mode):**
커밋 [327c12b](https://github.com/Earnft/dropbox-smart-contracts/commit/327c12b4b9351fb87df9f0b4706e6ac662cb59cc)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 생성자 입력에 상수 `TIER_IDS_LENGTH` 사용

**설명:** 생성자 입력에 상수 `TIER_IDS_LENGTH`를 사용하십시오:
```solidity
// DropBox:
  constructor(
    uint256 _subscriptionId,
    address _vrfCoordinator,
    address _feesReceiverAddress,
    string memory __baseTokenURI,
    string memory __contractURI,
    address _earmmERC20Token,
    bytes32 _keyHash,
    address _apiAddress,
    uint24[TIER_IDS_LENGTH] memory _tierTokens,
    uint16[TIER_IDS_LENGTH] memory _tierMaxBoxes,
    string[TIER_IDS_LENGTH] memory _tierNames
  )

// DropBoxFractalProtocol:
  constructor(uint24[TIER_IDS_LENGTH] memory tierTokens,
              uint16[TIER_IDS_LENGTH] memory tierMaxBoxes,
              string[TIER_IDS_LENGTH] memory tierNames) {
```

**모드 (Mode):**
커밋 [d8af391](https://github.com/Earnft/dropbox-smart-contracts/commit/d8af391035ca785e2c280de4577f79de2ffda59d)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `VRFHandler`는 `ConfirmedOwner`를 상속받는 `VRFConsumerBaseV2Plus`를 상속받으므로 `ConfirmedOwner`를 상속받지 않아야 함

**설명:** `VRFHandler`는 이미 `ConfirmedOwner`를 [상속](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol#L101)받는 `VRFConsumerBaseV2Plus`를 상속받으므로 `ConfirmedOwner`를 상속받지 않아야 합니다:
```diff
- contract VRFHandler is ConfirmedOwner, VRFConsumerBaseV2Plus, IVRFHandler {
+ contract VRFHandler is VRFConsumerBaseV2Plus, IVRFHandler {
```

**모드 (Mode):**
커밋 [7fe63a3](https://github.com/Earnft/dropbox-smart-contracts/commit/7fe63a32d61adf53ad722a24928f855b68626618)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 전체 네임스페이스를 가져오는 대신 명명된(named) import 사용

**설명:** 전체 네임스페이스를 가져오는 것에 비해 여러 가지 [이점](https://ethereum.stackexchange.com/a/117173)을 제공하므로 명명된 import를 사용하십시오.

파일: `DropBox.sol`
```solidity
// DropBoxFractalProtocol
import {DropBoxFractalProtocol} from "./DropBoxFractalProtocol.sol";

// VRF Handler Interface
import {IVRFHandler} from "./IVRFHandler.sol";
import {IVRFHandlerReceiver} from "./IVRFHandlerReceiver.sol";

// LimitBreak implementations of ERC721C and Programmable Royalties
import {OwnableBasic, OwnablePermissions} from "@limitbreak/creator-token-standards/src/access/OwnableBasic.sol";
import {ERC721C, ERC721OpenZeppelin} from "@limitbreak/creator-token-standards/src/erc721c/ERC721C.sol";
import {BasicRoyalties, ERC2981} from "@limitbreak/creator-token-standards/src/programmable-royalties/BasicRoyalties.sol";

// OpenZeppelin
import {ReentrancyGuard} from "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";
```

파일: `VRFHandler.sol`
```solidity
// VRF Handler Interface
import {IVRFHandler} from "./IVRFHandler.sol";
import {IVRFHandlerReceiver} from "./IVRFHandlerReceiver.sol";

// Chainlink
import {VRFConsumerBaseV2Plus} from "@chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol";
import {IVRFCoordinatorV2Plus} from "@chainlink/contracts/src/v0.8/vrf/dev/interfaces/IVRFCoordinatorV2Plus.sol";
import {VRFV2PlusClient} from "@chainlink/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";
```

**모드 (Mode):**
커밋 [9afe010](https://github.com/Earnft/dropbox-smart-contracts/commit/9afe0109afb01cd30f00393bf00f817c3a39a205)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 명명된 `mapping` 매개변수 사용

**설명:** Solidity 0.8.18은 명명된 `mapping` 매개변수를 [도입](https://soliditylang.org/blog/2023/02/01/solidity-0.8.18-release-announcement/)했습니다. `DropBox`에서 더 명확한 매핑을 위해 이 기능을 사용하십시오:
```solidity
  /// @notice Mappings
  mapping(uint256 boxId => Box boxData) internal boxIdToBox;
  mapping(bytes32 code => bool usedInd) internal oneTimeCodeUsed; // ensure uniqueness of one-time codes
  mapping(bytes32 codeAddressHash => OneTimeCode codeData) internal oneTimeCode; // ensure integrity of one-time code data
  mapping(bytes32 codeAddressHash => uint256[] randomWords) internal oneTimeCodeRandomWords;
  mapping(uint256 vrfRequestId => bytes32 codeAddressHash) internal activeVrfRequests;
  mapping(uint256 vrfRequestId => bool fulfilledInd) internal fulfilledVrfRequests;
```

**모드 (Mode):**
커밋 [8f6a168](https://github.com/Earnft/dropbox-smart-contracts/commit/8f6a168a9ddf17d0b0611c305fa8a0f3069ea56d)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### 기본값으로 초기화하지 말 것

**설명:** 기본값으로 초기화하지 마십시오:

파일: `src/DropBox.sol`
```solidity
142:    isEarnmClaimAllowed = false;
143:    isRevealAllowed = false;
265:      for (uint32 i = 0; i < boxAmount; i++) {
424:    uint256 amountToClaim = 0;
437:    for (uint256 i = 0; i < _boxIds.length; i++) {
478:    for (uint256 i = 0; i < _boxIds.length; i++) {
537:    for (uint256 i = 0; i < _randomWords.length; i++) {
620:    for (uint32 i = 0; i < boxAmount; i++) {
676:    uint256 totalLiability = 0;
784:    for (uint256 i = 0; i < boxIds.length; i++) {
```

**모드 (Mode):**
커밋 [16ecc3b](https://github.com/Earnft/dropbox-smart-contracts/commit/16ecc3ba4ef8463d6b805d8ad95a7fa15c96d0db)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 같은 값을 여러 번 읽을 때 저장소 변수 캐시

**설명:** 같은 값을 여러 번 읽을 때 저장소 변수를 캐시하십시오:

파일: `src/DropBox.sol`
```solidity
232:    if (remainingBoxesAmount == 0) revert NoMoreBoxesToMint();
235:    if (remainingBoxesAmount < boxAmount) revert NoMoreBoxesToMint();

329:    if (remainingBoxesAmount == 0) revert NoMoreBoxesToMint();
354:    if (remainingBoxesAmount < oneTimeCodeData.boxAmount) revert NoMoreBoxesToMint();

357:    if (oneTimeCodeRandomWords[otpHash].length == 0) revert InvalidVrfState();
360:    uint256[] memory randomWords = oneTimeCodeRandomWords[otpHash];
```

**모드 (Mode):**
커밋 [3cfec7f](https://github.com/Earnft/dropbox-smart-contracts/commit/3cfec7fe5d6c912725e8db5284399b7177df58a3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `amountToClaim == 0`인 경우 이전에 revert 되므로 `DropBox::claimDropBoxes`에서 `amountToClaim > 0` 확인 제거

**설명:** `amountToClaim == 0`인 경우 이전에 revert 되므로 `DropBox::claimDropBoxes`에서 `amountToClaim > 0` 확인을 제거하십시오:

```solidity
// [Safety check] In case the amount to claim is 0, revert
if (amountToClaim == 0) revert AmountOfEarnmToClaimIsZero();
// ---------------------------------------------------------------------------------------------

//
//
//
//  Contract Earnm Balance and Safety Checks
// ---------------------------------------------------------------------------------------------
// Get the balance of Earnm ERC20 tokens of the contract
uint256 balance = EARNM.balanceOf(address(this));

// [Safety check] Validate there is more than 0 balance of Earnm ERC20 tokens
// [Safety check] Validate there is enough balance of Earnm ERC20 tokens to claim
// @audit remove `amountToClaim > 0` due to prev check which enforced that
// amountToClaim != 0 and amountToClaim is unsigned so can't be negative
if (!(amountToClaim > 0 && balance > 0 && balance >= amountToClaim)) revert InsufficientEarnmBalance();
```

**모드 (Mode):**
커밋 [e6231a7](https://github.com/Earnft/dropbox-smart-contracts/commit/e6231a756a50ac36c9e10076d947dcc16b5696ea)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 부호 없는 변수는 음수가 될 수 없으므로 `< 0` 확인 제거

**설명:** 부호 없는 변수는 음수가 될 수 없으므로 `< 0` 확인을 제거하십시오:

파일: `src/DropBox.sol`
```solidity
538:      if (_randomWords[i] <= 0) revert InvalidRandomWord();
```

**모드 (Mode):**
커밋 [ccc358a](https://github.com/Earnft/dropbox-smart-contracts/commit/ccc358a88e27ed08c85a37c7770af4dea24f9155)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 0으로 기본 초기화를 피하기 위해 `DropBox::boxIdCounter`를 1로 초기화

**설명:** 0으로 기본 초기화를 피하기 위해 `DropBox::boxIdCounter`를 1로 초기화하십시오:

```diff
- uint256 internal boxIdCounter; // Counter for the box ids, starting from 1 (see _assignTierAndMint function)
+ uint256 internal boxIdCounter = 1; // Counter for the box ids
```

그리고 `_assignTierAndMint`에서 다음 라인을 변경하십시오:

```diff
- uint256 newBoxId = ++boxIdCounter;
+ uint256 newBoxId = boxIdCounter++;
```

**모드 (Mode):**
커밋 [4e2d90b](https://github.com/Earnft/dropbox-smart-contracts/commit/4e2d90b7f18c3bcc3810754941e460bbe6894189)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::claimDropBoxes`에서 더 효율적인 중복 방지를 구현하기 위해 boxId의 오름차순 적용

**설명:** `DropBox::claimDropBoxes`에서 더 효율적인 중복 방지를 구현하기 위해 boxId의 오름차순을 적용하십시오:
```diff
-      for (uint256 j = i + 1; j < _boxIds.length; j++) {
-        if (_boxIds[i] == _boxIds[j]) revert BadRequest();
-      }
+     if(i > 0 && _boxIds[i-1] >= _boxIds[i]) revert BadRequest();
```

**모드 (Mode):**
커밋 [756fc82](https://github.com/Earnft/dropbox-smart-contracts/commit/756fc827eb5265c09e7e1eca8c7a55832e58d1f2)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 외부 함수 매개변수에 대해 `memory`보다 `calldata` 선호

**설명:** 외부 함수 매개변수에 대해 `memory`보다 `calldata`를 선호하십시오:

파일: `src/DropBox.sol`:
```solidity
158:  function removeOneTimeCodeToAddress(string memory code, address allowedAddress) external only(apiAddress) {
203:    string memory code,
308:  function revealDropBoxes(string memory code) external payable nonReentrant {
406:  function claimDropBoxes(uint256[] memory _boxIds) external payable nonReentrant {
781:  function getBoxTierAndBlockTsOfIds(uint256[] memory boxIds) external view returns (uint256[3][] memory) {
798:  function getOneTimeCodeData(address sender, string memory code) external view returns (OneTimeCode memory) {
805:  function getOneTimeCodeToAddress(address sender, string memory code) external view returns (address) {
929:  function setContractURI(string memory __contractURI) external onlyOwner {
939:  function setBaseTokenURI(string memory __baseTokenURI) external onlyOwner {
```

**모드 (Mode):**
커밋 [9776bc3](https://github.com/Earnft/dropbox-smart-contracts/commit/9776bc3f3be227afb45e22356c9887cd4c428b43) & [b0cc24a](https://github.com/Earnft/dropbox-smart-contracts/commit/b0cc24a8450b21c86c92ee8e0056a89d3f0a80b5)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 명명된 반환 변수에 할당하고 명시적 `return` 문 제거 선호

**설명:** 명명된 반환 변수에 할당하고 명시적 `return` 문을 제거하는 것을 [선호](https://x.com/DevDacian/status/1796396988659093968)하십시오.

이는 코드 베이스 대부분에 적용되지만, 명명된 반환 변수를 사용하여 `tokens` 변수와 `return` 문을 제거하도록 `DropBoxFractalProtocol::_calculateAmountToClaim`을 리팩터링하는 방법에 대한 한 가지 예를 제공합니다:

```solidity
  /// @return tokens The amount of Earnm tokens to claim.
  function _calculateAmountToClaim(uint256 tier, uint256 daysPassed) internal view returns (uint256 tokens) {
      if (tier == 6) tokens = TIER_6_TOKENS;
      if (tier == 5) tokens = TIER_5_TOKENS;
      if (tier == 4) tokens = TIER_4_TOKENS;
      if (tier == 3) tokens = TIER_3_TOKENS;
      if (tier == 2) tokens = TIER_2_TOKENS;
      if (tier == 1) tokens = TIER_1_TOKENS;

      // The maximum amount of tokens to claim is the total amount of tokens
      // Which is obtained after 4 years
      if (daysPassed > FOUR_YEARS_IN_DAYS) {
          tokens *= (10 ** EARNM_DECIMALS);
      }
      else {
          tokens = (tokens * (10 ** EARNM_DECIMALS)) * daysPassed / FOUR_YEARS_IN_DAYS;
      }
  }
```

**모드 (Mode):**
커밋 [88c5f55](https://github.com/Earnft/dropbox-smart-contracts/commit/88c5f55d75d69aa2ba66779e27cddf47bddfac14)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `uint128`을 사용하도록 `Box` 구조체를 변경하여 `Box` 당 1개의 스토리지 슬롯 절약

**설명:** 현재 `Box` 구조체는 다음과 같습니다:
```solidity
  struct Box {
    uint256 blockTs;
    uint256 tier;
  }
```

이는 각 `Box`마다 2개의 스토리지 슬롯을 필요로 합니다. 그러나 `blockTs`와 `tier` 모두 `uint256`이 필요하지 않습니다. 둘 다 `uint128`로 변경할 수 있으며, 이렇게 하면 각 `Box`를 1개의 스토리지 슬롯에 압축할 수 있습니다:

```solidity
  struct Box {
    uint128 blockTs;
    uint128 tier;
  }
```

이렇게 하면 상자를 스토리지에서 읽거나 쓸 때 스토리지 읽기/쓰기 횟수가 절반으로 줄어듭니다.

**모드 (Mode):**
커밋 [88c5f55](https://github.com/Earnft/dropbox-smart-contracts/commit/88c5f55d75d69aa2ba66779e27cddf47bddfac14#diff-3c794fa0376b06334bb77ee613047755475945d090c0853efd4cd33ddd57d6d4L9-R10)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 이전 확인이 `true`일 때 후속 확인을 방지하기 위해 순차적 `if` 문에 `else if` 사용

**설명:** 이전 확인이 `true`일 때 후속 확인을 방지하기 위해 순차적 `if` 문에 `else if`를 사용하십시오. 예를 들어 `DropBoxFractalProtocol::_determineTier`에는 다음 코드가 있습니다:
```solidity
if (tierId == 6) maxBoxesPerTier = TIER_6_MAX_BOXES;
if (tierId == 5) maxBoxesPerTier = TIER_5_MAX_BOXES;
if (tierId == 4) maxBoxesPerTier = TIER_4_MAX_BOXES;
if (tierId == 3) maxBoxesPerTier = TIER_3_MAX_BOXES;
if (tierId == 2) maxBoxesPerTier = TIER_2_MAX_BOXES;
if (tierId == 1) maxBoxesPerTier = TIER_1_MAX_BOXES;
```

여기서 첫 번째 조건이 참이더라도 다른 모든 `if` 문은 참이 될 수 없음에도 불구하고 여전히 실행됩니다. 다음과 같이 리팩터링하십시오:
```solidity
if (tierId == 6) maxBoxesPerTier = TIER_6_MAX_BOXES;
else if (tierId == 5) maxBoxesPerTier = TIER_5_MAX_BOXES;
else if (tierId == 4) maxBoxesPerTier = TIER_4_MAX_BOXES;
else if (tierId == 3) maxBoxesPerTier = TIER_3_MAX_BOXES;
else if (tierId == 2) maxBoxesPerTier = TIER_2_MAX_BOXES;
else if (tierId == 1) maxBoxesPerTier = TIER_1_MAX_BOXES;
```

동일한 문제가 확인 순서는 다르지만 동일한 함수의 두 번째 루프와 `DropBoxFractalProtocol::_calculateAmountToClaim`에도 적용됩니다.

**모드 (Mode):**
커밋 [fc8466e](https://github.com/Earnft/dropbox-smart-contracts/commit/fc8466edb470680ef7b7606ca34cdda444d18f33)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 이미 `onlyOwner` 수정자가 있으므로 `DropBox::setDefaultRoyalty` 및 `setTokenRoyalty`에서 `_requireCallerIsContractOwner` 호출 제거

**설명:** 이미 `onlyOwner` 수정자가 있으므로 `DropBox::setDefaultRoyalty` 및 `setTokenRoyalty`에서 `_requireCallerIsContractOwner` 호출을 제거하십시오:
```diff
function setDefaultRoyalty(address receiver, uint96 feeNumerator) external onlyOwner {
-  _requireCallerIsContractOwner();
  _setDefaultRoyalty(receiver, feeNumerator);
}

function setTokenRoyalty(uint256 tokenId, address receiver, uint96 feeNumerator) external onlyOwner {
-  _requireCallerIsContractOwner();
  _setTokenRoyalty(tokenId, receiver, feeNumerator);
}
```

**모드 (Mode):**
커밋 [2939602](https://github.com/Earnft/dropbox-smart-contracts/commit/2939602b20ff093cb21f37b151aa7f5638e109a2)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `randomNumber >= TIER_1_MAX_BOXES`일 때 `DropBoxFractalProtocol::_determineTier`에서 첫 번째 `for` 루프 우회

**설명:** 티어 1은 가장 많은 상자를 가진 가장 낮은 티어이며, 첫 번째 `for` 루프 내부의 이 조건은 `randomNumber >= TIER_1_MAX_BOXES`일 때 첫 번째 `for` 루프가 항상 `tierId` 할당에 실패하도록 보장합니다:
```solidity
// Check if the randomNumber falls within the current tier's range and hasn't exceeded the mint limit.
if ((randomNumber < maxBoxesPerTier) && hasCapacity) return tierId;
```

따라서 `randomNumber >= TIER_1_MAX_BOXES`일 때 첫 번째 `for` 루프에 진입할 이유가 없습니다; 첫 번째 `for` 루프를 건너뛰고 바로 두 번째 루프로 이동하십시오:
```diff
  function _determineTier(
    uint256 randomNumber,
    uint256[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache
  )
    internal
    view
    returns (uint128 /* tierId, function returns before the end of the execution */ )
  {
-    // Iterate backwards from the highest tier to the lowest.
-    for (uint128 tierId = TIER_IDS_LENGTH; tierId > 0; tierId--) {

+    // impossible for first loop to allocate tierId when randomNumber is
+    // >= lowest tier max boxes, so in that case skip to second loop
+    if(randomNumber < TIER_1_MAX_BOXES) {
+      // Iterate backwards from the highest tier to the lowest.
+      for (uint128 tierId = TIER_IDS_LENGTH; tierId > 0; tierId--) {
```

**모드 (Mode):**
커밋 [a3d7641](https://github.com/Earnft/dropbox-smart-contracts/commit/a3d7641b472130cadc8502426ee49dbbc4d947d2)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::claimDropBoxes`에서 중복된 `tierId` 유효성 검사 제거

**설명:** `DropBox::claimDropBoxes`는 `DropBoxFractalProtocol::_calculateVestingPeriodPerBox`를 호출하기 전에 다음 확인을 수행합니다:
```solidity
// [Safety check] Validate that the box tier is valid
if (!(box.tier > 0 && box.tier <= TIER_IDS_LENGTH)) revert InvalidTierId();
```

그러나 `DropBoxFractalProtocol::_calculateVestingPeriodPerBox`가 동일한 확인을 수행하므로, 이 확인은 중복되며 `DropBox::claimDropBoxes`에서 제거되어야 합니다.

**모드 (Mode):**
커밋 [7ebd568](https://github.com/Earnft/dropbox-smart-contracts/commit/7ebd5682e5263ce42a0432548e4b7ed5b801b7be)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::claimDropBoxes`에서 중복된 `balance > 0` 확인 제거

**설명:** `DropBox::claimDropBoxes`에는 다음 확인이 있습니다:
```solidity
if (!(balance > 0 && balance >= amountToClaim)) revert InsufficientEarnmBalance();
```

그러나 `amountToClaim == 0`인 경우 함수가 이미 revert 되었으므로 `balance > 0` 구성 요소는 중복됩니다. 따라서 확인을 다음과 같이 단순화할 수 있습니다:
```solidity
if (!(balance >= amountToClaim)) revert InsufficientEarnmBalance();
```

이것은 최종 `!` 부정 연산자의 필요성을 제거하여 다음과 같이 더 단순화할 수 있습니다:
```solidity
if (balance < amountToClaim) revert InsufficientEarnmBalance();
```

**모드 (Mode):**
커밋 [a1e6435](https://github.com/Earnft/dropbox-smart-contracts/commit/a1e6435e922db96194cd74490600e23f8a56e20f)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 더 빨리 실패하고 최종 부울 결과에 대한 부정 연산자를 제거하도록 확인 단순화

**설명:** 코드의 많은 확인이 다음 패턴을 사용합니다:
```solidity
if( !(check_1 && check_2 && check_3) ) revert Error();
```

종종 이러한 확인은 다음과 같이 다시 작성하여 효율성을 높이고 단순화할 수 있습니다:
* `&&` 대신 `||`를 사용하여 확인이 더 빨리 실패하도록 하고 후속 구성 요소의 평가를 피함
* 최종 `!` 부정 연산자의 필요성 제거

파일: `src/DropBox.sol`
```diff
// L218
- if (!(boxAmount > 0 && boxAmount <= MAX_BOX_AMOUNT_PER_REVEAL)) revert InvalidBoxAmount();
+ if(boxAmount == 0 || boxAmount > MAX_BOX_AMOUNT_PER_REVEAL) revert InvalidBoxAmount();

// L347
- if (!(oneTimeCodeData.boxAmount > 0 && oneTimeCodeData.boxAmount <= MAX_BOX_AMOUNT_PER_REVEAL)) {
+ if (oneTimeCodeData.boxAmount == 0 || oneTimeCodeData.boxAmount > MAX_BOX_AMOUNT_PER_REVEAL) {

// L417
- if (!(_boxIds.length > 0)) revert Unauthorized();
+ if (boxIds.length == 0) revert Unauthorized();

// L442
- if (!(ownerOf(_boxIds[i]) == msg.sender)) revert Unauthorized();
+ if (ownerOf(_boxIds[i]) != msg.sender) revert Unauthorized();
```

파일: `src/DropBoxFractalProtocol.sol`
```diff
// L170
- if (!(boxTierId > 0 && boxTierId <= tierIdsLength)) revert InvalidTierId();
+ if (boxTierId == 0 || boxTierId > tierIdsLength) revert InvalidTierId();

// L173
- if (!(boxMintedTimestamp <= block.timestamp && boxMintedTimestamp > 0)) revert InvalidBlockTimestamp();
+ if (boxMintedTimestamp > block.timestamp || boxMintedTimestamp == 0) revert InvalidBlockTimestamp();
```

**모드 (Mode):**
커밋 [96773b9](https://github.com/Earnft/dropbox-smart-contracts/commit/96773b972fafd3086a18ad03e65c7199695b5577)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBoxFractalProtocol::_determineTier`에 대해 단순화되고 더 효율적인 구현 사용

**설명:** `DropBoxFractalProtocol::_determineTier`에 대해 다음과 같이 단순화되고 더 효율적인 구현을 사용하십시오:
```solidity
  function _determineTier(
    uint256 randomNumber,
    uint256[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache
  )
    internal
    view
    returns (uint128 /* tierId, function returns before the end of the execution */ )
  {
    // if the randomNumber is smaller than TIER_1_MAX_BOXES, first attempt
    // tier selection which prioritizes the most valuable tiers for selection where:
    // 1) randomNumber falls within one or more tiers AND
    // 2) the tier(s) have not been exhausted
    if(randomNumber < TIER_1_MAX_BOXES) {
      if(randomNumber < TIER_6_MAX_BOXES && mintedTierAmountCache[6] < TIER_6_MAX_BOXES) return 6;
      if(randomNumber < TIER_5_MAX_BOXES && mintedTierAmountCache[5] < TIER_5_MAX_BOXES) return 5;
      if(randomNumber < TIER_4_MAX_BOXES && mintedTierAmountCache[4] < TIER_4_MAX_BOXES) return 4;
      if(randomNumber < TIER_3_MAX_BOXES && mintedTierAmountCache[3] < TIER_3_MAX_BOXES) return 3;
      if(randomNumber < TIER_2_MAX_BOXES && mintedTierAmountCache[2] < TIER_2_MAX_BOXES) return 2;
      if(randomNumber < TIER_1_MAX_BOXES && mintedTierAmountCache[1] < TIER_1_MAX_BOXES) return 1;
    }

    // if we get here it means that either:
    // 1) randomNumber >= TIER_1_MAX_BOXES OR
    // 2) the tier(s) randomNumber fell into had been exhausted
    //
    // in this case we attempt to allocate based on what is available
    // prioritizing from the least valuable tiers
    if(mintedTierAmountCache[1] < TIER_1_MAX_BOXES) return 1;
    if(mintedTierAmountCache[2] < TIER_2_MAX_BOXES) return 2;
    if(mintedTierAmountCache[3] < TIER_3_MAX_BOXES) return 3;
    if(mintedTierAmountCache[4] < TIER_4_MAX_BOXES) return 4;
    if(mintedTierAmountCache[5] < TIER_5_MAX_BOXES) return 5;
    if(mintedTierAmountCache[6] < TIER_6_MAX_BOXES) return 6;

    // if we get here it means that all tiers have been exhausted
    revert NoMoreBoxesToMint();
  }
```

**모드 (Mode):**
커밋 [0e3410b](https://github.com/Earnft/dropbox-smart-contracts/commit/0e3410b6bdf8150e3ed613af2713747cd93084f8)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::_assignTierAndMint`에서 스토리지 위치 `boxIdCounter` 및 `mintedTierAmount`에 대해 한 번만 읽고 쓰기

**설명:** `DropBox::_assignTierAndMint`에서 `boxIdCounter`와 `mintedTierAmount`는 루프 전에 캐시할 수 있으며, 루프 중에 캐시된 버전을 사용하고 마지막에 한 번 업데이트할 수 있습니다:
```solidity
  function _assignTierAndMint(
    uint32 boxAmount,
    uint256[] memory randomNumbers,
    address claimer,
    bytes memory code
  )
    internal
  {
    uint256[] memory boxIdsToEmit = new uint256[](boxAmount);

    // [Gas] Cache mintedTierAmount, update cache during the loop and record
    // which tiers were updated. At the end of the loop write once to storage
    // only for tiers which were updated. Also prevents `_determineTier` from
    // re-reading storage multiple times
    bool[TIER_IDS_ARRAY_LEN] memory tierChangedInd;
    uint256[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache = mintedTierAmount;

    // [Gas] cache the current boxId, use cache during the loop then write once
    // at the end; this results in only 1 storage read and 1 storage write for `boxIdCounter`
    uint256 boxIdCounterCache = boxIdCounter;

    // For each box to mint
    for (uint32 i; i < boxAmount; i++) {
      // Generate a new box id, starting from 1
      uint256 newBoxId = boxIdCounterCache++;

      // The box seed is directly derived from the random number without any
      // additional values like the box id, the claimer address, or the block timestamp.
      // This is made like this to ensure the tier assignation of the box is purely random
      // and the tier that the box is assigned to, won't change after the randomness is fulfilled.
      // The only way to predict the seed is to know the random number.
      // Random number validation is done in the _fulfillRandomWords function.
      uint256 boxSeed = uint256(keccak256(abi.encode(randomNumbers[i])));

      // Generate a random number between 0 and TOTAL_MAX_BOXES - 1 using the pure random seed
      // Modulo ensures that the random number is within the range of [0, TOTAL_MAX_BOXES) (upper bound exclusive),
      // providing a uniform distribution across the possible range. This ensures that each tier has a
      // probability of being selected proportional to its maximum box count. The randomness is normalized
      // to fit within the predefined total boxes, maintaining the intended probability distribution for each tier.
      //
      // Example with the following setup:
      // _tierMaxBoxes -> [1, 10, 100, 1000, 2000, 6889]
      // _tierNames -> ["Mythical Box", "Legendary Box", "Epic Box", "Rare Box", "Uncommon Box", "Common Box"]
      //
      // The `TOTAL_MAX_BOXES` is the sum of `_tierMaxBoxes`, which is 10_000.
      // The probability distribution is as follows:
      //
      // Mythical Box:   1    / 10_000  chance.
      // Legendary Box:  10   / 10_000  chance.
      // Epic Box:       100  / 10_000  chance.
      // Rare Box:       1000 / 10_000  chance.
      // Uncommon Box:   2000 / 10_000  chance.
      // Common Box:     6889 / 10_000  chance.
      //
      // By using the modulo operation with TOTAL_MAX_BOXES, the random number's probability
      // to land in any given tier is determined by the number of max boxes available for that tier
      uint256 randomNumber = boxSeed % TOTAL_MAX_BOXES;

      // Determine the tier of the box based on the random number
      uint128 tierId = _determineTier(randomNumber, mintedTierAmountCache);

      // Increment the cached amount for this tier and mark this tier as changed
      ++mintedTierAmountCache[tierId];
      tierChangedInd[tierId] = true;

      // [Memory] Add the box id to the boxes ids array
      boxIdsToEmit[i] = newBoxId;

      // Save the box to the boxes mappings
      _saveBox(newBoxId, tierId);

      // Mint the box; explicitly not using `_safeMint` to prevent
      // re-entrancy issues
      _mint(claimer, newBoxId);
    }

    // update storage for boxIdCounter
    boxIdCounter = boxIdCounterCache;

    // update storage for mintedTierAmount only for tiers which were changed
    // hence each tier from mintedTierAmount storage is read only once and only
    // tiers which changed are written to storage only once
    for(uint128 tierId = 1; tierId<TIER_IDS_ARRAY_LEN; tierId++) {
      if(tierChangedInd[tierId]) {
        mintedTierAmount[tierId] = mintedTierAmountCache[tierId];
      }
    }

    // Emit an event indicating the boxes have been minted
    emit BoxesMinted(claimer, boxIdsToEmit, code);
  }
```

제공된 코드에는 루프 내부에 몇 가지 다른 유용한 변경 사항도 있습니다:
* `mintedTierAmountCache`와 `boxIdsToEmit`은 `_saveBox`와 `_mint`를 호출하기 전에 변경됩니다 (Interactions 전 Effects)
* `_safeMint` 대신 `_mint`가 명시적으로 사용되고 있음을 알리는 주석 추가

또한 `mintedTierAmount`에 대한 단위 테스트가 부족하여 코드 라인을 주석 처리해도 모든 테스트가 통과하는 것을 발견했으므로, 이 영역에 대한 단위 테스트를 더 추가했습니다:

파일: `test/mocks/DropBoxMock.sol`
```solidity
// couple of helper functions
  /// @dev [Test purposes] Test get the minted tier amounts
  function mock_getMintedTierAmount(uint128 tierId) public view returns (uint256) {
    return mintedTierAmount[tierId];
  }
  function mock_getMintedTierAmounts() public view returns (uint256[TIER_IDS_ARRAY_LEN] memory) {
    return mintedTierAmount;
  }
```

파일: `test/DropBox/behaviors/revealDropBox/revealDropBoxes.t.sol`
```solidity
// firstly over-write the `_validateMinting` function to perform additional verification
  function _validateMinting(
    uint256[TIER_IDS_ARRAY_LEN] memory prevMintedTierAmounts,
    uint32 boxAmount,
    string memory code) internal
  {
    uint256[] memory expectedBoxIds = new uint256[](boxAmount);
    for (uint256 i; i < boxAmount; i++) {
      expectedBoxIds[i] = i + 1;
    }

    vm.expectEmit(address(dropBox));
    emit DropBoxFractalProtocolLib.BoxesMinted(users.stranger, expectedBoxIds, abi.encode(code));
    _fundAndReveal(users.stranger, code, 1 ether);

    (uint256 blockTs, uint256 tier, address owner) = dropBox.getBox(1);
    assertEq(owner, users.stranger);
    assertGt(tier, 0);
    assertLt(tier, 7);

    // iterate over the boxes to get their types and update the
    // previous minted tier amounts
    for (uint256 i; i < expectedBoxIds.length; i++) {
      (,uint128 boxTierId, address owner) = dropBox.getBox(expectedBoxIds[i]);
      assertEq(owner, users.stranger);

      // increment previously minted tierId for this tier
      ++prevMintedTierAmounts[boxTierId];
    }

    // verify DropBox::mintedTierAmount has been correctly changed
    for(uint128 tierId = 1; tierId < TIER_IDS_ARRAY_LEN; tierId++) {
      assertEq(dropBox.mock_getMintedTierAmount(tierId), prevMintedTierAmounts[tierId]);
    }
  }

// next in the tests where _validateMinting is called, simply replace the call with this line
_validateMinting(dropBox.mock_getMintedTierAmounts(), boxAmount, code);
```

**모드 (Mode):**
커밋 [5badac3](https://github.com/Earnft/dropbox-smart-contracts/commit/5badac3a5a6de31d0ad84decdefd0e4938e07be3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::associateOneTimeCodeToAddress`에서 반복적인 동일 계산을 방지하기 위해 `_getOneTimeCodeHash` 결과 캐시

**설명:** `DropBox::associateOneTimeCodeToAddress`는 현재 `_getOneTimeCodeHash(code, allowedAddress)`를 두 번 계산합니다:
```solidity
    // [Safety check] Validate that the one-time code is not already associated to an address
    // @audit 1) calculating _getOneTimeCodeHash for first time
    address claimerAddress = oneTimeCode[_getOneTimeCodeHash(code, allowedAddress)].claimer;
    if (claimerAddress != address(0x0)) revert OneTimeCodeAlreadyAssociated(claimerAddress);

    // [Safety check] Validate that the one-time code has not been used
    if (oneTimeCodeUsed[keccak256(bytes(code))]) revert OneTimeCodeAlreadyUsed();

    uint256 remainingBoxesAmountCache = remainingBoxesAmount;

    // [Safety check] Validate that there are still balance of boxes to mint
    if (remainingBoxesAmountCache == 0) revert NoMoreBoxesToMint();

    // [Safety check] Validate that the remaining boxes to mint are greater or equal to the box amount
    if (remainingBoxesAmountCache < boxAmount) revert NoMoreBoxesToMint();
    // ---------------------------------------------------------------------------------------------

    // Decrease the amount of boxes remaining to mint, since they are now associated to be minted
    remainingBoxesAmount -= boxAmount;

    // Generate the hash of the code and the sender address
    // @audit 2) calculating _getOneTimeCodeHash again with identical inputs
    bytes32 otpHash = _getOneTimeCodeHash(code, allowedAddress);
```

`code`와 `allowedAddress`가 변경되지 않으므로 다시 계산할 이유가 없습니다. 한 번만 수행하고 캐시된 결과를 사용하십시오:
```diff
+   // Generate the hash of the code and the sender address
+   bytes32 otpHash = _getOneTimeCodeHash(code, allowedAddress);

    // [Safety check] Validate that the one-time code is not already associated to an address
-   address claimerAddress = oneTimeCode[_getOneTimeCodeHash(code, allowedAddress)].claimer;
+   address claimerAddress = oneTimeCode[otpHash].claimer;
    if (claimerAddress != address(0x0)) revert OneTimeCodeAlreadyAssociated(claimerAddress);

    // [Safety check] Validate that the one-time code has not been used
    if (oneTimeCodeUsed[keccak256(bytes(code))]) revert OneTimeCodeAlreadyUsed();

    uint256 remainingBoxesAmountCache = remainingBoxesAmount;

    // [Safety check] Validate that there are still balance of boxes to mint
    if (remainingBoxesAmountCache == 0) revert NoMoreBoxesToMint();

    // [Safety check] Validate that the remaining boxes to mint are greater or equal to the box amount
    if (remainingBoxesAmountCache < boxAmount) revert NoMoreBoxesToMint();
    // ---------------------------------------------------------------------------------------------

    // Decrease the amount of boxes remaining to mint, since they are now associated to be minted
    remainingBoxesAmount -= boxAmount;

-   // Generate the hash of the code and the sender address
-   bytes32 otpHash = _getOneTimeCodeHash(code, allowedAddress);
```

**모드 (Mode):**
커밋 [1c983c2](https://github.com/Earnft/dropbox-smart-contracts/commit/1c983c2597be511b22fad5a33ee45bcce000dada)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::claimDropBoxes`에서 두 번째 `for` 루프 제거 및 삭제된 상자당 1회의 스토리지 읽기 감소

**설명:** `DropBox::claimDropBoxes` 내부의 첫 번째 `for` 루프는 다음과 같이 작성할 수 있어 두 번째 `for` 루프를 제거할 수 있습니다. 즉, 입력 `_boxIds`에 대해 한 번만 반복하면 됩니다:
```solidity
    for (uint256 i; i < _boxIds.length; i++) {
      // [Safety check] Duplicate values are not allowed
      if (i > 0 && _boxIds[i - 1] >= _boxIds[i]) revert BadRequest();

      // [Safety check] Validate that the box owner is the sender
      if (ownerOf(_boxIds[i]) != msg.sender) revert Unauthorized();

      // Copy the box from the boxes mapping
      Box memory box = boxIdToBox[_boxIds[i]];

      // Increment the amount of Earnm tokens to send to the sender
      amountToClaim += _calculateVestingPeriodPerBox(box.tier, box.blockTs, TIER_IDS_LENGTH);

      // Delete the box from the boxes mappings
      _deleteBox(_boxIds[i], box.tier);

      // Burn the ERC721 token corresponding to the box
      _burn(_boxIds[i]);
    }
```

위의 코드는 삭제된 상자당 1회의 스토리지 읽기를 절약하는 수정된 `_deleteBox` 함수를 필요로 합니다:
```solidity
  /// @param _boxTierId the tier id of the box to delete
  function _deleteBox(uint256 _boxId, uint128 _boxTierId) internal {
    // Delete from address -> _boxId mapping
    delete boxIdToBox[_boxId];

    // Decrease the unclaimed amount of boxes minted per tier
    unclaimedTierAmount[_boxTierId]--;
  }
```

이로 인해 추가 `_boxTierId` 매개변수를 전달하기 위해 테스트 스위트에 2가지 변경이 필요합니다:
```solidity
// mocks/DropBoxMock.sol
  function mock_deleteBox(uint256 boxId, uint128 tierId) public {
    _deleteBox(boxId, tierId);
  }

// DropBox/behaviors/boxStorage/deleteBox.t.sol
dropBox.mock_deleteBox(boxId, tierId);
```

**모드 (Mode):**
커밋 [541f929](https://github.com/Earnft/dropbox-smart-contracts/commit/541f9297b5e0f072e8140d97f23031971e6fff21)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `DropBox::claimDropBoxes`에서 변경된 티어당 스토리지 위치 `unclaimedTierAmount`에 대해 한 번만 읽고 쓰기

**설명:** `DropBox::claimDropBoxes`는 다음과 같이 `unclaimedTierAmount` 스토리지 위치에 대한 스토리지 읽기/쓰기 횟수를 획기적으로 줄이도록 다시 작성할 수 있습니다:
```solidity
  function claimDropBoxes(uint256[] calldata _boxIds) external payable nonReentrant {
    //
    //
    //
    //  Request Checks
    // ---------------------------------------------------------------------------------------------
    // [Safety check] Only allowed to claim if the isEarnmClaimAllowed variable is true
    // Can be modified by the owner as a safety measure
    if (!(isEarnmClaimAllowed)) revert Unauthorized();

    // [Safety check] Validate that the boxIds array is not empty
    if (_boxIds.length == 0) revert Unauthorized();

    // Require that the user sends a fee of "claimFee" amount
    if (msg.value != claimFee) revert InvalidClaimFee();
    // ---------------------------------------------------------------------------------------------

    // Amount of Earnm tokens to claim
    uint256 amountToClaim;

    // [Gas] Cache unclaimedTierAmount and update cache during the loop then
    // write to storage at the end to prevent multiple update storage writes
    uint256[TIER_IDS_ARRAY_LEN] memory unclaimedTierAmountCache;

    // [Gas] Cache the tier changed indicator to update storage only for tiers which changed
    bool[TIER_IDS_ARRAY_LEN] memory tierChangedInd;

    //
    //
    //
    //  Earnm Calculation and Safety Checks
    // ---------------------------------------------------------------------------------------------
    // Iterate over box IDs to calculate rewards and validate ownership and uniqueness.
    // - Validate that the sender is the owner of the boxes
    // - Validate that the sender has enough boxes to claim
    // - Validate that the box ids are valid
    // - Validate that the box ids are not duplicated
    // - Calculate the amount of Earnm tokens to claim given the box ids
    for (uint256 i; i < _boxIds.length; i++) {
      // [Safety check] Duplicate values are not allowed
      if (i > 0 && _boxIds[i - 1] >= _boxIds[i]) revert BadRequest();

      // [Safety check] Validate that the box owner is the sender
      if (ownerOf(_boxIds[i]) != msg.sender) revert Unauthorized();

      // Copy the box from the boxes mappings
      Box memory box = boxIdToBox[_boxIds[i]];

      // Increment the amount of Earnm tokens to send to the sender
      amountToClaim += _calculateVestingPeriodPerBox(box.tier, box.blockTs, TIER_IDS_LENGTH);

      // if the unclaimed cache hasn't been updated for this tier type
      if(!tierChangedInd[box.tier]) {
          // then set it as updated
          tierChangedInd[box.tier] = true;

          // cache the current amount once from storage
          unclaimedTierAmountCache[box.tier] = unclaimedTierAmount[box.tier];
      }

      // decrement the unclaimed cached amount for this tier
      --unclaimedTierAmountCache[box.tier];

      // Delete the box from the boxIdToBox mapping
      delete boxIdToBox[_boxIds[i]];

      // Burn the ERC721 token corresponding to the deleted box
      _burn(_boxIds[i]);
    }

    // [Safety check] In case the amount to claim is 0, revert
    if (amountToClaim == 0) revert AmountOfEarnmToClaimIsZero();

    // Update tier storage only for tiers which were changed
    for (uint128 tierId = 1; tierId < TIER_IDS_ARRAY_LEN; tierId++) {
      if (tierChangedInd[tierId]) unclaimedTierAmount[tierId] = unclaimedTierAmountCache[tierId];
    }

    // ---------------------------------------------------------------------------------------------

    //
    //
    //
    //  Contract Earnm Balance and Safety Checks
    // ---------------------------------------------------------------------------------------------
    // Get the balance of Earnm ERC20 tokens of the contract
    uint256 balance = EARNM.balanceOf(address(this));

    // [Safety check] Validate that the contract has enough Earnm tokens to cover the claim
    if (balance < amountToClaim) revert InsufficientEarnmBalance();
    // ---------------------------------------------------------------------------------------------

    // Emit an event indicating the boxes have been claimed
    emit BoxesClaimed(msg.sender, _boxIds, amountToClaim);

    // Transfer the tokens to the sender
    EARNM.safeTransfer(msg.sender, amountToClaim);
    // ---------------------------------------------------------------------------------------------

    //
    //
    //
    //  Claim Fee Transfer
    // ---------------------------------------------------------------------------------------------
    _sendFee(msg.value);
    // ---------------------------------------------------------------------------------------------
  }
```

이 버전에서는 `DropBox::_deleteBox` 함수가 더 이상 필요하지 않으므로 `DropBoxMock.sol`의 관련 테스트 파일 및 도우미 함수와 함께 삭제해야 합니다.

또한 변이 테스트(mutation testing)를 통해 청구 프로세스 중에 발생하는 `unclaimedTierAmount` 및 `boxIdToBox`에 대한 스토리지 업데이트가 기존 테스트 스위트에 의해 검증되지 않았음을 알게 되었으므로, 이를 커버하기 위해 테스트 스위트에 새로운 기능을 추가했습니다:

파일: `test/mocks/DropBoxMock.sol`
```solidity
// new helper function
  /// @dev [Test purposes] Test get all the unclaimed tier amounts
  function mock_getUnclaimedTierAmounts() public view returns (uint256[TIER_IDS_ARRAY_LEN] memory) {
    return unclaimedTierAmount;
  }
```

파일: `test/DropBox/behaviors/claimDropBox/claimDropBox.t.sol`
```solidity
// updated this helper function to return the expected unclaimed amounts
  function _calculateExpectedClaimAmount(uint256[] memory boxIds) internal view
  returns (uint256 totalClaimAmount, uint256[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts) {
    // first load the existing unclaimed tier amounts
    expectedUnclaimedTierAmounts = dropBox.mock_getUnclaimedTierAmounts();

    // iterate through the boxes that will be claimed
    for (uint256 i; i < boxIds.length; i++) {
      // get timestamp & tier
      (uint128 blockTs, uint128 tier,) = dropBox.getBox(boxIds[i]);

      // calculate expected token claim amount
      totalClaimAmount += dropBox.mock_calculateVestingPeriodPerBox(tier, blockTs, TIER_IDS_LENGTH);

      // update expected unclaimed tier amount
      --expectedUnclaimedTierAmounts[tier];
    }
  }

// updated this helper function to validate the expected amounts
  function _validateClaimedBoxes(
    uint256[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts,
    uint256[] memory boxIds,
    uint256 expectedAmount,
    address claimer) internal
  {
    for (uint256 i; i < boxIds.length; i++) {
      // get info for deleted box
      (uint128 blockTs, uint128 tier, address owner) = dropBox.getBox(boxIds[i]);

      // box should no longer be owned
      assertEq(owner, address(0));

      // box should be deleted from boxIdToBox mapping
      assertEq(blockTs, 0);
      assertEq(tier, 0);
    }

    // verify DropBox::unclaimedTierAmount has been correctly decreased
    for(uint128 tierId = 1; tierId < TIER_IDS_ARRAY_LEN; tierId++) {
      assertEq(dropBox.mock_getUnclaimedTierAmount(tierId), expectedUnclaimedTierAmounts[tierId]);
    }

    // verify expected earnm contract balance after claims processed
    uint256 earnmBalance = earnmERC20Token.balanceOf(claimer);
    assertEq(earnmBalance, expectedAmount);

    emit DropBoxFractalProtocolLib.BoxesClaimed(claimer, boxIds, expectedAmount);
  }

// updated 2 test functions to use the new updated helper functions
  function test_claimDropBoxes_valid() public {
    string memory code = "validCode";
    address allowedAddress = users.stranger;
    uint32 boxAmount = 5;
    uint256[] memory boxIds = new uint256[](boxAmount);
    uint256 earnmAmount = 1000 * 1e18;

    _setupClaimEnvironment(code, allowedAddress, boxAmount, boxIds, earnmAmount);

    (uint256 expectedAmount, uint256[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts)
      = _calculateExpectedClaimAmount(boxIds);

    _fundAndClaim(allowedAddress, boxIds, 1 ether);
    _validateClaimedBoxes(expectedUnclaimedTierAmounts, boxIds, expectedAmount, allowedAddress);
  }

  function test_claimDropBoxes_ClaimAndBurn() public {
    string memory code = "validCode";
    address allowedAddress = users.stranger;
    uint32 boxAmount = 5;
    uint256[] memory boxIds = new uint256[](boxAmount);
    uint256 earnmAmount = 1000 * 1e18;

    _setupClaimEnvironment(code, allowedAddress, boxAmount, boxIds, earnmAmount);
    (uint256 expectedAmount, uint256[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts)
      = _calculateExpectedClaimAmount(boxIds);

    _fundAndClaim(allowedAddress, boxIds, 1 ether);
    _validateClaimedBoxes(expectedUnclaimedTierAmounts, boxIds, expectedAmount, allowedAddress);
  }
```

**모드 (Mode):**
커밋 [82a2365](https://github.com/Earnft/dropbox-smart-contracts/commit/82a23652e1f806cfc10bafce389c8ef31ecf020a).

**Cyfrin:** 확인됨.


### `DropBox::_assignTierAndMint`에서 변경된 티어당 스토리지 위치 `unclaimedTierAmount`에 대해 한 번만 읽고 쓰기

**설명:** `DropBox::_assignTierAndMint`는 다음과 같이 `unclaimedTierAmount` 스토리지 위치에 대한 스토리지 읽기/쓰기 횟수를 획기적으로 줄이도록 다시 작성할 수 있습니다:
```solidity
  function _assignTierAndMint(
    uint32 boxAmount,
    uint256[] memory randomNumbers,
    address claimer,
    bytes memory code
  )
    internal
  {
    uint256[] memory boxIdsToEmit = new uint256[](boxAmount);

    // [Gas] Cache mintedTierAmount and update cache during the loop to prevent
    // _determineTier() from re-reading it from storage multiple times
    uint32[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache = mintedTierAmount;

    // [Gas] Cache unclaimedTierAmount and update cache during the loop then
    // write to storage at the end to prevent multiple update storage writes
    uint256[TIER_IDS_ARRAY_LEN] memory unclaimedTierAmountCache = unclaimedTierAmount;

    // [Gas] Cache the tier changed indicator to update storage only for tiers which changed
    bool[TIER_IDS_ARRAY_LEN] memory tierChangedInd;

    // [Gas] cache the current boxId, use cache during the loop then write once
    // at the end; this results in only 1 storage read and 1 storage write for `boxIdCounter`
    uint32 boxIdCounterCache = boxIdCounter;

    // For each box to mint
    for (uint32 i; i < boxAmount; i++) {
      // [Memory] Add the box id to the boxes ids array
      boxIdsToEmit[i] = boxIdCounterCache++;

      // The box seed is directly derived from the random number without any
      // additional values like the box id, the claimer address, or the block timestamp.
      // This is made like this to ensure the tier assignation of the box is purely random
      // and the tier that the box is assigned to, won't change after the randomness is fulfilled.
      // The only way to predict the seed is to know the random number.
      // Random number validation is done in the _fulfillRandomWords function.
      uint256 boxSeed = uint256(keccak256(abi.encode(randomNumbers[i])));

      // Generate a random number between 0 and TOTAL_MAX_BOXES - 1 using the pure random seed
      // Modulo ensures that the random number is within the range of [0, TOTAL_MAX_BOXES) (upper bound exclusive),
      // providing a uniform distribution across the possible range. This ensures that each tier has a
      // probability of being selected proportional to its maximum box count. The randomness is normalized
      // to fit within the predefined total boxes, maintaining the intended probability distribution for each tier.
      //
      // Example with the following setup:
      // _tierMaxBoxes -> [1, 10, 100, 1000, 2000, 6889]
      // _tierNames -> ["Mythical Box", "Legendary Box", "Epic Box", "Rare Box", "Uncommon Box", "Common Box"]
      //
      // The `TOTAL_MAX_BOXES` is the sum of `_tierMaxBoxes`, which is 10_000.
      // The probability distribution is as follows:
      //
      // Mythical Box:   1    / 10_000  chance.
      // Legendary Box:  10   / 10_000  chance.
      // Epic Box:       100  / 10_000  chance.
      // Rare Box:       1000 / 10_000  chance.
      // Uncommon Box:   2000 / 10_000  chance.
      // Common Box:     6889 / 10_000  chance.
      //
      // By using the modulo operation with TOTAL_MAX_BOXES, the random number's probability
      // to land in any given tier is determined by the number of max boxes available for that tier
      uint256 randomNumber = boxSeed % TOTAL_MAX_BOXES;

      // Determine the tier of the box based on the random number
      uint128 tierId = _determineTier(randomNumber, mintedTierAmountCache);

      // if the tier amount cache hasn't been updated for this tier type
      if(!tierChangedInd[tierId]) {
          // then set it as updated
          tierChangedInd[tierId] = true;

          // cache the current amounts once from storage
          unclaimedTierAmountCache[tierId] = unclaimedTierAmount[tierId];
      }

      // [Gas] Adjust the cached amounts for this tier
      ++mintedTierAmountCache[tierId];
      ++unclaimedTierAmountCache[tierId];

      // Save the box to the boxIdToBox mapping
      boxIdToBox[boxIdsToEmit[i]] = Box({ blockTs: uint128(block.timestamp), tier: tierId });

      // Mint NFT for this box; explicitly not using `_safeMint` to prevent re-entrancy issues
      _mint(claimer, boxIdsToEmit[i]);
    }

    // Update storage for boxIdCounter
    boxIdCounter = boxIdCounterCache;

    // Update tier storage only for tiers which were changed
    for (uint128 tierId = 1; tierId < TIER_IDS_ARRAY_LEN; tierId++) {
      if (tierChangedInd[tierId]) {
        mintedTierAmount[tierId] = mintedTierAmountCache[tierId];
        unclaimedTierAmount[tierId] = unclaimedTierAmountCache[tierId];
      }
    }

    // Emit an event indicating the boxes have been minted
    emit BoxesMinted(claimer, boxIdsToEmit, code);
  }
```

이 변경을 수행했을 때 `test_assignTierAndMint_InvalidBoxAmount`가 실패하기 시작했지만, `boxAmount` 입력이 `_assignTierAndMint`가 호출되기 전에 검증되므로 이 테스트는 유효하지 않은 것으로 보입니다. 따라서 현재 코드베이스와 관련이 없으므로 이 테스트는 삭제되어야 한다고 판단합니다.

**모드 (Mode):**
커밋 [79eea07](https://github.com/Earnft/dropbox-smart-contracts/commit/79eea078baf6bf855e5a5854981ac60c88f46118)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 발행 및 미청구 티어 상자 수량을 추적할 때 더 효율적인 스토리지 패킹을 위해 `uint32` 사용

**설명:** 먼저 `DropBox` 스토리지를 다음과 같이 변경하여 더 효율적인 스토리지 슬롯 패킹을 수행하십시오:
* `vrfHandler`를 public 변수의 시작 부분으로 이동
* 일부 상수가 아닌 변수를 `uint32`로 변경
* 모든 `uint32` 상수가 아닌 변수를 `bool` 상수가 아닌 변수 바로 뒤에 그룹화:
```solidity
contract DropBox is
  IVRFHandlerReceiver,
  DropBoxFractalProtocol,
  ReentrancyGuard,
  OwnableBasic,
  BasicRoyalties,
  ERC721C
{
  using SafeERC20 for IERC20;

  /// @notice - EARNM ERC20 token
  IERC20 internal immutable EARNM;

  /// @notice Public variables
  IVRFHandler public vrfHandler; // VRF Handler to handle the Chainlink VRF requests
  string public _contractURI; // OpenSea Contract-level metadata
  string public _baseTokenURI; // ERC721 Base Token metadata
  address public apiAddress;
  address public feesReceiverAddress;
  uint128 public revealFee = 1 ether;
  uint128 public claimFee = 1 ether;
  bool public isEarnmClaimAllowed;
  bool public isRevealAllowed;

  /// @notice Internal variables
  uint32 internal boxIdCounter = 1; // Counter for the box ids
  uint32 internal remainingBoxesAmount; // Remaining boxes amount to mint
  /// @notice this array stores index: tierId, value: total amount minted
  /// element [0] never used since tierId : [1,6]
  uint32[TIER_IDS_ARRAY_LEN] internal mintedTierAmount;
  uint32[TIER_IDS_ARRAY_LEN] internal unclaimedTierAmount;

  /// @notice Mappings
  mapping(uint256 => Box) internal boxIdToBox; // boxId -> Box
  mapping(bytes32 => OneTimeCode) internal oneTimeCode; // code+address hash -> one-time code data; ensure integrity of
    // one-time code data
  mapping(bytes32 => bool) internal oneTimeCodeUsed; // plain text code -> used (true/false); ensure uniqueness of
    // one-time codes
  mapping(bytes32 => uint256[]) internal oneTimeCodeRandomWords; // code+address hash -> random words
  /// [Chainlink VRF]
  mapping(uint256 => bytes32) internal activeVrfRequests; // vrf request -> code+address hash
  mapping(uint256 => bool) internal fulfilledVrfRequests; // vrf request -> fulfilled

  /// @notice Constants - DropBox
  uint32 internal constant MAX_BOX_AMOUNT_PER_REVEAL = 100;
  uint128 internal constant MAX_MINT_FEE = 1000 ether;
  uint128 internal constant MAX_CLAIM_FEE = 1000 ether;
```

그런 다음 `DropBox.sol` 전체에서 적절한 유형을 변경하십시오:
```solidity
// function associateOneTimeCodeToAddress,
uint32 remainingBoxesAmountCache = remainingBoxesAmount;

// function claimDropBoxes - see optimized code at end of this issue

// function _assignTierAndMint - see optimized code at end of this issue

// view function return parameters
  function getTotalMaxBoxes() external view returns (uint32 totalMaxBoxes) {
```

그리고 `DropBoxFractalProtocol.sol` 내부:
```solidity
// events
event OneTimeCodeAssociated(address indexed claimerAddress, bytes code, uint32 boxAmount);

  /// @notice Constants - Tier max boxes
  uint32 internal immutable TIER_6_MAX_BOXES;
  uint32 internal immutable TIER_5_MAX_BOXES;
  uint32 internal immutable TIER_4_MAX_BOXES;
  uint32 internal immutable TIER_3_MAX_BOXES;
  uint32 internal immutable TIER_2_MAX_BOXES;
  uint32 internal immutable TIER_1_MAX_BOXES;
  uint32 internal immutable TOTAL_MAX_BOXES;

  function _determineTier(
    uint256 randomNumber,
    uint32[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache
  )
```

테스트 스위트에도 몇 가지 변경이 필요합니다:

파일: `test/mocks/DropBoxMock.sol`
```solidity
function mock_setUnclaimedTierAmount(uint128 tierId, uint32 amount) public {
function mock_getUnclaimedTierAmount(uint128 tierId) public view returns (uint32) {
function mock_setAllBoxesMinted(uint32 amount) public {

function mock_remainingBoxesAmount() public view returns (uint32) {

  function mock_determineTier(
    uint256 randomNumber,
    uint32[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache

function mock_getMintedTierAmount(uint128 tierId) public view returns (uint32) {
function mock_getMintedTierAmounts() public view returns (uint32[TIER_IDS_ARRAY_LEN] memory) {
function mock_getUnclaimedTierAmounts() public view returns (uint32[TIER_IDS_ARRAY_LEN] memory) {

  function mock_getMaxBoxesPerTier(uint128 tierId) external view returns (uint32 maxBoxesPerTier) {
```

파일: `test/DropBox/behaviours/claimDropBox/claimDropBox.t.sol`
```solidity
  function _calculateExpectedClaimAmount(uint256[] memory boxIds)
    internal
    view
    returns (uint256 totalClaimAmount, uint32[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts)

  function _validateClaimedBoxes(
    uint32[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts,

// occurs in 2 few places
    (uint256 expectedAmount, uint32[TIER_IDS_ARRAY_LEN] memory expectedUnclaimedTierAmounts) =
      _calculateExpectedClaimAmount(boxIds);

// near the bottom
    for (uint32 i; i < boxAmount; i++) {
```

파일: `test/DropBox/behaviours/earnmInVesting/liability.t.sol`
```solidity
  function _setUnclaimedTierAmounts(uint32[] memory unclaimedAmounts) internal {
  function _calculateExpectedLiability(uint32[] memory unclaimedAmounts) internal view returns (uint256)

    // occurs in a few places
    uint32[] memory unclaimedAmounts = new uint32[](TIER_IDS_LENGTH);
    for (uint32 i = 1; i <= TIER_IDS_LENGTH; i++) {
```

파일: `test/DropBox/behaviours/revealDropBox/revealDropBoxes.t.sol`
```solidity
  function _validateMinting(
    address allowedAddress,
    uint32[TIER_IDS_ARRAY_LEN] memory prevMintedTierAmounts,

    for (uint32 i; i < boxAmount; i++) {
```

파일: `test/FractalProtocol/FractalProtocol.sol`
```solidity
// occurs many times
    uint32[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache;
```

파일: `test/DropBoxFractalProtocolLib.sol`
```solidity
  event OneTimeCodeAssociated(address indexed claimerAddress, bytes code, uint32 boxAmount);
```

위의 변경 사항을 적용한 후 `foundry.toml`에서 `gas_limit = "18446744073709551615" # u64::MAX`를 사용하여 모든 상자를 공개/청구하려는 테스트에 대한 가스 한도를 높이면 모든 것이 컴파일되고 모든 테스트가 통과합니다.

`forge inspect DropBox storage`를 사용하면 일반 코드가 46개의 스토리지 슬롯을 사용하는 반면 위의 변경 사항으로 수정된 버전은 33개의 스토리지 슬롯만 사용한다는 것을 알 수 있습니다.

이것의 또 다른 이점은 전체 배열 `unclaimedTierAmount` 및 `mintedTierAmount`가 이제 배열 요소당 1개의 스토리지 슬롯을 사용하는 대신 각각 1개의 스토리지 슬롯에 저장된다는 것입니다. 즉, `claimDropBoxes` 및 `_assignTierAndMint`가 조건부 티어 기반 로직 없이 한 번에 배열을 읽고 쓸 수 있도록 더욱 단순화하고 최적화될 수 있습니다:

```solidity
  function claimDropBoxes(uint256[] calldata _boxIds) external payable nonReentrant {
    //
    //  Request Checks
    // ---------------------------------------------------------------------------------------------
    // [Safety check] Only allowed to claim if the isEarnmClaimAllowed variable is true
    // Can be modified by the owner as a safety measure
    if (!(isEarnmClaimAllowed)) revert Unauthorized();

    // [Safety check] Validate that the boxIds array is not empty
    if (_boxIds.length == 0) revert Unauthorized();

    // Require that the user sends a fee of "claimFee" amount
    if (msg.value != claimFee) revert InvalidClaimFee();
    // ---------------------------------------------------------------------------------------------

    // Amount of Earnm tokens to claim
    uint256 amountToClaim;

    // [Gas] Cache unclaimedTierAmount and update cache during the loop then
    // write to storage at the end to prevent multiple update storage writes
    uint32[TIER_IDS_ARRAY_LEN] memory unclaimedTierAmountCache = unclaimedTierAmount;

    //  Earnm Calculation and Safety Checks
    // ---------------------------------------------------------------------------------------------
    // Iterate over box IDs to calculate rewards and validate ownership and uniqueness.
    // - Validate that the sender is the owner of the boxes
    // - Validate that the sender has enough boxes to claim
    // - Validate that the box ids are valid
    // - Validate that the box ids are not duplicated
    // - Calculate the amount of Earnm tokens to claim given the box ids
    for (uint256 i; i < _boxIds.length; i++) {
      // [Safety check] Duplicate values are not allowed
      if (i > 0 && _boxIds[i - 1] >= _boxIds[i]) revert BadRequest();

      // [Safety check] Validate that the box owner is the sender
      if (ownerOf(_boxIds[i]) != msg.sender) revert Unauthorized();

      // Copy the box from the boxes mappings
      Box memory box = boxIdToBox[_boxIds[i]];

      // Increment the amount of Earnm tokens to send to the sender
      amountToClaim += _calculateVestingPeriodPerBox(box.tier, box.blockTs, TIER_IDS_LENGTH);

      // Decrement the unclaimed cached amount for this tier
      --unclaimedTierAmountCache[box.tier];

      // Delete the box from the boxIdToBox mapping
      delete boxIdToBox[_boxIds[i]];

      // Burn the ERC721 token corresponding to the box
      _burn(_boxIds[i]);
    }

    // [Safety check] In case the amount to claim is 0, revert
    if (amountToClaim == 0) revert AmountOfEarnmToClaimIsZero();

    // Update unclaimed tier storage
    unclaimedTierAmount = unclaimedTierAmountCache;

    // ---------------------------------------------------------------------------------------------
    //  Contract Earnm Balance and Safety Checks
    // ---------------------------------------------------------------------------------------------
    // Get the balance of Earnm ERC20 tokens of the contract
    uint256 balance = EARNM.balanceOf(address(this));

    // [Safety check] Validate that the contract has enough Earnm tokens to cover the claim
    if (balance < amountToClaim) revert InsufficientEarnmBalance();
    // ---------------------------------------------------------------------------------------------

    // Emit an event indicating the boxes have been claimed
    emit BoxesClaimed(msg.sender, _boxIds, amountToClaim);

    // Transfer the tokens to the sender
    EARNM.safeTransfer(msg.sender, amountToClaim);
    // ---------------------------------------------------------------------------------------------

    //  Claim Fee Transfer
    // ---------------------------------------------------------------------------------------------
    _sendFee(msg.value);
    // ---------------------------------------------------------------------------------------------
  }

  function _assignTierAndMint(
    uint32 boxAmount,
    uint256[] memory randomNumbers,
    address claimer,
    bytes memory code
  )
    internal
  {
    uint256[] memory boxIdsToEmit = new uint256[](boxAmount);

    // [Gas] Cache mintedTierAmount and update cache during the loop to prevent
    // _determineTier() from re-reading it from storage multiple times
    uint32[TIER_IDS_ARRAY_LEN] memory mintedTierAmountCache = mintedTierAmount;

    // [Gas] Cache unclaimedTierAmount and update cache during the loop then
    // write to storage at the end to prevent multiple update storage writes
    uint32[TIER_IDS_ARRAY_LEN] memory unclaimedTierAmountCache = unclaimedTierAmount;

    // [Gas] cache the current boxId, use cache during the loop then write once
    // at the end; this results in only 1 storage read and 1 storage write for `boxIdCounter`
    uint32 boxIdCounterCache = boxIdCounter;

    // For each box to mint
    for (uint32 i; i < boxAmount; i++) {
      // [Memory] Add the box id to the boxes ids array
      boxIdsToEmit[i] = boxIdCounterCache++;

      // The box seed is directly derived from the random number without any
      // additional values like the box id, the claimer address, or the block timestamp.
      // This is made like this to ensure the tier assignation of the box is purely random
      // and the tier that the box is assigned to, won't change after the randomness is fulfilled.
      // The only way to predict the seed is to know the random number.
      // Random number validation is done in the _fulfillRandomWords function.
      uint256 boxSeed = uint256(keccak256(abi.encode(randomNumbers[i])));

      // Generate a random number between 0 and TOTAL_MAX_BOXES - 1 using the pure random seed
      // Modulo ensures that the random number is within the range of [0, TOTAL_MAX_BOXES) (upper bound exclusive),
      // providing a uniform distribution across the possible range. This ensures that each tier has a
      // probability of being selected proportional to its maximum box count. The randomness is normalized
      // to fit within the predefined total boxes, maintaining the intended probability distribution for each tier.
      //
      // Example with the following setup:
      // _tierMaxBoxes -> [1, 10, 100, 1000, 2000, 6889]
      // _tierNames -> ["Mythical Box", "Legendary Box", "Epic Box", "Rare Box", "Uncommon Box", "Common Box"]
      //
      // The `TOTAL_MAX_BOXES` is the sum of `_tierMaxBoxes`, which is 10_000.
      // The probability distribution is as follows:
      //
      // Mythical Box:   1    / 10_000  chance.
      // Legendary Box:  10   / 10_000  chance.
      // Epic Box:       100  / 10_000  chance.
      // Rare Box:       1000 / 10_000  chance.
      // Uncommon Box:   2000 / 10_000  chance.
      // Common Box:     6889 / 10_000  chance.
      //
      // By using the modulo operation with TOTAL_MAX_BOXES, the random number's probability
      // to land in any given tier is determined by the number of max boxes available for that tier
      uint256 randomNumber = boxSeed % TOTAL_MAX_BOXES;

      // Determine the tier of the box based on the random number
      uint128 tierId = _determineTier(randomNumber, mintedTierAmountCache);

      // [Gas] Adjust the cached amounts for this tier
      ++mintedTierAmountCache[tierId];
      ++unclaimedTierAmountCache[tierId];

      // Save the box to the boxIdToBox mapping
      boxIdToBox[boxIdsToEmit[i]] = Box({ blockTs: uint128(block.timestamp), tier: tierId });

      // Mint the box; explicitly not using `_safeMint` to prevent re-entrancy issues
      _mint(claimer, boxIdsToEmit[i]);
    }

    // Update storage for boxIdCounter
    boxIdCounter = boxIdCounterCache;

    // Update unclaimed & minted tier amounts
    mintedTierAmount = mintedTierAmountCache;
    unclaimedTierAmount = unclaimedTierAmountCache;

    // Emit an event indicating the boxes have been minted
    emit BoxesMinted(claimer, boxIdsToEmit, code);
  }
```

**모드 (Mode):**
커밋 [37d258c](https://github.com/Earnft/dropbox-smart-contracts/commit/37d258caa2b6ca07af7f8229951ebf7bc2cd6202)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 더 효율적인 스토리지 패킹을 위해 `revealFee` 및 `claimFee`에 `uint128` 사용

**설명:** 현재 `revealFee`와 `claimFee`는 `uint256`을 사용하며 최대값이 `1000 ether`인 `1 ether`로 설정되어 있습니다. 따라서 각각 1개의 스토리지 슬롯을 차지하지만 다음과 같으므로 그리 효율적이지 않습니다:
```solidity
1 ether           = 1000000000000000000
1000 ether        = 1000000000000000000000
type(uint128.max) = 340282366920938463463374607431768211455
```

따라서 `revealFee` 및 `claimFee`에 `uint128`을 사용하십시오:
```solidity
// storage
  uint128 public revealFee = 1 ether;
  uint128 public claimFee = 1 ether;
  uint128 internal constant MAX_MINT_FEE = 1000 ether;
  uint128 internal constant MAX_CLAIM_FEE = 1000 ether;

// function updates
function setRevealFee(uint128 _mintFee) external onlyOwner {
  function setClaimFee(uint128 _claimFee) external onlyOwner {
```

또한 `DropBoxFractalProtocol`에서 이벤트를 업데이트하십시오:
```solidity
  event RevealFeeUpdated(uint128 revealFee);
  event ClaimFeeUpdated(uint128 claimFee);
```

**모드 (Mode):**
커밋 [fb20301](https://github.com/Earnft/dropbox-smart-contracts/commit/fb20301f77c9ae0c7c2333a13a11ca4b02d88366)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `VRFHandler::constructor`에서 중복 확인 제거

**설명:** `VRFHandler::constructor`에서 중복 확인을 제거하십시오:
```diff
  constructor(
    address vrfCoordinator,
    bytes32 keyHash,
    uint256 subscriptionId
  )
    /**
     * ConfirmedOwner(msg.sender) is defined in @chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol:113:40
     * with "msg.sender".
     */
    VRFConsumerBaseV2Plus(vrfCoordinator)
  {
    if (keyHash == bytes32(0)) revert InvalidVrfKeyHash();
    if (subscriptionId == 0) revert InvalidVrfSubscriptionId();
-   if (vrfCoordinator == address(0)) revert InvalidVrfCoordinator();

    vrfKeyHash = keyHash;
    vrfSubscriptionId = subscriptionId;
  }
```

이 확인은 `VRFHandler`가 상속하는 `VRFConsumerBaseV2Plus`가 이미 자체 생성자에서 이 확인을 [수행](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol#L114-L115)하므로 중복됩니다.

**모드 (Mode):**
커밋 [cc45c9a](https://github.com/Earnft/dropbox-smart-contracts/commit/cc45c9a80980118107ede5001ee9ac5c7a5b6040)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `VRFHandler::fulfillRandomWords`에서 `vrfRequestIdToRequester`의 이행된 vrf 요청 삭제

**설명:** `VRFHandler::fulfillRandomWords`에서 `vrfRequestIdToRequester`의 이행된 vrf 요청을 삭제하십시오:
```diff
  function fulfillRandomWords(uint256 _requestId, uint256[] calldata _randomWords) internal override {
    // Check if the request has already been fulfilled
    if (vrfFulfilledRequests[_requestId]) revert InvalidVrfState();

    // Cache the contract that requested the random numbers based on the request ID
    address contractRequestor = vrfRequestIdToRequester[_requestId];

    // Revert if the contract that requested the random numbers is not found
    if (contractRequestor == address(0)) revert InvalidVrfState();

+   // delete the active requestId -> requestor record
+   delete vrfRequestIdToRequester[_requestId];

    // Mark the request as fulfilled
    vrfFulfilledRequests[_requestId] = true;

    // Decrement the counter of outstanding requests
    activeRequests--;

    // Call the contract that requested the random numbers with the random numbers
    IVRFHandlerReceiver(contractRequestor).fulfillRandomWords(_requestId, _randomWords);
  }
```

**모드 (Mode):**
커밋 [08d7ce4](https://github.com/Earnft/dropbox-smart-contracts/commit/08d7ce40f24e958b51528233f57b99f4333c7501)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage