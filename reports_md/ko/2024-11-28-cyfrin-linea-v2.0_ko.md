**Lead Auditors**

[Dacian](https://x.com/DevDacian)

**Assisting Auditors**

 


---

# Findings
## Medium Risk


### Operator can finalize for non-existent `finalShnarf`

**Description:** 이전 버전의 `LineaRollup::_finalizeBlocks`에는 최종 shnarf가 블록 번호와 연결되어 있는지 확인하는 다음과 같은 검사가 있었습니다:
```solidity
if (
  shnarfFinalBlockNumbers[_finalizationData.finalSubmissionData.shnarf] !=
  _finalizationData.finalSubmissionData.finalBlockInData
) {
  revert FinalBlockDoesNotMatchShnarfFinalBlock(
    _finalizationData.finalSubmissionData.finalBlockInData,
    shnarfFinalBlockNumbers[_finalizationData.finalSubmissionData.shnarf]
  );
}
```

새 버전에서는 `shnarfFinalBlockNumbers`가 `blobShnarfExists`로 변경되어 shnarf를 유효한 부울 플래그(이전 정의로 인해 uint이지만)에 연결하며, 위 검사는 제거되었지만 유사한 검사가 구현되지 않았습니다.

**Impact:** 운영자가 존재하지 않는 `finalShnarf`에 대해 최종화(finalize)할 수 있습니다.

**Recommended Mitigation:** `LineaRollup::_finalizeBlocks`에 계산된 `finalShnarf`가 존재하는지 확인하는 동등한 검사를 추가하십시오:
```solidity
finalShnarf = _computeShnarf(
  _finalizationData.shnarfData.parentShnarf,
  _finalizationData.shnarfData.snarkHash,
  _finalizationData.shnarfData.finalStateRootHash,
  _finalizationData.shnarfData.dataEvaluationPoint,
  _finalizationData.shnarfData.dataEvaluationClaim
);

// @audit prevent finalization for non-existent final shnarf
if(blobShnarfExists[finalShnarf] == 0) revert FinalBlobNotSubmitted();
```

**Linea:** [PR226](https://github.com/Consensys/linea-monorepo/pull/226) 커밋 [4286bdb](https://github.com/Consensys/linea-monorepo/pull/226/commits/4286bdbd03a0d447adf60dba6d26680503c2a14f)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## Low Risk


### Operator can submit data via `LineaRollup::submitDataAsCalldata` for invalid parent shnarf

**Description:** `LineaRollup::submitBlobs`에는 부모 shnarf가 존재하는지 검증하는 다음과 같은 검사가 있습니다:
```solidity
    if (blobShnarfExists[_parentShnarf] == 0) {
      revert ParentBlobNotSubmitted(_parentShnarf);
    }
```

그러나 `LineaRollup::submitDataAsCalldata`에는 유사한 검사가 없으므로 운영자가 `submitDataAsCalldata`를 호출하여 유효하지 않은 부모 shnarf에 대한 데이터를 제출할 수 있습니다.

**Linea:** [PR223](https://github.com/Consensys/linea-monorepo/pull/223) 커밋 [8800eaa](https://github.com/Consensys/linea-monorepo/pull/223/commits/8800eaa2fd9f45b9048b29caaeee939ade01e317)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## Informational


### Use SafeCast or document assumption that unsafe downcast in `SparkeMerkleTreeVerifier::_verifyMerkleProof` can't overflow

**Description:** `SparkeMerkleTreeVerifier::_verifyMerkleProof`에 다음과 같은 온전성 검사(sanity check)가 추가되었습니다:
```solidity
uint32 maxAllowedIndex = uint32((2 ** _proof.length) - 1);
if (_leafIndex > maxAllowedIndex) {
  revert LeafIndexOutOfBounds(_leafIndex, maxAllowedIndex);
}
```

`_proof.length` > 32인 경우 Solidity 캐스트는 되돌리지 않고(revert) 오버플로우되므로 오버플로우가 발생합니다.

팀은 다음과 같이 언급했습니다: _"finalization에서 오는 머클 트리 깊이(depth)를 기반으로 하며, 길이는 깊이와 대조 확인됩니다. 현재 5로 설정되어 있으며 변경될 가능성은 낮습니다"._

따라서 오버플로우는 실제로 불가능해 보이지만, 다음 중 하나를 권장합니다:

* 오버플로우가 발생하면 되돌리기 위해 [SafeCast](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeCast.sol#L509-L514) 사용
* 코드 내에서 오버플로우가 발생할 수 없다는 가정을 명시적으로 문서화

주석 접근 방식의 위험은 향후 개발자가 이 곳에서 오버플로우를 유발할 수 있다는 사실을 깨닫지 못한 채 finalization의 관련 코드를 변경할 수 있다는 것입니다.

**Linea:** [PR222](https://github.com/Consensys/linea-monorepo/pull/222) 커밋 [c20b938](https://github.com/Consensys/linea-monorepo/pull/222/commits/c20b9380dd55237a6cbc65826d17c85cb68afe5b) 및 [77a6e99](https://github.com/Consensys/linea-monorepo/pull/222/commits/77a6e99bc4413c8e445f795a35ff8b4cddbcadb9)에서 수정됨.

**Cyfrin:** 확인됨.


### Mark `L1MessageManagerV1::outboxL1L2MessageStatus` as deprecated

**Description:** `L1MessageManagerV1::outboxL1L2MessageStatus`는 테스트 스위트를 제외하고는 더 이상 읽거나 쓰지 않습니다:
```
$ rg "outboxL1L2MessageStatus"
test-contracts/TestL1MessageManager.sol
31:    outboxL1L2MessageStatus[_messageHash] = OUTBOX_STATUS_SENT;
39:      uint256 existingStatus = outboxL1L2MessageStatus[messageHash];
46:        outboxL1L2MessageStatus[messageHash] = OUTBOX_STATUS_RECEIVED;

messageService/l1/v1/L1MessageManagerV1.sol
22:  mapping(bytes32 messageHash => uint256 messageStatus) public outboxL1L2MessageStatus;

test-contracts/LineaRollupV5.sol
1944:  mapping(bytes32 messageHash => uint256 messageStatus) public outboxL1L2MessageStatus;

test-contracts/LineaRollupAlphaV3.sol
2011:  mapping(bytes32 messageHash => uint256 messageStatus) public outboxL1L2MessageStatus;

tokenBridge/mocks/MessageBridgeV2/MockMessageServiceV2.sol
40:    outboxL1L2MessageStatus[messageHash] = OUTBOX_STATUS_SENT;
```

따라서 `LineaRollup`이 더 이상 사용되지 않는 매핑을 처리하는 방식과 유사한 주석으로 지원 중단(deprecated) 표시를 해야 합니다.

이와 유사하게 `L1MessageManagerV1::inboxL2L1MessageStatus`는 삭제만 되고 새로운 매핑이 삽입되지 않습니다. 이상적으로는 이에 대한 주석도 표시해야 합니다.

**Linea:** [PR256](https://github.com/Consensys/linea-monorepo/pull/256) 커밋 [ac51e9e](https://github.com/Consensys/linea-monorepo/pull/256/commits/ac51e9e57050f1fae13f6446a890250403e74b10#diff-231d964c3a2bc0ac9159b40fa3ab1196ee50417232c63df839d83cec34250d49L19-R26)에서 수정됨.

**Cyfrin:** 확인됨.


### Remove comments which no longer apply

**Description:** 더 이상 적용되지 않는 주석은 오해의 소지가 있으므로 제거해야 합니다.

File: `L1MessageServiceV1.sol`
```solidity
// @audit `claimMessage` no longer uses `_messageSender` so these comments are incorrect
   * @dev _messageSender is set temporarily when claiming and reset post. Used in sender().
   * @dev _messageSender is reset to DEFAULT_SENDER_ADDRESS to be more gas efficient.
```

File: `L1MessageService.sol`
```solidity
// @audit `_messageSender` no longer initialized as it is not used anymore by L1 Messaging
   * @dev _messageSender is initialised to a non-zero value for gas efficiency on claiming.
```

**Linea:** [PR256](https://github.com/Consensys/linea-monorepo/pull/256) 커밋 [ac51e9e](https://github.com/Consensys/linea-monorepo/pull/256/commits/ac51e9e57050f1fae13f6446a890250403e74b10#diff-e9f6a0c3577321e5aa88a9d7e12499c2c4819146062e33a99576971682396bb5L101-R102) 및 [b875723](https://github.com/Consensys/linea-monorepo/commit/b875723765ceeeccbbdf1a0a747884ad7589001e)에서 수정됨.

**Cyfrin:** 확인됨.


### Use named mappings in `TokenBridge` and remove obsolete comments

**Description:** `TokenBridge`는 명명된 매핑(named mappings)을 사용하고 구식 주석을 제거해야 합니다:
```diff
-   /// @notice mapping (chainId => nativeTokenAddress => brigedTokenAddress)
-   mapping(uint256 => mapping(address => address)) public nativeToBridgedToken;
-   /// @notice mapping (brigedTokenAddress => nativeTokenAddress)
-   mapping(address => address) public bridgedToNativeToken;

+   mapping(uint256 chainId => mapping(address native => address bridged)) public nativeToBridgedToken;
+   mapping(address bridged => address native) public bridgedToNativeToken;
```

**Linea:** [PR256](https://github.com/Consensys/linea-monorepo/pull/256) 커밋 [ac51e9e](https://github.com/Consensys/linea-monorepo/pull/256/commits/ac51e9e57050f1fae13f6446a890250403e74b10#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L62-R74)에서 수정됨.

**Cyfrin:** 확인됨.


### `L2MessageService::reinitializePauseTypesAndPermissions` should use `reinitializer(2)`

**Description:** `TokenBridge::reinitializePauseTypesAndPermissions`는 `reinitializer(2)`를 사용합니다. 왜냐하면:
```
export ETH_RPC_URL=mainnet_rpc
cast storage 0x051F1D88f0aF5763fB888eC4378b4D8B29ea3319 0
0x0000000000000000000000000000000000000000000000000000000000000001
export ETH_RPC_URL=linea_rpc
cast storage 0x353012dc4a9A6cF55c941bADC267f82004A8ceB9 0
0x0000000000000000000000000000000000000000000000000000000000000001
```

`LineaRollup::reinitializeLineaRollupV6`는 `reinitializer(6)`를 사용합니다. 왜냐하면:
```
export ETH_RPC_URL=mainnet_rpc
cast storage 0xd19d4B5d358258f05D7B411E21A1460D11B0876F 0
0x0000000000000000000000000000000000000000000000000000000000000005
```

하지만 `L2MessageService::reinitializePauseTypesAndPermissions`는 다음 상황임에도 불구하고 `reinitializer(6)`를 사용합니다:
```
export ETH_RPC_URL=linea_rpc
cast storage 0x508Ca82Df566dCD1B0DE8296e70a96332cD644ec 0
0x0000000000000000000000000000000000000000000000000000000000000001
```

일관성을 위해 `L2MessageService::reinitializePauseTypesAndPermissions`는 `reinitializer(2)`를 사용해야 합니다.

**Linea:** [PR271](https://github.com/Consensys/linea-monorepo/pull/271) 커밋 [53f43d3](https://github.com/Consensys/linea-monorepo/pull/271/commits/53f43d3d6f6c49556e4da49af53e5db3aa2bfa24)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## Gas Optimization


### Remove redundant `L2MessageManagerV1::__L2MessageManager_init` and associated constant

**Description:** `L2MessageManagerV1::__L2MessageManager_init`은 새로운 `PermissionsManager` 컨트랙트를 사용하는 `L2MessageService::initialize`에 의해 더 이상 호출되지 않습니다.

따라서 관련 상수인 `L1_L2_MESSAGE_SETTER_ROLE`과 함께 제거되어야 합니다. 이 상수는 코드 전반의 주석에서 참조되므로 해당 주석도 업데이트되어야 합니다.

테스트 스위트에는 `L2MessageManagerV1::__L2MessageManager_init`에 대한 호출이 여전히 포함되어 있습니다; 테스트 스위트 또한 초기화에 새로운 방법만 사용하도록 업데이트되어야 합니다.
```solidity
// output of: rg "__L2MessageManager_init"
messageService/l2/v1/L2MessageManagerV1.sol
39:  function __L2MessageManager_init(address _l1l2MessageSetter) internal onlyInitializing {

test-contracts/TestL2MessageManager.sol
32:    __L2MessageManager_init(_l1l2MessageSetter);
41:    __L2MessageManager_init(_l1l2MessageSetter);

test-contracts/L2MessageServiceLineaMainnet.sol
1620:  function __L2MessageManager_init(address _l1l2MessageSetter) internal onlyInitializing {
1992:    __L2MessageManager_init(_l1l2MessageSetter);

// output of: rg "L1_L2_MESSAGE_SETTER_ROLE"
messageService/l2/L2MessageManager.sol
26:   * @dev Only address that has the role 'L1_L2_MESSAGE_SETTER_ROLE' are allowed to call this function.
40:  ) external whenTypeNotPaused(PauseType.GENERAL) onlyRole(L1_L2_MESSAGE_SETTER_ROLE) {

messageService/l2/v1/L2MessageManagerV1.sol
18:  bytes32 public constant L1_L2_MESSAGE_SETTER_ROLE = keccak256("L1_L2_MESSAGE_SETTER_ROLE");
37:   * @param _l1l2MessageSetter The address owning the L1_L2_MESSAGE_SETTER_ROLE role.
40:    _grantRole(L1_L2_MESSAGE_SETTER_ROLE, _l1l2MessageSetter);

interfaces/l2/IL2MessageManager.sol
48:   * @dev Only address that has the role 'L1_L2_MESSAGE_SETTER_ROLE' are allowed to call this function.

test-contracts/L2MessageServiceLineaMainnet.sol
1601:  bytes32 public constant L1_L2_MESSAGE_SETTER_ROLE = keccak256("L1_L2_MESSAGE_SETTER_ROLE");
1618:   * @param _l1l2MessageSetter The address owning the L1_L2_MESSAGE_SETTER_ROLE role.
1621:    _grantRole(L1_L2_MESSAGE_SETTER_ROLE, _l1l2MessageSetter);
1626:   * @dev Only address that has the role 'L1_L2_MESSAGE_SETTER_ROLE' are allowed to call this function.
1629:  function addL1L2MessageHashes(bytes32[] calldata _messageHashes) external onlyRole(L1_L2_MESSAGE_SETTER_ROLE) {
```

**Linea:** [PR212](https://github.com/Consensys/linea-monorepo/pull/212) 커밋 [3b30a8a](https://github.com/Consensys/linea-monorepo/pull/212/commits/3b30a8aa083bfe77fa6e73ca3950343f250f482b)에서 수정됨.

**Cyfrin:** 확인됨.


### Cheaper to not cache `calldata` array length

**Description:** 배열이 `calldata`로 전달될 때 [길이를 캐시하지 않는 것이 더 저렴합니다](https://x.com/DevDacian/status/1791490921881903468):
```diff
// PermissionsManager::__Permissions_init
  function __Permissions_init(RoleAddress[] calldata _roleAddresses) internal onlyInitializing {
-    uint256 roleAddressesLength = _roleAddresses.length;

-    for (uint256 i; i < roleAddressesLength; i++) {
+    for (uint256 i; i < _roleAddresses.length; i++) {
```

다음에도 동일하게 적용됩니다:
* `PauseManager::__PauseManager_init`
* `LineaRollup::submitBlobs`
* `L2MessageManager::anchorL1L2MessageHashes`

**Linea:** [PR247](https://github.com/Consensys/linea-monorepo/pull/247) 커밋 [8bf9d86](https://github.com/Consensys/linea-monorepo/pull/247/commits/8bf9d867bb0fa3f9f5956efa3d8e90f4e21cf4ee), [0ffc752](https://github.com/Consensys/linea-monorepo/pull/247/commits/0ffc752c126c419d67d70260065996d4ad2545b5) 및 커밋 [968b257](https://github.com/Consensys/linea-monorepo/pull/247/commits/968b25795d1323d83360cbc3be3480a861b12aac)에서 수정됨.

**Cyfrin:** 확인됨.


### Use named return variables to save at least 9 gas per variable

**Description:** [명명된 반환 변수](https://x.com/DevDacian/status/1796396988659093968)를 사용하면 변수당 최소 9 가스를 절약할 수 있습니다; 명명된 반환은 프로토콜의 일부 함수에서 이미 사용되고 있지만 다른 함수에서는 그렇지 않습니다:
```solidity
PauseManager.sol
136:  function isPaused(PauseType _pauseType) public view returns (bool)

l1/L1MessageManager.sol
98:  function isMessageClaimed(uint256 _messageNumber) external view returns (bool) {

l1/L1MessageService.sol
150:  function sender() external view returns (address addr) {

l2/v1/L2MessageServiceV1.sol
165:  function sender() external view returns (address) {

lib/SparseMerkleTreeVerifier.sol
32:  ) internal pure returns (bool) {

TokenBridge.sol
 function _safeName(address _token) internal view returns (string memory) {
  function _safeSymbol(address _token) internal view returns (string memory) {
  function _safeDecimals(address _token) internal view returns (uint8) {
  function _returnDataToString(bytes memory _data) internal pure returns (string memory) {
```

**Linea:** [PR247](https://github.com/Consensys/linea-monorepo/pull/247) 커밋 [968b257](https://github.com/Consensys/linea-monorepo/pull/247/commits/968b25795d1323d83360cbc3be3480a861b12aac)에서 수정됨.

**Cyfrin:** 확인됨.


### Cache storage variables to avoid multiple identical storage reads

**Description:** 동일한 스토리지 읽기 반복을 피하기 위해 스토리지 변수를 캐시하십시오:

File: `TokenBridge.sol`
```solidity
// @audit use `_initializationData.sourceChainId` instead of `sourceChainId`
147:        nativeToBridgedToken[sourceChainId][_initializationData.reservedTokens[i]] = RESERVED_STATUS;

// @audit cache 'sourceChainId' from storage and use cached copy
371:        nativeToBridgedToken[sourceChainId][_nativeTokens[i]] = DEPLOYED_STATUS;
```

**Linea:** [PR247](https://github.com/Consensys/linea-monorepo/pull/247) 커밋 [968b257](https://github.com/Consensys/linea-monorepo/pull/247/commits/968b25795d1323d83360cbc3be3480a861b12aac#diff-e5dcf44cdbba69f5a1f8fc58700577ce57caac0c15a5d5fb63e0620aeced62d4L147-R374)에서 수정됨.

**Cyfrin:** 확인됨.


### Fail fast in `LineaRollup::submitBlobs` and `submitDataAsCalldata`

**Description:** `LineaRollup::submitBlobs`는 많은 처리를 수행한 후 `for` 루프 다음에 계산된 shnarf가 제공된 예상 입력과 일치하는지 확인하는 첫 번째 검사를 수행합니다:
```solidity
if (_finalBlobShnarf != computedShnarf) {
  revert FinalShnarfWrong(_finalBlobShnarf, computedShnarf);
}
```

첫 번째 검사가 되돌려지지 않았다면, 이는 `_finalBlobShnarf == computedShnarf`임을 의미합니다.

그 다음 두 번째 검사는 이 shnarf가 이미 존재하는 경우 되돌립니다:
```solidity
if (blobShnarfExists[computedShnarf] != 0) {
  revert DataAlreadySubmitted(computedShnarf);
}
```

그러나 두 번째 검사는 `_finalBlobShnarf == computedShnarf`인 경우에만 실행될 수 있으므로, 두 번째 검사를 삭제하고 함수의 시작 부분에 다음과 같이 새로운 검사를 넣는 것이 훨씬 효율적입니다:
```solidity
if (blobShnarfExists[_finalBlobShnarf] != 0) {
  revert DataAlreadySubmitted(_finalBlobShnarf);
}
```

이상적으로는 함수의 시작 부분에 다른 작업을 수행하거나 선언하기 전에 다음 4가지 검사가 있어야 합니다:
```solidity
  function submitBlobs(
    BlobSubmission[] calldata _blobSubmissions,
    bytes32 _parentShnarf,
    bytes32 _finalBlobShnarf
  ) external whenTypeAndGeneralNotPaused(PauseType.BLOB_SUBMISSION) onlyRole(OPERATOR_ROLE) {
    uint256 blobSubmissionLength = _blobSubmissions.length;

    if (blobSubmissionLength == 0) {
      revert BlobSubmissionDataIsMissing();
    }

    if (blobhash(blobSubmissionLength) != EMPTY_HASH) {
      revert BlobSubmissionDataEmpty(blobSubmissionLength);
    }

    if (blobShnarfExists[_parentShnarf] == 0) {
      revert ParentBlobNotSubmitted(_parentShnarf);
    }

    if (blobShnarfExists[_finalBlobShnarf] != 0) {
      revert DataAlreadySubmitted(_finalBlobShnarf);
    }

    // variable declarations and processing follow
```

`submitDataAsCalldata`에도 동일하게 적용됩니다:
```solidity
  function submitDataAsCalldata(
    CompressedCalldataSubmission calldata _submission,
    bytes32 _parentShnarf,
    bytes32 _expectedShnarf
  ) external whenTypeAndGeneralNotPaused(PauseType.CALLDATA_SUBMISSION) onlyRole(OPERATOR_ROLE) {
    if (_submission.compressedData.length == 0) {
      revert EmptySubmissionData();
    }

    if (blobShnarfExists[_expectedShnarf] != 0) {
      revert DataAlreadySubmitted(_expectedShnarf);
    }

    // ...
```

**Linea:** [PR247](https://github.com/Consensys/linea-monorepo/pull/247) 커밋 [968b257](https://github.com/Consensys/linea-monorepo/pull/247/commits/968b25795d1323d83360cbc3be3480a861b12aac)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
