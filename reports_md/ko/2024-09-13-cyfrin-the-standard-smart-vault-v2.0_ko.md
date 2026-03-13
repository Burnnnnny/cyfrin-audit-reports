**수석 감사자**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Immeas](https://twitter.com/0ximmeas)

**보조 감사자**



---

# 발견 사항 (Findings)
## 중대 위험 (Critical Risk)


### 감마 볼트(Gamma Vaults)에 예치된 담보가 청산 중에 고려되지 않아 USDs 안정성이 손상될 수 있음

**설명:** The Standard의 사용자는 `SmartVaultV4` 인스턴스에 예치된 담보에 대해 `USDs` 스테이블코인 대출을 받을 수 있습니다. 스마트 볼트의 담보 가치가 `USDs` 부채 가치의 110% 미만으로 떨어지면 전액 청산될 수 있습니다. 사용자는 또한 Uniswap V3에서 LP 포지션을 보유하는 감마 볼트(일명 Hypervisors)로 담보 토큰을 이동하여 예치된 담보에 대해 추가 수익을 얻을 수 있습니다.

감마 볼트에서 수익 포지션으로 보유된 담보는 `SmartVaultV4` 계약으로 전송되어 보유되는 Hypervisor 토큰으로 표시됩니다. 그러나 이 토큰들은 청산에 영향을 받지 않습니다:

```solidity
function liquidate() external onlyVaultManager {
    if (!undercollateralised()) revert NotUndercollateralised();
    liquidated = true;
    minted = 0;
    liquidateNative();
    ITokenManager.Token[] memory tokens = ITokenManager(ISmartVaultManagerV3(manager).tokenManager()).getAcceptedTokens();
    for (uint256 i = 0; i < tokens.length; i++) {
        if (tokens[i].symbol != NATIVE) liquidateERC20(IERC20(tokens[i].addr));
    }
}
```

현재 `SmartVaultV4::hypervisors` 배열에 존재하는 Hypervisor 토큰은 체인링크 데이터 피드가 필요하므로 `TokenManager::getAcceptedTokens`에 의해 반환되는 배열에 포함되지 않습니다. 따라서 감마 볼트 내에서 수익 포지션으로 예치된 담보는 영향을 받지 않습니다.

사용자는 합리적으로 담보의 100%를 감마에 예치하고 최대 USDs의 100%를 발행한 스마트 볼트를 가질 수 있습니다. 이 시점에서 작은 시장 변동만 있어도 스마트 볼트는 담보 부족 상태가 되어 청산에 취약해집니다. 청산 성공 시 `minted` 상태 변수가 0으로 재설정된다는 점을 감안할 때, [`SmartVaultV4::canRemoveCollateral`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L171)의 유효성 검사로 인해 사용자는 [`SmartVaultV4::removeCollateral`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L186)을 통해 이 담보에 다시 접근할 수 있게 됩니다:

```solidity
function canRemoveCollateral(ITokenManager.Token memory _token, uint256 _amount) private view returns (bool) {
    if (minted == 0) return true;
    /* snip: collateral calculations */
}
```

결과적으로 사용자는 원래 `USDs` 대출을 상환하지 않고 담보를 인출할 수 있습니다. `liquidated` 상태 변수가 이제 `true`로 설정되므로 공격자는 담보를 다시 사용하기 전에 새 스마트 볼트를 생성해야 합니다.

**파급력:** 공격자는 청산된 후 수익 포지션에 예치된 담보에 대해 반복적으로 차입할 수 있으며, 이는 프로토콜에 악성 부채를 초래하고 대규모로 실행될 경우 `USDs`의 안정성을 손상시킬 가능성이 높습니다.

**개념 증명 (PoC):** 다음 테스트를 `SmartVault.js`에 추가할 수 있습니다:
```javascript
it('cant liquidate yield positions', async () => {
  const ethCollateral = ethers.utils.parseEther('0.1')
  await user.sendTransaction({ to: Vault.address, value: ethCollateral });

  let { collateral, totalCollateralValue } = await Vault.status();
  let preYieldCollateral = totalCollateralValue;
  expect(getCollateralOf('ETH', collateral).amount).to.equal(ethCollateral);

  depositYield = Vault.connect(user).depositYield(ETH, HUNDRED_PC.div(10));
  await expect(depositYield).not.to.be.reverted;
  await expect(depositYield).to.emit(YieldManager, 'Deposit').withArgs(Vault.address, MockWeth.address, ethCollateral, HUNDRED_PC.div(10));

  ({ collateral, totalCollateralValue } = await Vault.status());
  expect(getCollateralOf('ETH', collateral).amount).to.equal(0);
  expect(totalCollateralValue).to.equal(preYieldCollateral);

  const mintedValue = ethers.utils.parseEther('100');
  await Vault.connect(user).mint(user.address, mintedValue);

  await expect(VaultManager.connect(protocol).liquidateVault(1)).to.be.revertedWith('vault-not-undercollateralised')

  // drop price, now vault is liquidatable
  await CL_WBTC_USD.setPrice(1000);

  await expect(VaultManager.connect(protocol).liquidateVault(1)).not.to.be.reverted;
  ({ minted, maxMintable, totalCollateralValue, collateral, liquidated } = await Vault.status());

  // hypervisor tokens (yield position) not liquidated
  await expect(MockWETHWBTCHypervisor.balanceOf(Vault.address)).to.not.equal(0);

  // since minted is zero, the vault owner still has access to all collateral
  expect(minted).to.equal(0);
  expect(maxMintable).to.not.equal(0);
  expect(totalCollateralValue).to.not.equal(0);
  collateral.forEach(asset => expect(asset.amount).to.equal(0));
  expect(liquidated).to.equal(true);

  // price returns
  await CL_WBTC_USD.setPrice(DEFAULT_ETH_USD_PRICE.mul(20));

  // user exits yield position
  await Vault.connect(user).withdrawYield(MockWETHWBTCHypervisor.address, ETH);
  await Vault.connect(user).withdrawYield(MockUSDsHypervisor.address, ETH);

  // and withdraws assets
  const userBefore = await ethers.provider.getBalance(user.address);
  await Vault.connect(user).removeCollateralNative(await ethers.provider.getBalance(Vault.address), user.address);
  const userAfter = await ethers.provider.getBalance(user.address);

  // user should have all collateral back minus protocol fee from yield withdrawal
  expect(userAfter.sub(userBefore)).to.be.closeTo(ethCollateral, ethers.utils.parseEther('0.01'));

  // and user also has the minted USDs
  const usds = await USDs.balanceOf(user.address);
  expect(usds).to.equal(mintedValue);
});
```

**권장 완화 조치:** 수익 포지션에 보유된 담보도 청산 대상이 되도록 하십시오.

**The Standard DAO:** 커밋 [`c6af5d2`](https://github.com/the-standard/smart-vault/commit/c6af5d21fc20244531c0202b70eb9392a6ea9b6a)로 수정되었습니다.

**Cyfrin:** 확인됨, `SmartVault4::liquidate`는 이제 `SmartVaultV4::hypervisors` 배열도 순회합니다. 그러나 재진입(re-entrancy) 위험을 완화하기 위해 네이티브 청산은 마지막에 수행되어야 합니다. 마찬가지로 `SmartVaultManagerV6::liquidateVault`에서의 역할 취소는 `SmartVaultV4::liquidate`를 호출하기 전에 발생해야 합니다.

**The Standard DAO:** 커밋 [`23c573c`](https://github.com/the-standard/smart-vault/commit/23c573cba241b0dc6af276f56ee8772efc8b4a5c)로 수정되었습니다.

**Cyfrin:** 확인됨, 호출 순서가 수정되었습니다.


### USDs 부채를 상환하지 않고 볼트에서 직접 Hypervisor 토큰을 제거하여 담보를 훔칠 수 있어 USDs 안정성이 손상될 수 있음

**설명:** 스마트 볼트의 소유자가 [`SmartVaultV4::depositYield`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L167-L180)를 호출할 때, [`SmartVaultYieldManager::deposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L299-L310)이 호출되어 지정된 담보 토큰을 [`IUniProxy`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/interfaces/IUniProxy.sol#L6) 계약을 통해 주어진 감마 볼트(일명 Hypervisor)에 예치합니다. 감마 Hypervisor는 Uniswap V3 풀의 포지션을 보유하고 이 포지션의 지분을 소유한 예금자에게 대체 가능(fungible)하게 만드는 방식으로 작동하며, 이는 ERC-20 토큰으로 표시됩니다. 예치가 완료되면 Hypervisor 토큰은 호출한 `SmartVaultV4` 계약으로 다시 전송되어 발행된 `USDs` 부채에 대한 담보로 남습니다.

그러나 불충분한 입력 유효성 검사로 인해 [`SmartVaultV4::removeAsset`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L191-L196)을 호출하여 이러한 Hypervisor 담보 토큰을 스마트 볼트에서 제거할 수 있습니다:

```solidity
function removeAsset(address _tokenAddr, uint256 _amount, address _to) external onlyOwner {
    ITokenManager.Token memory token = getTokenManager().getTokenIfExists(_tokenAddr);
    if (token.addr == _tokenAddr && !canRemoveCollateral(token, _amount)) revert Undercollateralised();
    IERC20(_tokenAddr).safeTransfer(_to, _amount);
    emit AssetRemoved(_tokenAddr, _amount, _to);
}
```

Hypervisor 토큰은 [`SmartVaultV4::hypervisors`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L26) 배열에만 존재하며 `TokenManager` 계약에 의해 처리되지 않으므로 `token.addr`은 `address(0)`이 되고 담보 확인이 우회됩니다. 따라서 이러한 토큰은 계약에서 추출될 수 있으며, 스마트 볼트는 담보 부족 상태가 되고 프로토콜에는 악성 부채가 남게 됩니다.

**파급력:** 공격자는 예치된 수익 담보에 대해 최대 발행 가능한 `USDs` 금액을 차입한 다음, 단순히 담보 Hypervisor 토큰을 제거하여 스마트 볼트를 담보 부족 상태로 만들 수 있습니다. 공격자는 `USDs`와 그 담보를 모두 받고 프로토콜은 악성 부채를 떠안게 되며, 이는 대규모로 실행되거나 플래시 론에서 얻은 자금으로 원자적으로 실행될 경우 `USDs`의 안정성을 손상시킬 가능성이 높습니다.

**개념 증명 (PoC):** 다음 테스트를 `SmartVault.js`에 추가할 수 있습니다:

```javascript
it('can steal collateral hypervisor tokens', async () => {
  const ethCollateral = ethers.utils.parseEther('0.1')
  await user.sendTransaction({ to: Vault.address, value: ethCollateral });

  let { collateral, totalCollateralValue } = await Vault.status();
  let preYieldCollateral = totalCollateralValue;
  expect(getCollateralOf('ETH', collateral).amount).to.equal(ethCollateral);

  depositYield = Vault.connect(user).depositYield(ETH, HUNDRED_PC);
  await expect(depositYield).not.to.be.reverted;
  await expect(depositYield).to.emit(YieldManager, 'Deposit').withArgs(Vault.address, MockWeth.address, ethCollateral, HUNDRED_PC);

  ({ collateral, totalCollateralValue } = await Vault.status());
  expect(getCollateralOf('ETH', collateral).amount).to.equal(0);
  expect(totalCollateralValue).to.equal(preYieldCollateral);

  const mintedValue = ethers.utils.parseEther('100');
  await Vault.connect(user).mint(user.address, mintedValue);

  // Vault is fully collateralised after minting USDs
  expect(await Vault.undercollateralised()).to.be.equal(false);

  const hypervisorBalanceVault = await MockUSDsHypervisor.balanceOf(Vault.address);
  await Vault.connect(user).removeAsset(MockUSDsHypervisor.address, hypervisorBalanceVault , user.address);

  // Vault has no collateral left and as such is undercollateralised
  expect(await MockUSDsHypervisor.balanceOf(Vault.address)).to.be.equal(0);
  expect(await Vault.undercollateralised()).to.be.equal(true);

  // User has both the minted USDs and Hypervisor collateral tokens
  expect(await MockUSDsHypervisor.balanceOf(user.address)).to.be.equal(hypervisorBalanceVault);
  expect(await USDs.balanceOf(user.address)).to.be.equal(mintedValue);
});
```

**권장 완화 조치:** 제거된 자산이 `hypervisors` 배열에 존재하는 Hypervisor 토큰이 아님을 검증하십시오.

`TokenManager`에 Hypervisor 토큰을 담보로 추가하는 것을 고려하는 경우, `SmartVaultV4::usdCollateral` 내의 [이 루프](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L108-L111)에서 제외되고 `SmartVaultV4::getAssets`의 [가격 계산](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L131)도 그에 따라 업데이트되는지 확인하십시오.

**The Standard DAO:** 커밋 [`5862d8e`](https://github.com/the-standard/smart-vault/commit/5862d8e10ac8648b89a7e3a78498ff20dc31e42e)로 수정되었습니다.

**Cyfrin:** 확인됨, Hypervisor 토큰은 더 이상 담보 부족으로 인해 `SmartVault::removeAsset`이 revert 되지 않고 제거될 수 없습니다. 그러나 `SmartVaultV4::removeCollateralNative`에서 `remainCollateralised()` 수정자를 사용하면 재진입 취약점이 발생하여 스마트 볼트 소유자가 프로토콜 소각 수수료를 우회할 수 있습니다: 네이티브 담보 예치 -> USDs 발행 -> 네이티브 담보 제거 -> 재진입 및 자가 청산. Hypervisor 토큰에는 영향을 미치지 않으므로 여기서는 원래의 유효성 검사를 사용해야 합니다.

**The Standard DAO:** 커밋 [`d761d48`](https://github.com/the-standard/smart-vault/commit/d761d48e957d45c5d61eb494d41b7362f7001155)로 수정되었습니다.

**Cyfrin:** 확인됨, `SmartVaultV4::removeCollateralNative`는 더 이상 담보 부족 상태에서 재진입하는 데 사용될 수 없습니다.

\clearpage
## 고위험 (High Risk)


### `USDs` 자체 담보(self-backing)는 경제적 페그 유지 인센티브에 대한 가정을 깨뜨림

**설명:** `SmartVaultV4::depositYield`를 통해 `SmartVaultYieldManager::deposit`이 호출될 때, 예치된 담보의 [최소 10%](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L168)는 `USDs` Hypervisor(프로토콜 관리 `USDs/USDC` Ramses 풀에서 LP 포지션을 보유함)로 향해야 합니다:

```solidity
function deposit(address _collateralToken, uint256 _usdPercentage) external returns (address _hypervisor0, address _hypervisor1) {
    if (_usdPercentage < MIN_USDS_PERCENTAGE) revert StablePoolPercentageError();
    uint256 _balance = IERC20(_collateralToken).balanceOf(address(msg.sender));
    IERC20(_collateralToken).safeTransferFrom(msg.sender, address(this), _balance);
    HypervisorData memory _hypervisorData = hypervisorData[_collateralToken];
    if (_hypervisorData.hypervisor == address(0)) revert HypervisorDataError();
    _usdDeposit(_collateralToken, _usdPercentage, _hypervisorData.pathToUSDC);
    /* snip: other hypervisor deposit */
}
```

스마트 볼트의 담보 가치가 `SmartVaultV4::yieldVaultCollateral`에 의해 결정될 때, 스테이블코인을 $1로 하드코딩하는 문제를 무시하고 각 Hypervisor의 기본 토큰 가치가 사용됩니다:

```solidity
if (_token0 == address(USDs) || _token1 == address(USDs)) {
    // both USDs and its vault pair are € stablecoins, but can be equivalent to €1 in collateral
    _usds += _underlying0 * 10 ** (18 - ERC20(_token0).decimals());
    _usds += _underlying1 * 10 ** (18 - ERC20(_token1).decimals());
```

`USDs` Hypervisor 예치에 대한 문제는 `USDs`의 이 [기본 잔액](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L90-L93)이 스마트 볼트의 [총 담보 가치](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L113)에 포함되며, 이 Hypervisor로 향할 수 있는 담보의 양에 제한이 없다는 것입니다. 따라서 사용자는 `USDs/USDC` 풀에 예치된 총 담보의 최대 100%까지 사용하여 `USDs` 대출을 담보할 수 있습니다 (두 스테이블코인 토큰이 모두 페그 상태라고 가정할 때 `USDs`에서 50%).

**파급력:** 자체적으로 담보되는 것과 같이 내부적으로 담보되는 스테이블코인의 경우 페그가 손실되면 스테이블코인과 담보의 가치가 동시에 감소하므로 복구하기가 점점 어려워집니다. 이 자체 담보는 또한 페그 유지에 기여하기 위한 프로토콜의 경제적 인센티브를 둘러싼 가정을 깨뜨립니다.

**권장 완화 조치:** `USDs` Hypervisor 토큰을 담보로 사용하는 것을 금지하고 풀에 충분한 유동성을 보장하기 위해 다른 메커니즘을 구현하는 것을 고려하십시오. 대안으로 `USDs` Hypervisor로 향할 수 있는 담보의 비율을 제한할 수 있지만 이는 위험을 완전히 완화하지는 못합니다.

**The Standard DAO:** 커밋 [`cc86606`](https://github.com/the-standard/smart-vault/commit/cc86606ef6f8c1fea84f378e7f324e648f9bcbc8)으로 수정되었습니다.

**Cyfrin:** 확인됨, `USDs`는 더 이상 스마트 볼트 수익 담보에 기여하지 않습니다.


### USD 스테이블코인이 항상 페그 상태라고 잘못 가정됨

**설명:** [`SmartVaultV4::yieldVaultCollateral`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L79-L103)은 주어진 스마트 볼트에 대한 수익 포지션에 보유된 담보의 가치를 반환합니다. `USDs` Hypervisor 이외의 모든 감마 볼트의 경우, 담보가 예치된 각 감마 볼트에 대해 가져온 담보 토큰의 기본 금액의 달러 가치는 Chainlink에서 보고한 가격을 사용하여 계산됩니다.

```solidity
function yieldVaultCollateral(ITokenManager.Token[] memory _acceptedTokens) private view returns (uint256 _usds) {
    for (uint256 i = 0; i < hypervisors.length; i++) {
        IHypervisor _Hypervisor = IHypervisor(hypervisors[i]);
        uint256 _balance = _Hypervisor.balanceOf(address(this));
        if (_balance > 0) {
            uint256 _totalSupply = _Hypervisor.totalSupply();
            (uint256 _underlyingTotal0, uint256 _underlyingTotal1) = _Hypervisor.getTotalAmounts();
            address _token0 = _Hypervisor.token0();
            address _token1 = _Hypervisor.token1();
            uint256 _underlying0 = _balance * _underlyingTotal0 / _totalSupply;
            uint256 _underlying1 = _balance * _underlyingTotal1 / _totalSupply;
            if (_token0 == address(USDs) || _token1 == address(USDs)) {
                // both USDs and its vault pair are € stablecoins, but can be equivalent to €1 in collateral
                _usds += _underlying0 * 10 ** (18 - ERC20(_token0).decimals());
                _usds += _underlying1 * 10 ** (18 - ERC20(_token1).decimals());
            } else {
                for (uint256 j = 0; j < _acceptedTokens.length; j++) {
                    ITokenManager.Token memory _token = _acceptedTokens[j];
                    if (_token.addr == _token0) _usds += calculator.tokenToUSD(_token, _underlying0);
                    if (_token.addr == _token1) _usds += calculator.tokenToUSD(_token, _underlying1);
                }
            }
        }
    }
}
```

감마 볼트에 담보를 예치할 때마다 최소 금액이 `USDs/USDC` 쌍으로 향해야 합니다. 따라서 감마 볼트에 대해 다른 Hypervisor 토큰의 잔액이 0이 아닌 경우 `USDs` Hypervisor 토큰의 잔액은 항상 0이 아닙니다.

따라서 `USDs`에 대한 Chainlink 가격 피드가 없기 때문에 이 Hypervisor는 별도로 처리됩니다. 그러나 이 로직은 `USDC`와 `USDs`의 가격이 항상 $1로 동일할 것이라고 잘못 가정합니다. 이는 항상 사실이 아닙니다. USDC가 디페깅(de-pegging) 이벤트를 겪은 사례가 있었으며 규모와 기간 면에서 상당했습니다. 별도의 발견 사항에서 제기된 자체 담보 관련 문제를 무시하더라도 `USDs`에 대해 유사한 우려가 존재합니다.

두 스테이블코인 중 하나가 디페깅되는 경우, 스마트 볼트 소유자는 실제 담보 가치 이상으로 차입할 수 있습니다. 엄밀히 말하면 가설적이지만, 다음 작업을 통해 `USDs/USDC` 풀을 직접 조작하여 이 시나리오를 초래할 수도 있습니다:
- `WETH` 담보를 플래시 론으로 빌려 스마트 볼트에 예치.
- 담보의 100%를 `USDs` Hypervisor에 예치.
- 다량의 `USDs` 발행.
- `USDs/USDC` 풀에 매도.
- `USDs` Hypervisor의 기본 포지션 중 더 많은 부분이 `USDs`가 되도록 `USDs`가 디페깅된다고 가정하면, 이 "담보 가치 이상 차입" 효과가 증폭되고 스마트 볼트 수익 담보가 증가합니다.
- 부풀려진 수익 볼트 담보 계산을 사용하여 더 많은 `USDs`를 빌리고 대출을 갚고 반복합니다.

**파급력:** `USDC` 또는 `USDs`가 $1 페그 아래로 떨어지면 사용자는 `USDs/USDC` Hypervisor 예치로 담보된 `USDs`를 가능한 것보다 더 많이 발행할 수 있습니다. 또한 하나 또는 두 스테이블코인이 디페깅될 때 `USDs/USDC` 풀의 조건에 따라 `USDs`의 안정성에 직접적인 영향을 미칠 수도 있습니다.

또한 Hypervisor 토큰이 청산의 영향을 받지 않는다는 별도의 발견 사항을 무시하면, `USDs` Hypervisor로 완전히 향한 담보 예치는 하나 또는 두 스테이블코인이 디페깅되더라도 절대 청산될 수 없어 스마트 볼트가 실제로는 담보 부족 상태가 됩니다.

**권장 완화 조치:** `USDC`의 가격을 결정하기 위해 Chainlink 데이터 피드를 사용해야 합니다.

일반적인 권장 사항으로 `USDs` 가격 책정을 위해 조작 방지 대안을 활용해야 합니다. 그러나 이 발견 사항은 이 시나리오에서 프로토콜의 의도된 경제적 인센티브가 적용되지 않으므로 USDs 자체 담보 문제를 강조합니다.

**The Standard DAO:** 커밋 [`cc86606`](https://github.com/the-standard/smart-vault/commit/cc86606ef6f8c1fea84f378e7f324e648f9bcbc8)으로 수정되었습니다.

**Cyfrin:** 확인됨, `USDC` 가격은 이제 Chainlink에서 얻으며 `USDs`는 더 이상 수익 볼트 담보에 포함되지 않습니다. Chainlink 데이터 피드 소수점 자릿수를 `1e8`로 하드코딩하는 대신 쿼리하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`5febbc4`](https://github.com/the-standard/smart-vault/commit/5febbc4768602433db029d85aee29a4e4b1aa5f3)로 수정되었습니다.

**Cyfrin:** 확인됨, 소수점 자릿수는 이제 동적으로 쿼리됩니다.

\clearpage
## 중급 위험 (Medium Risk)


### 수익 예치금은 최대 10%의 손실을 입을 수 있음

**설명:** [`SmartVaultV4::depositYield`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L299-L310) 및 [`SmartVaultV4::withdrawYield`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L312-L322)가 호출될 때 여러 [[1](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L99), [2](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L127), [3](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L141), [4](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L151), [5](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L191), [6](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L197), [7](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L205)] 중간 DEX 스왑 및 감마 볼트 상호 작용을 통해 발생하는 슬리피지를 처리하기 위해 총 담보 가치가 90% 이상 감소하지 않아야 한다는 요구 사항이 있습니다:

```solidity
    function significantCollateralDrop(uint256 _preCollateralValue, uint256 _postCollateralValue) private pure returns (bool) {
    return _postCollateralValue < 9 * _preCollateralValue / 10;
}
```

이 설계는 완전하고 즉각적인 손실로부터 사용자를 성공적으로 보호하지만, 10%는 각 예치/인출 작업에서 잃기에 상당한 금액입니다.

현재 중앙 집중식 시퀀서의 존재로 인해 Arbitrum에는 일반적인 의미의 MEV가 존재하지 않습니다. 그러나 청산과 같이 예측 가능한 이벤트에 대해 대기 시간 중심 전략을 실행하는 것은 여전히 가능합니다. 따라서 MEV 봇이 담보 수익 예치/인출이 원래 담보 가치의 90%를 반환하게 하여 스마트 볼트를 불필요하게 청산에 가깝게 만드는 것이 여전히 가능할 수 있습니다.

**파급력:** 사용자는 감마 볼트에 예치하고 인출할 때 담보의 상당 부분을 잃을 수 있습니다.

**권장 완화 조치:** 기존 유효성 검사는 유지할 수 있지만, 사용자가 위에서 링크한 상호 작용에 대해 더 제한적인 담보 하락 비율과 더 세분화된 슬리피지 매개변수를 전달할 수 있도록 허용하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`cc86606`](https://github.com/the-standard/smart-vault/commit/cc86606ef6f8c1fea84f378e7f324e648f9bcbc8)으로 수정되었습니다.

**Cyfrin:** 확인됨, `SmartVaultV4::depositYield` 및 `SmartVaultV4::withdrawYield`는 이제 사용자가 제공한 최소 담보 비율 매개변수를 허용합니다. `SmartVaultV4::mint`의 유효성 검사 순서 변경으로 인해 이 함수에 `remainCollateralised()` 수정자를 사용할 수 있습니다.

**The Standard DAO:** 커밋 [e89daee](https://github.com/the-standard/smart-vault/commit/e89daee950d76180a969c6f93839addd8d43b195)로 수정되었습니다.

**Cyfrin:** 확인됨, 수정자가 이제 `mint()`에 사용됩니다.


### 하드코딩된 풀 수수료로 인해 슬리피지 증가 및 스왑 실패 발생 가능

**설명:** 이전 CodeHawks 콘테스트에서 보고 항목 [M-03](https://codehawks.the-standard.io/c/2023-12-the-standard/s/483)으로 제기된 문제는 `SmartVaultV4::swap`에 여전히 존재하며 여기서 풀 수수료는 `3000`으로 [하드코딩](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L267)되어 있습니다:

```solidity
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
        tokenIn: inToken,
        tokenOut: getTokenisedAddr(_outToken),
        fee: 3000, // @audit hardcoded pool fee
        recipient: address(this),
        deadline: block.timestamp + 60,
        amountIn: _amount - swapFee,
        amountOutMinimum: minimumAmountOut,
        sqrtPriceLimitX96: 0
    });
```

동일한 문제가 [`SmartVaultYieldManager::_usdDeposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L158) 및 [`SmartVaultYieldManager::_withdrawDeposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L198)에도 존재하며, 여기서 담보 토큰은 `500`의 하드코딩된 풀 수수료로 `USDC` 및 `USDs`와 스왑됩니다:

```solidity
function _usdDeposit(address _collateralToken, uint256 _usdPercentage, bytes memory _pathToUSDC) private {
    _swapToUSDC(_collateralToken, _usdPercentage, _pathToUSDC);
    _swapToRatio(USDC, usdsHypervisor, ramsesRouter, 500);
    _deposit(usdsHypervisor);
}
...
function _withdrawUSDsDeposit(address _hypervisor, address _token) private {
    IHypervisor(_hypervisor).withdraw(_thisBalanceOf(_hypervisor), address(this), address(this), [uint256(0),uint256(0),uint256(0),uint256(0)]);
    _swapToSingleAsset(usdsHypervisor, USDC, ramsesRouter, 500);
    _sellUSDC(_token);
}
```

**파급력:** CodeHawks 콘테스트의 [M-03](https://codehawks.the-standard.io/c/2023-12-the-standard/s/483)에서 언급했듯이, 프로토콜에 의해 생성 및 유지 관리되는 `USDs/USDC` 풀을 제외하고 유동성이 가장 높은 풀이 항상 하드코딩된 값과 같지는 않으므로 유동성이 낮은 풀에서 거래하면 슬리피지가 증가하거나 스왑이 실패할 수 있습니다. 손실이 담보 가치의 10%를 초과하면 [`SmartVaultV4::significantCollateralDrop`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L295-L297)의 유효성 검사로 인해 수익 예치/인출의 서비스 거부(DoS)가 발생합니다. `SmartVaultV4::swap` 호출의 경우 스마트 볼트가 불필요하게 청산에 가깝게 되는 것을 방지하기 위한 유효성 검사가 없습니다. 스왑에서 출력되는 최소 금액은 청산의 1% 이내로 담보 상태를 유지하는 데 필요한 금액입니다.

**권장 완화 조치:** CodeHawks 콘테스트의 [M-03](https://codehawks.the-standard.io/c/2023-12-the-standard/s/483)과 동일한 권장 사항이 여기에 적용됩니다. 사용자가 풀 수수료를 매개변수로 전달할 수 있도록 허용하는 것을 고려하십시오.

**The Standard DAO:** 담보 스왑 풀 수수료는 커밋 [`f9f7093`](https://github.com/the-standard/smart-vault/commit/f9f70930168499f2de6b7aadf49995b7a766f1a1)으로 수정되었습니다. Hypervisor 스왑 풀 수수료는 인지됨 – 이 스왑 경로는 `hypervisorData`의 관리자에 의해 관리되므로 수정되지 않았습니다.

**Cyfrin:** 확인됨, `SmartVaultV4::swap`은 이제 사용자가 제공한 풀 수수료 매개변수를 허용합니다.


### Chainlink 데이터 피드의 불충분한 유효성 검사

**설명:** `PriceCalculator`는 The Standard에서 사용하는 자산에 대한 Chainlink 오라클 가격을 제공하는 계약입니다. 여기서 자산 가격을 쿼리한 다음 호출자에게 반환하기 전에 18 소수점으로 정규화합니다:

```solidity
function tokenToUSD(ITokenManager.Token memory _token, uint256 _tokenValue) external view returns (uint256) {
    Chainlink.AggregatorV3Interface tokenUsdClFeed = Chainlink.AggregatorV3Interface(_token.clAddr);
    uint256 scaledCollateral = _tokenValue * 10 ** getTokenScaleDiff(_token.symbol, _token.addr);
    (,int256 _tokenUsdPrice,,,) = tokenUsdClFeed.latestRoundData();
    return scaledCollateral * uint256(_tokenUsdPrice) / 10 ** _token.clDec;
}

function USDToToken(ITokenManager.Token memory _token, uint256 _usdValue) external view returns (uint256) {
    Chainlink.AggregatorV3Interface tokenUsdClFeed = Chainlink.AggregatorV3Interface(_token.clAddr);
    (, int256 tokenUsdPrice,,,) = tokenUsdClFeed.latestRoundData();
    return _usdValue * 10 ** _token.clDec / uint256(tokenUsdPrice) / 10 ** getTokenScaleDiff(_token.symbol, _token.addr);
}
```

그러나 `AggregatorV3Interface::latestRoundData`에 대한 이러한 호출에는 프로토콜이 잘못된 피드를 나타낼 수 있는 오래되거나 잘못된 가격 데이터를 수집하지 않도록 하는 Chainlink 데이터 피드에 필요한 유효성 검사가 없습니다.

**파급력:** 오래된 가격은 불필요한 청산이나 불충분하게 담보된 포지션 생성으로 이어질 수 있습니다.

**권장 완화 조치:** 다음 유효성 검사를 구현하십시오:

```diff
-   (,int256 _tokenUsdPrice,,,) = tokenUsdClFeed.latestRoundData();
+   (uint80 _roundId, int256 _tokenUsdPrice, , uint256 _updatedAt, ) = tokenUsdClFeed.latestRoundData();
+   if(_roundId == 0) revert InvalidRoundId();
+   if(_tokenUsdPrice == 0) revert InvalidPrice();
+   if(_updatedAt == 0 || _updatedAt > block.timestamp) revert InvalidUpdate();
+   if(block.timestamp - _updatedAt > TIMEOUT) revert StalePrice();
```

이 계약을 Arbitrum에 배포하려는 의도를 감안할 때 시퀀서 가동 시간을 확인하는 것도 권장됩니다. 이를 구현하기 위한 문서는 [여기](https://docs.chain.link/data-feeds/l2-sequencer-feeds)에 있으며 [코드 예제](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code)가 있습니다.

**The Standard DAO:** 커밋 [`8e78f7c`](https://github.com/the-standard/smart-vault/commit/8e78f7c55cc321e789da3d9f6b818ea740b55dc8)로 수정되었습니다.

**Cyfrin:** 확인됨, Chainlink 가격 피드 데이터의 추가 유효성 검사가 추가되었습니다. 그러나 타임아웃은 피드별로 지정해야 하며 24시간은 대부분의 피드에 너무 깁니다. 시퀀서 가동 시간 피드도 구현되지 않았지만 이는 중요한 추가 사항입니다. `hardhat/console.sol` import는 `PriceCalculator.sol`에서 제거되어야 합니다.

**The Standard DAO:** 커밋 [`7dfbff1`](https://github.com/the-standard/smart-vault/commit/7dfbff1c6a36b184f71eebcf0763131e53dccfc9)로 수정되었습니다.

**Cyfrin:** 확인됨, 추가 타임아웃 로직과 시퀀서 가동 시간 확인이 추가되었습니다.

\clearpage
## 저위험 (Low Risk)


### `SmartVaultYieldManager::_sellUSDC`에서 잘못된 토큰에 대한 승인 재설정

**설명:** `SmartVaultYieldManager::_sellUSDC`에서 `USDC`를 스왑할 때 라우터에 승인이 부여됩니다:

```solidity
IERC20(USDC).safeApprove(uniswapRouter, _balance);
ISwapRouter(uniswapRouter).exactInput(ISwapRouter.ExactInputParams({
    /* snip: swap */
}));
IERC20(USDs).safeApprove(uniswapRouter, 0);
```
이 계약에서 수행되는 다른 모든 스왑과 마찬가지로 승인은 라우터와의 상호 작용 후 재설정됩니다. 그러나 이 경우 `USDC` 대신 `USDs`에 대해 승인이 `0`으로 [잘못 재설정](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L193)됩니다.

**파급력:** 라우터에 작은 `USDC` 잔여 승인(dust allowance)이 남을 수 있습니다.

**권장 완화 조치:** `USDs`를 `USDC`로 교체하십시오:

```diff
  IERC20(USDC).safeApprove(uniswapRouter, _balance);
  ISwapRouter(uniswapRouter).exactInput(ISwapRouter.ExactInputParams({
      /* snip: swap */
  }));
- IERC20(USDs).safeApprove(uniswapRouter, 0);
+ IERC20(USDC).safeApprove(uniswapRouter, 0);
```

**The Standard DAO:** 커밋 [`217de3a`](https://github.com/the-standard/smart-vault/commit/217de3a777ec692e3ecc781464d8644814df3ab9)로 수정되었습니다.

**Cyfrin:** 확인됨, 승인이 이제 `USDC`에 대해 재설정됩니다.


### 수익 포지션에서 담보 추가/제거 시 불충분한 데드라인 보호

**설명:** 스마트 볼트 소유자가 담보 자산을 지원되는 감마 볼트로/로부터 전송할 때 `block.timestamp + 60`의 데드라인으로 여러 스왑이 실행됩니다. 예를 들어 `SmartVaultYieldManager::_sellUSDC`에서:

```solidity
ISwapRouter(uniswapRouter).exactInput(ISwapRouter.ExactInputParams({
    path: _pathFromUSDC,
    recipient: address(this),
    deadline: block.timestamp + 60,
    amountIn: _balance,
    amountOutMinimum: 0
}));
```

이 데드라인은 트랜잭션이 블록에 포함될 때마다 항상 유효하며, 실행 타임스탬프는 항상 `block.timestamp`이므로 현재 타임스탬프에서 60초를 추가하는 것은 아무런 효과가 없습니다.

**파급력:** 적절한 데드라인이 없으면 의도한 것과 상당히 다른 시장 상황에서 스왑이 실행되어 불리한 결과를 초래할 수 있습니다. 이는 `SmartVaultV4`의 `significantCollateralDrop()` 보호에 의해 어느 정도 완화되지만, 이는 트랜잭션이 제출된 이후 변경되었을 수도 있는 스마트 볼트 담보 계산을 위한 Chainlink 오라클 값에 의존합니다.

**권장 완화 조치:** 사용자가 수익 포지션에서 담보를 추가/제거할 때 실행되는 스왑에 대해 데드라인을 지정할 수 있도록 허용하는 것을 고려하십시오. 데드라인은 모든 스왑 호출에 직접 전달될 필요는 없지만 `SmartVaultV4::depositYield` 및 `SmartVaultV4::withdrawYield`의 함수 본문에서 직접 한 번 확인할 수 있습니다.

**The Standard DAO:** 커밋 [`71bad0a`](https://github.com/the-standard/smart-vault/commit/71bad0a8cc4bf8ff60321e41c9acb1e7d7fe1b2c)로 수정되었습니다.

**Cyfrin:** 확인됨, `SmartVaultV4::depositYield`, `SmartVaultV4:withdrawtYield`, 및 `SmartVaultV4::swap`은 이제 사용자가 제공한 데드라인 매개변수를 허용합니다.


### Hypervisor 데이터 제거 시 예치된 스마트 볼트 담보 잠김

**설명:** 감마 볼트(일명 Hypervisor)는 Uniswap V3 유동성 포지션의 대체 가능한 지분을 유지하고 제공하는 외부 계약입니다. The Standard는 `USDs`를 뒷받침하는 담보가 수익을 얻을 수 있도록 여러 Hypervisor를 활용하며, 이는 [`SmartVaultYieldManager::addHypervisorData`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L222-L224)에 대한 관리자 호출로 구성됩니다. 스마트 볼트 담보가 이러한 Hypervisor 중 하나에 예치되면 기본 포지션의 지분을 나타내는 Hypervisor ERC-20 토큰이 발행되고 내부적으로 [`SmartVaultV4::addUniqueHypervisor`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L279-L284)를 호출하여 담보가 예치된 Hypervisor 목록을 유지합니다.

스마트 볼트가 여전히 포지션을 보유하고 있는 Hypervisor를 제거하기 위해 [`SmartVaultYieldManager::removeHypervisorData`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L226-L228)에 대한 관리자 호출이 이루어지면 기본 담보가 잠기게 됩니다. 이는 [`SmartVaultYieldManager::_withdrawOtherDeposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L202-L207)에서 Hypervisor 데이터가 유효하고 구성되어 있어야 하는 다음 유효성 검사 때문입니다:

```solidity
function _withdrawOtherDeposit(address _hypervisor, address _token) private {
    HypervisorData memory _hypervisorData = hypervisorData[_token];
    if (_hypervisorData.hypervisor != _hypervisor) revert IncompatibleHypervisor();
    /* snip: withdraw and swap */
}
```

그러나 제거된 Hypervisor에 잠긴 이 담보는 독립적으로 유지되는 `SmartVaultV4::hypervisors` 배열(담보가 인출될 때만 Hypervisor가 제거됨)을 반복하기 때문에 여전히 스마트 볼트의 담보 계산에 기여합니다.

**파급력:** 스마트 볼트에서 Hypervisor 토큰을 악의적으로 제거하는 것에 대한 별도의 발견 사항을 무시하면, 프로토콜 관리자가 Hypervisor 데이터를 다시 추가하지 않는 한 Hypervisor 토큰과 해당 스마트 볼트 담보는 무기한 잠길 수 있습니다.

**개념 증명 (PoC):** 다음 테스트를 `SmartVault.js`에 추가할 수 있습니다:
```javascript
it('locks collateral when hypervisor is removed', async () => {
  const ethCollateral = ethers.utils.parseEther('0.1')
  await user.sendTransaction({ to: Vault.address, value: ethCollateral });

  let { collateral, totalCollateralValue } = await Vault.status();
  let preYieldCollateral = totalCollateralValue;
  expect(getCollateralOf('ETH', collateral).amount).to.equal(ethCollateral);

  depositYield = Vault.connect(user).depositYield(ETH, HUNDRED_PC.div(10));
  await expect(depositYield).not.to.be.reverted;
  await expect(depositYield).to.emit(YieldManager, 'Deposit').withArgs(Vault.address, MockWeth.address, ethCollateral, HUNDRED_PC.div(10));

  ({ collateral, totalCollateralValue } = await Vault.status());
  expect(getCollateralOf('ETH', collateral).amount).to.equal(0);
  expect(totalCollateralValue).to.equal(preYieldCollateral);

  await YieldManager.connect(admin).removeHypervisorData(MockWeth.address);

  // collateral is still counted
  ({ collateral, totalCollateralValue } = await Vault.status());
  expect(getCollateralOf('ETH', collateral).amount).to.equal(0);
  expect(totalCollateralValue).to.equal(preYieldCollateral);

  // user cannot remove collateral
  await expect(Vault.connect(user).withdrawYield(MockWETHWBTCHypervisor.address, ETH))
    .to.be.revertedWithCustomError(YieldManager, 'IncompatibleHypervisor');
});
```

**권장 완화 조치:** Hypervisor를 제거하는 기능이 필요한 경우, 스마트 볼트 소유자가 `SmartVaultYieldManager`에서 목록이 제거되었더라도 여전히 충분히 담보가 설정되어 있다면 자신의 볼트에서 Hypervisor 토큰을 제거할 수 있도록 허용하는 것을 고려하십시오.

**The Standard DAO:** 인지함. 사용자가 `removeAsset()`으로 제거할 수 있다고 믿기 때문에 수정하지 않음. 볼트가 담보 상태를 유지하는 한 문제가 없어야 합니다. 또한 가능하면 Hypervisor를 제거할 의도가 없습니다.

**Cyfrin:** 인지함, 제거된 Hypervisor 토큰은 주어진 스마트 볼트의 담보 가치에 계속 기여하지만, 볼트가 충분히 담보된 상태로 유지되는 한 `SmartVaultV4::removeAsset`을 호출하여 제거할 수 있습니다.


### 스왑된 담보 토큰의 잔여(dust) 금액이 `SmartVaultYieldManager`에 남음

**설명:** 반올림으로 인해 정확한 입력 매개변수를 사용하는 Uniswap 스타일 라우터를 통한 스왑은 호출 계약에 잔여(dust) 금액을 남길 수 있습니다. 이는 모든 스왑된 토큰이 해당 Hypervisor 계약으로 전송되므로 감마 볼트 예치에는 문제가 되지 않습니다. 그러나 인출 중에 [`SmartVaultYieldManager::_withdrawUSDsDeposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L196-L200) 및 [`SmartVaultYieldManager::_withdrawOtherDeposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L202-L207)에서 호출되는 [`SmartVaultYieldManager::_swapToSingleAsset`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L113-L131)을 사용하면 입력 토큰의 잔여 금액이 남을 수 있습니다.

**파급력:** 담보 토큰의 잔여 금액이 `SmartVaultYieldManager`에 축적될 수 있으며 해당 토큰에 대한 다음 호출자에 의해 활용됩니다.

**권장 완화 조치:** 수익 포지션 인출 중 이루어진 스왑에 대한 입력 토큰의 0이 아닌 잔여 금액을 확인하고, 존재하는 경우 스마트 볼트로 반환하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`a62973e`](https://github.com/the-standard/smart-vault/commit/a62973ef32942bc74c364d20f03f03229fe8c3bb)로 수정되었습니다.

**Cyfrin:** 확인됨, 원치 않는 토큰의 잔여 금액은 이제 발신자에게 다시 전송됩니다.


### `WETH` 담보는 `SmartVaultV4`에서 스왑할 수 없음

**설명:** [`SmartVaultV4::swap`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L259-L277)은 스마트 볼트 담보( `bytes32` 심볼로 지정됨)를 다른 지원되는 담보 토큰으로 스왑할 수 있게 합니다. 주어진 심볼에 해당하는 토큰 주소는 [`SmartVaultV4::getToken`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L220-L226)의 출력을 기반으로 [`SmartVaultV4::getTokenisedAddr`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L228-L231)에 의해 반환됩니다:

```solidity
function getToken(bytes32 _symbol) private view returns (ITokenManager.Token memory _token) {
    ITokenManager.Token[] memory tokens = ITokenManager(ISmartVaultManagerV3(manager).tokenManager()).getAcceptedTokens();
    for (uint256 i = 0; i < tokens.length; i++) {
        if (tokens[i].symbol == _symbol) _token = tokens[i];
    }
    if (_token.symbol == bytes32(0)) revert InvalidToken();
}

function getTokenisedAddr(bytes32 _symbol) private view returns (address) {
    ITokenManager.Token memory _token = getToken(_symbol);
    return _token.addr == address(0) ? ISmartVaultManagerV3(manager).weth() : _token.addr;
}
```

네이티브 `ETH`는 허용된 토큰 목록에 있지만 `address(0)`을 반환합니다. 따라서 `ETH`와 `WETH`의 심볼은 모두 `WETH` 주소에 해당하며, 이는 Uniswap V3 라우터 스왑 명령의 [`tokenIn`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L265) 매개변수로 사용됩니다. 이는 Uniswap V3 라우터를 통해 네이티브 `ETH`를 스왑하는 올바른 방법이며, 라우터는 `amountIn`을 충당하기 위해 먼저 [모든 네이티브 잔액을 활용하려고 시도](https://github.com/Uniswap/v3-periphery/blob/0682387198a24c7cd63566a2c58398533860a5d1/contracts/base/PeripheryPayments.sol#L58-L61)합니다.

스왑 매개변수가 채워진 후 실제 스왑의 실행은 `inToken` 주소에 따라 [`SmartVaultV4::executeNativeSwapAndFee`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L233-L237) 또는 [`SmartVaultV4::executeERC20SwapAndFee`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L239-L248)를 기반으로 발생합니다:

```solidity
inToken == ISmartVaultManagerV3(manager).weth() ?
    executeNativeSwapAndFee(params, swapFee) :
    executeERC20SwapAndFee(params, swapFee);
```

여기서 첫 번째 조건 분기는 호출자가 `WETH` 또는 네이티브 `ETH`를 스왑하려는 경우 실행됩니다. 그러나 이 로직은 호출자가 전적으로 네이티브 `ETH`를 스왑하기를 원한다고 가정하므로, 스마트 볼트에 네이티브 `ETH` 스왑을 수행하기에 충분한 `ETH` 잔액이 없는 한 `WETH`에 대해서는 실패합니다.

**파급력:** `WETH` 담보는 스마트 볼트 내에서 직접 스왑할 수 없습니다.

**개념 증명 (PoC):** 다음 테스트를 `SmartVault.js`에 추가할 수 있습니다:

```javascript
it('cant swap WETH', async () => {
  const ethCollateral = ethers.utils.parseEther('0.1')
  await MockWeth.connect(user).deposit({value: ethCollateral});
  await MockWeth.connect(user).transfer(Vault.address, ethCollateral);

  let { collateral } = await Vault.status();
  expect(getCollateralOf('WETH', collateral).amount).to.equal(ethCollateral);

  await expect(
    Vault.connect(user).swap(
      ethers.utils.formatBytes32String('WETH'),
      ethers.utils.formatBytes32String('WBTC'),
      ethers.utils.parseEther('0.05'),
      0)
    ).to.be.revertedWithCustomError(Vault, 'TransferError');
});
```

**권장 완화 조치:** `SmartRouterV4::swap`의 조건부 로직을 수정하여 `SmartVaultV4::executeERC20SwapAndFee`로 `WETH`를 처리하는 것을 고려하십시오:

```diff
-   inToken == ISmartVaultManagerV3(manager).weth() ?
+   _inToken == NATIVE ?
        executeNativeSwapAndFee(params, swapFee) :
        executeERC20SwapAndFee(params, swapFee);
```

**The Standard DAO:** 커밋 [`fb965bd`](https://github.com/the-standard/smart-vault/commit/fb965bdee4036cb525e4df18f77ece7b32720a66)로 수정되었습니다.

**Cyfrin:** 확인됨, `WETH` 담보는 이제 스왑될 수 있습니다. 그러나 출력 토큰이 `NATIVE`로 지정된 경우 스마트 볼트의 기존 `WETH` 담보도 인출됩니다. 또한 `SmartVaultV4::executeNativeSwapAndFee`는 이제 더 이상 사용되지 않으므로 제거할 수 있습니다.

**The Standard DAO:** 커밋 [`589d645`](https://github.com/the-standard/smart-vault/commit/589d645eae5bc5a10aa0e32302942fcbc5a07491)로 수정되었습니다.

**Cyfrin:** 확인됨, 이제 스왑의 `WETH` 출력만 네이티브로 인출됩니다.


### revert 되는 ERC-20 전송으로 인해 청산이 차단될 수 있음

**설명:** `SmartVaultV4::liquidate`를 통해 청산이 수행될 때 ERC-20 담보 토큰은 루프 내에서 처리됩니다:

```solidity
function liquidate() external onlyVaultManager {
    /* snip: validation, state updates & native liquidation
    ITokenManager.Token[] memory tokens = ITokenManager(ISmartVaultManagerV3(manager).tokenManager()).getAcceptedTokens();
    for (uint256 i = 0; i < tokens.length; i++) {
        if (tokens[i].symbol != NATIVE) liquidateERC20(IERC20(tokens[i].addr));
    }
}
```

주어진 ERC-20의 계약 잔액이 0이 아니면, 아래와 같이 프로토콜 주소로 전송을 수행합니다:

```solidity
function liquidateERC20(IERC20 _token) private {
    if (_token.balanceOf(address(this)) != 0) _token.safeTransfer(ISmartVaultManagerV3(manager).protocol(), _token.balanceOf(address(this)));
}
```

그러나 이러한 전송 중 하나라도 revert 되면 전체 호출이 revert 되고 청산이 차단됩니다. 현재 지원될 예정인 담보 토큰을 분석한 결과 즉각적인 위험은 확인되지 않았지만 다음 사항에 유의해야 합니다:
- `GMX`는 전송 시 보상 분배 로직을 포함합니다(가능성은 낮지만 잠재적으로 revert 될 수 있음).
- `WETH` 및 `ARB`는 투명한 업그레이드 가능(Transparent Upgradeable) 프록시입니다.
- `WBTC`, `LINK`, `PAXG`, 및 `SUSHI`는 비콘(Beacon) 프록시입니다.
- `RDNT`는 LayerZero 브리지 토큰입니다.

**파급력:** `GMX` 담보 전송이 revert 되면 주어진 스마트 볼트에 대한 청산이 차단됩니다. 다른 담보 토큰이 새로운 전송 로직을 도입하도록 업그레이드되면 스마트 볼트가 이 문제에 취약해질 수 있습니다. 공격자가 단일 담보 토큰 전송을 강제로 revert 시킬 수 있다면 청산을 피할 수 있습니다.

**권장 완화 조치:** 차단된 청산을 방지하기 위해 `try/catch`로 각 ERC-20 전송을 별도로 처리하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`efda8d2`](https://github.com/the-standard/smart-vault/commit/efda8d2de7cb4406598d50099f52fc1275769c0a)로 수정되었습니다.

**Cyfrin:** 확인됨, 단일 전송이 실패해도 청산은 더 이상 revert 되지 않습니다. `SafeERC20::safeTransfer` 대신 `ERC20::transfer`를 직접 사용하는 것은 괜찮아 보입니다. 왜냐하면:
- 스마트 볼트는 허용된 토큰을 반복할 때 항상 코드가 있는 계약을 호출합니다.
- 현재 허용된 담보 토큰 목록은 모두 `true`를 반환하거나 실패한 전송에 대해 revert 합니다.


### 잠재적으로 잘못된 스왑 경로 인코딩

**설명:** 포크 테스트 중에 스왑 경로가 packed 인코딩을 사용해야 한다는 것이 분명해졌습니다. 그러나 [기존 모의(mocked) 테스트 스위트](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/test/SmartVault.js#L499-L509)는 다음을 수행합니다:

```javascript
// data about how yield manager converts collateral to USDC, vault addresses etc
await YieldManager.addHypervisorData(
  MockWeth.address, MockWETHWBTCHypervisor.address, 500,
  new ethers.utils.AbiCoder().encode(['address', 'uint24', 'address'], [MockWeth.address, 3000, USDC.address]),
  new ethers.utils.AbiCoder().encode(['address', 'uint24', 'address'], [USDC.address, 3000, MockWeth.address])
)
```

[ethers 문서](https://docs.ethers.org/v5/api/utils/hashing/#utils-solidityPack)를 참조하면 `AbiCoder::encode`가 packed 인코딩에 대한 올바른 방법이 아님을 알 수 있습니다. 배포된 계약에 대한 Hypervisor 데이터의 실제 구성으로 확장되면, 실패한 스왑으로 인해 모든 수익 예치 기능이 revert 될 것입니다.

**파급력:** Hypervisor 데이터의 잘못된 구성으로 인해 수익 예치 기능이 작동하지 않습니다.

**권장 완화 조치:** 스왑 경로에 대해 tightly packed 인코딩을 사용하십시오.

**The Standard DAO:** 인지함. 이러한 종류의 인코딩이 실제 라우터가 있는 프로덕션에서는 작동하지 않는다는 것을 알고 있지만, 모의 스왑 라우터에서 올바른 경로 유형을 디코딩하는 방법을 알아내지 못했습니다. 솔루션을 알고 있는 경우 테스트 및 모의 스왑 라우터를 수정할 것입니다.

**Cyfrin:** 인지함. 해결책은 Uniswap V3 [Path](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/Path.sol) 및 [BytesLib](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/BytesLib.sol) 라이브러리를 사용하는 것입니다. 그러나 모의 테스트에 이 추가 복잡성이 필요하지 않을 수 있습니다.


### 18 소수점 이상의 담보 토큰이 지원되지 않음

**설명:** [`PriceCalculator::getTokenScaleDiff`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/PriceCalculator.sol#L15-L17) 내의 기존 소수점 스케일링 로직으로 인해, 18 소수점 이상의 담보 토큰은 지원되지 않으며 스마트 볼트 기능의 서비스 거부(DoS)를 초래합니다:

```solidity
function getTokenScaleDiff(bytes32 _symbol, address _tokenAddress) private view returns (uint256 scaleDiff) {
    return _symbol == NATIVE ? 0 : 18 - ERC20(_tokenAddress).decimals();
}
```

유사한 스케일링이 [`SmartVaultV4::yieldVaultCollateral`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L92-L93)에 존재합니다. 그러나 이것은 문제가 있는 기본 토큰을 가진 또 다른 `USDs` Hypervisor가 추가되어야 하는데, 이는 가능성이 낮습니다.

**파급력:** 18 소수점 이상의 토큰이 허용된 토큰 목록에 추가되면 스마트 볼트 담보를 계산할 수 없어 서비스 거부가 발생합니다.


**권장 완화 조치:** 18 소수점 이상의 담보 토큰이 추가될 경우 더 큰 소수점으로 스케일링하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`cf871f7`](https://github.com/the-standard/smart-vault/commit/cf871f7950465904f3f8967e6504eacdd1cbc75c)로 수정되었습니다 – Hypervisor 예치에는 적합하지 않지만 담보에는 괜찮을 것입니다.

**Cyfrin:** 확인됨, 이제 18 소수점 이상의 담보 토큰을 지원합니다. 그러나 `scale < 0` 분기에 대해 곱하기 전 나눗셈은 문제가 될 수 있습니다. `tokenToUSD`의 반환 문에서 36으로 모든 소수점을 스케일링한 다음 18로 다시 나누는 것이 더 나을 수 있습니다.

**The Standard DAO:** 커밋 [`2342302`](https://github.com/the-standard/smart-vault/commit/23423024550bca2dbe182e079403b28cc8d1f6e9)에서 수정되었습니다.

**Cyfrin:** 확인됨, 이제 소수점을 36으로 스케일링한 다음 18로 다시 축소합니다.

\clearpage
## 정보성 (Informational)


### `msg.sender`를 `address`로 불필요하게 형변환

**설명:** `SmartVaultYieldManager::deposit`에 `msg.sender` 컨텍스트 변수가 불필요하게 `address`로 형변환되는 [인스턴스](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L169)가 있습니다:

```solidity
uint256 _balance = IERC20(_collateralToken).balanceOf(address(msg.sender));
```

**권장 완화 조치:** `msg.sender`는 이미 주소이므로 `address` 형변환을 제거하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`1a9dc5f`](https://github.com/the-standard/smart-vault/commit/1a9dc5fc3f553d1b3dbf285e863d0f8cf5f8bbc0)로 수정되었습니다.

**Cyfrin:** 확인됨, 형변환이 제거되었습니다.


### 주석이 `$`여야 할 때 잘못하여 `€`를 참조함

**설명:** `SmartVaultV4::yieldVaultCollateral`에서 스테이블코인 담보를 합산할 때 다음 주석이 [존재](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L91)합니다:

```solidity
// both USDs and its vault pair are € stablecoins, but can be equivalent to €1 in collateral
```

여기서 `€` 기호는 `$` 대신 USD에 사용됩니다.

**권장 완화 조치:** 주석을 업데이트하여 `$` 기호를 사용하십시오.

**The Standard DAO:** 더 이상 적용되지 않음. 커밋 [`5862d8e`](https://github.com/the-standard/smart-vault/commit/5862d8e10ac8648b89a7e3a78498ff20dc31e42e)에서 주석이 제거되었습니다.

**Cyfrin:** 확인됨, 주석이 제거되었습니다.


### `SmartVaultYieldManager::_withdrawUSDsDeposit`에서 동등한 함수 매개변수와 불변 변수의 일관성 없는 사용으로 인한 혼란

**설명:** `SmartVaultYieldManager::_withdrawUSDsDeposit`에서 `USDs` Hypervisor로부터 담보를 인출할 때, `SmartVaultYieldManager::withdraw`의 다음 [조건부 확인](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L211-L213)으로 인해 `_hypervisor` 매개변수는 항상 불변 `usdsHypervisor` 변수와 같습니다:

```solidity
_hypervisor == usdsHypervisor ?
    _withdrawUSDsDeposit(_hypervisor, _token) :
    _withdrawOtherDeposit(_hypervisor, _token);
```

그러나 [`SmartVaultYieldManager::_withdrawUSDsDeposit`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L196-L200) 내에서는 `_hypervisor` 매개변수와 동등한 불변 변수가 혼합되어 사용됩니다:

```solidity
    function _withdrawUSDsDeposit(address _hypervisor, address _token) private {
    IHypervisor(_hypervisor).withdraw(_thisBalanceOf(_hypervisor), address(this), address(this), [uint256(0),uint256(0),uint256(0),uint256(0)]);
    _swapToSingleAsset(usdsHypervisor, USDC, ramsesRouter, 500);
    _sellUSDC(_token);
}
```

이것은 `_hypervisor` 매개변수가 불변 `usdsHypervisor`와 다르다는 것을 암시할 수 있으므로 독자에게 혼란을 줍니다.

**권장 완화 조치:** `_hypervisor` 매개변수 또는 불변 `usdsHypervisor` 변수를 일관되게 사용하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [`f601a11`](https://github.com/the-standard/smart-vault/commit/f601a1173e0b2e2006e73c13339051ae7c7e6af1)로 수정되었습니다.

**Cyfrin:** 확인됨, 이제 불변 변수가 독점적으로 사용됩니다.


### `USDC`를 허용된 담보 토큰으로 추가할 수 없음

**설명:** 감마에 대한 각 담보 예치금의 최소 10%는 `USDs` Hypervisor의 기본이 되는 `USDs/USDC` 풀로 향해야 합니다:

```solidity
function _usdDeposit(address _collateralToken, uint256 _usdPercentage, bytes memory _pathToUSDC) private {
    _swapToUSDC(_collateralToken, _usdPercentage, _pathToUSDC);
    _swapToRatio(USDC, usdsHypervisor, ramsesRouter, 500);
    _deposit(usdsHypervisor);
}
```

이 과정에서 [`SmartVaultYieldManager::_swapToUSDC`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L133-L144)는 담보 토큰을 `USDC`로 스왑합니다. 그러나 `USDC`는 자신에 대한 경로가 없으므로 추가 처리 없이는 실패합니다. `USDs` Hypervisor 예치금을 USDC로 인출하려고 시도할 때 [`SmartVaultYieldManager::_sellUSDC`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L182-L194)에도 유사한 문제가 있습니다.

또한 이것이 올바르게 처리되었다고 가정하더라도 [`SmartVaultYieldManager::thisBalanceOf`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L49-L51)의 광범위한 사용은 후속 Hypervisor 예치를 고려하지 않고 [`SmartVaultYieldManager::_swapToRatio`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L60-L61) 내에서 `USDs` Hypervisor 예치에 USDC의 전체 잔액을 활용하게 합니다:

```solidity
uint256 _tokenBBalance = _thisBalanceOf(_tokenB);
(uint256 _amountStart, uint256 _amountEnd) = IUniProxy(uniProxy).getDepositAmount(_hypervisor, _tokenA, _thisBalanceOf(_tokenA));
```

또한 `USDC`가 허용된 담보 토큰으로 추가되면 블랙리스트에 오른 스마트 볼트에 대한 청산이 차단됩니다. 공격자는 불법적으로 획득한 `USDC`를 스마트 볼트에 예치하고 `USDs`를 차입하여, 프로토콜이 이 토큰을 전송하려는 시도가 실패함에 따라 청산되는 것을 피할 수 있습니다.

**파급력:** `USDC`는 허용된 담보로 추가될 수 없습니다.

**권장 완화 조치:** `USDC`를 허용된 담보 토큰으로 추가하려면 먼저 이러한 문제를 해결해야 합니다.

**The Standard DAO:** 인지함. `USDC`를 담보 유형으로 추가할 의도가 없기 때문에 수정하지 않음. 만약 추가하더라도 그것에 대한 hypervisor 데이터를 추가하지 않는 한 여전히 괜찮을 것이라고 믿습니다. 이것은 우리에게 허용 가능한 것으로 보입니다.

**Cyfrin:** 인지함.


### `SmartVaultV4::removeAsset`을 사용하여 네이티브 자산을 제거할 수 없음

**설명:** `SmartVault::removeAsset`은 스마트 볼트 소유자가 볼트가 완전히 담보된 상태로 유지되는 한 담보 자산을 포함하여 볼트에서 자산을 제거할 수 있도록 허용합니다. 이것은 현재 ERC-20 담보 토큰에 대해 작동합니다. 그러나 `_tokenAddr == address(0)`인 경우에 대한 처리가 없습니다. 이 주소는 허용된 `TokenManager` 토큰 목록의 `NATIVE` 심볼에 해당하지만, 이 엣지 케이스가 고려되지 않았기 때문에 `SafeERC20::safeTransfer`에 의한 네이티브 전송 시도는 실패합니다.

```solidity
function removeAsset(address _tokenAddr, uint256 _amount, address _to) external onlyOwner {
    ITokenManager.Token memory token = getTokenManager().getTokenIfExists(_tokenAddr);
    if (token.addr == _tokenAddr && !canRemoveCollateral(token, _amount)) revert Undercollateralised();
    IERC20(_tokenAddr).safeTransfer(_to, _amount);
    emit AssetRemoved(_tokenAddr, _amount, _to);
}
```

네이티브 담보 인출은 이미 `SmartVaultV4::removeCollateralNative`에 의해 올바르게 처리되지만, 이 엣지 케이스는 `SmartVault::removeAsset` 내에서 ERC-20과 네이티브 자산 전송 간의 비대칭을 초래합니다.

**파급력:** 스마트 볼트 소유자는 `SmartVault::removeAsset`을 사용하여 볼트에서 네이티브 토큰을 제거할 수 없습니다.

**권장 완화 조치:** 네이티브 자산 제거가 시도되는 경우를 처리하는 것을 고려하십시오. 또한 제거된 자산이 담보 자산인지 여부에 따라 이벤트 사용을 재고해야 합니다.

```diff
function removeAsset(address _tokenAddr, uint256 _amount, address _to) external onlyOwner {
    ITokenManager.Token memory token = getTokenManager().getTokenIfExists(_tokenAddr);
    if (token.addr == _tokenAddr && !canRemoveCollateral(token, _amount)) revert Undercollateralised();
+   if(_tokenAddr == address(0)) {
+       (bool sent,) = payable(_to).call{value: _amount}("");
+       if (!sent) revert TransferError();
+   } else {
-   IERC20(_tokenAddr).safeTransfer(_to, _amount);
+        IERC20(_tokenAddr).safeTransfer(_to, _amount);
+   }
    emit AssetRemoved(_tokenAddr, _amount, _to);
}
```

**The Standard DAO:** 커밋 [`8257c4c`](https://github.com/the-standard/smart-vault/commit/8257c4c267fa86c2c237ff6a2acdcfe94bcfeb20) & [`57d5db4`](https://github.com/the-standard/smart-vault/commit/57d5db47e072d8730c0d0988217db8d66ef565d9)로 수정되었습니다.

**Cyfrin:** 확인됨, 네이티브 담보는 이제 두 함수 중 하나를 통해 제거될 수 있습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 수익 포지션으로/로부터 담보 예치/인출 시 `SmartVaultV4::usdCollateral`에 대한 불필요한 호출

**설명:** `SmartVaultV4`에서 수익 포지션으로/로부터 담보를 예치/인출할 때, 스마트 볼트가 충분히 담보된 상태로 유지되는지 검증하고 담보 가치가 10% 이상 떨어지지 않았는지 검증합니다:

```solidity
if (undercollateralised() || significantCollateralDrop(_preDepositCollateral, usdCollateral())) revert Undercollateralised();
```

이 로직은 스마트 볼트 담보의 가치를 얻기 위해 [`SmartVaultV4::usdCollateral`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L105-L114)을 호출합니다. 그러나 이것은 담보 토큰에 대해 여러 루프를 수행하는 값비싼 호출이며 [`SmartVaultV4::undercollateralised`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultV4.sol#L142-L144) 내에서도 호출됩니다:

```solidity
function undercollateralised() public view returns (bool) {
    return minted > maxMintable(usdCollateral());
}
```

**권장 완화 조치:** 담보 예치/인출 후 `usdCollateral()`을 한 번만 호출한 다음, 그 값을 `undercollaterlised()` 및 `significantCollateralDrop()`에 전달하는 것을 고려하십시오. `undercollateralised()`의 현재 구현은 `usdCollateral()`을 호출하고 결과를 담보 가치를 인수로 취하는 내부 `_undercollateralised()` 함수에 전달하는 공개 함수로 리팩터링할 수 있습니다.

**The Standard DAO:** 커밋 [`3fdefc8`](https://github.com/the-standard/smart-vault/commit/3fdefc8d9b4f46aad33993a26ca5a04defdf740a)로 수정되었습니다.

**Cyfrin:** 확인됨, 비공개 함수가 도입되었습니다.


### 캐시된 `_token0`이 사용되지 않음

**설명:** [`SmartVaultYieldManager::_swapToSingleAsset`](https://github.com/the-standard/smart-vault/blob/c6837d4a296fe8a6e4bb5e0280a66d6eb8a40361/contracts/SmartVaultYieldManager.sol#L114-L117)에서 캐시된 주소 `_token0`은 3항 연산의 조건에서 사용되지 않습니다:

```solidity
address _token0 = IHypervisor(_hypervisor).token0();
address _unwantedToken = IHypervisor(_hypervisor).token0() == _wantedToken ?
    IHypervisor(_hypervisor).token1() :
    _token0;
```

**권장 완화 조치:** 비교에 캐시된 `_token0` 변수를 사용하십시오.

**The Standard DAO:** 커밋 [`1c30144`](https://github.com/the-standard/smart-vault/commit/1c3014465689d75d1fc057cadb5cdd75d8f18a2d)로 수정되었습니다.

**Cyfrin:** 확인됨, 이제 캐시된 주소가 사용됩니다.

\clearpage
