**Lead Auditors**

[Hans](https://twitter.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Critical Risk


### SecuritizeVault's share token is in wrong decimals

**Description:** `SecuritizeVault`는 OpenZeppelin의 `ERC4626Upgradeable`을 상속하며, 여기서 공유 토큰(share tokens)은 기본적으로 기본 자산 토큰과 동일한 소수점 자릿수(decimals)를 유지합니다. 그러나 이 볼트는 `_convertToShares()` 및 `_convertToAssets()`를 재정의(override)하여 청산 토큰(liquidation token) 소수점 자릿수에 있다고 가정되는 NAV 제공자 비율을 통합합니다. 이는 외부 상환이 사용되고 청산 토큰이 자산 토큰과 다를 때 소수점 처리에서 불일치를 생성합니다.

`_convertToShares()`에서 반환된 공유 금액은 NAV 비율을 사용하여 계산되며, 예상되는 공유 토큰 소수점 자릿수 대신 청산 토큰의 소수점 자릿수를 유지합니다. 마찬가지로 `_convertToAssets()`에서도 계산의 한 분기는 입력 공유가 청산 토큰 소수점 자릿수에 있다고 가정하는 반면, 다른 분기는 자산 토큰 소수점 자릿수를 가정합니다.

```solidity
SecuritizeVault.sol
310:     function _convertToShares(uint256 _assets, Math.Rounding rounding) internal view virtual override(ERC4626Upgradeable) returns (uint256) {
311:         uint256 rate = navProvider.rate();//@audit-info D{liquidationToken}
312:         uint256 decimalsFactor = 10 ** decimals();
313:         uint256 totalSharesAfterDeposit = rate.mulDiv(totalAssets() + _assets, decimalsFactor, rounding);//@audit-info D{liquidationToken} * D{asset} / D{asset} -> D{liquidationToken}
314:         if (totalSharesAfterDeposit >= totalSupply()) {
315:             return totalSharesAfterDeposit - totalSupply();//@audit-issue the return amount must be in the decimals of the share token but it is in liquidationToken's decimals
316:         }
317:         return 0;
318:     }

333:     function _convertToAssets(uint256 _shares, Math.Rounding rounding) internal view virtual override(ERC4626Upgradeable) returns (uint256) {
334:         if (totalSupply() == 0) {
335:             return 0;
336:         }
337:         uint256 rate = navProvider.rate();//@audit-info D{liquidationToken}
338:         uint256 decimalsFactor = 10 ** decimals();
339:         return Math.min(_shares.mulDiv(decimalsFactor, rate, rounding), _shares.mulDiv(totalAssets(), totalSupply(), rounding));//@audit-info D{share} * D{asset} / D{liquidationToken}, D{share} * D{asset} / D{share}
340:     }
```

**Impact:** 사용자는 청산 토큰과 공유 토큰 간의 소수점 불일치로 인해 민팅/소각(minting/burning) 작업 중 잘못된 공유 토큰 금액을 받게 되며, 이는 볼트 공유 토큰이 외부 DeFi 프로토콜에 통합될 때 상당한 가치 손실로 이어집니다.

**Proof Of Concept:**
아래는 Foundry로 작성된 테스트 케이스입니다.
```solidity
    function testShareDecimalsWrong() public {
        navProvider.setRate(1e6);// nav provider rate is 1 in the decimals of liquidationToken

        // deposit
        vm.startPrank(owner);
        uint256 depositAmount = 1000e18;// 1000 asset tokens
        assetToken.mint(owner, depositAmount);
        assetToken.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, owner);
        uint256 redeemShares = vault.balanceOf(owner); // 100% redeem
        emit log_named_uint("Share token decimals", vault.decimals()); // 18
        emit log_named_uint("Share amount", redeemShares); // 1000 * 1e6

        vm.stopPrank();
    }
```
출력 결과는 다음과 같습니다.
```bash
forge test -vvv --match-test testShareDecimalsWrong

Ran 1 test for test/forge/SecuritizeVaultTest_R.t.sol:SecuritizeVaultTest_R
[PASS] testShareDecimalsWrong() (gas: 159913)
Logs:
  Share token decimals: 18
  Share amount: 1000000000
```
**Recommended Mitigation:** 두 가지 해결책이 있을 수 있습니다.
1. `_decimalsOffset()` 또는 `decimals()`를 재정의하여 청산 토큰의 소수점 자릿수를 제공하도록 합니다.
    이는 더 간단한 수정일 수 있지만 권장되지 않습니다. 외부 상환이 사용되는 경우 청산 토큰은 소수점 6자리의 스테이블 코인이 됩니다. 따라서 공유 토큰도 소수점 6자리가 되며, 이는 여러 곳에서 정밀도 손실을 발생시킬 것입니다. 또한 OZ의 ERC4626Upgradeable의 다른 부분이 영향을 받지 않는지 확인해야 합니다.
2. `_convertToShares` 및 `_convertToAssets` 반환 값이 올바른 소수점 자릿수가 되도록 구현을 변경합니다.

또한 변수(예: `decimalFactor` 대신 `assetDecimalFactor` 또는 `shareDecimalFactor`)에 더 정확한 이름을 사용하고 모든 곳에서 값과 공식의 소수점 자릿수를 문서화하는 것이 좋습니다.

**Securitize:** 커밋 [9788de](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/9788decda7cdf9979179d4a1f19f0c6619ada46f)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## Medium Risk


### Lack of slippage protection in the SecuritizeVault's liquidation function

**Description:** SecuritizeVault의 `liquidate()` 함수는 사용자가 최소 출력 금액을 지정하도록 허용하지 않고 NAV 제공자의 비율을 사용하여 공유 토큰을 자산 토큰으로 변환합니다. 변환율은 동적이고 외부 상환이 포함될 수 있으므로, 사용자는 트랜잭션 제출과 실행 사이의 비율 변경으로 인한 잠재적 가치 손실에 노출됩니다. 이는 변동성이 큰 시장 상황이나 트랜잭션 처리에 지연이 많을 때 특히 우려됩니다.

**Impact:** 사용자는 비율 변경으로 인해 청산 중 예상보다 적은 자산을 받을 수 있으며, 이는 직접적인 금전적 손실로 이어집니다.

**Recommended Mitigation:**
1. 청산 함수에 `minOutputAmount` 매개변수를 추가하십시오:
```solidity
function liquidate(uint256 shares, uint256 minOutputAmount) public override(ISecuritizeVault) whenNotPaused {
...
    require(assets >= minOutputAmount, "Insufficient output amount");
...
}
```
2. 오래 대기 중인 트랜잭션으로부터 보호하기 위해 시간 제한 매개변수를 구현하는 것을 고려하십시오.
3. 모니터링 목적으로 실제 출력 금액을 추적하는 이벤트를 추가하십시오.

**Securitize:** 커밋 [42e651](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/42e6511931f03b266b3acd4ca220544246efb4ab)에서 수정됨.

**Cyfrin:** 확인됨.



### Inconsistent token amount in external redemption process of SecuritizeVault's liquidate function

**Description:** 외부 상환을 포함한 SecuritizeVault의 청산 프로세스에서, 상환 컨트랙트로부터 실제로 받은 스테이블 코인과 청산인에게 보내기 위해 계산된 금액 사이에 불일치가 발생할 가능성이 높습니다. 볼트는 외부 상환 후 `navProvider.rate()`를 사용하여 출력 금액을 계산하지만, 이 계산된 금액은 상환 프로세스 중의 반올림 차이 및 비율 변동으로 인해 실제로 받은 스테이블 코인과 다를 수 있습니다.
```solidity
SecuritizeVault.sol
261:         if (address(0) != address(redemption)) {
262:             IERC20(asset()).approve(address(redemption), assets);
263:             redemption.redeem(assets);//@audit-info this sends underlying token to the redemption, receives stablecoin into this contract
264:             uint256 rate = navProvider.rate();
265:             uint256 decimalsFactor = 10 ** decimals();
266:             // after external redemption, vault gets liquidity to supply msg.sender (assets * nav)
267:             // liquidationToken === stableCoin
268:             liquidationToken.safeTransfer(msg.sender, assets.mulDiv(rate, decimalsFactor, Math.Rounding.Floor));//@audit-issue possible inconsistency in the amoutns. consider sending the balance delta instead
269:         }

```
**Impact:** 사용자는 받아야 할 것보다 적은 스테이블 코인을 받거나(가치 손실), 계산된 금액이 받은 토큰을 초과할 때 불충분한 잔액으로 인해 트랜잭션이 되돌려져 청산 기능을 신뢰할 수 없게 만듭니다.

**Recommended Mitigation:** 상환 컨트랙트로부터 실제로 받은 스테이블 코인을 추적하여 전송하십시오.
```solidity
    if (address(0) != address(redemption)) {
        IERC20 liquidityToken = IERC20(redemption.liquidity());
        uint256 balanceBefore = liquidityToken.balanceOf(address(this));
        redemption.redeem(assets);
        uint256 receivedAmount = liquidityToken.balanceOf(address(this)) - balanceBefore;
        liquidityToken.transfer(msg.sender, receivedAmount);
    }
```

**Securitize:** 커밋 [ef761a](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/ef761a654b4015478c82cecd33cc54f7a97d37bb)에서 수정됨.

**Cyfrin:** 확인됨.


### Ambiguous owner terminology creates confusion in access control of SecuritizeVault

**Description:** SecuritizeVault는 접근 제어에 혼란을 주는 두 가지 다른 소유권 개념을 구현합니다. 볼트 작업(입금/상환)을 위해 사용자 정의 `OWNER_ROLE`을 사용하는 반면, `owner()` 함수를 제공하는 `OwnableUpgradeable`을 상속합니다. 이는 문맥에 따라 "소유자(owner)"가 다른 주소를 지칭하는 상황을 만듭니다. `OWNER_ROLE` 보유자는 볼트 작업을 수행할 수 있지만 `pause()`와 같은 `onlyOwner` 함수를 실행할 수 없으며, Ownable 소유자는 일시 중지할 수 있지만 볼트 작업을 수행할 수 없습니다.

**Impact:** 이러한 모호성은 개발자나 사용자가 어떤 "소유자"가 어떤 권한을 가지고 있는지 오해할 경우 보안 문제로 이어질 수 있으며, 잠재적으로 실패한 작업이나 종속 컨트랙트에서의 잘못된 접근 제어 구현을 초래할 수 있습니다.

**Proof Of Concept:**
```solidity
function testPause() public {
    // admin (the deployer) is Ownable owner - can pause/unpause
    vm.startPrank(admin);
    vault.pause();
    assertTrue(vault.paused());
    vault.unpause();
    vm.stopPrank();

    // OWNER_ROLE holder cannot pause
    vm.startPrank(owner);
    vm.expectRevert(abi.encodeWithSelector(OwnableUpgradeable.OwnableUnauthorizedAccount.selector, owner));
    vault.pause();
    vm.stopPrank();

    // Demonstrating the confusion:
    assertTrue(vault.isOwner(owner));     // True: has OWNER_ROLE
    assertFalse(vault.isOwner(admin));    // False: doesn't have OWNER_ROLE
    assertTrue(vault.owner() == admin);    // True: is Ownable owner
}
```

**Recommended Mitigation:**
1. 역할(roles) 또는 Ownable 패턴 중 하나만 독점적으로 사용하도록 소유권 모델을 통합하십시오.
2. 둘 다 필요한 경우 함수/역할 이름을 더 명시적으로 변경하십시오:
```solidity
// Instead of OWNER_ROLE
bytes32 public constant VAULT_OPERATOR_ROLE = keccak256("VAULT_OPERATOR_ROLE");

// Instead of isOwner()
function isVaultOperator(address account) public view returns (bool) {
    return hasRole(VAULT_OPERATOR_ROLE, account);
}
```
3. 다양한 유형의 소유권 간의 차이점을 설명하는 명확한 문서를 추가하십시오.

**Securitize:** 커밋 [887554](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/8875544d6d3c5e20cec43ba06f3cce157a7cfcec)에서 수정됨.

**Cyfrin:** 확인됨.


\clearpage
## Low Risk


### Missing input validation in privileged functions

**Description:** 관리자 함수에 적절한 매개변수 유효성 검사가 부족하여 잘못된 값이 제공될 경우 의도하지 않은 상태 변경이 발생할 수 있습니다. 관리자 사용자는 올바르게 행동할 것으로 예상되지만 사람의 실수는 여전히 가능합니다.

```solidity
SecuritizeRedemption.sol
75:     function initialize(address _asset, address _navProvider) public onlyProxy initializer navProviderNonZero(_navProvider) {
76:         __BaseDSContract_init();
77:         asset = IERC20(_asset);//@audit-issue INFO non zero check
78:         navProvider = ISecuritizeNavProvider(_navProvider);
79:     }
80:

SecuritizeRedemption.sol
81:     function updateLiquidityProvider(address _liquidityProvider) onlyOwner external {
82:         address oldProvider = address(liquidityProvider);
83:         liquidityProvider = ILiquidityProvider(_liquidityProvider);//@audit-issue INFO non zero check
84:         emit LiquidityProviderUpdated(oldProvider, address(liquidityProvider));
85:     }

SecuritizeRedemption.sol
87:     function updateNavProvider(address _navProvider) onlyOwner navProviderNonZero(_navProvider) external {
88:         address oldProvider = address(navProvider);
89:         navProvider = ISecuritizeNavProvider(_navProvider);//@audit-issue INFO non zero check
90:         emit NavProviderUpdated(oldProvider, address(navProvider));
91:     }
```

```solidity
CollateralLiquidityProvider.sol
66:     function initialize(address _recipient, address _liquidityToken, address _securitizeRedemption) public onlyProxy initializer {
67:         __BaseDSContract_init();
68:         recipient = _recipient;//@audit-issue LOW sanity check
69:         liquidityToken = IERC20(_liquidityToken);
70:         securitizeRedemption = ISecuritizeRedemption(_securitizeRedemption);
71:     }

98:
99:     function setCollateralProvider(address _collateralProvider) external onlyOwner {
100:         address oldAddress = address(collateralProvider);
101:         collateralProvider = _collateralProvider;//@audit-issue LOW sanity check
102:         emit CollateralProviderUpdated(oldAddress, address(collateralProvider));
103:     }
```

```solidity
AllowanceLiquidityProvider.sol
65:     function initialize(address _recipient, address _liquidityToken, address _securitizeRedemption) public onlyProxy initializer {
66:         __BaseDSContract_init();
67:         recipient = _recipient;//@audit-issue LOW sanity check
68:         liquidityToken = IERC20(_liquidityToken);
69:         securitizeRedemption = ISecuritizeRedemption(_securitizeRedemption);
70:     }

85:     function setAllowanceProviderWallet(address _liquidityProviderWallet) external onlyOwner {
86:         address oldAddress = liquidityProviderWallet;
87:         liquidityProviderWallet = _liquidityProviderWallet;//@audit-issue sanity check
88:         emit AllowanceLiquidityProviderWalletUpdated(oldAddress, liquidityProviderWallet);
89:     }
```

**Securitize:** 커밋 [8254e8](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/8254e84a8bd2579780cc7b3b1ffa4b9821bcd505)에서 수정됨.

**Cyfrin:** 확인됨.


### Insufficient storage gap in BaseSecuritizeSwap contract

**Description:** BaseSecuritizeSwap 컨트랙트는 40 슬롯(`uint256[40] __gap`)의 스토리지 갭을 구현하는데, 이는 업그레이드 가능한 컨트랙트에 대해 총 50개의 스토리지 슬롯을 예약하는 OpenZeppelin의 관행을 따르지 않습니다. 표준에서의 이러한 편차는 향후 업그레이드에서 잠재적으로 스토리지 충돌 문제를 일으킬 수 있습니다.

```solidity
abstract contract BaseSecuritizeSwap is BaseDSContract, PausableUpgradeable {
    // ... other contract code ...
    uint256[40] __gap; // @audit-issue LOW -> uint256[43]
}
```

아래는 현재 스토리지 레이아웃을 보여줍니다.
```
forge inspect BaseSecuritizeSwap storage --pretty
| Name | Type | Slot | Offset | Bytes | Contract |
|---|---|---|---|---|---|
| services | mapping(uint256 => address) | 0 | 0 | 32 | BaseSecuritizeSwap |
| __gap | uint256[49] | 1 | 0 | 1568 | BaseSecuritizeSwap |
| dsToken | contract IDSToken | 50 | 0 | 20 | BaseSecuritizeSwap |
| stableCoinToken | contract IERC20 | 51 | 0 | 20 | BaseSecuritizeSwap |
| erc20Token | contract IERC20 | 52 | 0 | 20 | BaseSecuritizeSwap |
| collateralToken | contract IERC20 | 53 | 0 | 20 | BaseSecuritizeSwap |
| issuerWallet | address | 54 | 0 | 20 | BaseSecuritizeSwap |
| liquidityProviderWallet | address | 55 | 0 | 20 | BaseSecuritizeSwap |
| externalCollateralRedemption | contract IRedemption | 56 | 0 | 20 | BaseSecuritizeSwap |
| swapMode | enum BaseSecuritizeSwap.SwapMode | 56 | 20 | 1 | BaseSecuritizeSwap |
| __gap | uint256[40] | 57 | 0 | 1280 | BaseSecuritizeSwap |
```

`BaseSecuritizeSwap`의 스토리지는 `dstoken`에서 시작하며 `__gap` 변수 앞에 7개의 슬롯이 사용됨을 알 수 있습니다.
OpenZeppelin의 일반적인 관행에 따르면 끝에 43 슬롯 갭을 추가해야 합니다.

**Recommended Mitigation:** OpenZeppelin의 모범 사례에 맞춰 스토리지 갭을 43 슬롯으로 늘리십시오:
```diff
abstract contract BaseSecuritizeSwap is BaseDSContract, PausableUpgradeable {
--    uint256[40] __gap;
++    uint256[43] __gap;
}
```

**Securitize:** 커밋 [0c2a6e](https://bitbucket.org/securitize_dev/securitize-swap/commits/0c2a6e94cea7d4ee30ed8bd0386530f533959c04)에서 수정됨.

**Cyfrin:** 확인됨.



### Missing PausableUpgradable initialization in SecuritizeSwap contract initialization

**Description:** `SecuritizeSwap` 컨트랙트는 (`BaseSecuritizeSwap`을 통해) `PausableUpgradeable`을 상속하지만 초기화 중에 `__Pausable_init()`을 호출하지 않습니다. 이는 현재 기본 상태(paused = false)가 초기화 상태와 일치하므로 기능에는 영향을 미치지 않지만, 모범 사례에서 벗어난 것입니다.

**Recommended Mitigation:** 완전성을 기하고 모범 사례를 따르기 위해 Pausable 초기화를 추가하십시오:
```solidity
function initialize(...) public override initializer onlyProxy {
    BaseSecuritizeSwap.initialize(
        _dsToken,
        _stableCoin,
        _erc20Token,
        _issuerWallet,
        _liquidityProvider,
        _externalCollateralRedemption,
        _collateralToken,
        _swapMode
    );
    __BaseDSContract_init();
    __Pausable_init();
}
```

**Securitize:** 커밋 [637bcc](https://bitbucket.org/securitize_dev/securitize-swap/commits/637bcce49acab125b54caeaa98fffc1790782b60)에서 수정됨.

**Cyfrin:** 확인됨.



### Potential replay attacks due to static DOMAIN_SEPARATOR

**Description:** DOMAIN_SEPARATOR는 초기화 중에 한 번만 설정되며 `block.chainid`를 포함합니다. 이는 이론적으로 하드 포크 시 크로스체인 재생 공격(replay attacks)을 가능하게 할 수 있습니다.
컨트랙트가 이러한 공격을 효과적으로 방지하는 논스 시스템(`mapping(string => uint256) internal noncePerInvestor`)을 구현하고 있지만, 도메인 구분자가 항상 올바른 값을 갖도록 보장하는 모범 사례를 따르는 것이 좋습니다.

```solidity
function initialize(...) public override initializer onlyProxy {
    // ... other initialization code ...

    DOMAIN_SEPARATOR = keccak256(
        abi.encode(
            EIP712_DOMAIN_TYPE_HASH,
            NAME_HASH,
            VERSION_HASH,
            block.chainid,  //@audit-issue LOW this can change in the case of hard fork
            this,
            SALT
        )
    );
}
```

**Recommended Mitigation:** 논스 보호로 인해 엄격하게 필요하지는 않지만, 모범 사례에 따라 DOMAIN_SEPARATOR를 동적으로 만드는 것을 고려하십시오. OpenZeppelin의 EIP712를 대신 사용할 수도 있습니다.
```solidity
function DOMAIN_SEPARATOR() public view returns (bytes32) {
    return keccak256(
        abi.encode(
            EIP712_DOMAIN_TYPE_HASH,
            NAME_HASH,
            VERSION_HASH,
            block.chainid,
            this,
            SALT
        )
    );
}
```

**Securitize:** 커밋 [e31353](https://bitbucket.org/securitize_dev/securitize-swap/commits/e313530ecd15a84471857089093c17817dfb8e79)에서 수정됨.

**Cyfrin:** 확인됨.



### Share tokens can be transferred

**Description:** `SecuritizeVault`는 예금자가 수취인과 동일해야 한다는 추가 제한(`_msgSender() == receiver`)을 사용하여 ERC4626의 `deposit` 함수를 구현합니다. 이는 볼트 토큰이 지정된 투자자에 의해서만 소유되도록 보장하기 위한 것이었지만, 공유 토큰이 ERC20 기능을 구현하고 민팅 후 자유롭게 전송될 수 있기 때문에 이 제한은 효과적이지 않습니다.
```solidity
SecuritizeVault.sol
205:     function deposit(uint256 assets, address receiver)
206:         public
207:         override(ERC4626Upgradeable, ISecuritizeVault)
208:         whenNotPaused
209:         onlyRole(OWNER_ROLE)
210:         returns (uint256)
211:     {
212:         require(_msgSender() == receiver, "Sender should be equal than receiver");//@audit-issue not meaningful because share tokens can be transferred
213:         return super.deposit(assets, receiver);
214:     }
215:
```

**Impact:** 이 제한은 잘못된 보안 의식을 제공하며, 토큰이 전송 가능한 상태로 유지되므로 의도한 접근 제어를 달성하지 못하면서 예금자가 다른 주소로 직접 입금하려는 정당한 사용 사례에 불필요한 마찰을 일으킵니다.

**Recommended Mitigation:**
1. 의미 있는 이점을 제공하지 않으므로 `_msgSender() == receiver` 확인을 제거하십시오.
2. 엄격한 소유권 제어가 필요한 경우 다음을 고려하십시오:
   - 공유 토큰에 전송 제한 구현
   - 양도 불가능한 토큰 표준 사용
   - 유효한 토큰 보유자에 대한 허용 목록 메커니즘 추가

**Securitize:** 인지됨.

**Cyfrin:** 인지됨.

\clearpage
## Informational


### Incorrect/misleading comments

**Description:** 코드의 기능을 정확하게 반영하지 않는 주석은 개발, 코드 검토 및 향후 유지 관리 중에 오해를 불러일으킬 수 있습니다.

```solidity
ISecuritizeNavProvider.sol
20: /**
21:  * @title ISecuritizeNavProvider
22:  * @dev Defines a common interface to get NAV (Native Asset Value) Rate to  //@audit-issue INFO incomplete comment
23:  */
```

아래의 경우 금액 환산 공식을 작성하는 것도 권장됩니다.
```solidity
ISecuritizeNavProvider.sol
38:
39:     /**
40:      * @dev Set rate. It is expressed with the same decimal numbers as stable coin//@audit-issue INFO it is called liquidity token in other places, not "stable coin"
41:      */
42:     function setRate(uint256 _rate) external;//@audit-info same decimal to liquidity token
43:
44:     /**
45:      * @dev The asset:liquidity rate.//@audit-info INFO liquidityAmount = assetAmount * rate() / assetDecimals
46:      * @return The asset:liquidity rate.
47:      */
48:     function rate() external view returns (uint256);
49: }
```

```solidity
SecuritizeRedemption.sol
64: /**
65:     * @dev Throws if called by any account other than the owner.//@audit-issue INFO incorrect comment
66:     */
67:     modifier navProviderNonZero(address _address) {
68:         require(_address != address(0) , "NAV rate provider address can not be zero");
69:         _;
70:     }
```

```solidity
SecuritizeRedemption.sol
72:     /**
73:     * @dev Throws if called by any account other than the owner.//@audit-issue INFO incorrect comment
74:     */
75:     function initialize(address _asset, address _navProvider) public onlyProxy initializer navProviderNonZero(_navProvider) {
76:         __BaseDSContract_init();
77:         asset = IERC20(_asset);
78:         navProvider = ISecuritizeNavProvider(_navProvider);
79:     }
```
```solidity
ISecuritizeRedemption.sol
60:     /**
61:      * @dev The NAV rate provider implementation.//@audit-issue INFO misleading comment, it's not necessarily an implementation.
62:      * @return The address of the NAV rate provider.
63:      */
64:     function navProvider() external view returns (ISecuritizeNavProvider);
65:
66:     /**
67:      * @dev Update the NAV rate provider implementation.//@audit-issue INFO misleading comment, it's not necessarily an implementation.
68:      * @param _navProvider The NAV rate provider implementation address//@audit-issue INFO misleading comment, it's not necessarily an implementation.
69:      */
70:     function updateNavProvider(address _navProvider) external;
```

```solidity
ISecuritizeRedemption.sol
42:     /**
43:      * @dev The liquidity provider implementation.//@audit-issue INFO misleading comment, it's not necessarily an implementation.
44:      * @return The address of the liquidity provider.
45:      */
46:     function liquidityProvider() external view returns (ILiquidityProvider);
47:
48:     /**
49:      * @dev Update the liquidity provider implementation.//@audit-issue INFO misleading comment, it's not necessarily an implementation.
50:      * @param _liquidityProvider The liquidity provider implementation address//@audit-issue INFO misleading comment, it's not necessarily an implementation.
51:      */
52:     function updateLiquidityProvider(address _liquidityProvider) external;
```

```solidity
CollateralLiquidityProvider.sol
56:     /**
57:      * @dev Throws if called by any account other than the owner.//@audit-issue INFO misleading comment, it's not the owner
58:      */
59:     modifier onlySecuritizeRedemption() {
60:         if (address(securitizeRedemption) != _msgSender()) {
61:             revert RedemptionUnauthorizedAccount(_msgSender());
62:         }
63:         _;
64:     }
```

```solidity
AllowanceLiquidityProvider.sol
55:     /**
56:      * @dev Throws if called by any account other than the owner.//@audit-issue INFO wrong comment
57:      */
58:     modifier onlySecuritizeRedemption() {
59:         if (address(securitizeRedemption) != _msgSender()) {
60:             revert RedemptionUnauthorizedAccount(_msgSender());
61:         }
62:         _;
63:     }
```

**Securitize:** 커밋 [8254e8](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/8254e84a8bd2579780cc7b3b1ffa4b9821bcd505)에서 수정됨.

**Cyfrin:** 확인됨.



### Naming improvements

**Description:** 기능적으로는 정확하지만, 특정 함수/수정자 이름은 그 목적과 구현을 더 잘 반영하기 위해 더 설명적일 수 있습니다.

```solidity
SecuritizeRedemption.sol
64:     /**
65:     * @dev Throws if called by any account other than the owner.
66:     */
67:     modifier navProviderNonZero(address _address) {//@audit-issue INFO better naming addressNonZero
68:         require(_address != address(0) , "NAV rate provider address can not be zero");
69:         _;
70:     }
```

**Securitize:** 커밋 [8254e8](https://bitbucket.org/securitize_dev/bc-redemption-sc/commits/8254e84a8bd2579780cc7b3b1ffa4b9821bcd505)에서 수정됨.

**Cyfrin:** 확인됨.


### Incorrect/misleading comments (2) - SecuritizeVault

**Description:** 코드의 기능을 정확하게 반영하지 않는 주석은 개발, 코드 검토 및 향후 유지 관리 중에 오해를 불러일으킬 수 있습니다.
```solidity
ISecuritizeVault.sol
72:     /**
73:      * @dev Grants the Redeemer role to an account. Emits a OwnerAdded event.//@audit-issue INFO Redeemer role is not defined in the interface. Must be Owner.
74:      *
75:      * @param _account The address to which the Owner role will be granted.
76:      */
77:     function addOwner(address _account) external;

86:     /**
87:      * @dev Checks if an account has the Redeemer role.//@audit-issue INFO Redeemer role is not defined in the interface. Must be Owner.
88:      *
89:      * @param _account The address to check for the Redeemer role.
90:      * @return bool Returns true if the account has the Redeemer role, false otherwise.
91:      */
92:     function isOwner(address _account) external view returns (bool);
```

```solidity
SecuritizeVault.sol
120:     /**
121:      * @dev Grants the owner role to an account. Emits a RedeemerAdded event.//@audit-issue INFO wrong comment, should be Owner
122:      * Owners can deposit and redeem. Emits OwnerAdded event
123:      *
124:      * @param _account The address to which the Redeemer role will be granted.
125:      */
126:     function addOwner(address _account) external addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
127:         grantRole(OWNER_ROLE, _account);
128:         emit OwnerAdded(_account);
129:     }
130:
131:     /**
132:      * @dev Revokes the Owner role from an account. Emits a OwnerRevoked event.
133:      *
134:      * @param _account The address from which the Redeemer role will be revoked.
135:      */
136:     function revokeOwner(address _account) external addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
137:         revokeRole(OWNER_ROLE, _account);
138:         emit OwnerRevoked(_account);
139:     }

```

```solidity
SecuritizeVault.sol
320:     /**
321:      * @dev Internal conversion function (from shares to assets) with support for rounding direction.
322:      *
323:      * This function overrides the default behavior in ERC4626Upgradeable
324:      * to ensure a conversion using NavProvider rate between assets and shares.
325:      *
326:      * min (xsSHARE / NAV, xsSHARE * tSHARE / tsSHARE) // <- xsSHARE & tASSSET / tsSHARE
327:      *
328:      * For more details, view ERC4626Upgradeable documentation.
329:      *
330:      * @param _shares The amount of shares to convert to assets.
331:      * @return uint256 The equivalent amount of assets.
332:      */
```

**Securitize:** 커밋 [172aed](https://bitbucket.org/securitize_dev/bc-securitize-vault-sc/commits/172aed24319d7bd2224cb3182f2150d90f43ac3a)에서 수정됨.

**Cyfrin:** 확인됨.



### Unused state variable in BaseSecuritizeSwap contract

**Description:** `BaseSecuritizeSwap` 컨트랙트에는 사용되지 않는 상태 변수 `dsToken`이 포함되어 있습니다. 이 변수는 선언되었지만 컨트랙트 또는 상속 컨트랙트 전체의 어떤 함수에서도 참조되지 않습니다. 이는 보안 위험을 초래하지는 않지만 배포 중 가스 비용을 불필요하게 증가시키고 코드베이스를 유지 관리하는 개발자에게 혼란을 줄 수 있습니다.

**Securitize:** 커밋 [ab46d0](https://bitbucket.org/securitize_dev/securitize-swap/commits/ab46d0e9be5184b0a4980a975f2afd68b33fa66b)에서 수정됨.

**Cyfrin:** 확인됨.


\clearpage
