## 낮은 중요도 (Low Risk)


### 만료된 청구(claim)는 만료일이 지난 후에도 계약 소유자에 의해 부활될 수 있음

**설명:** 계약 소유자는 `setClaims` 함수를 사용하여 `expiry` 및 `minFee`와 같은 설정을 포함하여 각 청구 계약에 대한 청구 매개변수를 구성할 수 있습니다. 청구가 유효한 것으로 간주되려면 청구자는 지정된 `minFee`를 초과하는 수수료를 지불하고 만료일 전에 청구해야 합니다.

그러나 `setClaims` 함수가 계약 소유자가 이미 지난 만료일로 청구를 추가하거나 업데이트할 수 있다는 점에 유의하는 것이 중요합니다. 청구자가 그러한 계약에서 청구할 수 없으므로 처음에는 무해한 부작용처럼 보일 수 있지만 더 중요한 의미가 있습니다. 계약 소유자는 이미 만료된 에어드랍을 효과적으로 부활시켜 더 이상 활성 상태가 아니어야 하는 에어드랍을 수행할 수 있습니다.

[`DropClaims::setClaims`의 영향을 받는 코드 라인](https://github.com/kadenzipfel/drop-claim/blob/9fb36aab457b1ad3ea27351b004ddcdc5ef30682/src/DropClaim.sol#L77-L83)

**파급력 (Impact):** 계약 소유자는 만료된 후에도 청구를 수정할 수 있습니다. 이를 통해 이미 종료되었어야 하는 청구의 유효 기간을 연장하여 청구자를 소급하여 추가할 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):** `setClaims` 함수는 계약 소유자로 제한되고 프로토콜 개발자가 특정 신뢰 가정을 하지만, 계약 소유자에게 부여된 무제한 권한을 완화하기 위해 추가 제어를 구현하는 것이 좋습니다.

기존 청구를 추가/업데이트하기 전에 청구 시간을 검증하는 것을 고려하십시오.

```diff
// DropClaim::setClaims
    for (uint256 i; i < arrLength;) {
+       if(expiries[i] <= block.timsetstamp) revert ERROR_EXPIRE_TIME_SHOULD_BE_GREATER_THAN_NOW;
        claims[claimContractHashes[i]] = ClaimData(uint64(expiries[i]), uint128(minFees[i]));

        unchecked {
            ++i;
        }
    }
```
**Bankless:** 확인했습니다. 의도된 기능이므로 팀은 이 발견 사항과 관련된 어떠한 변경도 하지 않을 것입니다.

**Cyfrin:** 확인했습니다.


### 활성 에어드랍 중간에 최소 수수료 및 만료일을 변경하면 기존/미래 청구자에게 불공평할 수 있음

**설명:** `DropClaim::setClaims`를 통해 사용자는 기존 청구 계약 해시의 청구 매개변수, `expiry` 및 `minimumFee`를 덮어쓸 수 있습니다.

```solidity
  function setClaims(bytes32[] calldata claimContractHashes, uint256[] calldata expiries, uint256[] calldata minFees)
        external
        onlyOwner
    {
        if (claimContractHashes.length != expiries.length) revert MismatchedArrayLengths();
        if (claimContractHashes.length != minFees.length) revert MismatchedArrayLengths();

        uint256 arrLength = claimContractHashes.length;
        for (uint256 i; i < arrLength;) {
            claims[claimContractHashes[i]] = ClaimData(uint64(expiries[i]), uint128(minFees[i]));
            // @audit -> claim parameters for an existing claim contract hash can be overwritten for an active airdrop
            unchecked {
                ++i;
            }
        }
    }
```

사용자가 에어드랍 청구를 시작할 때 프로토콜이 시작할 수 있는 다음 조치의 잠재적 결과를 고려하는 것이 중요합니다:

1. 유효 기간을 연장하거나 단축하여 미래 청구자에 대한 유효 기간 조정.
2. 수수료를 늘리거나 줄여서 미래 청구자에 대한 수수료 수정.

두 경우 모두 기존 청구자이든 미래 청구자이든 그러한 조치가 불공정함을 초래할 가능성이 있음을 인식하는 것이 중요합니다. 프로토콜 소유자조차도 에어드랍이 활성화되면 에어드랍 매개변수를 조작할 수 없어야 합니다.

청구에 `allowList` 모드를 사용하는 경우 `setMerkleRoot` 함수와 관련하여 유사한 우려가 발생합니다. 소유자는 에어드랍이 활성화되고 일부 사용자가 청구를 한 후에도 머클 루트를 업데이트할 수 있습니다.

**파급력 (Impact):** `minFees` 감소는 미래 청구자에게 이익이 되고 `minFees` 증가는 기존 청구자에게 이익이 됩니다. 마찬가지로 유효 기간 단축은 과거 청구자에게 이익이 되고 유효 기간 연장은 미래 청구자에게 이익이 됩니다. 에어드랍이 활성화된 후 소유자가 머클 루트를 업데이트할 수 있도록 허용하면 이전에 자격이 있던 청구자가 자격을 잃거나 그 반대의 경우가 발생할 수 있습니다.


**권장 완화 조치 (Recommended Mitigation):** `ClaimData` 구조체에서 추가 매개변수 `numClaimers`를 추적하는 것을 고려하십시오. 이 추가 사항이 있어도 `ClaimData`는 여전히 단일 슬롯에 맞습니다.

```diff
  struct ClaimData {
        uint64 expiry; // Timestamp beyond which claims are disallowed
        uint128 minFee; // Minimum ETH fee amount
+        uint64 numClaimers;// Number of users who already claimed //@audit -> add this variable to ClaimData struct
    }
```

새로운 청구가 성공할 때마다 `numClaimers`를 증가시킵니다. 기존 청구자가 없는 경우에만 `setClaims` 및 `setMerkleRoot`에서 변경을 허용합니다.

`DropClaim::setClaims`

```diff

    function setClaims(bytes32[] calldata claimContractHashes, uint256[] calldata expiries, uint256[] calldata minFees)
        external
        onlyOwner
    {
        if (claimContractHashes.length != expiries.length) revert MismatchedArrayLengths();
        if (claimContractHashes.length != minFees.length) revert MismatchedArrayLengths();

        uint256 arrLength = claimContractHashes.length;
        for (uint256 i; i < arrLength;) {
+         require(claims[claimContractHashes[i]].numClaimers ==0, "Airdrop already activated");
            claims[claimContractHashes[i]] = ClaimData(uint64(expiries[i]), uint128(minFees[i]));
            unchecked {
                ++i;
            }
        }
    }

```

**Bankless:** 확인했습니다. 팀은 트레이드 오프할 가치가 있으므로 변경하지 않을 것입니다.

**Cyfrin:** 확인했습니다.

\clearpage

## 정보성 (Informational)


### `allowlistClaim`을 통해 사용자가 `claimContract`에 무제한으로 액세스할 수 있어 중복 호출 가능성 생성

**설명:** `allowlistClaim` 함수를 사용하면 청구자가 `claimContract`의 함수에 반복해서 액세스할 수 있습니다. 그러나 청구자가 검증을 위해 이미 증명을 제출했는지 여부는 추적하지 않습니다. 이는 에어드랍과 관련된 계약의 경우 문제가 될 수 있는데, `claimContract`와의 성공적인 상호 작용 횟수를 제한하는 것이 중요하기 때문입니다.

현재 중복 상호 작용 확인은 전적으로 `claimContract`에 있습니다. 왜냐하면 `dropClaim` 계약은 머클 증명만 확인하기 때문입니다.

대조적으로, Uniswap과 1inch가 배포한 [MerkleDistributor 계약의 청구 함수](https://etherscan.io/address/0x090D4613473dEE047c3f2706764f49E0821D256e#code)는 청구자가 제출한 증명 확인과 각 청구자가 한 번만 성공적인 호출을 수행할 수 있도록 보장하는 것을 모두 처리합니다.

다음은 0x090D4613473dEE047c3f2706764f49E0821D256e의 Uniswap `Merkledistributor` 계약 스니펫입니다:


```solidity
  function claim(uint256 index, address account, uint256 amount, bytes32[] calldata merkleProof) external override {
        require(!isClaimed(index), 'MerkleDistributor: Drop already claimed.'); //@restricting each user to 1 successful claim
       //@audit -> this verifies if already claimed or not
        // Verify the merkle proof.
        bytes32 node = keccak256(abi.encodePacked(index, account, amount));
        require(MerkleProof.verify(merkleProof, merkleRoot, node), 'MerkleDistributor: Invalid proof.');

        // Mark it claimed and send the token.
        _setClaimed(index);
        require(IERC20(token).transfer(account, amount), 'MerkleDistributor: Transfer failed.');

        emit Claimed(index, account, amount);
    }
```

**파급력 (Impact):** 에어드랍 청구, NFT 민팅 또는 베스팅된 주식 청구와 같은 상황에서는 각 청구자가 수행하는 호출 수를 제한하는 것이 중요합니다. 중복에 대한 명시적인 처리 없이 `claim`과 같은 주요 기능에 대한 중복 액세스를 허용하면 프로토콜에 손실이 발생할 수 있습니다. 이러한 중복 처리를 전적으로 `claimContracts`에 의존하면 잠재적으로 공격 표면이 증가할 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):** 각 사용자에 대해 `claimContract`당 청구 횟수를 고려하는 `DropClaim` 로직 개선을 고려하는 것을 제안합니다. 이를 위해 사용자 주소와 성공적인 청구 횟수를 연결하는 매핑 시스템을 도입하는 것을 제안합니다.

각 `claimContract`에 대한 `minFee` 및 `expiry` 매개변수와 같은 기존 구성 옵션에 맞춰 `maxCallsPerContract`라는 새 매개변수를 추가하는 것을 권장합니다. 이 매개변수를 통해 소유자는 사용자가 특정 `claimContract`에 대해 수행할 수 있는 청구 횟수를 제한할 수 있습니다.

`DropClaim` 계약에서 제공하는 이 추가 보안 계층은 주어진 `claimContract`에 대해 사용자가 수행한 총 청구가 해당 특정 계약에 대해 설정된 `maxCallsPerContract` 값 미만인지 확인하는 것을 포함합니다. 가스 비용이 더 많이 들지만, 이 개선 사항은 중복 청구에 해당하는 공격 표면을 줄입니다.

**Bankless:** 확인했습니다. 의도된 기능이므로 팀은 이 발견 사항과 관련된 어떠한 변경도 하지 않을 것입니다.

**Cyfrin:** 확인했습니다.

\clearpage

## 가스 최적화 (Gas Optimization)


### 메모리에 복사하는 대신 저장소(storage) 포인터 사용

하나의 요소만 필요한 경우 구조체의 모든 요소를 복사하지 않으려면 메모리에 요소를 복사하는 대신 저장소 포인터를 사용하는 것이 더 효율적입니다.

```diff
// DropClaim::claim
-   ClaimData memory claimData = claims[getClaimContractHash(claimContract, salt)];
+   ClaimData storage claimData = claims[getClaimContractHash(claimContract, salt)];
```
[Line 100](https://github.com/kadenzipfel/drop-claim/blob/9fb36aab457b1ad3ea27351b004ddcdc5ef30682/src/DropClaim.sol#L100)

```diff
// DropClaim::batchClaim
-   ClaimData memory claimData = claims[getClaimContractHash(claimContracts[i], salts[i])];
+   ClaimData storage claimData = claims[getClaimContractHash(claimContracts[i], salts[i])];
```
[Line 131](https://github.com/kadenzipfel/drop-claim/blob/9fb36aab457b1ad3ea27351b004ddcdc5ef30682/src/DropClaim.sol#L131)

```diff
// DropClaim::_claim
-   ClaimData memory claimData = claims[getClaimContractHash(claimContract, salt)];
+   ClaimData storage claimData = claims[getClaimContractHash(claimContract, salt)];
```
[Line 200](https://github.com/kadenzipfel/drop-claim/blob/9fb36aab457b1ad3ea27351b004ddcdc5ef30682/src/DropClaim.sol#L200)

**Bankless:** 확인했으며 [commit 020bb4c49a281af32898f951b784d7748dac049f](https://github.com/kadenzipfel/drop-claim/commit/020bb4c49a281af32898f951b784d7748dac049f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### 상수 값에 대한 Bool 비교는 피해야 함

상수(true 또는 false)와 비교하는 것은 반환된 부울 값을 직접 확인하는 것보다 약간 더 비쌉니다.

```diff
// DropClaim::allowlistClaim
-   if (MerkleProof.verify(merkleProof, merkleRoot, bytes32(uint256(uint160(msg.sender)))) == false) {
+   if (!MerkleProof.verify(merkleProof, merkleRoot, bytes32(uint256(uint160(msg.sender))))) {
```
[Line 158](https://github.com/kadenzipfel/drop-claim/blob/9fb36aab457b1ad3ea27351b004ddcdc5ef30682/src/DropClaim.sol#L158)

```diff
// DropClaim::allowlistBatchClaim
-   if (MerkleProof.verify(merkleProof, merkleRoot, bytes32(uint256(uint160(msg.sender)))) == false) {
+   if (!MerkleProof.verify(merkleProof, merkleRoot, bytes32(uint256(uint160(msg.sender))))) {
```
[Line 181](https://github.com/kadenzipfel/drop-claim/blob/9fb36aab457b1ad3ea27351b004ddcdc5ef30682/src/DropClaim.sol#L181)

**Bankless:** 확인했으며 [commit 0d3ccd6eb7ad266be54598a52d321cb9bb17e7af](https://github.com/kadenzipfel/drop-claim/commit/0d3ccd6eb7ad266be54598a52d321cb9bb17e7af)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.
