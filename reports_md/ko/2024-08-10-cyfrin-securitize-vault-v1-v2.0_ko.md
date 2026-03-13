**Lead Auditors**

[Hans](https://twitter.com/hansfriese)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 심각도 높음 (Critical Risk)


### 입금 및 상환에 대한 접근 제어가 불완전함

**설명:** `SecuritizeVault`는 입금과 상환에 대한 접근을 제한하도록 설계되었지만, 필요한 오버라이드가 누락되어 접근 제어가 적용된 함수들을 우회해 사용할 수 있습니다.
이 볼트는 OpenZeppelin의 `ERC4626Upgradeable`을 상속하며, `ERC4626Upgradeable`은 기본적으로 여러 공개 함수를 노출합니다. 특히 `ERC4626Upgradeable`은 mint/deposit와 redeem/withdraw를 각각 두 가지 방식으로 제공합니다.
```solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256);
function mint(uint256 shares, address receiver) public virtual returns (uint256);

function withdraw(uint256 assets, address receiver, address owner) public virtual returns (uint256);
function redeem(uint256 shares, address receiver, address owner) public virtual returns (uint256);
```
여기서 `deposit()`와 `withdraw()`는 자산 수량을 입력으로 받고, `mint()`와 `redeem()`는 공유 수량을 입력으로 받습니다.

`SecuritizeVault`는 `deposit()`와 `redeem()`를 추가 접근 제어와 함께 오버라이드했습니다.
`deposit()`는 오직 볼트 자신에게만 예치할 수 있도록 제한했고, `redeem()`는 리디머(redeemer)에게만 허용했습니다.
하지만 `mint()`와 `withdraw()`는 오버라이드하지 않았기 때문에 이 제한을 우회할 수 있습니다.

또한 볼트가 소유자에 의해 일시 중지된 상태에서도 deposit과 redeem이 가능해집니다.

**영향:** 접근 제어가 깨져 누구나 다른 주소를 위해 입금할 수 있고, 프로토콜 의도와 달리 누구나 상환할 수 있습니다.
더 나아가 소유자가 볼트 컨트랙트를 일시 중지해도 deposit과 redeem은 계속 동작합니다.
영향은 CRITICAL로 평가됩니다.

**개념 증명 (Proof of Concept):**
다음 테스트를 `deposit.test.ts`에 추가하십시오.
```typescript
    it('Cyfrin: Can deposit using a mint function even if receiver is not the same as sender', async () => {
      const { vault, dsMock, redeemer, owner } = await loadFixture(deployRedemptionVault);
      await dsMock.mint(owner.address, amount);
      await dsMock.approve(vault.target, amount);

      console.log(await vault.balanceOf(redeemer.address));
      await vault.mint(amount, redeemer.address);
      console.log(await vault.balanceOf(redeemer.address));
    });
```
**권장 완화 조치:**
- 동일한 접근 제어를 적용하도록 `mint()`와 `withdraw()`도 오버라이드하십시오.
- 일반적으로 `deposit()`와 `redeem()` 조합을 함께 노출하는 것은 이상적이지 않습니다. 보통은 deposit/withdraw 또는 mint/redeem 중 하나의 쌍을 택합니다.
- `receiverSenderNotEqual(receiver)` 수정자는 기술적으로 큰 의미가 없습니다. 누구나 볼트 공유 토큰을 다른 사람에게 전송할 수 있기 때문입니다.

**Securitize:** 커밋 [52350aa](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/52350aa809c140edc6f794baabc7e53891b37852)에서 수정됨.

**Cyfrin:** 확인함.
- 공개 함수 `mint()`와 `withdraw()`는 이제 항상 revert하도록 오버라이드되었습니다.
- `receiverSenderNotEqual(receiver)` 수정자는 제거되었습니다.

\clearpage
## 낮은 위험 (Low Risk)


### 안전하지 않은 ERC20 연산을 사용하면 안 됨

**설명:** 현재 구현은 여러 곳에서 ERC20 토큰 전송에 `transfer()` 함수를 사용합니다.
```solidity
SecuritizeVault.sol
186: bool success = assetToken.transfer(_to, _transferAmount);
307: bool success = liquidationToken.transfer(msg.sender, assets);
```
하지만 모든 ERC20 토큰이 표준을 엄격히 준수하는 것은 아닙니다. 어떤 토큰은 boolean을 반환하지 않고, 어떤 토큰은 실패해도 revert하지 않습니다.

**권장 완화 조치:** 반환값 검사와 비표준 토큰 처리를 함께 제공하는 OpenZeppelin의 `SafeERC20`과 `safeTransfer`, `safeTransferFrom` 사용을 권장합니다.

**Securitize:** [3cd6413](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/3cd641372c614587ca3b9f7b6ca850397caa6e5c)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### 일부 함수 이름이 오해를 부를 수 있음

**설명:** 일부 함수 이름은 실제 용법과 맞지 않아 오해를 부를 수 있습니다.

- 수정자 `receiverSenderNotEqual()`는 `msg.sender`가 인자와 같은지 확인하는 데 사용되는데, 인자 이름을 그대로 포함한 현재 이름은 적절하지 않습니다. 특히 같은 수정자가 L273에서는 `msg.sender == owner` 확인에도 사용되기 때문입니다.
`senderEqualTo` 또는 `msgSenderEqualTo`가 더 적절합니다.
```solidity
37: modifier receiverSenderNotEqual(address _receiver) {
38:         require(_receiver == msg.sender, "Receiver must be equal to sender");
39:         _;
40:     }
```
- `impairedAsset()`는 실제 손상된 양을 반환하는 것이 아니라 상태를 boolean으로 반환하므로, `assetIsImpaired()`가 더 적절합니다.
- `impairedVault()`보다 `vaultIsImpaired()`가 더 적절합니다.

**Securitize:** [b337b12](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/b337b1271b72c5d53e46412c734672c45afc6039)에서 수정됨.

**Cyfrin:** 확인함.


### 일부 주석이 오해를 부를 수 있음

**설명:** 몇몇 주석이 잘못되었거나 오해를 부를 수 있습니다.

아래 스니펫에서 `impairedAssetBalance`는 플래그가 아니라 `impairedAsset()` 함수를 의미했어야 합니다.
```solidity
SecuritizeVault.sol
177: * - The `impairedAssetBalance` flag must be true, indicating that the asset balance is indeed impaired.
```

또한 아래 주석의 L321은 잘못되었습니다. 볼트 컨트랙트가 자산을 직접 사용자에게 전송하는 경우는 없기 때문입니다.
```solidity
SecuritizeVault.sol
320:      * This can occur if users transfer assets directly to the vault instead of using the deposit function,
321:      * or if the vault transfers assets directly to users instead of using the redeem function.
```

**Securitize:** [a2bae8](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/a2bae865466a79c9a079a0957efa05ea0d6f68a6) 및 [b337b1](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/b337b1271b72c5d53e46412c734672c45afc6039)에서 수정됨.

**Cyfrin:** 확인함.


### 불필요한 이벤트 발생을 피해야 함

**설명:** `setLiquidationOpenToPublic` 함수는 상태값 `liquidationOpenToPublic`을 설정할 때 현재 상태와 무관하게 전달된 파라미터로 덮어씁니다. 만약 전달된 값이 현재 값과 동일하다면, 불필요하게 `LiquidationOpenToPublic` 이벤트가 한 번 더 발생합니다.

**Securitize:** [a2bae86](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/a2bae865466a79c9a079a0957efa05ea0d6f68a6)에서 수정됨.

**Cyfrin:** 확인함.


### 설계 관련 일반 메모

**설명:**
1. 전반적으로 팀이 왜 ERC4626 볼트를 상속했는지 명확하지 않습니다.
ERC-4626 볼트의 핵심 목적은 "수익 발생형" 토큰화 볼트에 대한 표준화된 인터페이스를 제공하여 공유 가격, 즉 기초 자산 대비 볼트 공유의 가치를 일관되게 추적·계산하게 하는 것입니다.
하지만 `SecuritizeVault`는 공유와 자산 사이의 1:1 비율을 강제하므로, 그 핵심 목적이 사실상 무력화됩니다.
또한 이 볼트에는 ERC4626 표준에 없는 `liquidate()` 함수가 존재하며, 대출 프로토콜에서 흔히 쓰이는 용어와 겹쳐 사용자와 엔지니어를 혼란스럽게 할 수 있습니다.
현재 구현에서는 ERC4626을 반드시 써야 할 명확한 이유가 보이지 않습니다.
오히려 과한 선택일 수 있고, 실수 가능성도 높입니다.

2. 현재 구현에는 `impairedAssetBalance`와 `impairedVaultBalance`가 있고, 각각에 대응하는 전송 함수가 존재합니다. 왜 이런 것들이 필요한지 명확하지 않습니다. 팀과 통화 중 받은 맥락에 따르면, 이는 실수로 볼트 컨트랙트에 보내진 초과 자산 또는 공유를 쓸어내기 위해 도입된 것으로 보입니다. 일반적으로 이는 프로토콜 자체의 큰 우려 사항은 아닙니다. 결국 호출자의 실수이며, 프로토콜 책임은 아니기 때문입니다. 만약 프로토콜이 정말 사용자 실수를 우려한다면 단일 관리자 함수 하나면 충분합니다. (아래 [Compound의 sweep 함수](https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CErc20.sol#L124) 참조.) 참고로 현재 구현은 모든 거래마다 내부 함수 `_validateBalance()`를 호출하므로 불필요한 가스도 소모합니다.
```solidity
    /**
     * @notice A public function to sweep accidental ERC-20 transfers to this contract. Tokens are sent to admin (timelock)
     * @param token The address of the ERC-20 token to sweep
     */
    function sweepToken(EIP20NonStandardInterface token) override external {
        require(msg.sender == admin, "CErc20::sweepToken: only admin can sweep tokens");
        require(address(token) != underlying, "CErc20::sweepToken: can not sweep underlying token");
        uint256 balance = token.balanceOf(address(this));
        token.transfer(admin, balance);
    }
```


**Securitize:** 우리는 향후 변동형 순자산가치(NAV) 가격을 도입할 계획이 있기 때문에 단순한 wrapped ERC-20 대신 `ERC-4626` 표준을 구현하기로 했습니다. 이는 공유 대 자산 비율이 가변적이 된다는 뜻이며, 따라서 수익 발생형 토큰화 볼트를 위한 `ERC-4626`의 표준 인터페이스가 유리합니다. 현재 사용 사례에서는 1:1 비율을 강제하고 있지만, 구현이 발전하면서 이 부분은 바뀔 예정입니다.

`liquidate()` 함수는 szToken의 주요 사용처가 될 대출 프로토콜과의 통합 요구를 충족하기 위해 포함했습니다. `liquidate()`는 szToken을 USDC로 전환하는 과정을 돕습니다. 이 함수가 `ERC-4626` 표준의 일부가 아니어서 혼란을 줄 수 있다는 점은 인정하지만, 의도한 기능을 위해서는 필요합니다. 또한 모든 사용자가 자신의 szToken을 DS Token으로 상환할 수 있는 것은 아니므로, 일부 사용자에게는 liquidation이 유일한 출구가 됩니다.

우리는 `_validateBalance()`를 제거했고 impairment 관련 메서드도 아래와 같이 수정했습니다.

```solidity
    /**
     * @dev Checks if the vault's asset balance impairment.
     *
     * @return uint256 Returns true if the asset balance is impaired, false otherwise.
     */
    function assetImpairedBalance() public view returns (uint256) {
        uint256 assetBalance = IERC20(asset()).balanceOf(address(this));
        uint256 impairedAssetBalance;
        if (assetBalance > totalSupply()) {
            impairedAssetBalance = assetBalance - totalSupply();
        }
        return impairedAssetBalance;
    }

    /**
     * @dev Checks if the vault's balance imparement.
     *
     * @return uint256 Returns the impaired vault balance.
     */
    function vaultImpairedBalance() public view returns (uint256) {
        uint256 impairedVaultBalance;
        if (balanceOf(address(this)) > 0) {
            impairedVaultBalance = balanceOf(address(this));
        }
        return impairedVaultBalance;
    }
```
이제 이 로직은 view 함수이므로 필요할 때만 계산되며 가스를 소비하지 않습니다.
잘못 입금된 토큰을 꺼내야 하는 이유는 DS Token이 규제형 토큰이기 때문이며, 오류 발생 시 수정 메커니즘이 필요하기 때문입니다. 오류가 발생하면 1:1 비율 일관성을 유지해야 합니다. 만약 szToken이 잠기면 DS Token도 잠깁니다. Compound의 sweep 메서드는 그대로는 적합하지 않으며, 사실상 `transferImpairedXbalance` 메서드와도 매우 유사합니다.
현재 모델에서는 별도의 owner-admin이 존재하고, owner-admin이 필요 시 오류를 수정할 수 있습니다.
```solidity
    /**
     * @dev Transfers the impaired asset balance to a specified address.
     * This method is used when the asset balance of the vault is considered impaired,
     * meaning it does not reflect the expected balance due to incorrect transfers into the vault.
     * Only callable by an account with the DEFAULT_ADMIN_ROLE.
     *
     * Requirements:
     * - The `assetIsImpaired()` function must return a value greater than 0.
     *
     * @param _to The address to which the impaired asset balance will be transferred.
     */
    function transferImpairedAssetBalance(address _to) external addressNotZero(_to) onlyRole(DEFAULT_ADMIN_ROLE) {
        uint256 _assetImpairedBalance = assetImpairedBalance();
        require(_assetImpairedBalance>0, "Asset balance is not impaired");
        IERC20 assetToken = IERC20(asset());
        assetToken.safeTransfer(_to, _assetImpairedBalance);
    }

    /**
     * @dev Transfers the impaired vault balance to a specified address.
     * This method is used when the vault's balance is considered impaired,
     * meaning it does not reflect the expected balance due to incorrect transfers out of the vault.
     * Only callable by an account with the DEFAULT_ADMIN_ROLE.
     *
     * Requirements:
     * - The `vaultImpairedBalance()` function must be return a value greater than 0.
     *
     * @param _to The address to which the impaired vault balance will be transferred.
     */
    function transferImpairedVaultBalance(address _to) external addressNotZero(_to) onlyRole(DEFAULT_ADMIN_ROLE) {
        uint _vaultImpairedBalance = vaultImpairedBalance();
        require(_vaultImpairedBalance>0, "Vault balance is not impaired");
        IERC20(address(this)).safeTransfer(_to, _vaultImpairedBalance);
    }
```

**Cyfrin:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 불필요한 수정자

**설명:** `redeem()` 함수에는 `receiverSenderNotEqual(_owner)` 수정자가 붙어 있어 호출자가 실제 공유 소유자인지 확인합니다.
하지만 이 검사는 `ERC4626::_withdraw()` 함수에서 이미 수행되므로 여기서는 불필요합니다.
```solidity
openzeppelin-contracts-upgradeable\contracts\token\ERC20\extensions\ERC4626Upgradeable.sol
291:         ERC4626Storage storage $ = _getERC4626Storage();
292:         if (caller != owner) {
293:             _spendAllowance(owner, caller, shares);
294:         }
```

**Securitize:** [52350aa](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/52350aa809c140edc6f794baabc7e53891b37852)에서 수정자를 삭제함.

**Cyfrin:** 확인함.

\clearpage
