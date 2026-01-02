**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[immeas](https://twitter.com/0ximmeas)

**Assisting Auditors**



---

# Findings
## High Risk


### Misconfigured index boundaries prevent certain swaps in QuantAMMWeightedPool

**Description:** Balancer 풀은 가중치 풀(weighted pools)에서 최대 8개의 토큰을 처리할 수 있습니다. QuantAMM은 이 기능을 사용하지만 두 개의 슬롯만 사용하여 스토리지를 최적화합니다. 결과적으로 처음 4개의 토큰과 마지막 4개의 토큰은 별도로 처리됩니다.

스왑의 경우, 이 로직은 [`QuantAMMWeightedPool::onSwap`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L242-L265)에 구현되어 있습니다:

```solidity
// if both tokens are within the first storage element
if (request.indexIn < 4 && request.indexOut < 4) {
    QuantAMMNormalisedTokenPair memory tokenWeights = _getNormalisedWeightPair(
        request.indexIn,
        request.indexOut,
        timeSinceLastUpdate,
        totalTokens
    );
    tokenInWeight = tokenWeights.firstTokenWeight;
    tokenOutWeight = tokenWeights.secondTokenWeight;
} else if (request.indexIn > 4 && request.indexOut < 4) {
    // if the tokens are in different storage elements
    QuantAMMNormalisedTokenPair memory tokenWeights = _getNormalisedWeightPair(
        request.indexOut,
        request.indexIn,
        timeSinceLastUpdate,
        totalTokens
    );
    tokenInWeight = tokenWeights.firstTokenWeight;
    tokenOutWeight = tokenWeights.secondTokenWeight;
} else {
    tokenInWeight = _getNormalizedWeight(request.indexIn, timeSinceLastUpdate, totalTokens);
    tokenOutWeight = _getNormalizedWeight(request.indexOut, timeSinceLastUpdate, totalTokens);
}
```

문제는 두 토큰이 모두 두 번째 슬롯에 있는 경우를 처리하기 위한 두 번째 `else if` 블록에서 발생합니다. 그러나 조건 `request.indexOut < 4`는 올바르지 않습니다. 이는 `request.indexOut >= 4`여야 합니다. 결과적으로 [`_getNormalisedWeightPair`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L305-L325)에서 코드가 되돌아갑니다(revert):

```solidity
function _getNormalisedWeightPair(
    uint256 tokenIndexOne, // @audit indexOut
    uint256 tokenIndexTwo, // @audit indexIn
    // ...
) internal view virtual returns (QuantAMMNormalisedTokenPair memory) {
    uint256 firstTokenIndex = tokenIndexOne; // @audit indexOut
    uint256 secondTokenIndex = tokenIndexTwo; // @audit indexIn
    // ...
    if (tokenIndexTwo > 4) { // @audit indexIn will always be > 4 from the branch above
        firstTokenIndex = tokenIndexOne - 4; // @audit indexOut, must be < 4 hence underflow
        secondTokenIndex = tokenIndexTwo - 4;
        totalTokensInPacked -= 4;
        targetWrappedToken = _normalizedSecondFourWeights;
    } else {
```

여기서 `tokenIndexOne - 4`는 잘못된 조건으로 인해 언더플로가 발생합니다.

[`QuantAMMWeightedPool::_getNormalizedWeight`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L370-L404)에서도 비슷한 문제가 있는데, 조건 `tokenIndex > 4`는 `tokenIndex >= 4`여야 합니다:

```solidity
if (tokenIndex > 4) { // @audit should be >=
    // get the index in the second storage element
    index = tokenIndex - 4;
    targetWrappedToken = _normalizedSecondFourWeights;
    tokenIndexInPacked -= 4;
} else {
    // @audit first slot
    if (totalTokens > 4) {
         tokenIndexInPacked = 4;
    }
    targetWrappedToken = _normalizedFirstFourWeights;
```

이로 인해 [`QuantAMMWeightedPool::_calculateCurrentBlockWeight`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L352-L368)에서 승수(multiplier)를 가져올 때 실패가 발생합니다:

```solidity
int256 blockMultiplier = tokenWeights[tokenIndex + (tokensInTokenWeights)];
```

이 경우 `tokenWeights`에는 8개의 항목이 있지만 `tokenIndex + (tokensInTokenWeights)`는 8이 되어 배열 경계 초과 오류가 발생합니다.

동일한 문제가 다음 위치에도 존재하지만 현재는 영향을 미치지 않습니다:

- [`QuantAMMWeightedPool::onSwap`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L252)에서:
  ```solidity
  } else if (request.indexIn > 4 && request.indexOut < 4) { // @audit should be >=
  ```

- [`QuantAMMWeightedPool::_getNormalisedWeightPair`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L320)에서:
  ```solidity
  if (tokenIndexTwo > 4) { // @audit should be >=
  ```

**Impact:**
1. `indexIn > 4`이고 `indexOut < 4`인 스왑은 불가능합니다.
2. 토큰 인덱스 4와 관련된 스왑을 수행할 수 없습니다.
3. `indexIn > 4`이고 `indexOut < 4`인 스왑이 가능하다면 스왑 금액이 잘못 계산될 수 있습니다.

**Proof of Concept:** 문제를 보여주기 위해 `pkg/pool-quantamm/test/foundry/QuantAMMWeightedPool8Token.t.sol`에 다음 테스트를 추가하십시오:

```solidity
// cross swap not working correctly
function testGetNormalizedWeightOnSwapOutGivenInNBlocksAfterToken7Token3() public {
    testParam memory firstWeight = testParam(7, 0.1e18, 0.001e18); // indexIn > 4
    testParam memory secondWeight = testParam(3, 0.15e18, 0.001e18); // indexOut < 4
    // will revert on underflow
    _onSwapOutGivenInInternal(firstWeight, secondWeight, 2, 1.006410772600252500e18);
}
// index >= 4 not working correctly
function testGetNormalizedWeightOnSwapOutGivenInInitialToken0Token4() public {
    testParam memory firstWeight = testParam(0, 0.1e18, 0.001e18);
    testParam memory secondWeight = testParam(4, 0.15e18, 0.001e18);
    // fails with `panic: array out-of-bounds access`
    _onSwapOutGivenInInternal(firstWeight, secondWeight, 0, 0.499583703357018000e18);
}
// index >= 4 not working correctly
function testGetNormalizedWeightOnSwapOutGivenInInitialToken4Token0() public {
    testParam memory firstWeight = testParam(4, 0.1e18, 0.001e18);
    testParam memory secondWeight = testParam(0, 0.15e18, 0.001e18);
    // fails with `panic: array out-of-bounds access`
    _onSwapOutGivenInInternal(firstWeight, secondWeight, 0, 0.887902403682279000e18);
}
```

**Recommended Mitigation:**
1. `QuantAMMWeightedPool::onSwap`의 **`if` 블록을 다음과 같이 리팩터링**하십시오:

   ```solidity
   if (request.indexIn < 4 && request.indexOut < 4 || request.indexIn >= 4 && request.indexOut >= 4) {
       // Same slot; _getNormalisedWeightPair handles the correct logic
       tokenWeights = _getNormalisedWeightPair(...);
   } else {
       // Cross-slot handling
       tokenWeights = _getNormalizedWeight(...);
   }
   ```

2. 다음 라인에서 조건 `> 4`를 `>= 4`로 **업데이트**하십시오:

   - [Line 320](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L320)
   - [Line 383](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L383)

**QuantAmm:**
[`414d4bc`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/414d4bc3d8699f1e4620f0060226b4e23ff3b010)에서 수정됨

**Cyfrin:** 확인됨.


### Token Index error in `getNormalizedWeights` calculates incorrect weight leading to incorrect pool asset allocation

**Description:** `QuantAMMWeightedPool.sol`의 `_getNormalizedWeights()` 함수는 4개 이상의 토큰이 있는 풀에 대한 승수를 처리할 때 치명적인 인덱싱 오류를 포함하고 있습니다. 이 오류로 인해 토큰 4-7에 대한 보간된 가중치를 계산할 때 잘못된 승수 값이 사용되어 잠재적으로 심각한 자산 할당 오류가 발생할 수 있습니다.

이 문제는 4개 이상의 토큰이 있는 풀에 대한 두 번째 스토리지 슬롯을 처리할 때 `_getNormalizedWeights()` 함수에서 발생합니다:

```solidity
function _getNormalizedWeights() internal view virtual returns (uint256[] memory) {
    // ...
    uint256 tokenIndex = totalTokens;
    if (totalTokens > 4) {
        tokenIndex = 4;  // @audit Sets to 4 for first slot multipliers
    }

    // First slot handles correctly
    normalizedWeights[0] = calculateBlockNormalisedWeight(
        firstFourWeights[0],
        firstFourWeights[tokenIndex],  // Uses indices 4-7 for multipliers
        timeSinceLastUpdate
    );
    // ...

    // Second slot has indexing error
    if (totalTokens > 4) {
        tokenIndex -= 4;  // @audit Resets to 0, breaking multiplier indices
        int256[] memory secondFourWeights = quantAMMUnpack32(_normalizedSecondFourWeights);
        normalizedWeights[4] = calculateBlockNormalisedWeight(
            secondFourWeights[0],
            secondFourWeights[tokenIndex],  // @audit Uses wrong indices 0-3 for multipliers -> essentially weight and multiplier are same
            timeSinceLastUpdate
        );
        // ...
    }
}
```
두 번째 토큰 슬롯(인덱스 4-7)을 처리할 때:
- 토큰 합계 > 4인 경우 첫 번째 tokenIndex는 4로 설정됨
- tokenIndex가 4만큼 감소하여 0으로 재설정됨
- 모든 후속 가중치 계산의 경우 가중치 및 승수가 사실상 동일함

**Impact:**
- 4개 이상의 토큰이 있는 풀에서 토큰 4-7에 대한 잘못된 가중치 계산
- 의도한 풀 자산 할당 비율에서 크게 벗어남


**Recommended Mitigation:** 풀에 토큰이 4개 이상인 경우 tokenIndex를 4만큼 줄이지 마십시오.

**QuantAMM:** [`60815f2`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/60815f262748579cba99bafb640e7931d6f13c47)에서 수정됨

**Cyfrin:** 확인됨


### Incorrect handling of negative multipliers in `QuantAMMWeightedPool` leads to underflow in weight calculation

**Description:** [`QuantAMMWeightedPool::calculateBlockNormalisedWeight`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L505-L521)에서:
```solidity
if (multiplier > 0) {
    return uint256(weight) + FixedPoint.mulDown(uint256(multiplierScaled18), timeSinceLastUpdate);
} else {
    // @audit silent overflow, uint256(multiplierScaled18) of a negative value will result in a very large value
    return uint256(weight) - FixedPoint.mulUp(uint256(multiplierScaled18), timeSinceLastUpdate);
}
```

**Impact:** 업데이트가 발생한 동일한 블록에 있지 않은(`timeSinceLastUpdate != 0`) 음수 승수가 있는 모든 스왑은 언더플로로 인해 실패합니다.

**Proof of Concept:** 다음 테스트를 `pkg/pool-quantamm/test/foundry/QuantAMMWeightedPool8Token.t.sol`에 추가하십시오:
```solidity
function testGetNormalizeNegativeMultiplierOnSwapOutGivenInInitialToken0Token1() public {
    testParam memory firstWeight = testParam(0, 0.1e18, 0.001e18);
    testParam memory secondWeight = testParam(1, 0.15e18, -0.001e18);
    _onSwapOutGivenInInternal(firstWeight, secondWeight, 2, 1.332223208952048000e18);
}
```

**Recommended Mitigation:** 먼저 `-1`을 곱하는 것을 고려하십시오:
```diff
- return uint256(weight) - FixedPoint.mulUp(uint256(multiplierScaled18), timeSinceLastUpdate);
+ return uint256(weight) - FixedPoint.mulUp(uint256(-multiplierScaled18), timeSinceLastUpdate);
```

**QuantAMM:** 커밋 [`31636cf`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/31636cf736e9f6eda09f569bceb4e8d9bd3137bb)에서 수정됨

**Cyfrin:** 확인됨.


### Mismatch in multiplier position causes incorrect weight calculations in `QuantAMMWeightedPool`

**Description:** QuantAMM 가중치 풀의 주요 기능은 풀 생성 중에 설정된 다양한 거래 규칙에 따라 가중치를 주기적으로 업데이트하는 것입니다. 이러한 가중치는 `UpdateWeightRunner`라는 싱글톤 컨트랙트에 의해 업데이트되며, 이 컨트랙트는 새로운 가중치와 승수를 제공합니다. 승수는 다음 업데이트까지 블록 간 가중치를 조정하는 데 도움이 됩니다.

Balancer 가중치 풀에 비해 가스 사용량을 최적화하기 위해 QuantAMM은 가중치와 승수를 단 두 개의 스토리지 슬롯에 압축합니다.

업데이트가 발생하면 `UpdateWeightRunner`는 [`QuantAMMWeightedPool::setWeights`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L588-L615)에 새로운 가중치를 보냅니다. 그러나 가중치와 승수는 `QuantAMMWeightedPool::_splitWeightAndMultipliers`에 대한 [NatSpec 주석](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L696)에 설명된 대로 풀이 예상하는 것과는 다른 형식으로 전송됩니다:

```solidity
/// @dev Update weight runner gives all weights in a single array shaped like [w1,w2,w3,w4,w5,w6,w7,w8,m1,m2,m3,m4,m5,m6,m7,m8], we need it to be [w1,w2,w3,w4,m1,m2,m3,m4,w5,w6,w7,w8,m5,m6,m7,m8]
```

그런 다음 데이터는 다음과 같이 두 개의 스토리지 슬롯으로 분할됩니다:
- **첫 번째 슬롯:** `[w1, w2, w3, w4, m1, m2, m3, m4]`
- **두 번째 슬롯:** `[w5, w6, w7, w8, m5, m6, m7, m8]`

두 개의 토큰만 있는 풀의 경우 `UpdateWeightRunner`는 가중치와 승수를 `[w1, w2, m1, m2]`로 보내며, 이는 동일한 순서로 저장됩니다(끝에 0이 패딩됨). 그러나 더 많은 토큰(예: 6개 토큰)이 있는 풀의 경우 데이터를 두 개의 슬롯으로 분할하려면 더 복잡한 로직이 필요합니다. 이는 [`QuantAMMWeightedPool::_splitWeightAndMultipliers`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L694-L722)에서 처리합니다.

`UpdateWeightRunner`가 6개 토큰에 대한 가중치와 승수를 `[w1, w2, w3, w4, w5, w6, m1, m2, m3, m4, m5, m6]`로 보내면 [첫 번째 슬롯](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L700-L710)은 다음과 같이 채워집니다:

```solidity
uint256 tokenLength = weights.length / 2;
splitWeights = new int256[][](2);
splitWeights[0] = new int256[](8);
for (uint i; i < 4; ) {
    splitWeights[0][i] = weights[i];
    splitWeights[0][i + 4] = weights[i + tokenLength];


    unchecked {
        i++;
    }
}
```

이 결과 첫 번째 슬롯은 `[w1, w2, w3, w4, m1, m2, m3, m4]`가 됩니다.

[두 번째 슬롯](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L711-L721)은 다음 로직으로 처리됩니다:

```solidity
splitWeights[1] = new int256[](8);
uint256 moreThan4Tokens = tokenLength - 4;
for (uint i = 0; i < moreThan4Tokens; ) {
    uint256 i4 = i + 4;
    splitWeights[1][i] = weights[i4];
    splitWeights[1][i4] = weights[i4 + tokenLength];

    unchecked {
        i++;
    }
}
```

6개 토큰의 경우, 결과적으로 두 번째 슬롯은 `[w5, w6, 0, 0, m5, m6, 0, 0]`이 됩니다.

이 문제는 [`QuantAMMWeightedPool::_getNormalizedWeight`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L381-L388)에서 이 값을 읽을 때 발생합니다:

```solidity
uint256 tokenIndexInPacked = totalTokens;

if (tokenIndex > 4) {
    //get the index in the second storage int
    index = tokenIndex - 4;
    targetWrappedToken = _normalizedSecondFourWeights;
    tokenIndexInPacked -= 4;
} else {
```

여기서 `tokenIndexInPacked`는 `6 - 4 = 2`가 되며, 이는 [`QuantAMMWeightedPool::_calculateCurrentBlockWeight`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L364)에서 사용됩니다:

```solidity
int256 blockMultiplier = tokenWeights[tokenIndex + (tokensInTokenWeights)];
```
토큰 인덱스 5 또는 두 번째 슬롯의 인덱스 1의 경우, `1 + 2 = 3` 위치의 승수에 액세스하려고 시도하지만 승수는 실제로는 5번째 위치에 있습니다. 이 불일치로 인해 잘못된 승수가 사용되거나 승수가 사용되지 않습니다.

**Impact:** 승수를 잘못 읽으면 토큰 스왑 중 잘못된 가중치가 적용됩니다. 결과적으로 스왑 금액이 올바르지 않아 사용자가 의도하지 않고 잠재적으로 유해한 결과를 초래할 수 있습니다.

**Proof of Concept:** 이 테스트를 `pkg/pool-quantamm/test/foundry/QuantAMMWeightedPool8Token.t.sol`에 추가하십시오:
```solidity
function testGetNormalizedWeightsInitial6Tokens() public {
    int256 weight = int256(1e18) / 6 + 1;
    PoolRoleAccounts memory roleAccounts;
    IERC20[] memory tokens = [
        address(dai),
        address(usdc),
        address(weth),
        address(wsteth),
        address(veBAL),
        address(waDAI)
    ].toMemoryArray().asIERC20();
    MockMomentumRule momentumRule = new MockMomentumRule(owner);

    int256[] memory initialWeights = new int256[](6);
    initialWeights[0] = weight;
    initialWeights[1] = weight;
    initialWeights[2] = weight - 1;
    initialWeights[3] = weight - 1;
    initialWeights[4] = weight;
    initialWeights[5] = weight;

    uint256[] memory initialWeightsUint = new uint256[](6);
    initialWeightsUint[0] = uint256(weight);
    initialWeightsUint[1] = uint256(weight);
    initialWeightsUint[2] = uint256(weight - 1);
    initialWeightsUint[3] = uint256(weight - 1);
    initialWeightsUint[4] = uint256(weight);
    initialWeightsUint[5] = uint256(weight);

    uint64[] memory lambdas = new uint64[](1);
    lambdas[0] = 0.2e18;

    int256[][] memory parameters = new int256[][](1);
    parameters[0] = new int256[](1);
    parameters[0][0] = 0.2e18;

    address[][] memory oracles = new address[][](1);
    oracles[0] = new address[](1);
    oracles[0][0] = address(chainlinkOracle);

    QuantAMMWeightedPoolFactory.NewPoolParams memory params = QuantAMMWeightedPoolFactory.NewPoolParams(
        "Pool With Donation",
        "PwD",
        vault.buildTokenConfig(tokens),
        initialWeightsUint,
        roleAccounts,
        MAX_SWAP_FEE_PERCENTAGE,
        address(0),
        true,
        false, // Do not disable unbalanced add/remove liquidity
        ZERO_BYTES32,
        initialWeights,
        IQuantAMMWeightedPool.PoolSettings(
            new IERC20[](6),
            IUpdateRule(momentumRule),
            oracles,
            60,
            lambdas,
            0.01e18,
            0.01e18,
            0.01e18,
            parameters,
            address(0)
        ),
        initialWeights,
        initialWeights,
        3600,
        0,
        new string[][](0)
    );

    address quantAMMWeightedPool = quantAMMWeightedPoolFactory.create(params);

    uint256[] memory weights = QuantAMMWeightedPool(quantAMMWeightedPool).getNormalizedWeights();

    int256[] memory newWeights = new int256[](12);
    newWeights[0] = weight;
    newWeights[1] = weight;
    newWeights[2] = weight - 1;
    newWeights[3] = weight - 1;
    newWeights[4] = weight;
    newWeights[5] = weight;
    newWeights[6] = 0.001e18;
    newWeights[7] = 0.001e18;
    newWeights[8] = 0.001e18;
    newWeights[9] = 0.001e18;
    newWeights[10] = 0.001e18;
    newWeights[11] = 0.001e18;

    vm.prank(address(updateWeightRunner));
    QuantAMMWeightedPool(quantAMMWeightedPool).setWeights(
        newWeights,
        quantAMMWeightedPool,
        uint40(block.timestamp + 5)
    );

    uint256[] memory balances = new uint256[](6);
    balances[0] = 1000e18;
    balances[1] = 1000e18;
    balances[2] = 1000e18;
    balances[3] = 1000e18;
    balances[4] = 1000e18;
    balances[5] = 1000e18;

    PoolSwapParams memory swapParams = PoolSwapParams({
        kind: SwapKind.EXACT_IN,
        amountGivenScaled18: 1e18,
        balancesScaled18: balances,
        indexIn: 0,
        indexOut: 1,
        router: address(router),
        userData: abi.encode(0)
    });

    vm.warp(block.timestamp + 5);

    vm.prank(address(vault));
    uint256 swap1Balance = QuantAMMWeightedPool(quantAMMWeightedPool).onSwap(swapParams);

    swapParams.indexOut = 5;

    vm.prank(address(vault));
    uint256 swap5Balance = QuantAMMWeightedPool(quantAMMWeightedPool).onSwap(swapParams);

    assertEq(swap1Balance, swap5Balance);
}
```

**Recommended Mitigation:** 두 번째 슬롯을 첫 번째 슬롯과 동일한 방식으로 저장하는 것을 고려하십시오:
```diff
- splitWeights[1][i4] = weights[i4 + tokenLength];
+ splitWeights[1][i + moreThan4Tokens] = weights[i4 + tokenLength];
```
이것은 두 번째 가중치 슬롯에서 4의 거리를 가정하므로 `getNormalizedWeights`에 대한 변경도 필요합니다.


**[Project]:**
커밋 [`31636cf`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/31636cf736e9f6eda09f569bceb4e8d9bd3137bb) 및 [`8a6b3b2`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/8a6b3b2d8ecd8cb52ecd652a0c3e02ab4964e8df)에서 수정됨

**Cyfrin:** 확인됨.

\clearpage
## Medium Risk


### Incorrect range validation in `quantAMMPackEight32` allows invalid values to be packed

**Description:** `ScalarQuantAMMBaseStorage::quantAMMPackEight32` 함수는 8개 입력 매개변수 중 6개에 대한 최소 경계를 올바르게 확인하지 못하는 잘못된 유효성 검사 로직을 포함하고 있습니다. require 문은 각 입력 매개변수를 확인하는 대신 최소 경계 검사에 `_firstInt`를 잘못 재사용합니다.

```solidity
require(
    _firstInt <= MAX32 &&
        _firstInt >= MIN32 &&
        _secondInt <= MAX32 &&
        _secondInt >= MIN32 &&
        _thirdInt <= MAX32 &&
        _firstInt >= MIN32 &&  // @audit should be _thirdInt
        _fourthInt <= MAX32 &&
        _firstInt >= MIN32 &&  // @audit should be _fourthInt
        _fifthInt <= MAX32 &&
        _firstInt >= MIN32 &&  // @audit should be _fifthInt
        _sixthInt <= MAX32 &&
        _firstInt >= MIN32 &&  // @audit should be _sixthInt
        _seventhInt <= MAX32 &&
        _firstInt >= MIN32 &&  // @audit should be _seventhInt
        _eighthInt <= MAX32 &&
        _firstInt >= MIN32,    // @audit should be _eighthInt
    "Overflow"
);
```
이 함수는 `quantAMMPack32Array`에 의해 호출되며, 이는 차례로 `QuantAMMWeightedPool::setWeights` 함수에서 가중치 값을 압축하는 데 사용됩니다.

**Impact:** `setWeights` 함수는 접근 제어에 의해 보호되지만 (`updateWeightRunner`에 의해서만 호출 가능), 잘못된 값을 압축하도록 허용하면 가중치 계산에서 예상치 못한 동작이 발생할 수 있습니다.

**Proof of Concept:** TBD

**Recommended Mitigation:** 모든 입력 매개변수에 대해 최소 경계를 적절하게 검증하도록 require 문을 업데이트하는 것을 고려하십시오:

**QuantAmm:**
우리는 initialise 함수에서 사용되는 setInitialWeights에서 팩을 사용합니다. 절대 가드 레일 로직 등을 감안할 때 풀이 실행되는 동안 이것이 트리거되는 것은 수학적으로 불가능하다고 생각합니다. 풀이 나쁜 가중치로 초기화되면 풀의 배포가 실패할 수 있습니다. 커밋 [`408ba20`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/408ba203c1b6c4abfd3474e4baa448af3752ea6e)에서 수정됨.

**Cyfrin:** 확인됨.


### Lack of permission check allows unauthorized updates to `lastPoolUpdateRun`

**Description:** 규칙에 의해 계산된 가중치 변경을 적용하기 위해, 모든 사용자는 [`UpdateWeightRunner::performUpdate`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L256-L279)를 호출할 수 있습니다.

각 풀에는 `QuantAMMWeightedPool.poolRegistry`의 배포 중에 구성된 권한 세트가 있습니다. 이 레지스트리의 각 비트는 특정 권한을 나타냅니다. `performUpdate`를 실행하려면 `MASK_POOL_PERFORM_UPDATE` 비트가 설정되어야 합니다.

`performUpdate`가 너무 자주 호출되는 것을 방지하기 위해(사용된 알고리즘의 스무딩에 영향을 줄 수 있음), 각 풀에는 업데이트 사이의 최소 시간을 나타내는 `updateInterval`이 구성되어 있습니다.

관리자는 [`UpdateWeightRunner::InitialisePoolLastRunTime`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L289-L304)을 호출하여 이 제한을 무시할 수 있습니다.

`InitialisePoolLastRunTime`을 호출하는 기능은 권한에 따라 제한됩니다. 풀은 `MASK_POOL_DAO_WEIGHT_UPDATES` 또는 `MASK_POOL_OWNER_UPDATES` 비트가 설정되어 있어야 [하며](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L295-L302), 그에 따라 호출자의 신원을 확인합니다:

```solidity
// Current breakglass settings allow DAO or pool creator to trigger updates. This is subject to review.
if (poolRegistryEntry & MASK_POOL_DAO_WEIGHT_UPDATES > 0) {
    address daoRunner = QuantAMMBaseAdministration(quantammAdmin).daoRunner();
    require(msg.sender == daoRunner, "ONLYDAO");
} else if (poolRegistryEntry & MASK_POOL_OWNER_UPDATES > 0) {
    require(msg.sender == poolRuleSettings[_poolAddress].poolManager, "ONLYMANAGER");
}
poolRuleSettings[_poolAddress].timingSettings.lastPoolUpdateRun = _time;
```

이러한 권한 중 어느 것도 설정되지 않은 경우 누구나 `InitialisePoolLastRunTime`을 호출할 수 있습니다.

**Impact:** 악의적인 사용자는 다음을 수행할 수 있습니다:

1. `lastPoolUpdateRun`을 부적절한 시간으로 수정하여 **합법적인 업데이트를 방지**합니다.
2. `lastPoolUpdateRun`을 먼 과거 시간으로 설정하여 `updateInterval` 제한을 **우회**하고 의도한 것보다 빨리 업데이트가 수행되도록 허용합니다.


**Proof of Concept:** 다음 테스트를 `pkg/pool-quantamm/test/foundry/UpdateWeightRunner.t.sol`에 추가하십시오:
```solidity
function testAnyoneCanUpdatePoolLastRunTime() public {
    // pool lacks MASK_POOL_DAO_WEIGHT_UPDATES and MASK_POOL_OWNER_UPDATES but has MASK_POOL_PERFORM_UPDATE
    mockPool.setPoolRegistry(1);

    vm.startPrank(owner);
    // Deploy oracles with fixed values and delay
    chainlinkOracle1 = deployOracle(1001, 0);
    chainlinkOracle2 = deployOracle(1002, 0);

    updateWeightRunner.addOracle(chainlinkOracle1);
    updateWeightRunner.addOracle(chainlinkOracle2);
    vm.stopPrank();

    address[][] memory oracles = new address[][](2);
    oracles[0] = new address[](1);
    oracles[0][0] = address(chainlinkOracle1);
    oracles[1] = new address[](1);
    oracles[1][0] = address(chainlinkOracle2);

    uint64[] memory lambda = new uint64[](1);
    lambda[0] = 0.0000000005e18;
    vm.prank(address(mockPool));
    updateWeightRunner.setRuleForPool(
        IQuantAMMWeightedPool.PoolSettings({
            assets: new IERC20[](0),
            rule: IUpdateRule(mockRule),
            oracles: oracles,
            updateInterval: 10,
            lambda: lambda,
            epsilonMax: 0.2e18,
            absoluteWeightGuardRail: 0.2e18,
            maxTradeSizeRatio: 0.2e18,
            ruleParameters: new int256[][](0),
            poolManager: addr2
        })
    );

    address anyone = makeAddr("anyone");

    vm.prank(anyone);
    updateWeightRunner.InitialisePoolLastRunTime(address(mockPool), uint40(block.timestamp));

    vm.expectRevert("Update not allowed");
    updateWeightRunner.performUpdate(address(mockPool));
}
```

**Recommended Mitigation:** 적절한 권한이 설정되지 않은 경우 되돌리도록(revert) `else` 절을 추가하십시오. 이렇게 하면 승인된 사용자만 `lastPoolUpdateRun`을 수정할 수 있습니다:

```diff
}
+ else {
+     revert("No permission to set lastPoolUpdateRun");
+ }
poolRuleSettings[_poolAddress].timingSettings.lastPoolUpdateRun = _time;
```

**QuantAMM:** [`a69c06f`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a69c06f27ce52a2a7c42bfad23b0de9da7d766e9) 및 [`84f506f`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/84f506f7e61e07c36d354f428b0e0f7d226de0d9)에서 수정됨

**Cyfrin:** 확인됨. 승인되지 않은 사용자에 대한 `InitialisePoolLastRunTime` 호출이 이제 되돌려집니다.


### Faulty permission check in `UpdateWeightRunner::getData` allows unauthorized access

**Description:** 풀에 대한 특정 작업은 권한에 의해 보호됩니다. 그러한 권한 중 하나는 `POOL_GET_DATA`로, 이는 [`UpdateWeightRunner::getData`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L315-L366) 함수에 대한 액세스를 제어합니다:

```solidity
bool internalCall = msg.sender != address(this);
require(internalCall || approvedPoolActions[_pool] & MASK_POOL_GET_DATA > 0, "Not allowed to get data");
```

문제는 `internalCall` 조건이 강제되는 방식에 있습니다. 변수 `internalCall`은 호출이 컨트랙트에 의해 내부적으로 이루어졌는지 확인하기 위한 것입니다. 그러나 이 로직은 잘못 동작합니다:

- `UpdateWeightRunner`는 호출 컨텍스트를 변경하는 방식(즉, `address(this).getData()`)으로 `getData`를 호출하지 않으므로 `internalCall`은 항상 `true`가 됩니다. 결과적으로 `msg.sender`는 `address(this)`와 절대 같지 않습니다.
- 결과적으로 `msg.sender != address(this)` 문은 항상 `true`로 평가되어 풀의 권한에 관계없이 `require` 조건이 통과됩니다.

결과적으로 `POOL_GET_DATA` 권한이 효과적으로 우회됩니다.

**Impact:** `UpdateWeightRunner::getData` 함수는 `POOL_GET_DATA` 권한이 없는 풀에서도 호출될 수 있습니다.

**Proof of Concept:** 다음 테스트를 `pkg/pool-quantamm/test/foundry/UpdateWeightRunner.t.sol`에 추가하십시오:
```solidity
function testUpdateWeightRunnerGetDataPermissionCanByBypassed() public {
    vm.prank(owner);
    updateWeightRunner.setApprovedActionsForPool(address(mockPool), 1);

    assertEq(updateWeightRunner.getPoolApprovedActions(address(mockPool)) & 2 /* MASK_POOL_GET_DATA */, 0);

    // this should revert as the pool is not allowed to get data
    updateWeightRunner.getData(address(mockPool));
}
```

**Recommended Mitigation:** `internalCall` 부울을 완전히 제거하는 것을 고려하십시오. 이 변경으로 올바른 `POOL_GET_DATA` 권한이 있는 호출만 허용됩니다. 또한 이 수정으로 끝부분의 [이벤트 방출](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L363-L365)이 불필요해지므로 `getData`를 `view` 함수로 만들 수 있습니다.

권장되는 코드 변경 사항은 다음과 같습니다:
```diff
- function getData(address _pool) public returns (int256[] memory outputData) {
+ function getData(address _pool) public view returns (int256[] memory outputData) {
-     bool internalCall = msg.sender != address(this);
-     require(internalCall || approvedPoolActions[_pool] & MASK_POOL_GET_DATA > 0, "Not allowed to get data");
+     require(approvedPoolActions[_pool] & MASK_POOL_GET_DATA > 0, "Not allowed to get data");
// ...
-     if(!internalCall){
-         emit GetData(msg.sender, _pool);
-     }
```

**QuantAMM:** [`a1a333d`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a1a333d28b0aae82338f4ba440af1a473ef1eda6)에서 수정됨

**Cyfrin:** 확인됨. 코드가 리팩터링되었으며 `internalCall`은 이제 새로운 `UpdateWeightRunner::_getData`에 매개변수로 전달됩니다. 원래 `UpdateWeightRunner::getData`는 권한이 필요하지만 업데이트 중 내부 호출은 그렇지 않습니다.


### Stale Oracle prices accepted when no backup oracles available

**Description:** `UpdateWeightRunner::getData` 함수는 가격 데이터가 최신 상태인지 확인하기 위해 오라클 가격에 대한 최신성 검사를 수행합니다. 그러나 최적화된(기본) 오라클이 오래된 데이터를 반환하고 구성된 백업 오라클이 없는 경우, 함수는 되돌리는 대신 오래된 가격 데이터를 조용히 수락하고 반환합니다.

이는 로직이 기본 오라클의 최신성을 확인하지만 오래된 경우 백업 오라클 확인 섹션으로 이동하기 때문에 발생합니다.
백업 오라클이 존재하지 않으면 배열 길이 확인으로 인해 이 섹션을 건너뛰고 함수는 기본 오라클의 오래된 데이터를 계속 사용합니다.

```solidity
// getData function
function getData(address _pool) public returns (int256[] memory outputData) {
         // .... code
        outputData = new int256[](oracleLength);
        uint oracleStalenessThreshold = IQuantAMMWeightedPool(_pool).getOracleStalenessThreshold();

        for (uint i; i < oracleLength; ) {
            // Asset is base asset
            OracleData memory oracleResult;
            oracleResult = _getOracleData(OracleWrapper(optimisedOracles[i]));
            if (oracleResult.timestamp > block.timestamp - oracleStalenessThreshold) {
                outputData[i] = oracleResult.data; //@audit skips this correctly when data is stale
            } else {
                unchecked {
                    numAssetOracles = poolBackupOracles[_pool][i].length;
                }

                for (uint j = 1 /*0 already done via optimised poolOracles*/; j < numAssetOracles; ) { //@audit for loop is skipped when no backup oracle
                    oracleResult = _getOracleData(
                        // poolBackupOracles[_pool][asset][oracle]
                        OracleWrapper(poolBackupOracles[_pool][i][j])
                    );
                    if (oracleResult.timestamp > block.timestamp - oracleStalenessThreshold) {
                        // Oracle has fresh values
                        break;
                    } else if (j == numAssetOracles - 1) {
                        // All oracle results for this data point are stale. Should rarely happen in practice with proper backup oracles.

                        revert("No fresh oracle values available");
                    }
                    unchecked {
                        ++j;
                    }
                }
                outputData[i] = oracleResult.data; //@audit BUG - here it accepts the same stale data that was rejected before
            }

            unchecked {
                ++i;
            }
        }

        if(!internalCall){
            emit GetData(msg.sender, _pool);
        }
    }

```

**Impact:** 오래된 오라클 가격이 풀 가중치 업데이트를 계산하는 데 사용될 수 있으며, 이는 오래된 시장 정보에 기반한 잘못된 가중치 조정으로 이어질 수 있습니다.

**Recommended Mitigation:** `numAssetOracles == 1`일 때 `else` 블록에서 되돌리는 것을 고려하십시오.

```solidity
 else {
    unchecked {
        numAssetOracles = poolBackupOracles[_pool][i].length;
    }

    if (numAssetOracles == 1) {  // @audit revert when No backups available (1 accounts for primary oracle)
        revert("No fresh oracle values available");
    }

    for (uint j = 1; j < numAssetOracles; ) {
        // ... rest of the code
    }
}
```

**QuantAMM:** 오래된 업데이트는 추정자가 가중치가 적용된 가격과 일정 수준의 스무딩을 캡슐화하는 방식 때문에 없는 것보다 낫습니다. 가격이 멀어질수록 덜 중요해집니다. 따라서 업데이트를 트리거하는 것은 현재의 오래된 가격일 수 있지만, 역사적 가격이 중요도 측면에서 지수 스무딩에 포함되어 궤도를 유지함을 의미합니다.

**Cyfrin:** 인지됨.


### Missing admin functions prevent execution of key `UpdateWeightRunner` actions

**Description:** `UpdateWeightRunner`의 여러 함수는 `quantammAdmin` 역할을 필요로 하지만, 이 함수들은 `QuantAMMBaseAdministration`에 누락되어 있습니다. 영향을 받는 함수는 다음과 같습니다:

* [`UpdateWeightRunner::addOracle`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L182-L195)
* [`UpdateWeightRunner::removeOracle`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L197-L203)
* [`UpdateWeightRunner::setApprovedActionsForPool`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L205-L209)
* [`UpdateWeightRunner::setETHUSDOracle`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L281-L287)

이 함수들은 `quantammAdmin`에 의존하므로, 해당 호출이 `QuantAMMBaseAdministration`에 포함되지 않는 한 실행될 수 없습니다.

**Impact:** `QuantAMMBaseAdministration` 컨트랙트는 위의 함수를 실행할 수 없습니다. 결과적으로 오라클 추가 또는 제거, 풀에 대한 승인된 작업 설정, ETH/USD 오라클 업데이트와 같은 관리 작업을 수행할 수 없어 시스템의 기능과 유연성이 제한됩니다.

**Recommended Mitigation:** 다음 접근 방식 중 하나를 고려하십시오:

1. 누락된 함수를 `QuantAMMBaseAdministration`에 추가
2. OpenZeppelin의 `TimeLock` 컨트랙트를 직접 사용:
   `QuantAMMBaseAdministration`이 중복된다면, 관리 기능을 안전하고 효과적으로 관리하기 위해 OpenZeppelin의 `TimeLock` 컨트랙트로 교체하십시오.

**QuantAMM:** [`a299ce7`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a299ce72076b58503386bc146b81014560536d9e)에서 수정됨

**Cyfrin:** 확인됨. `QuantAMMBaseAdministration`이 제거되었습니다.


### `QuantAMMBaseAdministration::onlyExecutor` modifier does not enforce a time lock on actions

**Description:** `QuantAMMBaseAdministration`에서 특정 함수는 [`onlyExecutor`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMBaseAdministration.sol#L77-L81) 수정자를 사용하여 제한됩니다:
```solidity
// Modifier to check for EXECUTOR_ROLE using `timelock`
modifier onlyExecutor() {
    require(timelock.hasRole(timelock.EXECUTOR_ROLE(), msg.sender), "Not an executor");
    _;
}
```
이 수정자의 문제점은 작업에 타임락을 강제하지 않는다는 것입니다. 대신 호출자가 타임락 컨트랙트에서 `EXECUTOR_ROLE`을 가지고 있는지만 확인합니다. OpenZeppelin [`TimelockController` 코드](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v5.0/contracts/governance/TimelockController.sol#L351-L371)에서 볼 수 있듯이, 이 확인을 통해 `EXECUTOR_ROLE`을 가진 사람은 타임락을 우회하고 `QuantAMMBaseAdministration`에서 직접 명령을 실행할 수 있습니다.

**Impact:** 타임락 컨트랙트에서 작업을 실행할 권한이 있는 사람은 누구나 타임락을 우회하고 `QuantAMMBaseAdministration` 컨트랙트에서 동일한 명령을 직접 실행할 수 있습니다. 이는 중요한 작업에 지연을 제공하려는 타임락의 목적을 약화시킵니다.

**Recommended Mitigation:**
1. **타임락을 강제하도록 `onlyExecutor` 수정자 수정:**
   호출자가 타임락 컨트랙트 자체여야 하도록 `onlyExecutor` 수정자를 변경하십시오:

   ```solidity
   modifier onlyTimelock() {
       require(address(timelock) == msg.sender, "Only timelock");
       _;
   }
   ```

2. **`timelock`을 공개(Public)로 설정:**
   작업 제안을 위해 타임락의 주소를 쿼리할 수 있도록 `timelock` 변수를 `public`으로 만드십시오:

   ```diff
   - TimelockController private timelock;
   + TimelockController public timelock;
   ```

3. **대안 솔루션:**
   `QuantAMMBaseAdministration`을 사용하는 대신, OpenZeppelin `TimelockController` 컨트랙트를 `quantammAdmin`으로 직접 사용하는 것을 고려하십시오. 이 접근 방식은 프로세스를 간소화하고 중복 확인 없이 타임락 시행을 보장합니다.

**QuantAMM:** [`a299ce7`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a299ce72076b58503386bc146b81014560536d9e)에서 수정됨

**Cyfrin:** 확인됨. `QuantAMMBaseAdministration`이 제거되었습니다.

### Unauthorized role grants prevent `QuantAMMBaseAdministration` deployment

**Description:** `QuantAMMBaseAdministration`은 특정 유사시(break-glass) 기능을 관리하고 다른 QuantAMM 컨트랙트에 대한 호출을 용이하게 하는 컨트랙트입니다.

그러나 이 컨트랙트는 제안자(proposers)와 실행자(executors)에게 역할을 부여하는 [`QuantAMMBaseAdministration` 생성자](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMBaseAdministration.sol#L68-L74)의 문제로 인해 배포에 실패합니다:

```solidity
for (uint256 i = 0; i < proposers.length; i++) {
    timelock.grantRole(timelock.PROPOSER_ROLE(), proposers[i]);
}

for (uint256 i = 0; i < executors.length; i++) {
    timelock.grantRole(timelock.EXECUTOR_ROLE(), executors[i]);
}
```
`QuantAMMBaseAdministration` 컨트랙트가 이러한 `grantRole` 호출을 성공시키려면 `timelock` 컨트랙트에서 `DEFAULT_ADMIN_ROLE`을 가지고 있어야 합니다. 이 역할을 가지고 있지 않으므로 호출은 `AccessControlUnauthorizedAccount` 오류와 함께 [되돌아갑니다](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v5.0/contracts/access/AccessControl.sol#L122-L124).

**Impact:** `QuantAMMBaseAdministration`의 배포는 승인되지 않은 역할 할당으로 인해 실패하여 컨트랙트가 성공적으로 배포되지 못하게 합니다.

**Proof of Concept:** 다음 테스트 파일을 `pkg/pool-quantamm/test/foundry/` 폴더에 추가하십시오:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "../../contracts/QuantAMMBaseAdministration.sol";

contract QuantAMMBaseAdministrationTest is Test {

    address daoRunner = makeAddr("daoRunner");

    address proposer = makeAddr("proposer");
    address executor = makeAddr("executor");

    QuantAMMBaseAdministration admin;

    function testSetupQuantAMMBaseAdministration() public {
        address[] memory proposers = new address[](1);
        proposers[0] = proposer;

        address[] memory executors = new address[](1);
        executors[0] = executor;

        vm.expectRevert(); // AccessControlUnauthorizedAccount(address(admin),DEFAULT_ADMIN_ROLE)
        admin = new QuantAMMBaseAdministration(
            daoRunner,
            1 hours,
            proposers,
            executors
        );
    }
}
```

**Recommended Mitigation:** `TimeLockController` [생성자](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v5.0/contracts/governance/TimelockController.sol#L125-L133)에서 이러한 할당이 이미 처리되므로 생성자에서 역할 부여를 제거하십시오:

**QuantAMM:** [`a299ce7`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a299ce72076b58503386bc146b81014560536d9e)에서 수정됨

**Cyfrin:** 확인됨. `QuantAMMBaseAdministration`이 제거되었습니다.

\clearpage
## Low Risk


### `UpdateWeightRunner::performUpdate` reverts on underflow when new multiplier is zero

**Description:** [`UpdateWeightRunner::_calculateMultiplerAndSetWeights`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L433-L504)에서 새로운 승수는 [시간에 따른 가중치 변화](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L447)를 계산하여 결정됩니다.

예상치 못한 값으로부터 보호하기 위해 승수에 대한 "마지막 유효 타임스탬프"가 [여기](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L453-L485) 섹션에서 계산됩니다.

가중치 변화가 없으면 승수는 0이 됩니다. 이 시나리오는 다음 코드의 특수 케이스로 처리됩니다:
```solidity
int256 currentLastInterpolationPossible = type(int256).max;

for (uint i; i < local.currentWeights.length; ) {
    // ...

    if (blockMultiplier > int256(0)) {
        weightBetweenTargetAndMax = upperGuardRail - local.currentWeights[i];
        // ...
        blockTimeUntilGuardRailHit = weightBetweenTargetAndMax / blockMultiplier;
    } else if (blockMultiplier == int256(0)) {
        blockTimeUntilGuardRailHit = type(int256).max; // @audit if blockMultiplier is 0 this is max
    } else {
        weightBetweenTargetAndMax = local.currentWeights[i] - local.absoluteWeightGuardRail18;
        // ...
        blockTimeUntilGuardRailHit = weightBetweenTargetAndMax / int256(uint256(-1 * blockMultiplier));
    }
    // ...
}
// ...

//next expected update + time beyond that
currentLastInterpolationPossible += int40(uint40(block.timestamp));
```
가중치가 변경되지 않고 승수가 `0`이면 `currentLastInterpolationPossible`은 `type(int256).max`로 설정됩니다. 결과적으로 문 `currentLastInterpolationPossible += int40(uint40(block.timestamp))`는 정수 오버플로로 인해 되돌아갑니다.

승수가 다른 분기에서 매우 작을 경우 `weightBetweenTargetAndMax / blockMultiplier`가 폭발할 수 있으므로 동일한 오버플로가 잠재적으로 발생할 수 있습니다.

**Impact:** 새 승수가 `0`(또는 매우 큰 값)일 때마다 `UpdateWeightRunner::performUpdate` 호출이 되돌아갑니다. 이는 가중치가 일정하게 유지되거나 특정 엣지 케이스에서 업데이트가 실행되지 못하게 합니다.

**Proof of Concept:** 다음 테스트를 `pkg/pool-quantamm/test/foundry/UpdateWeightRunner.t.sol`에 추가하십시오:
```solidity
function testUpdatesFailsWhenMultiplierIsZero() public {
    vm.startPrank(owner);
    updateWeightRunner.setApprovedActionsForPool(address(mockPool), 3);
    vm.stopPrank();
    int256[] memory initialWeights = new int256[](4);
    initialWeights[0] = 0;
    initialWeights[1] = 0;
    initialWeights[2] = 0;
    initialWeights[3] = 0;

    // Set initial weights
    mockPool.setInitialWeights(initialWeights);

    int216 fixedValue = 1000;
    chainlinkOracle = deployOracle(fixedValue, 3601);

    vm.startPrank(owner);
    updateWeightRunner.addOracle(OracleWrapper(chainlinkOracle));
    vm.stopPrank();

    vm.startPrank(address(mockPool));

    address[][] memory oracles = new address[][](1);
    oracles[0] = new address[](1);
    oracles[0][0] = address(chainlinkOracle);

    uint64[] memory lambda = new uint64[](1);
    lambda[0] = 0.0000000005e18;
    updateWeightRunner.setRuleForPool(
        IQuantAMMWeightedPool.PoolSettings({
            assets: new IERC20[](0),
            rule: IUpdateRule(mockRule),
            oracles: oracles,
            updateInterval: 1,
            lambda: lambda,
            epsilonMax: 0.2e18,
            absoluteWeightGuardRail: 0.2e18,
            maxTradeSizeRatio: 0.2e18,
            ruleParameters: new int256[][](0),
            poolManager: addr2
        })
    );
    vm.stopPrank();

    vm.startPrank(addr2);
    updateWeightRunner.InitialisePoolLastRunTime(address(mockPool), 1);
    vm.stopPrank();

    vm.warp(block.timestamp + 10000000);
    mockRule.CalculateNewWeights(
        initialWeights,
        new int256[](0),
        address(mockPool),
        new int256[][](0),
        new uint64[](0),
        0.2e18,
        0.2e18
    );
    vm.expectRevert();
    updateWeightRunner.performUpdate(address(mockPool));
}

```

**Recommended Mitigation:** 오버플로를 방지하는 것을 고려하십시오:
```diff
  //next expected update + time beyond that
+ if(currentLastInterpolationPossible < int256(type(int40).max)) {
      currentLastInterpolationPossible += int40(uint40(block.timestamp));
+ }
```

**QuantAMM:** [`9b7a998`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/9b7a998fca97fcb239c72d2cd762d9dc59e6ce8e)에서 수정됨

**Cyfrin:** 확인됨. 오버플로가 이제 방지됩니다.


### Missing weight sum validation in `setWeightsManually` function

**Description:** `UpdateWeightRunner::setWeightsManually`는 가중치가 1의 합계가 되는지 확인하지 않고 권한이 있는 사용자(풀 레지스트리에 따라 DAO, 풀 관리자 또는 관리자)가 풀의 가중치를 수동으로 설정할 수 있도록 허용합니다. 이 확인은 가중치가 `_setInitialWeights`에서 초기화될 때 처음에는 올바르게 수행됩니다:

```solidity
// In QuantAMMWeightedPool.sol - _setInitialWeights
// Ensure that the normalized weights sum to ONE
if (uint256(normalizedSum) != FixedPoint.ONE) {
    revert NormalizedWeightInvariant();
}
```

UpdateWeightRunner.sol의 setWeightsManually에는 유사한 유효성 검사가 누락되어 있어 가중치를 수동으로 입력할 때 가중치 검증이 처리되는 방식에 불일치가 발생합니다.


**Impact:** 이 함수는 액세스 제어되지만, 권한이 있는 사용자가 합계가 1이 아닌 가중치를 설정하도록 허용하면 다음과 같은 문제가 발생할 수 있습니다:

- 잘못된 풀 평가
- 스왑 금액 계산 오류로 인한 불공정 거래

참고로 `setInitialWeights`도 권한 있는 액세스 권한을 가지고 있지만 해당 함수에는 가중치 합계 확인이 존재합니다.


**Recommended Mitigation:** `setWeightsManually`에 가중치 합계 유효성 검사를 추가하는 것을 고려하십시오.


**QuantAMM:** 실제로 그리고 이론적으로 가중치 합계는 1이 될 필요가 없습니다. 거래는 가중치의 절대값이 아니라 비율로 수행됩니다. 모두가 그렇게 하지만 0.5/0.5 대신 50/50으로 설정해도 풀의 수학에 의해 허용되는 스왑에는 아무런 변화가 없습니다. 유사시(break glass) 조치에 대한 우리의 일반적인 관행은 어떤 유리가 깨져야 할지 알 수 없는 미지의 것일 수 있으므로 유효성 검사를 포함하지 않는 것입니다. 그러나 잠재적인 문제를 검토한 결과 실제로 1>x>0(특히 0보다 큰)을 예상하는 고정 소수점 수학 라이브러리를 사용한다는 점을 고려할 때 확인을 추가하는 것이 합리적이므로 문제에 대한 설명만 변경될 수 있습니다. 우리는 가드 레일을 확인하지 않을 것입니다. 그것은 이상한 미지의 상황에서 유효한 유사시 조치가 될 수 있기 때문입니다.

**Cyfrin:** 인지됨.


### Extreme weight ratios combined with large balances can cause denial-of-service for unbalanced liquidity operations

**Description:** 가중치 풀 수학은 가중치가 매우 작고 잔액이 큰 풀 토큰에 대해 불균형한 유동성 조정을 할 때 잠재적으로 오버플로될 수 있습니다.

문제는 `WeightedMath::computeBalanceOutGivenInvariant`에서 발생합니다:

```solidity
 newBalance = oldBalance * (invariantRatio ^ (1/weight))
```

가중치가 작고 불변 비율이 최대값(300%)에 가까우면 현실적인 잔액 변경으로도 계산이 오버플로될 수 있습니다.

퍼즈 테스트에서 가져온 예:

```solidity
// Balance computation scenario:
balance = 7500e21  (7.5 million tokens)
weight = 0.01166 (1.166%) // Just above absoluteWeightGuardRail minimum of 1% proposed for Balancer pools
invariantRatio = 3.0 // maximum value

calculation = 7500e21 * (3.0 ^ (1/0.01166))
           = 7500e21 * (3.0 ^ 85.76)
           = OVERFLOW
```

`QuantAMMWeightedPool`은 `QuantAMMWeightedPool::_setRule`에서 볼 수 있듯이 가중치를 `0.1 %`까지 허용합니다:
```solidity
//applied both as a max (1 - x) and a min, so it cant be more than 0.49 or less than 0.01
//all pool logic assumes that absolute guard rail is already stored as an 18dp int256
require(
    int256(uint256(_poolSettings.absoluteWeightGuardRail)) <
        PRBMathSD59x18.fromInt(1) / int256(uint256((_initialWeights.length))) &&
        int256(uint256(_poolSettings.absoluteWeightGuardRail)) > 0.001e18, // @audit minimum guard rail allowed is 0.1 %
    "INV_ABSWGT"
); //Invalid absoluteWeightGuardRail value
```

**Impact:** 가중치가 작고 잔액이 큰 토큰에 대한 유동성 작업이 되돌아갈(revert) 수 있습니다. 결과적으로 가중치가 가드 레일 최소값에 접근함에 따라 풀을 일시적으로 사용할 수 없게 될 수 있습니다. 사용자는 이 시나리오를 완화하기 위해 항상 비례적으로 유동성을 추가/제거하도록 선택할 수 있다는 점을 강조할 가치가 있습니다.

**Recommended Mitigation:** 오버플로의 위험은 최소 가중치가 1%까지 올라갈 수 있을 때 존재합니다. 최소 가중치를 3%로 조금만 늘려도 이 시나리오를 효과적으로 방지할 수 있습니다. 가장 낮은 시행 가드 레일을 더 높게 변경하는 것을 고려하십시오.

**QuantAMM:** Balancer 팀의 응답과 우리는 동의합니다:

> computeBalance는 불균형 유동성 작업(단일 토큰 정확한 출력 추가 또는 단일 토큰 정확한 입력 제거)에만 사용됩니다.
정확한 출력 추가는 배치 스왑에서만 사용됩니다. 정확한 입력 제거는 단일 측면으로 종료하려는 경우 사용할 수 있다고 생각합니다.
하지만 어떤 경우에도 항상 비례적으로 종료할 수 있습니다.

> 위의 예는 3의 불변 비율을 사용하는데, 이는 유동성 추가(>1)입니다. 해당 비율에 대한 잔액 계산은 단일 토큰 정확한 출력 추가에만 발생합니다. 풀을 중첩하지 않는 한(어차피 [quantamm] 가중치 풀에서는 기본적으로 지원되지 않음) 그것을 사용하지 않을 것입니다.

[`4beffda`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/4beffda73e0c38b9dc719d78147c06f1d59644bd)에서 가장 낮은 가드 레일로 1%가 시행됨

**Cyfrin:** 확인됨.

\clearpage
## Informational


### Use of solidity `assert` instead of foundry asserts in test suite

**Description:** [`QuantAMMWeightedPool8Token`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/test/foundry/QuantAMMWeightedPool8Token.t.sol#L312) 테스트에서 foundry `assertEq` 대신 `assert`가 사용됩니다.

Foundry [`assertEq`](https://book.getfoundry.sh/reference/forge-std/std-assertions)는 비교된 두 값과 같이 실패할 때 더 나은 오류 정보를 제공합니다. Solidity `assert` 대신 `assertEq`를 사용하는 것을 고려하십시오.

**QuantAMM:** [`414d4bc`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/414d4bc3d8699f1e4620f0060226b4e23ff3b010) 및 [`ca1e441`](https://github.com/QuantAMMProtocol/QuantAMM-V1/pull/27/commits/ca1e441d4feb2a1ffe202121857207f3e15fbc2f)에서 수정됨

**Cyfrin:** 확인됨


### Consider adding a getter for `QuantAMMWeightedPool.poolDetails`

**Description:** [`QuantAMMWeightedPool.poolDetails`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L170)는 중첩 배열입니다:
```solidity
string[][] public poolDetails;
```
중첩 배열은 요소에 액세스하려면 두 인덱스가 모두 필요합니다. 전체 배열을 쿼리하려면 사용자 지정 getter가 필요합니다. 이는 오프체인 모니터링에 유용할 수 있습니다.

감사 중 프로토콜에 의해 보고되었습니다.

**QuantAMM:** [`ee1fbb8`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/ee1fbb8db9d113f04c0b99fe766800dc80cf1ff2)에서 수정됨

**Cyfrin:** 확인됨.


### `AntimomentumUpdateRule.parameterDescriptions` initialized too long

**Description:** [`AntimomentumUpdateRule.parameterDescriptions`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/rules/AntimomentumUpdateRule.sol#L16-L18)는 길이 3으로 초기화되지만 두 개의 항목만 사용됩니다:
```solidity
parameterDescriptions = new string[](3); // @audit should be 2
parameterDescriptions[0] = "Kappa: Kappa dictates the aggressiveness of response to a signal change TODO";
parameterDescriptions[1] = "Use raw price: 0 = use moving average, 1 = use raw price to be used as the denominator";
```
`new string[](2)`로 초기화하는 것을 고려하십시오.

**QuantAMM:** [`91241ca`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/91241ca9e72b8a4fc964cc2df34f5aeb82f7ca39)에서 수정됨

**Cyfrin:** 확인됨.


###  `TODO`s left in rule parameter descriptions

**Description:** 세 가지 규칙 [`AntimomentumUpdateRule`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/rules/AntimomentumUpdateRule.sol#L17), [`MinimumVarianceUpdateRule`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/rules/MinimumVarianceUpdateRule.sol#L15) 및 [`MomentumUpdateRule`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/rules/MomentumUpdateRule.sol#L15)의 매개변수 설명에 `TODO`가 남아 있습니다.

이러한 매개변수에 대한 설명을 마무리하는 것을 고려하십시오.

**QuantAMM:** [`e3e0d5e`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/e3e0d5eab5f81d2141ce69daffb249c223385b2f), [`8c3a4f3`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/8c3a4f3df43ba8731808c226b2fe4eb68b3e7c8b), [`f56796e`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/f56796ee75615a28c14c3678d984d5b55d3cf864)에서 수정됨

**Cyfrin:** 확인됨.


### Unnecessary `_initialWeights` sum check

**Description:** `QuantAMMWeightedPool`을 처음 초기화할 때 풀 생성자는 초기 가중치를 제공합니다. 이러한 초기 가중치의 합은 1(`1e18`)이어야 합니다. 그러나 문제는 이것이 [`QuantAMMWeightedPool::_setRule`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L767-L778)과:
```solidity
int256 sumWeights;
for (uint i; i < _initialWeights.length; ) {
    sumWeights += int256(_initialWeights[i]);
    unchecked {
        ++i;
    }
}

require(
    sumWeights == 1e18, // 1 for int32 sum
    "SWGT!=1"
); //Initial weights must sum to 1
```
그리고 [`QuantAMMWeightedPool::_setInitialWeights`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMWeightedPool.sol#L625-L644)에서:
```solidity
int256 normalizedSum;
int256[] memory _weightsAndBlockMultiplier = new int256[](_weights.length * 2);
for (uint i; i < _weights.length; ) {
    if (_weights[i] < int256(uint256(absoluteWeightGuardRail))) {
        revert MinWeight();
    }

    _weightsAndBlockMultiplier[i] = _weights[i];
    normalizedSum += _weights[i];
    //Initially register pool with no movement, first update will come and set block multiplier.
    _weightsAndBlockMultiplier[i + _weights.length] = int256(0);
    unchecked {
        ++i;
    }
}

// Ensure that the normalized weights sum to ONE
if (uint256(normalizedSum) != FixedPoint.ONE) {
    revert NormalizedWeightInvariant();
}
```
두 번 모두 확인된다는 것입니다.

`_setRule`에서의 확인은 규칙을 확인하기 위한 것이므로 제거를 고려하고, 또한 `absoluteWeightGuardRail` 요구 사항을 준수하는지 확인하므로 `_setInitialWeights`에서의 확인이 더 철저합니다.

**QuantAMM:** [`6b891e8`](https://github.com/QuantAMMProtocol/QuantAMM-V1/pull/36/commits/6b891e877f8dc1ec50b562d744c5fbb7c5f70eef)에서 수정됨

**Cyfrin:** 확인됨.

\clearpage
