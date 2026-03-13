**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)


**Assisting Auditors**

[Alex Roan](https://twitter.com/alexroan)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

---

# Findings
## Informational


### 상태 변경 시 이벤트 방출 부족 (Lack of events emitted on state changes)

**설명:** 투명성과 추적성을 높이기 위해 다음 함수들은 이벤트를 방출하는 것이 이상적입니다:


[`Vault::setDelegateRegistry`](https://github.com/stakedotlink/audit-2025-02-linkpool/blob/046c65a9c771315816bc59533183f52661af8e5e/contracts/linkStaking/base/Vault.sol#L180-L186) 및 [`VaultControllerStrategy::setDelegateRegistry`](https://github.com/stakedotlink/audit-2025-02-linkpool/blob/046c65a9c771315816bc59533183f52661af8e5e/contracts/linkStaking/base/VaultControllerStrategy.sol#L708-L714):

```diff
  function setDelegateRegistry(address _delegateRegistry) external onlyOwner {
      delegateRegistry = _delegateRegistry;
+     emit SetDelegateRegistry(_delegateRegistry);
  }
```

[`FundFlowController::setNonLINKRewardReceiver`](https://github.com/stakedotlink/audit-2025-02-linkpool/blob/046c65a9c771315816bc59533183f52661af8e5e/contracts/linkStaking/FundFlowController.sol#L325-L331):

```diff
  function setNonLINKRewardReceiver(address _nonLINKRewardReceiver) external onlyOwner {
      nonLINKRewardReceiver = _nonLINKRewardReceiver;
+     emit SetNonLINKRewardReceiver(_nonLINKRewardReceiver);
  }
```

추가적으로, [`FundFlowController::withdrawTokenRewards`](https://github.com/stakedotlink/audit-2025-02-linkpool/blob/046c65a9c771315816bc59533183f52661af8e5e/contracts/linkStaking/FundFlowController.sol#L307-L323)에서 보상을 인출할 때 이벤트를 방출할 수 있습니다:
```diff
  function withdrawTokenRewards(address[] calldata _vaults, address[] calldata _tokens) external {
      // ...
+     emit WithdrawTokenRewards(msg.sender, _vaults, _tokens);
  }
```

이러한 함수에 이벤트를 추가하여 언제 누가 이러한 작업을 실행했는지에 대한 명확한 온체인 기록을 제공하는 것을 고려하세요. 이는 투명성을 개선하고 변경 사항을 더 쉽게 추적할 수 있게 합니다.

**Stake.Link:** 인지함.

**Cyfrin:** 인지함.

\clearpage
## Gas Optimization


### 보상 토큰 인출 시 불필요한 토큰 전송 (Unnecessary token transfer when withdrawing reward tokens)

**설명:** LINK가 아닌 보상 토큰을 청구할 때, 토큰은 `Vault -> FundFlowController -> nonLINKRewardReceiver`로 전송됩니다:

[`Vault::withdrawTokenRewards`](https://github.com/stakedotlink/audit-2025-02-linkpool/blob/046c65a9c771315816bc59533183f52661af8e5e/contracts/linkStaking/base/Vault.sol#L168-L178)는 `msg.sender` (`FundFlowController`)에게 전송합니다:
```solidity
function withdrawTokenRewards(address[] calldata _tokens) external onlyFundFlowController {
    for (uint256 i = 0; i < _tokens.length; ++i) {
        IERC20Upgradeable rewardToken = IERC20Upgradeable(_tokens[i]);
        uint256 balance = rewardToken.balanceOf(address(this));
        if (balance != 0) rewardToken.safeTransfer(msg.sender, balance);
    }
}
```

그리고 [`FundFlowController::withdrawTokenRewards`](https://github.com/stakedotlink/audit-2025-02-linkpool/blob/046c65a9c771315816bc59533183f52661af8e5e/contracts/linkStaking/FundFlowController.sol#L312-L323)는 프로토콜 지갑 `nonLINKRewardReceiver`에게 전송합니다:
```solidity
function withdrawTokenRewards(address[] calldata _vaults, address[] calldata _tokens) external {
    for (uint256 i = 0; i < _vaults.length; ++i) {
        IVault(_vaults[i]).withdrawTokenRewards(_tokens);
    }

    for (uint256 i = 0; i < _tokens.length; ++i) {
        IERC20Upgradeable rewardToken = IERC20Upgradeable(_tokens[i]);
        if (address(rewardToken) == linkToken) revert InvalidToken();
        uint256 balance = rewardToken.balanceOf(address(this));
        if (balance != 0) rewardToken.safeTransfer(nonLINKRewardReceiver, balance);
    }
}
```

이는 볼트가 `nonLINKRewardReceiver`로 직접 전송하도록 하여 흐름에서 한 번의 토큰 전송을 제거함으로써 최적화될 수 있습니다:

```solidity
function withdrawTokenRewards(address[] calldata _vaults, address[] calldata _tokens) external {
    // cache linkToken
    address _linkToken = linkToken;

    // check for LINK token
    for (uint256 i = 0; i < _tokens.length; ) {
        if (_tokens[i] == _linkToken) revert InvalidToken();
        unchecked { ++i; }
    }

    for (uint256 i = 0; i < _vaults.length; ++i) {
        // add `nonLINKRewardReceiver` in the call to vault.withdrawTokenRewards
        IVault(_vaults[i]).withdrawTokenRewards(_tokens, nonLINKRewardReceiver);
    }
}
```

```diff
- function withdrawTokenRewards(address[] calldata _tokens) external onlyFundFlowController {
+ function withdrawTokenRewards(address[] calldata _tokens, address _receiver) external onlyFundFlowController {
      for (uint256 i = 0; i < _tokens.length; ++i) {
          IERC20Upgradeable rewardToken = IERC20Upgradeable(_tokens[i]);
          uint256 balance = rewardToken.balanceOf(address(this));
-         if (balance != 0) rewardToken.safeTransfer(msg.sender, balance);
+         if (balance != 0) rewardToken.safeTransfer(_receiver, balance);
      }
  }
```

`Vault::withdrawTokenRewards`는 이미 `onlyFundFlowController`에 의해 보호되므로 추가적인 위험을 초래하지 않습니다.

**Stake.Link:** 인지함.

**Cyfrin:** 인지함.

\clearpage
