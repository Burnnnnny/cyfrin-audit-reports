**수석 감사자 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)

**보조 감사자 (Assisting Auditors)**

[0kage](https://twitter.com/0kage_eth)


---

# 발견 사항 (Findings)
## 높은 중요도 (High Risk)

### 공격자가 악성 수익 토큰을 사용하여 사용자 자금을 탈취할 수 있음 (Attackers can use a malicious yield token to steal funds from users)

**심각도:** 높음 (High)

**설명:** 문서 및 현재 구현에 따르면 누구나 새로운 StakePet 컨트랙트를 생성하고 `YIELD_TOKEN`에 대한 주소를 제공할 수 있습니다. 컨트랙트가 `IYieldToken` 인터페이스를 구현하는 한 문제 없이 생성됩니다.

공격자는 악성 `IYieldToken` 구현을 생성하고 이를 사용하여 사용자 자금을 훔칠 수 있습니다.
StakePet 컨트랙트는 회계를 위해 여러 곳에서 `YIELD_TOKEN.toToken()` 및 `YIELD_TOKEN.toValue()`에 의존합니다.
소유자의 숨겨진 플래그에 따라 `toToken()` 및 `toValue()`에서 다른 로직을 구현한 컨트랙트를 고려해 보세요.
공격자는 StakePet 컨트랙트가 충분한 예치금을 받을 때까지 악성 토큰 컨트랙트가 정상적으로 작동하도록 놔둘 가능성이 높습니다.
그런 다음 필요에 따라 숨겨진 플래그를 전환하여 회계를 엉망으로 만들고 이익을 취할 수 있습니다.
최악의 경우 `IYieldToken::ERC20_TOKEN()`의 출력을 조작할 수도 있습니다(사용자 자금을 영구적으로 동결하기 위해).

**파급력:** 사용자 자금이 도난당하거나 영구적으로 잠길 수 있습니다.

**권장 완화 방안:** YIELD_TOKEN의 화이트리스트를 유지하고 허용된 수익 토큰에 대해서만 StakePet 생성을 허용하는 것을 고려하십시오.

**클라이언트:** [308672e](https://github.com/Ranama/StakePet/commit/308672e914651ca2300f2b585d91f16764994bf7) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 인플레이션 공격으로 인해 초기 사용자가 예치금을 잃을 수 있음 (Inflation attack can cause early users to lose their deposit)

**심각도:** 높음 (High)

**설명:** 악의적인 `StakePet` 컨트랙트 생성자는 전형적인 인플레이션 공격을 시작하여 예금자로부터 자금을 훔칠 수 있습니다. 공격을 실행하기 위해 생성자는 먼저 `1 wei`를 예치하여 `1 wei`의 소유권을 얻을 수 있습니다. 그 후 생성자는 `StakePet` 컨트랙트로 직접 많은 양의 담보를 보낼 수 있습니다. 이는 단일 지분의 가치를 엄청나게 부풀릴 것입니다.

이제 담보를 예치하는 모든 후속 애완동물 소유자는 대가로 소유권을 얻지 못합니다. `StakePet::ownershipToMint` 함수는 `StakePet::totalValue`를 사용하여 새 예금자의 소유권을 계산합니다. `s_totalOwnership`으로 표시되는 총 소유권은 동일한 `1 wei`로 유지되지만, 생성자가 수행한 대규모 직접 예치 덕분에 `totalValueBefore`는 엄청난 숫자가 됩니다. 이는 1 wei의 지분이 엄청난 가치의 담보를 나타내도록 보장하고 새 예금자의 소유권이 0으로 반올림되도록 합니다.

**파급력:** 예치된 토큰에 대한 대가로 소유권을 받지 못하므로 새로운 예금자의 자금 전체 손실 가능성이 있습니다.

**개념 증명 (Proof of Concept):**
- 악의적인 행위자 Bob이 StakePet 컨트랙트를 시작합니다.
- Bob은 `StakePet::create`를 호출하여 단 `1 wei`를 예치하여 애완동물을 생성하고 `1 wei`의 소유권을 얻습니다.
- 그런 다음 Bob은 10 이더와 같은 상당한 금액을 `StakePet` 컨트랙트로 직접 전송합니다.
- 결과적으로 단일 `1 wei` 지분은 `10 이더`와 동일해집니다.
- 무고한 사용자 Pete가 `StakePet::create`를 호출하여 애완동물을 생성하려고 시도하고 1 이더를 예치합니다.
- 불행히도 Pete는 0의 소유권을 받는 반면 그의 예치금은 컨트랙트 내에 남습니다.

**권장 완화 방안:** 인플레이션 공격에는 알려진 방어책이 있습니다. 포괄적인 논의는 [여기](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3706)에서 찾을 수 있습니다.

Uniswap V2에서 구현한 주목할만한 방법 중 하나는 컨트랙트에 최소 유동성을 예치하고 소유권을 널 주소로 전송하여 "죽은 지분(dead shares)"을 생성하는 것입니다. 이 기술은 후속 예금자를 잠재적인 인플레이션 공격으로부터 보호합니다.

이 경우 컨트랙트 시작 시 최소 담보 요구 사항을 도입하고 이에 따라 `s_totalOwnership`을 이 미리 설정된 담보와 일치하도록 조정하는 것이 유익할 수 있습니다.

**클라이언트:** [a692abc](https://github.com/Ranama/StakePet/commit/a692abc038fdd8992916f93d213a38c30e3a9764) 및 [21dd15b](https://github.com/Ranama/StakePet/commit/21dd15b1fceecddb9caf47739b6df1a4d1856367) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

## 중간 중요도 (Medium Risk)

### 악성 사용자가 엄청난 수의 애완동물을 생성하여 `StakePet` 컨트랙트를 괴롭힐 수 있음 (A malicious user can grief a `StakePet` contract by creating massive number of pets)

**심각도:** 중간 (Medium)

**설명:** `StakePet::create` 함수는 담보를 예치하여 애완동물 NFT의 발행을 용이하게 합니다. 그러나 발행에 대한 최소 예치 요구 사항이 없어서 잠재적인 남용에 노출됩니다. 악성 사용자는 과도한 수의 NFT를 발행하여 이를 악용할 수 있습니다. 특히 이 동작은 `StakePetManager::buryAllDeadPets`와 같은 함수에 부담을 줄 수 있으며, 이 함수는 차례로 `StakePetManager::getDeadNonBuriedPets`를 호출합니다. 후자의 함수는 모든 애완동물 ID를 반복하여 죽었지만 아직 매장되지 않은 애완동물을 식별합니다.

**파급력:** 함수가 광범위하고 잠재적으로 무제한인 애완동물 ID 목록을 처리할 때 사용 가능한 모든 가스를 소비할 위험이 있습니다. 결과적으로 가스 부족 예외를 발생시키며 실패할 수 있어 컨트랙트와 상호 작용하려는 사용자에게 부정적인 영향을 미칩니다.

**권장 완화 방안:** 이러한 괴롭힘(griefing) 공격을 막으려면 새로운 애완동물 생성에 대한 최소 예치 요구 사항을 도입하는 것이 좋습니다. 이 임계값을 설정하면 공격자의 대량 발행 전략이 비용이 많이 들게 됩니다.

**클라이언트:** [a692abc](https://github.com/Ranama/StakePet/commit/a692abc038fdd8992916f93d213a38c30e3a9764) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

## 낮은 중요도 (Low Risk)

### 폐쇄 조건이 다수결 합의라는 명시된 문서와 일치하지 않음 (Closedown condition is inconsistent with the stated documentation of majority agreement)

**심각도:** 낮음 (Low)

**설명:** [문서](https://hackmd.io/CPINxScvSE2vo-t8mwY_Og#Risks)에는 다음과 같이 명시되어 있습니다:

_"컨트랙트 폐쇄(Closing the Contract): 대다수의 애완동물이 동의하면 투표를 통해 컨트랙트를 폐쇄할 수 있습니다. 폐쇄되면 남은 자금은 생존한 애완동물에게 분배됩니다. 기본 보상, 조기 인출 보상, 죽은 애완동물의 보상을 받으므로 귀하에게 가장 유리한 시나리오입니다."_

[`StakePet::closedown`](https://github.com/Ranama/StakePet/blob/9ba301823b5062d657baa3462224da498dc4bb46/src/StakePet.sol#L398C2-L398C2) 함수에 대한 인라인 주석은 다음과 같이 명시합니다:

```solidity
    /// @notice Close down the contract if majority wants it, after closedown everyone can withdraw without getting a yield cut and no pet can die.
    function closedown(uint256[] memory _idsOfMajorityThatWantsClosedown) external {
...
}
```

두 경우 모두 폐쇄 조건은 `애완동물의 대다수(majority of pets)`가 폐쇄에 동의하는 것입니다. 그러나 `closedown`에 사용된 확인은 폐쇄를 원하는 애완동물의 총 담보가 총 담보의 최소 50%여야 한다는 것입니다. 이는 대규모 담보 예치금을 가진 단일 또는 소수의 애완동물 소유자가 대다수의 애완동물 소유자가 동의하지 않더라도 폐쇄를 트리거할 수 있음을 의미합니다.

가치 합의의 50%를 갖는 것과 다수결 합의를 갖는 것은 서로 다른 두 가지일 수 있습니다.

**파급력:** 현재 모델은 원할 때마다 컨트랙트 폐쇄를 트리거할 수 있는 고래에 의해 탈취될 수 있습니다. 이는 컨트랙트에 남기를 원하는 대다수의 애완동물 소유자에게 나쁜 사용자 경험을 줄 수 있습니다.

**권장 완화 방안:** 문서를 Stake Pet의 비전과 일치하도록 만드십시오.

**클라이언트:** [54a4dcb](https://github.com/Ranama/StakePet/commit/54a4dcbb696da3138dc0fdd8e7032d664d32b7da)에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 종료 수수료 구현이 문서와 일치하지 않음 (Exit fees implementation is inconsistent with documentation)

**심각도:** 낮음 (Low)

**설명:** `StakePet` 컨트랙트의 인라인 주석은 종료 수수료가 담보의 %로 청구됨을 나타냅니다.

```
The contract also has an early exit fee, which is a percentage of the collateral taken if a participant chooses to exit early.
```

그러나 구현은 종료 수수료가 [수익의 퍼센트](https://github.com/Ranama/StakePet/blob/9ba301823b5062d657baa3462224da498dc4bb46/src/StakePet.sol#L559)로 청구됨을 보여줍니다.

```
uint256 earlyExitFee = (uint256(yieldToWithdraw) * EARLY_EXIT_FEE) / BASIS_POINT
```

**권장 완화 방안:** 실제 구현을 반영하도록 코드 문서를 수정하는 것을 고려하십시오.

**클라이언트:** [54a4dcb](https://github.com/Ranama/StakePet/commit/54a4dcbb696da3138dc0fdd8e7032d664d32b7da)에서 수정되었습니다.

**Cyfrin:** 검증됨.

## 가스 최적화 (Gas Optimizations)

### 스토리지에 bool을 사용하면 오버헤드 발생 (Using bools for storage incurs overhead)

true/false에 대해 uint256(1) 및 uint256(2)를 사용하여 Gwarmaccess(100 가스)를 피하고, 과거에 'true'였던 후 'false'에서 'true'로 변경할 때 Gsset(20000 가스)을 피하십시오. [소스](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27)를 참조하세요.

```solidity
File: StakePet.sol

91:     bool public constant TESTING = true; // TODO: Remove this when not testing

107:     bool public immutable HARDCORE; // Whether the initial collateral is taken if failing to proof of life or not

```

**클라이언트:** [aea1f74](https://github.com/Ranama/StakePet/commit/aea1f7464339cb16008143440bd427b6f0a14669)에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 루프 외부에서 배열 길이 캐시 (Cache array length outside of loop)

캐시되지 않으면 solidity 컴파일러는 각 반복 중에 항상 배열의 길이를 읽습니다. 즉, 스토리지 배열인 경우 이는 추가 sload 작업(첫 번째를 제외한 각 반복에 대해 100 추가 가스)이고 메모리 배열인 경우 추가 mload 작업(첫 번째를 제외한 각 반복에 대해 3 추가 가스)입니다.

```solidity
File: StakePet.sol

410:         for (uint256 i = 0; i < _idsOfMajorityThatWantsClosedown.length; i++) {

```

```solidity
File: StakePetManager.sol

73:         for (uint256 i = 0; i < _contractIDs.length; i++) {

75:             for (uint256 j = 0; j < _petIDs[i].length; j++) {

108:         for (uint256 i = 0; i < _contractIDs.length; i++) {

110:             for (uint256 j = 0; j < _petIDs[i].length; j++) {

147:         for (uint256 i = 0; i < _contractIDs.length; i++) {

```

**클라이언트:** [627d09c](https://github.com/Ranama/StakePet/commit/627d09c34bb4853418e8c22ed8ce291efd7ad087)에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 기본값으로 변수 초기화하지 않기 (Don't initialize variables with default value)

```solidity
File: StakePet.sol

128:     uint256 public s_closedAtTimestamp = 0; // The timestamp that the contract was closed down

409:         uint256 _totalValueWantsClosedown = 0;

410:         for (uint256 i = 0; i < _idsOfMajorityThatWantsClosedown.length; i++) {

```

```solidity
File: StakePetManager.sol

73:         for (uint256 i = 0; i < _contractIDs.length; i++) {

75:             for (uint256 j = 0; j < _petIDs[i].length; j++) {

108:         for (uint256 i = 0; i < _contractIDs.length; i++) {

110:             for (uint256 j = 0; j < _petIDs[i].length; j++) {

123:         uint256 j = 0;

137:         for (uint256 i = 0; i < j; i++) {

147:         for (uint256 i = 0; i < _contractIDs.length; i++) {

```

**클라이언트:** [970b71c](https://github.com/Ranama/StakePet/commit/970b71cfe73760dc694b0c0e1e5a3a77dc704c8c)에서 수정되었습니다.

**Cyfrin:** 검증됨.

### `i++`보다 `++i`가 가스 비용이 적음 (`++i` costs less gas than `i++`)

특히 `for`-루프에서 사용될 때 그렇습니다 (`--i`/`i--` 도 마찬가지).

```solidity
File: StakePet.sol

410:         for (uint256 i = 0; i < _idsOfMajorityThatWantsClosedown.length; i++) {

```

```solidity
File: StakePetManager.sol

73:         for (uint256 i = 0; i < _contractIDs.length; i++) {

75:             for (uint256 j = 0; j < _petIDs[i].length; j++) {

108:         for (uint256 i = 0; i < _contractIDs.length; i++) {

110:             for (uint256 j = 0; j < _petIDs[i].length; j++) {

127:         for (uint256 i = 1; i <= currentPetId; i++) {

131:                 j++;

137:         for (uint256 i = 0; i < j; i++) {

147:         for (uint256 i = 0; i < _contractIDs.length; i++) {

```

**클라이언트:** [27225c2](https://github.com/Ranama/StakePet/commit/27225c256c3173cc306045949584b66be7f60c0f)에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 가능하다면 나눗셈/곱셈 대신 시프트 오른쪽/왼쪽 사용 (Use shift Right/Left instead of division/multiplication if possible)

```solidity
File: StakePet.sol

420:         if (_totalValueWantsClosedown <= totalValue() / 2) {

```

**클라이언트:** [540cca1](https://github.com/Ranama/StakePet/commit/540cca16669ee8575806d0f3430723726e3d9c2e)에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 부호 없는 정수 비교에 > 0 대신 != 0 사용 (Use != 0 instead of > 0 for unsigned integer comparison)

```solidity
File: StakePet.sol

269:         if (_amount > 0) {

299:         if (!petAlive && pet.ownership > 0) {

432:         if (totYield > 0) {

497:             if (_milkAmount > 0) {

541:         if (yieldToWithdraw > 0) {

558:                 require(yieldToWithdraw > 0); // This should never be hit and is maybe not needed, but just in case.

562:                 require(yieldToWithdraw > 0); // This should never be hit and is maybe not needed, but just in case.

709:         if (_totalYieldNoMilk > 0) {

723:         if (s_totalOwnership > 0) {

741:         if (s_totalOwnership > 0) {

```

```solidity
File: StakePetManager.sol

129:             if (!stakePetContract.alive(pet.lastProofOfLife) && pet.ownership > 0) {

```

**클라이언트:** [9e3d0d0](https://github.com/Ranama/StakePet/commit/9e3d0d0c1b6a324e22e0e3f70453c6d411cd9101)에서 수정되었습니다.

**Cyfrin:** 검증됨.

