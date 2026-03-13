**Lead Auditors**

[0kage](https://x.com/0kage_eth)

[Farouk](https://x.com/Ubermensh3dot0)

**Assisting Auditors**



---

# Findings
## High Risk


### InfraVault의 무허가 보상 청구로 인해 누구나 MetaVault에 보상을 잠글 수 있음 (InfraVault's Permissionless Reward Claiming Can Allow Anyone to Lock Rewards in the MetaVault)

**설명:** `InfraredBGTMetaVault`는 `_performDepositRewardByRewardType`을 사용하여 새로 청구된 보상을 처리하며, 이를 DolomiteMargin에 예치하거나 볼트 소유자에게 직접 전송합니다. 그러나 온체인 Infrared 볼트 [계약](https://berascan.com/address/0x67b4e6721ad3a99b7ff3679caee971b07fd85cd1#code)은 누구나 모든 사용자의 보상에 대해 `getRewardForUser`를 호출할 수 있도록 허용합니다. 이 함수는 Infrared 볼트가 상속받는 `MultiRewards.sol`에 정의되어 있습니다.

이로 인해 보상이 예상치 못하게 MetaVault로 전송될 수 있습니다. 이러한 토큰을 예치하거나 전달하는 코드(`_performDepositRewardByRewardType`)는 일반적인 "자가 청구(self-claim)" 흐름에서만 실행되므로, 제3자의 호출을 통해 트리거된 보상은 의도한 예치 또는 분배 로직을 거치지 않게 됩니다.

```solidity
/// @inheritdoc IMultiRewards
function getRewardForUser(address _user)
    public
    nonReentrant
    updateReward(_user)
{
    onReward();
    uint256 len = rewardTokens.length;
    for (uint256 i; i < len; i++) {
        address _rewardsToken = rewardTokens[i];
        uint256 reward = rewards[_user][_rewardsToken];
        if (reward > 0) {
            (bool success, bytes memory data) = _rewardsToken.call{
                gas: 200000
            }(
                abi.encodeWithSelector(
                    ERC20.transfer.selector, _user, reward
                )
            );
            if (success && (data.length == 0 || abi.decode(data, (bool)))) {
                rewards[_user][_rewardsToken] = 0;
                emit RewardPaid(_user, _rewardsToken, reward);
            } else {
                continue;
            }
        }
    }
}
```

**영향:** 공격자는 `_performDepositRewardByRewardType`을 트리거하지 않고 보상이 MetaVault의 주소로 전송되도록 강제할 수 있습니다. 결과적으로, 새로 도착한 토큰들은 MetaVault 계약에 머물게 되며, 스테이킹되거나 DolomiteMargin에 예치되거나 볼트 소유자에게 분배되지 않습니다.

`InfraredBGTMetaVault` 계약은 업그레이드 가능하므로 토큰 손실은 영구적이지 않습니다. 그럼에도 불구하고 모든 사용자가 독립적인 볼트를 가지고 있어 각 볼트를 업그레이드하는 것은 번거로울 수 있으므로 지연이 발생할 수 있습니다. 그동안 볼트 소유자는 수령한 보상을 Dolomite 프로토콜 내에서 사용할 수 없습니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려해 보세요:
1. 공격자가 `infravault.getRewardForUser(metaVaultAddress)`를 호출합니다.
2. 보상은 `_performDepositRewardByRewardType` 로직을 거치지 않고 `metaVaultAddress`로 전송됩니다.
3. 토큰을 이동하거나 다시 스테이킹하는 대체 메커니즘이 없다면 토큰은 MetaVault 계약에 갇히게 됩니다.

**권장 완화 방법:** `_performDepositRewardByRewardType`을 수정하여 볼트 내의 토큰 잔액을 보상 금액에 추가하고, 모든 BGT 토큰 볼트 스테이킹을 metavault를 통해 Infrared 볼트로 라우팅하는 것을 고려하세요.

```diff
  function _performDepositRewardByRewardType(
        IMetaVaultRewardTokenFactory _factory,
        IBerachainRewardsRegistry.RewardVaultType _type,
        address _token,
        uint256 _amount
    ) internal {
++ _amount += IERC20(token).balanceOf(address(this));
}
```

**Dolomite:** [d0a638a](https://github.com/dolomite-exchange/dolomite-margin-modules/commit/d0a638aefdda72925329b2da60f405cd4450f78a)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Medium Risk


### `InfraredBGTIsolationModeTokenVaultV1::exit()` 함수 호출 후 사용자 보상이 청구되지 않은 상태로 남을 수 있음 (User rewards may remain unclaimed after calling `InfraredBGTIsolationModeTokenVaultV1::exit()` function)

**설명:** 보상이 동일한 토큰(iBGT)인 시나리오에서 사용자가 `InfraredBGTIsolationModeTokenVaultV1::exit`을 호출하면, 함수는 원래 예치금을 올바르게 언스테이킹하지만 다음과 같은 문제가 발생합니다:

- 보상이 사용자의 Dolomite Margin 계정 잔액으로 입금됩니다.
- 동일한 토큰이 동시에 Infrared 볼트에 다시 스테이킹됩니다.

이로 인해 사용자는 프로토콜에서 완전히 나갔다고 믿을 수 있지만, 실제로는 보상이 여전히 스테이킹되어 있는 상황이 발생합니다. `_exit()` 함수는 획득한 보상을 처리하는 `_handleRewards()`를 호출합니다. 문제는 `_handleRewards()`가 iBGT 보상을 자동으로 Dolomite Margin에 다시 예치하고 재스테이킹할 때 발생합니다:

```solidity
function _exit() internal {
    IInfraredVault vault = registry().iBgtStakingVault();

    IInfraredVault.UserReward[] memory rewards = vault.getAllRewardsForUser(address(this));
    vault.exit();

    _handleRewards(rewards);
}

function _handleRewards(IInfraredVault.UserReward[] memory _rewards) internal {
    IIsolationModeVaultFactory factory = IIsolationModeVaultFactory(VAULT_FACTORY());
    for (uint256 i = 0; i < _rewards.length; ++i) {
        if (_rewards[i].amount > 0) {
            if (_rewards[i].token == UNDERLYING_TOKEN()) {
                _setIsDepositSourceThisVault(true);
                factory.depositIntoDolomiteMargin(
                    DEFAULT_ACCOUNT_NUMBER,
                    _rewards[i].amount
                ); //@audit 보상 금액을 다시 스테이킹합니다.
                assert(!isDepositSourceThisVault());
            } else {
                // 다른 토큰 처리...
            }
        }
    }
}
```

**영향:**
- 사용자는 프로토콜에 자산이 여전히 스테이킹되어 있다는 사실을 깨닫지 못할 수 있습니다.
- 보상은 사용자가 확인해야 할지 모르는 별도의 볼트에 잠긴 채로 남게 됩니다.
- 소액의 보상인 경우 가스 비용이 가치를 초과하여 사실상 자금이 갇힐 수 있습니다.
- "exit(종료)"가 프로토콜을 완전히 종료하지 않아 사용자 경험이 저하됩니다.

**개념 증명 (Proof of Concept):** `InfraredBGTIsolationModeTokenVaultV1.ts`의 `#exit` 테스트 클래스에 다음 테스트를 추가하세요.

```typescript
    it('should demonstrate that iBGT rewards are re-staked after exit', async () => {
      await testInfraredVault.setRewardTokens([core.tokens.iBgt.address]);
      await core.tokens.iBgt.connect(iBgtWhale).approve(testInfraredVault.address, rewardAmount);
      await testInfraredVault.connect(iBgtWhale).addReward(core.tokens.iBgt.address, rewardAmount);
      await registry.connect(core.governance).ownerSetIBgtStakingVault(testInfraredVault.address);

      await iBgtVault.depositIntoVaultForDolomiteMargin(defaultAccountNumber, amountWei);
      await expectProtocolBalance(core, iBgtVault, defaultAccountNumber, iBgtMarketId, amountWei);

      // 종료 전 초기 스테이킹 잔액 확인
      const initialStakingBalance = await testInfraredVault.balanceOf(iBgtVault.address);
      expect(initialStakingBalance).to.eq(amountWei);

      // exit 호출, 원래 금액은 언스테이킹되지만 보상은 다시 스테이킹됨
      await iBgtVault.exit();

      // 스테이킹 잔액이 보상 금액과 같은지 확인 (보상이 다시 스테이킹됨)
      const finalStakingBalance = await testInfraredVault.balanceOf(iBgtVault.address);
      expect(finalStakingBalance).to.eq(rewardAmount, "Staking balance should equal reward amount after exit");

      // 지갑에 원래 예치금이 있는지 확인 (보상은 없음)
      await expectWalletBalance(iBgtVault, core.tokens.iBgt, amountWei);

      // 프로토콜 잔액에 원래 예치금과 보상이 모두 포함되어 있는지 확인
      await expectProtocolBalance(core, iBgtVault, defaultAccountNumber, iBgtMarketId, amountWei.add(rewardAmount));
    });
```

**권장 완화 방법:** 사용자가 명시적으로 `exit`을 호출할 때 재스테이킹을 피하는 것을 고려하세요. 사용자가 볼트를 완전히 종료하려는 의사를 표시했을 때 동일한 볼트에 자산을 다시 스테이킹하는 것은 UX 관점에서도 직관적이지 않습니다. 또한 다양한 볼트에서 보상이 어떻게 처리되는지 설명하는 명확한 문서를 추가하세요.

**Dolomite:** [d0a638a](https://github.com/dolomite-exchange/dolomite-margin-modules/commit/d0a638aefdda72925329b2da60f405cd4450f78a)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Low Risk


### `POLIsolationModeWrapperUpgradeableProxy` 생성자에서 초기화 calldata에 대한 유효성 검사 누락 (Missing validation for initialization calldata in `POLIsolationModeWrapperUpgradeableProxy` constructor)

**설명:** `POLIsolationModeWrapperUpgradeableProxy` 생성자는 `delegatecall`을 통해 구현 계약으로 실행하기 전에 내용에 대한 유효성 검사를 수행하지 않고 초기화 `calldata`를 수락합니다. 구체적으로:

- 생성자는 `calldata`가 예상되는 `initialize(address)` 함수를 대상으로 하는지 확인하지 않습니다.
- 생성자는 제공된 `vaultFactory` 주소 파라미터가 0이 아닌지 확인하지 않습니다.

```solidity
// POLIsolationModeWrapperUpgradeableProxy.sol
constructor(
    address _berachainRewardsRegistry,
    bytes memory _initializationCalldata
) {
    BERACHAIN_REWARDS_REGISTRY = IBerachainRewardsRegistry(_berachainRewardsRegistry);
    Address.functionDelegateCall(
        implementation(),
        _initializationCalldata,
        "POLIsolationModeWrapperProxy: Initialization failed"
    );
}
```
이러한 유효성 검사 부족은 생성자가 모든 calldata를 맹목적으로 실행하게 하여, 배포 중에 중요한 계약 파라미터를 잘못 설정할 수 있음을 의미합니다.

`POLIsolationModeUnwrapperUpgradeableProxy`에도 유사한 문제가 존재한다는 점에 유의하세요.

**영향:** 프록시가 0 또는 유효하지 않은 `vaultFactory` 주소로 초기화되어 작동하지 않거나 안전하지 않게 될 수 있습니다. 또한 구현 계약이 업그레이드되어 더 약한 접근 제어를 가진 새로운 함수가 도입되는 경우, 이 패턴은 새로운 프록시 초기화 중에 해당 함수가 호출될 수 있도록 허용할 수 있습니다.

**권장 완화 방법:** 생성자에서 함수 선택자(selector)와 파라미터 모두에 대해 명시적인 유효성 검사를 추가하는 것을 고려하세요:

```solidity
constructor(
    address _berachainRewardsRegistry,
    bytes memory _initializationCalldata
) {
    BERACHAIN_REWARDS_REGISTRY = IBerachainRewardsRegistry(_berachainRewardsRegistry);

    // 함수 선택자가 initialize(address)인지 확인
    require(
        _initializationCalldata.length == 36 &&
        bytes4(_initializationCalldata[0:4]) == bytes4(keccak256("initialize(address)")),
        "Invalid initialization function"
    );

     // vaultFactory 주소가 0이 아닌지 디코딩 및 확인
    address vaultFactory = abi.decode(_initializationCalldata[4:], (address));
    require(vaultFactory != address(0), "Zero vault factory address");

    Address.functionDelegateCall(
        implementation(),
        _initializationCalldata,
        "POLIsolationModeWrapperProxy: Initialization failed"
    );
}
```

**Dolomite:**
인지함.

**Cyfrin:** 인지함.


### 보상 볼트 유형 전환 시 보상 손실 위험 (Reward loss risk when transitioning between reward vault types)

**설명:** `InfraredBGTMetaVault` 계약에서 사용자의 자산에 대한 기본 보상 볼트 유형이 변경될 때(예: NATIVE 또는 BGTM에서 INFRARED로), 현재 로직은 유형을 전환하기 전에 미수령 보상을 청구하려고 시도합니다. 그러나 구현상 사용자의 현재 볼트 유형에서 보상을 가져오지 못해 누적된 보상이 영구적으로 손실될 수 있습니다.

이 문제는 `_setDefaultRewardVaultTypeByAsset` 및 `_getReward` 함수 간의 관계에서 발생합니다:

- `_stake`를 통해 토큰을 스테이킹할 때, 함수는 자산이 INFRARED 보상 유형을 사용하도록 보장하기 위해 `_setDefaultRewardVaultTypeByAsset`을 호출합니다.
- 자산의 현재 보상 유형이 INFRARED가 아닌 경우, `_setDefaultRewardVaultTypeByAsset`은 유형을 변경하기 전에 자산에 대한 보류 중인 보상을 청구하기 위해 `_getReward`를 호출합니다.
- 그러나 `_getReward`는 사용자의 현재 보상 유형을 사용하는 대신 보상 볼트 유형을 INFRARED로 하드코딩합니다.

```solidity
function _getReward(address _asset) internal {
    IBerachainRewardsRegistry.RewardVaultType rewardVaultType = IBerachainRewardsRegistry.RewardVaultType.INFRARED;
    IInfraredVault rewardVault = IInfraredVault(REGISTRY().rewardVault(
        _asset,
        rewardVaultType  // 사용자의 현재 유형을 무시하고 항상 INFRARED를 사용
    ));
    // ... 보상 청구 로직 ...
}
```

즉, 다른 보상 유형(예: NATIVE 또는 BGTM)에서 INFRARED로 전환할 때, 사용자의 보상이 다른 볼트 유형에 누적되어 있음에도 불구하고 계약은 INFRARED 볼트에서 보상을 청구하려고 시도합니다.

또한, 사용자가 스테이킹 잔액을 가지고 있을 때 유형 변경을 방지해야 하는 assertion은 비-INFRARED 유형에 대해 효과가 없습니다:

```solidity
assert(getStakedBalanceByAssetAndType(_asset, currentType) == 0);
```

이 assertion은 모든 스테이킹 함수에 있는 `onlyInfraredType` modifier로 인해 `getStakedBalanceByAssetAndType`이 INFRARED에 대해서만 0이 아닌 잔액을 가질 수 있기 때문에 비-INFRARED 유형에 대해 항상 통과합니다.

**영향:** 프로토콜은 현재 INFRARED 보상 유형만 지원할 의도이지만, 이 문제는 향후 확장을 위한 잠재적 위험을 초래합니다.

프로토콜이 추가 보상 유형(NATIVE, BGTM)에 대한 지원을 추가하고 사용자가 이러한 볼트에서 보상을 누적하는 경우, INFRARED 유형으로 전환할 때 이 보상을 영구적으로 잃게 됩니다. 레지스트리가 INFRARED를 기본 보상 유형으로 사용하도록 업데이트되면, 원래 볼트에 있는 보상은 정상적인 계약 상호 작용을 통해 접근할 수 없게 됩니다.

이 문제의 심각성은 다중 보상 유형 지원에 대한 프로토콜의 로드맵에 따라 달라집니다. INFRARED 이외의 확장에 대한 명확한 계획이 있다면 이는 사용자에게 상당한 보상 손실 위험을 나타냅니다.

**개념 증명 (Proof of Concept):** 다음 시나리오를 고려해 보세요:
- 향후 버전에서 사용자가 NATIVE 보상 볼트에 누적된 보상을 가지고 있다고 가정합니다.
- 사용자가 INFRARED로 전환하기 위해 `_setDefaultRewardVaultTypeByAsset`을 트리거하는 함수를 호출합니다.
- 이 계약의 잔액은 INFRARED에 대해서만 추적되므로 `assert(getStakedBalanceByAssetAndType(_asset, currentType) == 0)` assertion이 통과합니다.
- `_getReward(_asset)`이 호출되지만 NATIVE 볼트 대신 INFRARED 볼트에서 가져옵니다.
- 레지스트리가 `REGISTRY().setDefaultRewardVaultTypeFromMetaVaultByAsset(_asset, _type)`을 통해 업데이트됩니다.
- 사용자의 NATIVE 볼트에 있는 보상은 이제 접근할 수 없습니다.

**권장 완화 방법:** 프로토콜이 향후 여러 보상 유형을 지원할 계획이라면, `_getReward` 함수를 수정하여 사용자의 현재 보상 볼트 유형에서 보상을 청구하도록 하세요.

또한 여러 유형이 지원될 경우 모든 보상 유형에 대한 적절한 잔액 추적을 구현하거나, 보상 유형 간 전환 시 먼저 수동으로 보상을 청구해야 함을 명확히 문서화하는 것을 고려하세요.

INFRARED 볼트만 지원되는 경우 `_setDefaultRewardVaultTypeByAsset`을 제거하는 것을 고려하세요. 이 함수는 `stake` 함수에서 호출되므로, 이를 제거하면 코드가 단순화되고 가스가 절약됩니다.

**Dolomite:**
인지함. 여러 보상 유형을 허용하려면 코드를 상당히 변경해야 한다는 것을 알고 있음.

**Cyfrin**
인지함.



### Infrared 볼트 스테이킹이 일시 중지되었을 때 iBGT 보상을 상환할 수 없음 (Rewards in iBGT cannot be redeemed when infrared vault staking is paused)

**Description:** 현재 보상 처리 메커니즘은 스테이킹이 비활성화되었을 때 대안을 제공하지 않고 iBGT 보상을 스테이킹 풀에 자동으로 재투자하도록 강제합니다.

Infrared 프로토콜이 스테이킹을 일시 중지하면(보안 비상 사태나 기술적 문제 등으로 인해 발생 가능), 사용자는 획득한 보상에 접근할 방법이 없어 스테이킹이 재개될 때까지 사실상 자산이 동결됩니다. [InfraredVault](https://berascan.com/address/0x67b4e6721ad3a99b7ff3679caee971b07fd85cd1#code)는 스테이킹이 일시 중지된 경우에도 보상 상환/언스테이킹을 막지 않는다는 점에 주목할 필요가 있습니다.

문제는 iBGT 보상을 자동으로 재투자하려고 시도하는 `_handleRewards` 함수에 있습니다:

```solidity
function _handleRewards(IInfraredVault.UserReward[] memory _rewards) internal {
    IIsolationModeVaultFactory factory = IIsolationModeVaultFactory(VAULT_FACTORY());
    for (uint256 i = 0; i < _rewards.length; ++i) {
        if (_rewards[i].amount > 0) {
            if (_rewards[i].token == UNDERLYING_TOKEN()) {
                _setIsDepositSourceThisVault(true);
                factory.depositIntoDolomiteMargin(
                    DEFAULT_ACCOUNT_NUMBER,
                    _rewards[i].amount
                );
                assert(!isDepositSourceThisVault());
            } else {
                // ... 다른 토큰 유형 처리 ...
            }
        }
    }
}
```

`pauseStaking()` 함수를 통해 Infrared 볼트의 스테이킹 함수가 일시 중지되면...

```solidity
/// @inheritdoc IInfraredVault
function pauseStaking() external onlyInfrared {
    if (paused()) return;
    _pause();
}
```

...`executeDepositIntoVault`의 스테이킹 작업은 InfraredVault의 `whenNotPaused` modifier로 인해 실패합니다:

```solidity
function stake(uint256 amount) external whenNotPaused {
     // 코드
```

**영향:** Infrared 볼트에서 스테이킹이 일시 중지된 기간 동안, Infrared 볼트에서는 상환이 허용됨에도 불구하고 Dolomite 사용자는 획득한 보상에 접근할 수 없습니다.

**개념 증명 (Proof of Concept):** `TestInfraredVault`는 온체인 Infrared 볼트 계약에 맞춰 `Pausable`로 만들고 `whenNotPaused` modifier를 `stake` 함수에 추가했습니다.

```solidity
contract TestInfraredVault is ERC20, Pausable {

   function unpauseStaking() external {
        if (!paused()) return;
        _unpause();
    }

    function pauseStaking() external {
        if (paused()) return;
        _pause();
    }

    function stake(uint256 amount) external whenNotPaused {
        _mint(msg.sender, amount);
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
    }
}
```

`InfraredBGTIsolationModeTokenVaultV1.ts`의 `#getReward` 테스트 클래스에 다음 테스트를 추가하세요.

```typescript
  it('should revert when staking is paused and rewards are in iBGT', async () => {
      await testInfraredVault.setRewardTokens([core.tokens.iBgt.address]);

      // iBGT를 보상 토큰으로 추가하고 보상 자금 지원
      await core.tokens.iBgt.connect(iBgtWhale).approve(testInfraredVault.address, rewardAmount);
      await testInfraredVault.connect(iBgtWhale).addReward(core.tokens.iBgt.address, rewardAmount);
      await registry.connect(core.governance).ownerSetIBgtStakingVault(testInfraredVault.address);

      // iBGT를 볼트에 예치
      await iBgtVault.depositIntoVaultForDolomiteMargin(defaultAccountNumber, amountWei);
      await expectProtocolBalance(core, iBgtVault, defaultAccountNumber, iBgtMarketId, amountWei);

      // 보상 누적을 위해 시간 진행
      await increase(ONE_DAY_SECONDS * 30);

      // InfraredVault에서 스테이킹 일시 중지
      await testInfraredVault.pauseStaking();
      expect(await testInfraredVault.paused()).to.be.true;

      // getReward 호출은 스테이킹 일시 중지로 인해 iBGT 보상 재투자가 실패하므로 revert되어야 함
      await expectThrow(
        iBgtVault.getReward()
      );


      // 잔액이 변경되지 않았는지 확인
      await expectWalletBalance(iBgtVault, core.tokens.iBgt, ZERO_BI);
      await expectProtocolBalance(core, iBgtVault, defaultAccountNumber, iBgtMarketId, amountWei);

      // 정상적인 작동을 계속할 수 있도록 일시 중지 해제
      await testInfraredVault.unpauseStaking();
      expect(await testInfraredVault.paused()).to.be.false;

      // 이제 getReward가 성공해야 함
      await iBgtVault.getReward();
      await expectWalletBalance(iBgtVault, core.tokens.iBgt, ZERO_BI);
      await expectProtocolBalance(core, iBgtVault, defaultAccountNumber, iBgtMarketId, amountWei.add(rewardAmount));

      // 보상이 재스테이킹되었는지 확인
      expect(await testInfraredVault.balanceOf(iBgtVault.address)).to.eq(amountWei.add(rewardAmount));
    });
```

**권장 완화 방법:** 보상 재투자를 시도하기 전에 BGT 스테이킹 볼트가 `paused` 상태인지 확인하는 것을 고려하세요. 볼트가 일시 중지된 경우, 보상을 볼트에 보관하거나 볼트 소유자에게 다시 전송할 수 있습니다.

**Dolomite:** [7b83e77](https://github.com/dolomite-exchange/dolomite-margin-modules/commit/7b83e778d739c9afb039a8a8d4fe06d931f4bb22)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Informational


### `_handleRewards` 함수에서 0이 아닌 보상 금액에 대한 중복 확인 (Redundant check for non-zero reward amount in `_handleRewards` function)

**설명:** `InfraredBGTIsolationModeTokenVaultV1.sol`에서 `_handleRewards` 함수는 아래와 같이 양수 보상 금액에 대한 중복 확인을 포함하고 있습니다:

```solidity
//InfraredBGTIsolationModeTokenVaultV1.sol
function _handleRewards(IInfraredVault.UserReward[] memory _rewards) internal {
    IIsolationModeVaultFactory factory = IIsolationModeVaultFactory(VAULT_FACTORY());
    for (uint256 i = 0; i < _rewards.length; ++i) {
        if (_rewards[i].amount > 0) {  // <-- 이 확인은 중복됨
            if (_rewards[i].token == UNDERLYING_TOKEN()) {
                _setIsDepositSourceThisVault(true);
                factory.depositIntoDolomiteMargin(
                    DEFAULT_ACCOUNT_NUMBER,
                    _rewards[i].amount
                );
                assert(!isDepositSourceThisVault());
            } else {
                // ... 나머지 함수
            }
        }
    }
}
```

이 확인이 중복되는 이유는 [InfraredVault](https://berascan.com/address/0x67b4e6721ad3a99b7ff3679caee971b07fd85cd1#code)의 getAllRewardsForUser 함수가 양수 금액의 보상만 반환하기 때문입니다:

```solidity
// InfraredVault.sol
function getAllRewardsForUser(address _user) external view returns (UserReward[] memory) {
    uint256 len = rewardTokens.length;
    UserReward[] memory tempRewards = new UserReward[](len);
    uint256 count = 0;
    for (uint256 i = 0; i < len; i++) {
        uint256 amount = earned(_user, rewardTokens[i]);
        if (amount > 0) {  // @audit <-- 이미 양수 금액 필터링 중
            tempRewards[count] = UserReward({token: rewardTokens[i], amount: amount});
            count++;
        }
    }
    // 0이 아닌 보상의 정확한 크기로 새 배열 생성
    UserReward[] memory userRewards = new UserReward[](count);
    for (uint256 j = 0; j < count; j++) {
        userRewards[j] = tempRewards[j];
    }
    return userRewards;
}
```

**권장 완화 방법:** 중복 확인을 제거하는 것을 고려하세요.

**Dolomite:**
더 이상 적용되지 않음. 보상 관련 수정으로 인해 코드가 많이 변경됨.

**Cyfrin**
인지함.



### 프록시 계약에서 일관되지 않은 ETH 처리 패턴 (Inconsistent ETH handling pattern in Proxy Contracts)

**설명:** 프록시 계약이 `receive()` 및 `fallback()` 함수를 통해 들어오는 ETH 트랜잭션을 처리하는 방식에 일관성이 없습니다. 일부 프록시 계약은 두 함수 모두 구현 계약에 위임하는 반면, 다른 계약은 `receive()` 함수를 비워두고 `fallback()` 함수만 위임합니다.

예를 들어, `MetaVaultUpgradeableProxy.sol`에서는 두 함수 모두 위임합니다:

```solidity
// MetaVaultUpgradeableProxy
receive() external payable requireIsInitialized {
    _callImplementation(implementation());
}

fallback() external payable requireIsInitialized {
    _callImplementation(implementation());
}
```

반면 `POLIsolationModeWrapperUpgradeableProxy.sol` 및 `POLIsolationModeUnwrapperUpgradeableProxy`에서는 `fallback()` 함수만 위임합니다:

```solidity
// POLIsolationModeWrapperUpgradeableProxy
receive() external payable {} // solhint-disable-line no-empty-blocks

fallback() external payable {
    _callImplementation(implementation());
}
```

이는 설계상의 선택이며 보안 문제는 아니지만, 모든 프록시가 유사한 방식으로 ETH 전송을 처리할 것으로 기대하는 개발자들에게 잠재적인 혼란을 줄 수 있습니다.

**권장 완화 방법:** 다른 개발자를 위해 의도된 동작을 명확히 하기 위해 선택한 접근 방식과 이유를 계약 주석에 문서화하는 것을 고려하세요.

**Dolomite:** [d4ceeef](https://github.com/dolomite-exchange/dolomite-margin-modules/commit/d4ceeefc9c2a5b8c51c8ea77512e499a2e0bc811)에서 수정됨.

**Cyfrin:** 검증됨.


### 일부 중요한 상태 변경에 대한 이벤트 방출 누락 (Missing event emissions for some important state changes)

**설명:** POL 계약에서 몇 가지 중요한 상태 변경 작업에 대한 이벤트 방출이 누락되었습니다.


_1. POLIsolationModeTokenVaultV1_

- `prepareForLiquidation`: 청산 준비 시 이벤트 없음
- `stake/unstake`: (언)스테이킹 작업 시 이벤트 없음
- `getReward`: 보상 청구 시 이벤트 없음
- `exit`:  포지션 종료 시 이벤트 없음

_2. InfraredBGTMetaVault_
- `chargeDTokenFee`: 수수료 부과 시 이벤트 없음

_3. POLIsolationModeTraderBaseV2_
- `_POLIsolationModeTraderBaseV2__initialize`:  볼트 팩토리 설정 시 이벤트 없음

_4. MetaVaultRewardTokenFactory_
- `depositIntoDolomiteMarginFromMetaVault`: 메타 볼트에서 예치 시 이벤트 없음
- `depositIntoDolomiteMarginFromMetaVault`: 메타 볼트에서 다른 토큰 예치 시 이벤트 없음


**권장 완화 방법:** 코드베이스를 검토하고 중요한 상태 변경을 추적하는 이벤트를 추가하는 것을 고려하세요.


**Dolomite:** [e556252](https://github.com/dolomite-exchange/dolomite-margin-modules/commit/e556252bc49d222ea80540242f832fc996711c26) 및 [ccfcd12](https://github.com/dolomite-exchange/dolomite-margin-modules/commit/ccfcd1278afafae355020bdee4673c792687a109)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Gas Optimization


### `_handleRewards` 가스 최적화 (`_handleRewards` gas optimisation)

**설명:** `InfraredBGTMetaVault._performDepositRewardByRewardType`은 Infrared 볼트에서 보상을 가져올 때마다 호출되는 함수입니다. 이 함수는 다음과 같이 최적화할 수 있습니다:

- 보상 금액 > 0 확인 제거 ([*`_handleRewards` 함수에서 0이 아닌 보상 금액에 대한 중복 확인*](#redundant-check-for-nonzero-reward-amount-in-handlerewards-function)에 나열된 대로)
- 자주 사용되는 함수 `DOLOMITE_MARGIN()`, `OWNER()` 캐싱
- 루프 시작 시 보상 토큰 및 보상 금액 캐싱
- 보상 카운터 증가 시 unchecked integer 사용


**권장 완화 방법:** 아래의 최적화된 버전을 사용하는 것을 고려하세요:

```solidity
function _handleRewards(IInfraredVault.UserReward[] memory _rewards) internal {
    IIsolationModeVaultFactory factory = IIsolationModeVaultFactory(VAULT_FACTORY());
    address owner = OWNER();
    IDolomiteMargin dolomiteMargin = DOLOMITE_MARGIN();

    for (uint256 i = 0; i < _rewards.length;) {
        //@audit InfraredVault는 0이 아닌 보상만 보내므로 중복 확인 제거
        address rewardToken = _rewards[i].token;
        uint256 rewardAmount = _rewards[i].amount;

        if (rewardToken == UNDERLYING_TOKEN()) {
            _setIsDepositSourceThisVault(true);
            factory.depositIntoDolomiteMargin(
                DEFAULT_ACCOUNT_NUMBER,
                rewardAmount
            );
            assert(!isDepositSourceThisVault());
        } else {
        try dolomiteMargin.getMarketIdByTokenAddress(rewardToken) returns (uint256 marketId) {
                        IERC20(rewardToken).safeApprove(address(dolomiteMargin), rewardAmount);
                        try factory.depositOtherTokenIntoDolomiteMarginForVaultOwner(
                            DEFAULT_ACCOUNT_NUMBER,
                            marketId,
                           rewardAmount
                        ) {} catch {
                            IERC20(rewardToken).safeApprove(address(dolomiteMargin), 0);
                            IERC20(rewardToken).safeTransfer(owner, rewardAmount);
                        }
                    } catch {
                        IERC20(rewardToken).safeTransfer(owner,  rewardAmount);
                    }
        }
        unchecked { ++i; }
    }
}
```


**Dolomite:**
더 이상 적용되지 않음. 보상 관련 수정으로 인해 코드가 많이 변경됨.

**Cyfrin:** 인지함.

\clearpage
