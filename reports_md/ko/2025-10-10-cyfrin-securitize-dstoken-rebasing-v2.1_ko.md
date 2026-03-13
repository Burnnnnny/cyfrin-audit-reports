**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Stalin](https://x.com/0xStalin)

[Jorge](https://x.com/TamayoNft)

**Assisting Auditors**



---

# Findings
## Critical Risk


### Investors can steal tokens from other investors since `StandardToken::transferFrom` never checks spending approvals

**Description:** `StandardToken::transferFrom`이 지출 승인(spending approvals)을 확인하지 않기 때문에 투자자가 다른 투자자의 토큰을 훔칠 수 있습니다.

**Proof of Concept:** `test/dstoken-regulated.test.ts`에 다음 PoC를 추가하세요:
```javascript
  describe('TransferFrom', function () {
    it('Investors can steal tokens from other investors', async function () {
      // setup 2 investors
      const [investor, investor2] = await hre.ethers.getSigners();
      const { dsToken, registryService, rebasingProvider } = await loadFixture(deployDSTokenRegulated);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, investor, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, investor2, registryService);

      // give first investor some tokens
      await dsToken.issueTokens(investor, 500);

      const valueToTransfer = 100;
      const shares = await rebasingProvider.convertTokensToShares(valueToTransfer);
      const multiplier = await rebasingProvider.multiplier();

      // connect as second investor
      const dsTokenFromInvestor = await dsToken.connect(investor2);

      // use `transferFrom` to steal tokens from first investor, even though
      // first investor never approved second investor as a spender
      await expect(dsTokenFromInvestor.transferFrom(investor, investor2, valueToTransfer))
        .to.emit(dsToken, 'TxShares')
        .withArgs(investor.address, investor2.address, shares, multiplier);
    });
  });
```

실행: `npx hardhat test --grep "Investors can steal tokens from other investors"`.

**Recommended Mitigation:** `StandardToken::transferFrom`은 반드시 지출 승인을 강제해야 합니다.

**Securitize:** 커밋 [aefb895](https://github.com/securitize-io/dstoken/commit/aefb895e520d93ef0a8278ce3a7e88b2808478f5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Medium Risk


### Country updates is not reducing from the storage the prev country

**Description:** `ComplianceServiceRegulated.sol`의 `adjustInvestorCountsAfterCountryChange` 함수에는 국가 간 투자자 수를 이중으로 계산하는 회계 결함이 포함되어 있습니다. `RegistryService.setCountry()`를 통해 투자자가 국가를 변경할 때, 이 함수는 새 국가의 투자자 수를 올바르게 증가시키지만 이전 국가의 수를 감소시키지 못합니다:

```solidity
function setCountry(string calldata _id, string memory _country) public override onlyExchangeOrAbove investorExists(_id) returns (bool) {
        string memory prevCountry = getCountry(_id); <----------

        getComplianceService().adjustInvestorCountsAfterCountryChange(_id, _country, prevCountry);
        ...
    }
```

`adjustInvestorCountsAfterCountryChange`에서 `prevCountry`는 무시됩니다:
```solidity
function adjustInvestorCountsAfterCountryChange(
    string memory _id,
    string memory _country,
    string memory /*_prevCountry*/  // Previous country parameter is ignored
) public override onlyRegistry returns (bool) {
    if (getToken().balanceOfInvestor(_id) == 0) {
        return false;
    }

    // Only increases counts for new country
    adjustInvestorsCountsByCountry(_country, _id, CommonUtils.IncDec.Increase);
    // Missing: adjustInvestorsCountsByCountry(_prevCountry, _id, CommonUtils.IncDec.Decrease);

    return true;
}
```

그런 다음 `adjustInvestorsCountsByCountry`에서 여러 투자자 집계 변수가 이전 국가를 감소시키지 않고 증가합니다.

국가가 변경될 때마다 다음 카운터가 이전 국가를 감소시키지 않고 팽창합니다:
* `accreditedInvestorsCount`
* `usInvestorsCount`
* euRetailInvestorsCount[country]
* jpInvestorsCount

이 변수들은 최대 투자자 한도를 부과하는 데 사용되므로 중요합니다.

**Impact:** * 관할권별 투자자 한도의 잘못된 계산
* 부정확한 보고로 인한 잠재적 규제 위반

**Proof of Concept:** `file:compliance-service-regulated.ts`에서 다음 개념 증명을 실행하세요:
```typescript
 it('PoC: Country change double counting bug', async function() {
      const [wallet] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceConfigurationService, complianceService } = await loadFixture(deployDSTokenRegulated);

      // Setup
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.JAPAN, INVESTORS.Compliance.JP);

      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, wallet, registryService);
      await registryService.setAttribute(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, 2, 1, 0, ""); // Make accredited

      // Set initial country and issue tokens
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.USA);
      await dsToken.setCap(1000);
      await dsToken.issueTokens(wallet, 100);

      // Verify initial state: 1 US investor
      expect(await complianceService.getUSInvestorsCount()).to.equal(1);
      expect(await complianceService.getJPInvestorsCount()).to.equal(0);

      // Change country from USA to JAPAN
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.JAPAN);

      // VULNERABILITY: Both countries now count the same investor
      expect(await complianceService.getUSInvestorsCount()).to.equal(1); // Should be 0
      expect(await complianceService.getJPInvestorsCount()).to.equal(1);  // Should be 1
    });
```

**Recommended Mitigation:** `adjustInvestorCountsAfterCountryChange` 함수를 수정하여 새 국가의 수를 증가시키기 전에 이전 국가의 수를 적절히 감소시키도록 하세요:
```diff
function adjustInvestorCountsAfterCountryChange(
    string memory _id,
    string memory _country,
    string memory _prevCountry  // Use this parameter
) public override onlyRegistry returns (bool) {
    if (getToken().balanceOfInvestor(_id) == 0) {
        return false;
    }

+    if (bytes(_prevCountry).length > 0) {
+        adjustInvestorsCountsByCountry(_prevCountry, _id, CommonUtils.IncDec.Decrease);
+    }

    adjustInvestorsCountsByCountry(_country, _id, CommonUtils.IncDec.Increase);

    return true;
}
```

**Securitize:** 커밋 [2c76e2f](https://github.com/securitize-io/dstoken/commit/2c76e2f1862182930e5c8f7686d1ef21dba7dd4c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Incorrect EIP-712 `TXTYPE_HASH` in `SecuritizeSwap`

**Description:** `SecuritizeSwap::TXTYPE_HASH`가 EIP-712 서명 검증을 위해 인코딩되는 실제 메시지 구조와 일치하지 않습니다. 상수는 다음의 해시로 정의됩니다:
```solidity
"ExecutePreApprovedTransaction(string memory _senderInvestor, address _destination,address _executor,bytes _data, uint256[] memory _params)"
```

그러나 `SecuritizeSwap::doExecuteByInvestor`의 실제 `abi.encode` 호출은 다른 구조를 인코딩합니다:

```solidity
function doExecuteByInvestor(
        uint8 _securitizeHsmSigV,
        bytes32 _securitizeHsmSigR,
        bytes32 _securitizeHsmSigS,
        string memory _senderInvestorId,
        address _destination,
        bytes memory _data,
        address _executor,
        uint256[] memory _params
    ) internal {
        bytes32 txInputHash = keccak256(
            abi.encode(
                TXTYPE_HASH,
                _destination,
                _params[0], //value
                keccak256(_data),
                noncePerInvestor[_senderInvestorId],
                _executor,
                _params[1], //gasLimit
                keccak256(abi.encodePacked(_senderInvestorId))
            )
        );
        ...
    }
```

타입 문자열(type string)은 다음 순서로 5개의 매개변수를 선언합니다:

1) string _senderInvestor
2) address _destination
3) address _executor
4) bytes _data
5) uint256[] _params

그러나 실제 인코딩에는 7개의 매개변수가 있습니다:

1) address destination
2) uint256 value (from params[0])
3) bytes32 dataHash (keccak of _data)
4) uint256 nonce
5) address executor
6) uint256 gasLimit (from params[1])
7) bytes32 investorIdHash (keccak of _senderInvestor)

이는 예상되는 메시지 타입과 실제 인코딩된 매개변수 간의 중대한 불일치로, EIP-712 구조화 데이터 표준을 위반합니다.

또 다른 문제는 EIP-712 사양이 타입 해시 문자열에 `storage` 및 `memory`와 같은 저장소 한정자를 포함하지 않는다는 것입니다. 따라서 위의 불일치가 발생하지 않았더라도 현재 해시는 잘못된 타입 해시 문자열을 사용하여 생성되었으므로 여전히 올바르지 않습니다.

**Impact:** 구현이 EIP-712 표준을 위반하여 적절한 EIP-712 준수를 기대하는 지갑 및 서명 도구와 호환성 문제를 일으킬 수 있습니다.

**Recommended Mitigation:** `TXTYPE_HASH` 수정:
* 타입 해시 문자열에서 `memory` 키워드 제거
* 실제 인코딩된 매개변수와 일치하도록 타입 해시 문자열 변경
* 수정된 타입 해시 문자열을 사용하여 새로운 해시 생성

잠재적인 수정 사항은 다음과 같습니다:
```solidity
// keccak256("ExecutePreApprovedTransaction(address destination,uint256 value,bytes32 data,uint256 nonce,address executor,uint256 gasLimit,bytes32 investorIdHash)")
bytes32 constant TXTYPE_HASH = 0xf13da213cea16aa5bb997703966334f85e5aa4d2b25964a7191e3a7bc08b7690;
```

**Securitize:** `SecuritizeSwap`은 더 이상 사용되지 않으므로 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### Changing investor country to the same country inflates investor count erroneously triggering max investor errors

**Description:** 투자자의 국가를 동일한 국가로 변경하면 투자자 수가 부풀려집니다.

**Impact:** 투자자 수를 부풀리면 `ComplianceServiceRegulated::completeTransferCheck`에서 `MAX_INVESTORS_IN_CATEGORY` 오류가 잘못 트리거됩니다. 관리자가 `RegistryService::updateInvestor`를 호출하여 국가는 동일하게 유지하면서 다른 투자자 속성을 변경할 수 있고, 이로 인해 `ComplianceServiceRegulated::adjustInvestorCountsAfterCountryChange`가 호출되어 투자자 수가 부풀려지므로 관리자의 실수 없이도 발생할 수 있습니다.

**Proof of Concept:** `test/compliance-service-regulated.test.ts`에 다음 PoC를 추가하세요:
```typescript
    it('Changing to the same country inflates investor count', async function() {
      const [wallet] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceConfigurationService, complianceService } = await loadFixture(deployDSTokenRegulated);

      // Setup
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);

      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, wallet, registryService);
      await registryService.setAttribute(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, 2, 1, 0, ""); // Make accredited

      // Set initial country and issue tokens
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.USA);
      await dsToken.setCap(1000);
      await dsToken.issueTokens(wallet, 100);

      // Verify initial state: 1 US investor
      expect(await complianceService.getUSInvestorsCount()).to.equal(1);

      // Change country from USA to USA
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.USA);

      // bug: us investor count increased even though it is the same investor
      //      and their country hasn't actually changed
      expect(await complianceService.getUSInvestorsCount()).to.equal(2);
    });
```

실행: `npx hardhat test --grep "Changing to the same country inflates investor count"`

**Recommended Mitigation:** `ComplianceServiceRegulated::adjustInvestorCountsAfterCountryChange`는 `_country`와 `_prevCountry`가 동일한 경우 되돌리거나(revert) 단순히 아무것도 조정하지 않아야 합니다.

두 번째 옵션(아무것도 조정하지 않음)이 선호될 수 있습니다. 국가는 동일하게 유지하면서 다른 투자자 속성이 변경되는 `RegistryService::updateInvestor`에 대한 원래 호출로 인해 이 함수가 호출될 수 있기 때문입니다.

**Securitize:** 커밋 [86d4135](https://github.com/securitize-io/dstoken/commit/86d413548c5a41cc662681c1d7a67c8135d808e7)에서 `RegistryService::setCountry`(이 함수는 `ComplianceServiceRegulated::adjustInvestorCountsAfterCountryChange`를 호출함)가 국가가 동일할 경우 처리하지 않도록 변경하여 수정되었습니다.

다른 문제의 일부로 `ComplianceServiceRegulated::adjustInvestorCountsAfterCountryChange`도 이전 국가를 올바르게 감소시키도록 변경되었으므로, 동일한 국가로 이 함수를 직접 호출하면 감소 후 증가하여 결과적으로 투자자 수에 순 변화가 없어 올바르게 작동합니다.

**Cyfrin:** 검증되었습니다.


### Historical token issuances become subject to new lock-up requirements when users change residency

**Description:** `checkHoldUp` 함수와 `getComplianceTransferableTokens` 함수는 잠금 기간(lock periods)이 토큰 발행에 적용되는 방식에 결함을 포함하고 있습니다. 이 시스템은 각 토큰이 원래 발행되었을 때 적용 가능했던 잠금 기간 대신, 모든 과거 발행에 대해 하나의 현재 잠금 기간을 잘못 적용합니다.
이 취약점은 두 가지 주요 문제에서 비롯됩니다:

1. 잠금 기간은 발행 시점의 국가가 아니라 전송 시점의 투자자 현재 국가를 기준으로 계산됩니다(`checkHoldUp` 내부):
```solidity
function checkHoldUp(
    address[] memory _services,
    address _from,
    uint256 _value,
    bool _isUSLockPeriod,      // ← Determined at TRANSFER time
    bool _isPlatformWalletFrom
) internal view returns (bool) {
    uint256 lockPeriod;
    if (_isUSLockPeriod) {
        lockPeriod = getUSLockPeriod();     // ← Current US lock period
    } else {
        lockPeriod = getNonUSLockPeriod();  // ← Current Non-US lock period
    }

    // ← Applies same lock period to ALL issuances!
    return complianceService.getComplianceTransferableTokens(_from, block.timestamp, uint64(lockPeriod)) < _value;
}

```
2. 토큰이 언제 발행되었는지 또는 당시 적용 규정이 무엇이었는지에 관계없이 투자자의 모든 과거 발행에 대해 동일한 잠금 기간이 적용됩니다(`getComplianceTransferableTokens()` 내부):
```solidity
 function getComplianceTransferableTokens(
        address _who,
        uint256 _time,
        uint64 _lockTime
    ) public view override returns (uint256) {
        ...

        for (uint256 i = 0; i < investorIssuancesCount; i++) {
            uint256 issuanceTimestamp = issuancesTimestamps[investor][i];
             // Uses same _lockTime for ALL issuances <-------------
            if (uint256(_lockTime) > _time || issuanceTimestamp > (_time - uint256(_lockTime))) {
                uint256 tokens = getRebasingProvider().convertSharesToTokens(issuancesValues[investor][i]);
                totalLockedTokens = totalLockedTokens + tokens;
            }
        }

        //there may be more locked tokens than actual tokens, so the minimum between the two
        uint256 transferable = balanceOfInvestor - Math.min(totalLockedTokens, balanceOfInvestor);

        return transferable;
    }

```

**Impact:** * 투자자는 법적으로 잠금 해제되어야 하는 토큰을 전송할 수 없게 되어(일시적인 자금 동결), 유동성 문제가 발생할 수 있습니다.
* 잠금 기간의 잘못된 적용은 발행 당시 투자자 상태에 따라 특정 잠금 기간을 요구하는 증권 규정을 위반하는 결과를 초래할 수 있습니다.

**Proof of Concept:** 다음 실제 시나리오를 고려해 보세요:

1. 2024년 1월 1일: Alice는 독일 투자자입니다(비미국, 6개월 잠금 기간)
2. 2024년 6월 1일: Alice는 미국으로 이주하고 투자자 프로필을 업데이트합니다(미국, 12개월 잠금 기간)
3. 2024년 12월 1일: Alice가 토큰 전송을 시도합니다

Alice의 발행 내역:

발행 1: 2024년 1월 1일 - 1,000 토큰 (Alice가 독일인일 때 발행)
발행 2: 2024년 3월 1일 - 500 토큰 (Alice가 독일인일 때 발행)
발행 3: 2024년 8월 1일 - 200 토큰 (Alice가 미국 거주자일 때 발행)

Alice가 독일에 있을 때 발행된 토큰은 6개월 후에 잠금 해제되어야 하고, Alice가 미국에서 발행한 토큰은 12개월 동안 잠겨야 합니다. 그러나 계약은 발행 시점의 국가가 아닌 투자자의 현재 국가에 따른 잠금 기간을 사용하므로, Alice의 모든 토큰이 잘못 잠기게 됩니다.

올바른 동작:
2024년 12월 1일 전송 확인:
- 발행 1: 1월 1일 + 6개월 = 2024년 7월 1일     잠금 해제됨 (독일 규칙에 따라 발행)
- 발행 2: 3월 1일 + 6개월 = 2024년 9월 1일     잠금 해제됨 (독일 규칙에 따라 발행)
- 발행 3: 8월 1일 + 12개월 = 2025년 8월 1일    잠금 상태 (미국 규칙에 따라 발행)

총 양도 가능: 1,500 토큰
총 잠김: 200 토큰

현재 동작:
2024년 12월 1일 전송 확인:
- 발행 1: 1월 1일 + 12개월 = 2025년 1월 1일    잠금 상태
- 발행 2: 3월 1일 + 12개월 = 2025년 3월 1일    잠금 상태
- 발행 3: 8월 1일 + 12개월 = 2025년 8월 1일    잠금 상태


**Recommended Mitigation:** 발행 시점의 잠금 기간을 저장하는 것을 고려하세요:

```diff
// Add new mapping to ComplianceServiceDataStore.sol
mapping(string => mapping(uint256 => uint256)) issuanceLockPeriods;

function createIssuanceInformation(
    string memory _investor,
    uint256 _shares,
    uint256 _issuanceTime
) internal returns (bool) {
    ...

    issuancesValues[_investor][issuancesCount] = _shares;
    issuancesTimestamps[_investor][issuancesCount] = _issuanceTime;
+  issuanceLockPeriods[_investor][issuancesCount] = lockPeriod; // <----------Store lock period
    issuancesCounters[_investor] = issuancesCount + 1;

    return true;
}
```
그런 다음 `getComplianceTransferableTokens`에서:

```diff
function getComplianceTransferableTokens(
    address _who,
    uint256 _time,
    uint64 _lockTime // This parameter can be removed
) public view override returns (uint256) {
    ...

    for (uint256 i = 0; i < investorIssuancesCount; i++) {
        uint256 issuanceTimestamp = issuancesTimestamps[investor][i];
+       uint256 applicableLockPeriod = issuanceLockPeriods[investor][i]; // Use stored lock period

-         if (uint256(_lockTime) > _time || issuanceTimestamp > (_time - uint256(_lockTime))) {
+        if (_time < issuanceTimestamp + applicableLockPeriod) {
            uint256 tokens = getRebasingProvider().convertSharesToTokens(issuancesValues[investor][i]);
            totalLockedTokens = totalLockedTokens + tokens;
        }
    }

    ...
}
```

**Securitize:** 인정함(Acknowledged).


### No way to revert `setInvestorLiquidateOnly`

**Description:** `InvestorLockManagerBase.sol`의 `setInvestorLiquidateOnly` 함수에는 청산 전용(liquidate-only) 모드가 한 번 활성화되면 비활성화하는 것을 방지하는 논리 오류가 포함되어 있습니다. 이 함수는 투자자가 이미 청산 전용 모드에 있는지 확인하고 그렇다면 되돌리는 require 문을 포함하고 있어 상태를 다시 false로 전환하는 것이 불가능합니다.

``` solidity
function setInvestorLiquidateOnly(string memory _investorId, bool _enabled) public onlyTransferAgentOrAbove returns (bool) {
    require(!investorsLiquidateOnly[_investorId], "Investor is already in liquidate only mode");
    investorsLiquidateOnly[_investorId] = _enabled;
    emit InvestorLiquidateOnlySet(_investorId, _enabled);
    return true;
}
```

**Impact:** 투자자가 청산 전용 모드로 설정되면 이 상태를 비활성화할 방법이 없습니다.


**Recommended Mitigation:** 청산 전용 상태 전환을 허용하도록 require 문을 제거하세요:

```diff
function setInvestorLiquidateOnly(string memory _investorId, bool _enabled) public onlyTransferAgentOrAbove returns (bool) {
-   require(!investorsLiquidateOnly[_investorId], "Investor is already in liquidate only mode");
    investorsLiquidateOnly[_investorId] = _enabled;
    emit InvestorLiquidateOnlySet(_investorId, _enabled);
    return true;
}
```

**Securitize:** 커밋 [74a6675](https://github.com/securitize-io/dstoken/commit/74a66753c15a2cdddc41a29ae8d736711b141939)에서 현재 상태가 입력 상태와 같을 경우 되돌리도록 변경하여 수정되었습니다; 이를 통해 상태를 on/off로 전환할 수 있습니다.

**Cyfrin:** 검증되었습니다.


### `euRetailInvestorsCount` is susceptible to underflow when transferring tokens between retail and qualified investors of the same country on the EU region

**Description:** 이 문제의 근본 원인은 잔액이 있는 자격을 갖춘(qualified) EU 투자자가 소매(retail) 투자자로 다운그레이드될 때 `euRetailInvestorsCount`를 증가시키지 않는다는 것입니다.
이는 시스템을 일관성 없는 상태로 만듭니다. 자격을 갖춘 투자자는 자격이 있을 때 토큰을 받았을 수 있으며, 소매 투자자로 다운그레이드된 후 자신의 잔액 전액을 다른 투자자에게 전송할 수 있습니다. 이렇게 되면 소매 투자자가 더 이상 보유자가 아니기 때문에 `euRetailInvestorsCount`가 감소하게 됩니다.
- 이 문제는 자격을 갖춘 투자자가 잔액이 있는 상태에서 소매 투자자로 다운그레이드되었지만, `euRetailInvestorsCount`가 증가하지 않았기 때문에 발생합니다.

```solidity
    function adjustInvestorsCountsByCountry(
        ...
    ) internal {
        ...
//@audit-info => euRetailInvestorsCount are not modified for Qualified EU Investors
        } else if (countryCompliance == EU && !getRegistryService().isQualifiedInvestor(_id)) {
            if(_increase == CommonUtils.IncDec.Increase) {
                euRetailInvestorsCount[_country]++;
            }
            else {
                euRetailInvestorsCount[_country]--;
            }
        }
        ...
    }
```

PoC에서 시연된 바와 같이, 프랑스의 `euRetailInvestorsCount`는 다음과 같은 시나리오에서 엉망이 됩니다:
1. 프랑스 소매 EU 투자자가 잔액의 일부(부분 전송)를 자격을 갖춘(비소매) 프랑스 투자자에게 전송합니다.
2. 자격을 갖춘 프랑스 투자자가 소매 투자자가 되고 잔액 전액을 다시 프랑스 소매 투자자 1에게 전송합니다.
    - 여기서 `euRetailInvestorsCount`가 엉망이 됩니다.
3. 소매 프랑스 투자자 1이 잔액 전액을 외부로 전송하려고 시도하지만 프랑스의 소매 카운터가 이미 0이기 때문에 언더플로우가 발생합니다.

**Impact:** 언더플로우 상태에 도달할 위험 외에도, `euRetailInvestorsCount`는 추적해야 할 실제 투자자 수와 달라지게 됩니다.

**Proof of Concept:** `dstoken-regulated.test.ts` 테스트 파일에 다음 PoC를 추가하세요:
```js
    it.only('underflow on euRetailInvestorsCount PoC', async function () {
      const [investor, investor2, usInvestor] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceService, complianceConfigurationService } = await loadFixture(deployDSTokenRegulated);
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.FRANCE, INVESTORS.Compliance.EU);
      await complianceConfigurationService.setEURetailInvestorsLimit(10);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, investor, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, investor2, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, usInvestor, registryService);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.FRANCE);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.Country.FRANCE);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, INVESTORS.Country.USA);

      //@audit-info => Set investor2 as a qualified investor!
      await registryService.setAttribute(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, 4, 1, 0, 'abcde');

      await dsToken.issueTokens(investor, 500);
      const valueToTransfer = 100;

      //@audit-info => a retail EU investor transfers to a qualified (non-retail) EU investor
      const dsTokenFromInvestor = await dsToken.connect(investor);
      await dsTokenFromInvestor.transfer(investor2, valueToTransfer);
      expect(await complianceService.getEURetailInvestorsCount(INVESTORS.Country.FRANCE)).equal(1);

      //@audit-info => Set investor2 as retail investor - non-qualified!
      await registryService.setAttribute(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, 4, 0, 0, 'abcde');
      const dsTokenFromInvestor2 = await dsToken.connect(investor2);
      //@audit-info => The old qualified investor, currently retail, transfers all its balance to another retail EU investor
      await dsTokenFromInvestor2.transfer(investor, valueToTransfer);
      expect(await complianceService.getEURetailInvestorsCount(INVESTORS.Country.FRANCE)).equal(0);

      //@audit-info => investor1 tries to transfer all of its balance to an investor on another country and reverts bcs of underflow on euRetailInvestorsCount
      await expect(dsTokenFromInvestor.transfer(usInvestor, 500)).to.be.reverted;
    });
```

**Recommended Mitigation:** 자격을 갖춘 EU 투자자를 소매 EU 투자자로 전환할 때, 해당 투자자가 investorBalance를 가지고 있다면 `euRetailInvestorsCount`를 증가시키는 것을 고려하세요.

**Securitize:** 인정함(Acknowledged); 자격을 갖춘 상태와 자격을 갖추지 못한 상태 간의 변경을 설명하는 것은 현실적이지 않습니다. 이 로직을 구현하면 투자자 및 속성 업데이트에 복잡성이 추가되며, 어쨌든 운영적으로 해결할 수 있습니다.


### `ComplianceServiceRegulated` applies double locking to investor transferable tokens leading to DoS of transfers

**Description:** `ComplianceServiceRegulated::getComplianceTransferableTokens()`는 투자자의 양도 가능한 토큰에 이중 잠금(double locking)을 적용하여 실제보다 더 많은 토큰이 잠긴 것으로 간주하게 만들어, 투자자가 토큰을 전송할 수 있어야 할 때 DoS(서비스 거부)가 발생하게 합니다.

근본적인 문제는 `ComplianceServiceRegulated::getComplianceTransferableTokens()`에서 사용되는 investorBalance가 실제 investorBalance가 아니라 양도 가능한 토큰이라는 것입니다. 이로 인해 `InvestorLockManager`에서 시행되는 잠금에 규정 준수(Compliance) 계약에서 시행되는 잠금이 누적됩니다.

예를 들어, Investor1은 WalletA와 WalletB를 소유하고 있습니다.
- WalletA는 500개의 양도 가능한 토큰을 가지고 있습니다.
- WalletB는 1000개의 잠긴 토큰을 가지고 있습니다.
전체적으로 Investor1은 1500개의 토큰을 가지고 있으며, 그 중 1000개는 잠겨 있고 500개는 양도 가능합니다.

`getComplianceTransferableTokens()`에서 일어나는 일은 다음과 같습니다:
1. `Investor.getTransferableTokens()`가 Investor1이 500개의 잠금 해제된 토큰(총 1500개 - 잠긴 1000개)을 가지고 있다고 결정하기 때문에 balanceOfInvestor는 500으로 설정됩니다.
2. `getComplianceTransferableTokens()`는 issuancesCounters를 반복하고 1000개의 `totalLockedTokens`를 계산합니다.
3. `getComplianceTransferableTokens()`는 `transferable`을 0으로 계산합니다. `balanceOfInvestor (500)` - `min(totalLocked, balanceOfInvestor` => `500 - min(1000,500)` => `500 - 500 => 0`.

이로 인해 규정 준수 계약에 의해 이중 계산이 적용되어 투자자의 잠긴 잔액이 양도 가능한 토큰보다 커지기 때문에 투자자는 500개의 양도 가능한 토큰을 전송하지 못하게 됩니다(DoS).

```solidity
    function getComplianceTransferableTokens(
        address _who,
        uint256 _time,
        uint64 _lockTime
    ) public view override returns (uint256) {
        ...

//@audit-info => amount of transferable tokens that are not locked anymore (500)
        uint256 balanceOfInvestor = getLockManager().getTransferableTokens(_who, _time);

        ...

        for (uint256 i = 0; i < investorIssuancesCount; i++) {
            uint256 issuanceTimestamp = issuancesTimestamps[investor][i];

            if (uint256(_lockTime) > _time || issuanceTimestamp > (_time - uint256(_lockTime))) {
                uint256 tokens = getRebasingProvider().convertSharesToTokens(issuancesValues[investor][i]);
//@audit-info => amount of locked tokens (1000)
                totalLockedTokens = totalLockedTokens + tokens;
            }
        }

//@audit-issue => transferable is calculated as 0 even though the investor has 500 transferable tokens
        uint256 transferable = balanceOfInvestor - Math.min(totalLockedTokens, balanceOfInvestor);

        return transferable;
    }
```

**Note: 이 문제는 LockManager가 `LockManager` 계약의 인스턴스인 경우에도 발생합니다. 이 계약에서 잠금은 지갑 수준에서 적용되지만, 그럼에도 불구하고 이중 잠금 계산은 실제 양도 가능한 토큰의 계산을 엉망으로 만듭니다. 개발자는 `LockManager` 계약이 더 이상 사용되지 않을 것이라고 밝혔으므로, 이 문제를 해결하기 위한 권장 완화 조치는 `InvestorLockManager` 계약에만 집중하는 것입니다.**

**Impact:** 투자자는 실제 보유한 것보다 적은 양도 가능한 토큰을 갖게 되어 전송이 완전히 차단(DoS)될 수 있습니다.

**Proof of Concept:** `dstoken-regulated.test.ts`에 다음 PoC를 추가하세요:

```js
    it.only('Double Locking Count leads to DoS even when using InvestorLockManager', async function () {
      const [owner, wallet1, wallet2, walletOtherInvestor] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceConfigurationService, lockManager, complianceService } = await loadFixture(deployDSTokenRegulated);

      const usLockPeriod = 1000;
      await complianceConfigurationService.setUSLockPeriod(usLockPeriod);
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);

      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, '');
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.USA);
      await registryService.addWallet(wallet1, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);//wallet1 => INVESTOR_ID_1
      await registryService.addWallet(wallet2, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);//wallet2 -> INVESTOR_ID_1

      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, '');
      await registryService.addWallet(walletOtherInvestor, INVESTORS.INVESTOR_ID.INVESTOR_ID_2);//walletOtherInvestor -> INVESTOR_ID_2

      const currentTime = await time.latest();

      await dsToken.issueTokensCustom(wallet1, 500, currentTime, 500, 'TEST', currentTime + 1_000); //lock for INVESTOR_ID_1

      //@audit-info => forward time so 500 issued tokens to Wallet1 are transferable
      await time.increaseTo(currentTime + 1000);

      //@audit-info => Any of the investor's wallet has 500 transferable tokens bcs the Investor as a whole has 500 transferable tokens!
      expect(await lockManager.getTransferableTokens(wallet1, await time.latest())).equal(500);
      expect(await lockManager.getTransferableTokens(wallet2, await time.latest())).equal(500);

      //@audit-info => complianceTransferableTokens for any of the wallets is 500 before the new issuance
      expect(await complianceService.getComplianceTransferableTokens(wallet1, time.latest(), usLockPeriod)).equals(500);
      expect(await complianceService.getComplianceTransferableTokens(wallet2, time.latest(), usLockPeriod)).equals(500);

      //@audit-info => issue 1000 locked tokens to Wallet2
      await dsToken.issueTokensCustom(wallet2, 1000, await time.latest(), 1000, 'TEST', await time.latest() + 1_000); //lock for INVESTOR_ID_1

      //@audit-info => Any of the investor's wallet has 500 transferable tokens bcs the Investor as a whole has 500 transferable tokens!
      expect(await lockManager.getTransferableTokens(wallet1, await time.latest())).equal(500);
      expect(await lockManager.getTransferableTokens(wallet2, await time.latest())).equal(500);

//@audit-issue => Both wallets have 0 transferable tokens because of the double locking count
      expect(await complianceService.getComplianceTransferableTokens(wallet1, time.latest(), usLockPeriod)).equals(0);
      expect(await complianceService.getComplianceTransferableTokens(wallet2, time.latest(), usLockPeriod)).equals(0);

//@audit-issue => Investor as a whole (can't transfer from any of his wallets) is DoS from transferring even though it has transferable tokens because of the double locking count
      const dsTokenWallet1 = await dsToken.connect(wallet1);
      await expect(dsTokenWallet1.transfer(walletOtherInvestor, 500)).revertedWith('Under lock-up');

      const dsTokenWallet2 = await dsToken.connect(wallet2);
      await expect(dsTokenWallet2.transfer(walletOtherInvestor, 500)).revertedWith('Under lock-up');
    });
```

**Recommended Mitigation:** 기본 `balanceOfInvestor`를 설정하기 위해 `InvestorLockManager::getTransferableTokens()`를 쿼리하는 대신, `token::balanceOfInvestor()`를 쿼리하세요.
이렇게 하면 이중 잠금 계산이 제거되고 투자자의 실제 잔액을 규정 준수 잠금의 경우 양도 가능한 토큰을 결정하는 기준으로 사용하게 됩니다.
- 규정 준수 잠금은 없지만 잔액의 일부가 LockManager에 잠겨 있는 경우, 해당 잠금 계산은 `InvestorLockManager` 자체에서 처리되어 투자자 수준에서 적용됩니다.

```diff
function getComplianceTransferableTokens(
        address _who,
        uint256 _time,
        uint64 _lockTime
    ) public view override returns (uint256) {
        require(_time != 0, "Time must be greater than zero");
        string memory investor = getRegistryService().getInvestor(_who);

-       uint256 balanceOfInvestor = getLockManager().getTransferableTokens(_who, _time);
+      uint256 balanceOfInvestor = getToken().balanceOfInvestor(investor);

        ...
}
```

**Securitize:** 인정함(Acknowledged); 이것은 의도된 대로 작동합니다. 투자자: 레벨 잠금은 투자자의 양도 가능한 토큰의 현재 금액에 적용됩니다. 이는 잔액에서 아직 잠금 기간(lockup period) 하에 있는 발행된 토큰의 총액을 뺀 금액입니다.

커밋 [7bbe0b1](https://github.com/securitize-io/dstoken/commit/7bbe0b1f5fbbd14ec61da0ea4b44248ade54b905)에서 주석을 업데이트했으며 `LockManager` 및 `ComplianceServiceNotRegulated`는 더 이상 사용되지 않으므로 제거되었습니다.


### Token value locks done as part of issuance restrict subsequently transferred unlocked tokens after negative rebasing

**Description:** 토큰 발행이 발생할 때, 해당 토큰 발행의 일부로 두 가지 유형의 잠금이 제정될 수 있습니다:
* 주식 가치 기반 잠금:
```solidity
TokenIssuer::issueTokens
-- DSToken::issueTokensWithMultipleLocks
---- TokenLibrary::issueTokensCustom
------ ComplianceService::validateIssuance
-------- ComplianceServiceRegulated::recordIssuance -> createIssuanceInformation
```
* 토큰 가치 기반 잠금:
```solidity
TokenIssuer::issueTokens
-- DSToken::issueTokensWithMultipleLocks
---- TokenLibrary::issueTokensCustom
------ InvestorLockManager::addManualLockRecord -> createLock -> createLockForInvestor
```

**Impact:** 다음과 같은 시나리오에서:
* Investor1은 해당 발행에 대해 주식 기반 및 토큰 가치 기반 잠금이 모두 있는 1000 토큰을 발행받습니다.
* 네거티브 리베이스(Negative rebasing)가 발생합니다.
* Investor2는 1000개의 잠금 해제된 토큰을 Investor1에게 전송합니다.

Investor1의 원래 토큰 발행에서 발생한 토큰 가치 잠금은 Investor2가 이후에 Investor1에게 전송한 잠금 해제된 토큰의 전송을 제한합니다.

**Proof of Concept:** `test/token-issuer.test.ts`에 PoC를 추가하세요:
```typescript
  it('Token value locks done as part of issuance restrict subsequently transferred tokens after negative rebasing', async function() {
    const [ owner, investor1, investor2 ] = await hre.ethers.getSigners();
    const { dsToken, tokenIssuer, lockManager, rebasingProvider, registryService } = await loadFixture(deployDSTokenRegulated);

    // Register investors
    await registryService.registerInvestor('INVESTOR1', '');
    await registryService.setCountry('INVESTOR1', 'US');
    await registryService.addWallet(investor1.address, 'INVESTOR1');

    await registryService.registerInvestor('INVESTOR2', '');
    await registryService.setCountry('INVESTOR2', 'US');
    await registryService.addWallet(investor2.address, 'INVESTOR2');

    const currentTime = await time.latest();
    const lockRelease = currentTime + 100000;

    // Issue 1000 tokens to Investor1 with 1000 manually locked
    await tokenIssuer.issueTokens(
      'INVESTOR1',
      investor1.address,
      [ 1000, 1 ],
      '',
      [ 1000 ], // Lock ALL tokens
      [ lockRelease ],
      'INVESTOR1',
      'US',
      [0, 0, 0],
      [0, 0, 0]
    );

    // Issue 2000 tokens to Investor2 (no locks)
    await tokenIssuer.issueTokens(
      'INVESTOR2',
      investor2.address,
      [ 2000, 1 ],
      '',
      [],
      [],
      'INVESTOR2',
      'US',
      [0, 0, 0],
      [0, 0, 0]
    );

    // Verify initial state
    expect(await dsToken.balanceOf(investor1.address)).to.equal(1000);
    expect(await lockManager.getTransferableTokens(investor1.address, currentTime)).to.equal(0);

    // Get current multiplier and halve it
    const currentMultiplier = await rebasingProvider.multiplier();
    const halfMultiplier = currentMultiplier / BigInt(2);
    await rebasingProvider.setMultiplier(halfMultiplier);

    // After rebasing, Investor1 has 500 tokens but 1000 still locked
    expect(await dsToken.balanceOf(investor1.address)).to.equal(500);
    const transferableAfterRebasing = await lockManager.getTransferableTokens(investor1.address, currentTime);
    expect(transferableAfterRebasing).to.equal(0); // Over-locked

    // Investor2 transfers 1000 tokens to Investor1
    await dsToken.connect(investor2).transfer(investor1.address, 1000);

    // Final state
    const finalBalance = await dsToken.balanceOf(investor1.address);
    const finalTransferable = await lockManager.getTransferableTokens(investor1.address, currentTime);

    expect(finalBalance).to.equal(1500); // 500 from rebasing + 1000 transfer

    // Investor1 has 500 tokens from their own issuance and 1000 tokens
    // they received as a transfer with no associated lock
    // But the token value lock placed as part of their original issuance
    // affects subsequent transfers which had no locks attached
    //
    // Unlocked tokens received via transfer from Investor2 are partially
    // locked by Investor1's token-value based lock from Investor1's original
    // token issuance
    expect(finalTransferable).to.equal(500);
  });
```

**Recommended Mitigation:** `InvestorLockManager` 및 `LockManager`의 잠금 시스템을 토큰이 아닌 주식(shares) 단위로 모든 잠금 금액을 저장하도록 변환하세요. 또는 현재 동작을 수용하도록 문제를 단순히 인정할 수도 있습니다.

**Securitize:** 인정함(Acknowledged); 잠금은 항상 토큰으로 계산되며, 네거티브 리베이스 시나리오의 경우 발생할 수 있습니다. 하지만 운영 절차를 통해 관리할 수도 있습니다.


### Investors transferring all their balances among their wallets or self-transferring on the same wallet causes to incorrectly decrement the investor counters causing DoS for other investors transfers

**Description:** `ComplianceServiceRegulated::recordTransfer()`는 수신자의 투자자가 새로운 경우, 또는 발신자의 투자자가 잔액 전액을 전송하는 경우 투자자 카운터를 조정하는 역할을 합니다. 하지만 투자자 카운터가 잘못 감소되는 엣지 케이스가 있습니다.
1. 첫 번째 엣지 케이스는 기존 투자자가 한 지갑에서 다른 지갑으로 잔액 전액을 전송하는 경우입니다.
2. 투자자가 동일한 지갑에서 자기 자신에게 전송을 실행하는 경우입니다.

문제의 근본 원인은 `ComplianceServiceRegulated::recordTransfer()`에 있습니다. 이 함수는 발신자와 수신자 투자자가 동일한지 확인하지 않으며, 수신자가 동일한 투자자인지에 관계없이 발신자의 투자자가 잔액 전액을 전송하는지 바로 확인합니다.
```solidity
    function compareInvestorBalance(
        address _who,
        uint256 _value,
        uint256 _compareTo
    ) internal view returns (bool) {
       //@audit => true when the sender is transferring all of its balance
        return (_value != 0 && getToken().balanceOfInvestor(getRegistryService().getInvestor(_who)) == _compareTo);
    }

    function recordTransfer(
        address _from,
        address _to,
        uint256 _value
    ) internal override returns (bool) {
//@audit-issue => Not checking if sender is the same as the receiver

        if (compareInvestorBalance(_to, _value, 0)) {
            adjustTransferCounts(_to, CommonUtils.IncDec.Increase);
        }
```solidity
        if (compareInvestorBalance(_from, _value, _value)) {
//@audit=> When from's investor transfers all of its balance, investor counters are decremented.
            adjustTotalInvestorsCounts(_from, CommonUtils.IncDec.Decrease);
        }

        ...
    }

```

**Impact:**
- 투자자 카운터가 감소하지 않아야 할 때 감소하여 다른 투자자에 대한 전송 DoS가 발생합니다.
- 추가적인 영향으로 투자자 카운터가 시스템의 실제 투자자 수를 올바르게 추적하지 못하기 때문에 투자자 제한을 우회할 수 있다는 것입니다.

**Proof of Concept:** PoC를 `dstoken-regulated.test.ts`에 추가하세요. 첫 번째 PoC는 동일한 투자자가 소유한 지갑 간의 전송으로 인해 발생하는 문제를 보여줍니다:
```javascript
    it.only('Mess up counters via transfers among wallets owned by the same investor', async function () {
      const [owner, wallet1Investor1, wallet2Investor1, walletInvestor2, walletInvestor3] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceConfigurationService, lockManager, complianceService } = await loadFixture(deployDSTokenRegulated);

      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);
      await complianceConfigurationService.setUSInvestorsLimit(10);

      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, '');
      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, '');
      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3, '');

      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3, INVESTORS.Country.USA);

      await registryService.addWallet(wallet1Investor1, INVESTORS.INVESTOR_ID.US_INVESTOR_ID);//wallet1Investor1 => US_INVESTOR_ID
      await registryService.addWallet(wallet2Investor1, INVESTORS.INVESTOR_ID.US_INVESTOR_ID);//wallet2Investor1 -> US_INVESTOR_ID

      await registryService.addWallet(walletInvestor2, INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2);//walletInvestor2 -> US_INVESTOR_ID_2
      await registryService.addWallet(walletInvestor3, INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3);//walletInvestor3 -> US_INVESTOR_ID_3

      const currentTime = await time.latest();

      //@audit-info => Issue unlocked tokens to investors

      await dsToken.issueTokensCustom(wallet1Investor1, 10_000, currentTime, 0, 'TEST', 0);
      await dsToken.issueTokensCustom(walletInvestor2, 10_000, currentTime, 0, 'TEST', 0);
      await dsToken.issueTokensCustom(walletInvestor3, 10_000, currentTime, 0, 'TEST', 0);

      expect(await complianceService.getUSInvestorsCount()).equal(3);
      expect(await complianceService.getTotalInvestorsCount()).equal(3);

      //@audit-info => investor1 transfers all the balance among his wallets
      const dsTokenWallet1Investor1 = await dsToken.connect(wallet1Investor1);
      const dsTokenWallet2Investor1 = await dsToken.connect(wallet2Investor1);

      await dsTokenWallet1Investor1.transfer(wallet2Investor1, 10_000);
      await dsTokenWallet2Investor1.transfer(wallet1Investor1, 10_000);
      await dsTokenWallet1Investor1.transfer(wallet2Investor1, 10_000);

      //@audit-issue => Investor counters have been brought down to 0 while the 3 investors still have balances!
      expect(await complianceService.getUSInvestorsCount()).equal(0);
      expect(await complianceService.getTotalInvestorsCount()).equal(0);

      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID)).equal(10_000);
      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2)).equal(10_000);
      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3)).equal(10_000);

      //@audit-info => Will revert because investor counter has been brought down to 0
      const dsTokenWalletInvestor2 = await dsToken.connect(walletInvestor2);
      await expect(dsTokenWalletInvestor2.transfer(walletInvestor3, 10_000)).to.be.revertedWith('Not enough investors');
    });
```

두 번째 PoC는 동일한 지갑에서의 자기 전송으로 인한 문제를 보여줍니다:
```javascript
    it.only('self transfer from the same wallet messes up investor counters', async function () {
      const [owner, wallet1Investor1, wallet2Investor1, walletInvestor2, walletInvestor3] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceConfigurationService, lockManager, complianceService } = await loadFixture(deployDSTokenRegulated);

      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);
      await complianceConfigurationService.setUSInvestorsLimit(10);

      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, '');
      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, '');
      await registryService.registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3, '');

      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3, INVESTORS.Country.USA);

      await registryService.addWallet(wallet1Investor1, INVESTORS.INVESTOR_ID.US_INVESTOR_ID);//wallet1Investor1 => US_INVESTOR_ID

      await registryService.addWallet(walletInvestor2, INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2);//walletInvestor2 -> US_INVESTOR_ID_2
      await registryService.addWallet(walletInvestor3, INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3);//walletInvestor3 -> US_INVESTOR_ID_3

      const currentTime = await time.latest();

      //@audit-info => Issue unlocked tokens to investors

      await dsToken.issueTokensCustom(wallet1Investor1, 10_000, currentTime, 0, 'TEST', 0);
      await dsToken.issueTokensCustom(walletInvestor2, 10_000, currentTime, 0, 'TEST', 0);
      await dsToken.issueTokensCustom(walletInvestor3, 10_000, currentTime, 0, 'TEST', 0);

      expect(await complianceService.getUSInvestorsCount()).equal(3);
      expect(await complianceService.getTotalInvestorsCount()).equal(3);

      //@audit-info => investor1 transfers all the balance among his wallets
      const dsTokenWallet1Investor1 = await dsToken.connect(wallet1Investor1);

      await dsTokenWallet1Investor1.transfer(wallet1Investor1, 10_000);
      await dsTokenWallet1Investor1.transfer(wallet1Investor1, 10_000);
      await dsTokenWallet1Investor1.transfer(wallet1Investor1, 10_000);

      //@audit-issue => Investor counters have been brought down to 0 while the 3 investors still have balances!
      expect(await complianceService.getUSInvestorsCount()).equal(0);
      expect(await complianceService.getTotalInvestorsCount()).equal(0);

      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID)).equal(10_000);
      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2)).equal(10_000);
      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_3)).equal(10_000);

      //@audit-info => Will revert because investor counter has been brought down to 0
      const dsTokenWalletInvestor2 = await dsToken.connect(walletInvestor2);
      await expect(dsTokenWalletInvestor2.transfer(walletInvestor3, 10_000)).to.be.revertedWith('Not enough investors');
    });
```

**Recommended Mitigation:** `from` 지갑과 `receiver` 지갑의 투자자가 동일한 경우 투자자 카운터의 확인 및 감소를 건너뛰는 것을 고려하세요:
```diff
    function recordTransfer(
        address _from,
        address _to,
        uint256 _value
    ) internal override returns (bool) {
        if (compareInvestorBalance(_to, _value, 0)) {
            adjustTransferCounts(_to, CommonUtils.IncDec.Increase);
        }

-       if (compareInvestorBalance(_from, _value, _value)) {
-           adjustTotalInvestorsCounts(_from, CommonUtils.IncDec.Decrease);
-       }

+       string memory investorFrom = getRegistryService().getInvestor(_from);
+       string memory investorTo = getRegistryService().getInvestor(_to);

+       if(!CommonUtils.isEqualString(investorFrom, investorTo)) {
+           if (compareInvestorBalance(_from, _value, _value)) {
+               adjustTotalInvestorsCounts(_from, CommonUtils.IncDec.Decrease);
+           }
+       }

        cleanupInvestorIssuances(_from);
        cleanupInvestorIssuances(_to);
        return true;
    }
```

이 수정 사항의 더 가스 최적화된 버전은 `compareInvestorBalance` 및 `cleanupInvestorIssuances` 함수를 리팩토링하여 지갑 주소를 수신하고 내부적으로 각 함수에서 투자자 ID를 가져오는 대신 투자자 ID를 입력으로 받도록 하는 것입니다.

**Securitize:** 커밋 [6b94242](https://github.com/securitize-io/dstoken/commit/6b94242f913d5c5b94e5dd50ed7cf84723f2a8a1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Low Risk


### Protocol will leak value to users due to rounding in `RebasingLibrary`

**Description:** 프로토콜은 항상 프로토콜에 유리하고 사용자에게 불리하도록 반올림해야 합니다. `RebasingLibrary`는 사용자에게 가치를 유출하는 "가장 가까운 정수로 반올림(rounding to nearest)" 방법을 사용합니다:
```solidity
// In convertTokensToShares
return (_tokens * DECIMALS_FACTOR + _rebasingMultiplier / 2) / _rebasingMultiplier;
                                 // ^^^^^^^^^^^^^^^^^^^^
                                 // This rounds to nearest

// In convertSharesToTokens
return (_shares * _rebasingMultiplier + DECIMALS_FACTOR / 2) / DECIMALS_FACTOR;
                                     // ^^^^^^^^^^^^^^^^^^^^
                                     // This also rounds to nearest
```

**Impact:** 반올림은 두 방향 모두에서 사용자에게 도움이 됩니다:
* 예치 시: 사용자가 1개의 추가 지분(share)을 얻을 수 있습니다.
* 출금 시: 사용자가 1개의 추가 토큰을 얻을 수 있습니다.

수천 건의 거래에 걸쳐 이러한 wei 수준의 손실이 누적됩니다.

**Proof of Concept:** 먼저 [Foundry 통합을 추가하세요](https://getfoundry.sh/config/hardhat/#adding-foundry-to-a-hardhat-project). 그런 다음 `test/RebasingRoundingTest.t.sol`에 새 PoC 컨트랙트를 추가하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

import "forge-std/Test.sol";
import "../contracts/rebasing/RebasingLibrary.sol";

contract RebasingRoundingTest is Test {
    using RebasingLibrary for *;

    function testRoundingInconsistency() public {
        // Setup: multiplier = 1.5e18 (1.5x rebasing)
        uint256 multiplier = 1.5e18;
        uint8 decimals = 18;

        // Test 1: Convert 1 token to shares and back
        uint256 initialTokens = 1e18;

        // Convert tokens to shares
        uint256 shares = RebasingLibrary.convertTokensToShares(
            initialTokens,
            multiplier,
            decimals
        );

        // Convert shares back to tokens
        uint256 finalTokens = RebasingLibrary.convertSharesToTokens(
            shares,
            multiplier,
            decimals
        );

        console.log("Initial tokens:", initialTokens);
        console.log("Shares received:", shares);
        console.log("Final tokens:", finalTokens);

        // This should fail if there's inconsistency
        assertEq(initialTokens, finalTokens, "Rounding inconsistency detected");
    }

    function testAccumulatedRoundingErrors() public {
        uint256 multiplier = 1.7e18; // Non-round multiplier
        uint8 decimals = 18;

        uint256 totalSharesIssued;
        uint256 totalTokensIn;

        // Simulate 1000 small deposits
        for(uint i = 0; i < 1000; i++) {
            // Each user deposits a small odd amount
            uint256 tokens = 1e15 + i; // 0.001 tokens + i wei

            uint256 shares = RebasingLibrary.convertTokensToShares(
                tokens,
                multiplier,
                decimals
            );

            totalTokensIn += tokens;
            totalSharesIssued += shares;
        }

        // Now convert total shares back to tokens
        uint256 totalTokensOut = RebasingLibrary.convertSharesToTokens(
            totalSharesIssued,
            multiplier,
            decimals
        );

        console.log("Total tokens in:", totalTokensIn);
        console.log("Total tokens out:", totalTokensOut);
        console.log("Difference:", totalTokensOut > totalTokensIn ?
            totalTokensOut - totalTokensIn : totalTokensIn - totalTokensOut);

        // Check if protocol lost tokens
        if(totalTokensOut > totalTokensIn) {
            console.log("Protocol LOST tokens due to rounding!");
        }
    }
}
```

실행: `forge test --match-contract RebasingRoundingTest -vv`

**Recommended Mitigation:** 반올림은 항상 프로토콜에 유리해야 합니다. 일반적으로 사용자에게 토큰을 발행할 때는 내림(rounding down)하고, 사용자에게 수수료를 부과할 때는 올림(rounding up)하는 것이 유리합니다. 하지만 다음을 고려하세요:

* `convertTokensToShares`는 발행 중(`TokenLibrary::issueTokensCustom`)에 사용되며, 여기서 내림은 사용자에게 약간 적은 지분을 제공하여 프로토콜에 유리합니다.
* `convertTokensToShares`는 소각 중(`TokenLibrary::burn`)에도 사용되며, 여기서 내림은 실제로 사용자의 지분을 약간 덜 소각하게 하여 사용자에게 유리합니다.

따라서 이것은 까다롭습니다. 최신 프로토콜이 수행하는 작업은 다음과 같습니다:

* 코드 전체에서 사용되는 `convertTokensToShares`와 같은 함수에서 [입력 반올림 매개변수](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/Math.sol#L13-L18)를 추가합니다.
* `convertTokensToShares`의 모든 호출자는 추가 입력 매개변수를 통해 명시적인 반올림 방향을 지정해야 합니다.
* `convertTokensToShares`의 모든 호출자에는 지정된 반올림 방향이 올바른 이유에 대한 로직을 설명하는 주석이 호출 위에 있어야 합니다.

이것은 개발자가 모든 호출의 맥락에서 반올림 방향을 신중하게 고려하도록 강제하며, 이는 따르기에 좋은 관행이며 반올림 버그를 제거하는 데 도움이 될 수 있습니다.

`convertSharesToTokens`에 대해서도 동일하게 수행하세요. 모든 호출자가 반올림 방향을 지정하고 해당 컨텍스트에서 반올림 방향이 올바른 이유를 설명하는 주석을 모든 호출 앞에 넣으세요.

**Securitize:** 인정함(Acknowledged); 현재 구현은 악용 가능한 것으로 보이지 않습니다. 실제로는 소수점으로 인해 발생하지 않는 경우가 매우 많으며, 이를 수정하려고 하면 다른 잠재적인 문제가 발생할 수 있습니다. 지금은 그대로 두지만 커밋 [8b74550](https://github.com/securitize-io/dstoken/commit/8b7455093963caefacc8694ee87d0725c83451c6)에 설명 주석과 추가 테스트를 추가했습니다.


### Multiplication could overflow in `RebasingLibrary` for tokens with greater than 18 decimals

**Description:** `RebasingLibrary`에는 18자리 이상의 소수를 가진 토큰에 대한 특별한 처리가 포함되어 있습니다:
```solidity
// convertTokensToShares
        } else {
            uint256 scale = 10**(_tokenDecimals - 18);
            return (_tokens * DECIMALS_FACTOR + (_rebasingMultiplier * scale) / 2) / (_rebasingMultiplier * scale);
        }

// convertSharesToTokens
        } else {
            uint256 scale = 10**(_tokenDecimals - 18);
            return (_shares * _rebasingMultiplier * scale + DECIMALS_FACTOR / 2) / DECIMALS_FACTOR;
        }
```

**Impact:** 높은 소수점 값을 가진 토큰을 사용할 때, 여기서 곱셈이 오버플로우되어 서비스 거부가 발생할 수 있습니다.

**Recommended Mitigation:** OpenZeppelin의 [Math::mulDiv](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/Math.sol#L204)를 사용하세요. 수정된 코드는 또한 L-1에 대한 제안된 수정을 통합합니다:
```solidity
pragma solidity 0.8.22;

import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

library RebasingLibrary {
    uint256 private constant DECIMALS_FACTOR = 1e18;

    function convertTokensToShares(
        uint256 _tokens,
        uint256 _rebasingMultiplier,
        uint8 _tokenDecimals
    ) internal pure returns (uint256 shares) {
        require(_rebasingMultiplier > 0, "Invalid rebasing multiplier");

        if (_tokenDecimals == 18) {
            return Math.mulDiv(_tokens, DECIMALS_FACTOR, _rebasingMultiplier);
        } else if (_tokenDecimals < 18) {
            uint256 scale = 10**(18 - _tokenDecimals);
            // tokens * scale * DECIMALS_FACTOR / multiplier
            return Math.mulDiv(_tokens * scale, DECIMALS_FACTOR, _rebasingMultiplier);
        } else {
            uint256 scale = 10**(_tokenDecimals - 18);
            // tokens * DECIMALS_FACTOR / (multiplier * scale)
            return Math.mulDiv(_tokens, DECIMALS_FACTOR, _rebasingMultiplier * scale);
        }
    }

    function convertSharesToTokens(
        uint256 _shares,
        uint256 _rebasingMultiplier,
        uint8 _tokenDecimals
    ) internal pure returns (uint256 tokens) {
        require(_rebasingMultiplier > 0, "Invalid rebasing multiplier");

        if (_tokenDecimals == 18) {
            return Math.mulDiv(_shares, _rebasingMultiplier, DECIMALS_FACTOR);
        } else if (_tokenDecimals < 18) {
            uint256 scale = 10**(18 - _tokenDecimals);
            // (shares * multiplier / DECIMALS_FACTOR) / scale
            return Math.mulDiv(_shares, _rebasingMultiplier, DECIMALS_FACTOR * scale);
        } else {
            uint256 scale = 10**(_tokenDecimals - 18);
            // shares * multiplier * scale / DECIMALS_FACTOR
            return Math.mulDiv(_shares * scale, _rebasingMultiplier, DECIMALS_FACTOR);
        }
    }
}
```

**Securitize:** 커밋 [9b81e76](https://github.com/securitize-io/dstoken/commit/9b81e76c6d75f8e550f719a27c344cb337377d79)에서 18자리 이상의 소수를 가진 토큰에 대해 되돌리는(revert) 것으로 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Asymmetry enforcement between `TokenIssuer::registerInvestor`, `WalletRegistrar::registerWallet` and `SecuritizeSwap::_registerNewInvestor`

**Description:** `TokenIssuer::registerInvestor`에서, 사용자가 이미 투자자가 아닌 경우 등록되고 3가지 특정 속성을 설정해야 합니다:
```solidity
if (!getRegistryService().isInvestor(_id)) {
    getRegistryService().registerInvestor(_id, _collisionHash);
    getRegistryService().setCountry(_id, _country);

    if (_attributeValues.length > 0) {
        require(_attributeValues.length == 3, "Wrong length of parameters");
        getRegistryService().setAttribute(_id, KYC_APPROVED, _attributeValues[0], _attributeExpirations[0], "");
        getRegistryService().setAttribute(_id, ACCREDITED, _attributeValues[1], _attributeExpirations[1], "");
        getRegistryService().setAttribute(_id, QUALIFIED, _attributeValues[2], _attributeExpirations[2], "");
    }
```

그러나 `WalletRegistrar::registerWallet` 및 `SecuritizeSwap::_registerNewInvestor`에서는 사용자가 이미 투자자가 아닌 경우 등록되지만 동일한 속성 로직이 없습니다. 대신 더 일반적이어서 기존에 존재하는 것을 덮어쓰고 KYC_APPROVED, ACCREDITED 또는 QUALIFIED 속성의 존재를 강제하지 않는 것처럼 보입니다:
```solidity
if (!registryService.isInvestor(_id)) {
    registryService.registerInvestor(_id, _collisionHash);
    registryService.setCountry(_id, _country);
}

for (uint256 i = 0; i < _wallets.length; i++) {
    if (registryService.isWallet(_wallets[i])) {
        require(CommonUtils.isEqualString(registryService.getInvestor(_wallets[i]), _id), "Wallet belongs to a different investor");
    } else {
        registryService.addWallet(_wallets[i], _id);
    }
}

for (uint256 i = 0; i < _attributeIds.length; i++) {
    registryService.setAttribute(_id, _attributeIds[i], _attributeValues[i], _attributeExpirations[i], "");
}
```

**Impact:** `WalletRegistrar::registerWallet` 또는 `SecuritizeSwap::_registerNewInvestor`를 통하면 투자자는 필수 속성 없이 등록될 수 있습니다.

**Recommended Mitigation:** 투자자 등록 프로세스를 조화시켜 중복 코드를 제거하고 동일한 요구 사항을 강제하세요.

**Securitize:** 커밋 [72e54d2](https://github.com/securitize-io/dstoken/commit/72e54d2863f87cba2eda1535a4fc3f8902839ddc)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Protocol will leak value to users due to rounding in `SecuritizeSwap::buy`

**Description:** 프로토콜은 항상 프로토콜에 유리하고 사용자에게 불리하도록 반올림해야 합니다. `SecuritizeSwap.sol`의 `buy` 함수는 필요한 스테이블 코인을 사용자에게 유리하게 반올림하고 있습니다:

```solidity
 function buy(uint256 _dsTokenAmount, uint256 _maxStableCoinAmount) public override whenNotPaused {
        ...
        uint256 stableCoinAmount = calculateStableCoinAmount(_dsTokenAmount); <----
        dsToken.issueTokensCustom(msg.sender, _dsTokenAmount, block.timestamp, 0, "", 0);

        emit Buy(msg.sender, _dsTokenAmount, stableCoinAmount, navProvider.rate());
    }

  function calculateStableCoinAmount(uint256 _dsTokenAmount) public view returns (uint256) {
        return _dsTokenAmount * navProvider.rate() / (10 ** ERC20(address(dsToken)).decimals()); // rounding down
    }

```

이 내림(rounding down)으로 인해 이론적으로 사용자는 무료 DSToken을 발행할 수 있습니다. 그러나 DSToken은 소수점이 2자리뿐이므로 불가능합니다.

**Impact:** 반올림은 두 방향 모두에서 사용자에게 도움이 됩니다:

* 예치 시: 사용자가 1개의 추가 지분(share)을 얻을 수 있습니다.
* 출금 시: 사용자가 1개의 추가 토큰을 얻을 수 있습니다.

수천 건의 거래에 걸쳐 이러한 wei 수준의 손실이 누적됩니다.

**Recommended Mitigation:** 반올림은 항상 프로토콜에 유리해야 합니다. 반올림 방향 지정을 지원하는 [`Math::mulDiv`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/Math.sol#L280-L282)를 사용하세요.

**Securitize:** `SecuritizeSwap`은 더 이상 사용되지 않으므로 삭제되었습니다.

**Cyfrin:** 검증되었습니다.


### Missing `checkWalletsForList` in `issueTokensWithNoCompliance`

**Description:** `DSToken.sol`의 `issueTokensWithNoCompliance` 함수는 다른 모든 발행 함수와 달리 토큰 발행 후 `checkWalletsForList(address(0), _to)`를 호출하지 않습니다. 이로 인해 관리 작업에 사용되는 계약의 내부 열거형 목록에 지갑이 추가되지 않습니다.

**Impact:** * `walletCount()`가 잘못된 값을 반환하고 `getWalletAt()`이 규정 준수 없이(no-compliance) 발행을 통해 토큰을 받은 지갑을 검색할 수 없습니다.
* 대량 작업 실패: 지갑 열거에 의존하는 관리 기능은 이러한 토큰 보유자를 놓치게 됩니다.


**Recommended Mitigation:** `issueTokensWithNoCompliance`에 `checkWalletsForList`를 추가하세요.

**Securitize:** `DSToken:: issueTokensWithNoCompliance`가 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### Transferring all the investor balance from a non-us investor to a new us investor allows to bypass the `usInvestorLimit`

**Description:** `ComplianceServiceLibrary.completeTransferCheck()`는 전송 수신자가 새로운 투자자이고 투자자 수가 한도에 도달한 경우 투자자 한도(지역 및 글로벌)가 우회되는 것을 방지하기 위해 일련의 유효성 검사를 실행합니다.

`usInvestorsLimit`를 검증할 때 발신자의 지역도 미국인지 여부는 검증되지 않습니다.
- 즉, 비미국 투자자의 투자자 잔액 전액을 전송하면 `usInvestorsLimit`를 우회하게 됩니다.

`ComplianceServiceRegulated.completeTransferCheck()`의 확인을 우회하기 때문에(아래 스니펫 설명 참조), `usInvestorLimit`는 `ComplianceServiceRegulated.adjustInvestorsCountsByCountry()`에서 증가하게 됩니다. 이 함수는 `to` 수신자가 새로운 투자자이기 때문에 투자자 카운터를 증가시키기 위해 `ComplianceServiceRegulated.recordTransfer()`에서 호출됩니다(실행 흐름은 두 번째 스니펫 설명 참조).

```solidity
//completeTransferCheck()//
        } else if (toRegion == US) {
            ...

            uint256 usInvestorsLimit = getUSInvestorsLimit(_services);
            if (
                usInvestorsLimit != 0 &&
//@audit-info => Transferring the full balance turns out the entire conditional to evaluate to false
//@audit => A single false on any of the individual conditions causes all the conditions to evaluate to false because all of them are &&
                _args.fromInvestorBalance > _args.value &&
                ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE]).getUSInvestorsCount() >= usInvestorsLimit &&
                isNewInvestor(toInvestorBalance)
            ) {
                return (40, MAX_INVESTORS_IN_CATEGORY);
            }

            ...
        }
```

```solidity
//ComplianceServiceRegulated.sol//

    function recordTransfer(
        address _from,
        address _to,
        uint256 _value
    ) internal override returns (bool) {
//@audit => The `to` recipient is a new investor, therefore, calls adjustTransferCounts to increase the counters
//@audit => `to` investor is a us investor, therefore, the usInvestorLimit will be incremented
        if (compareInvestorBalance(_to, _value, 0)) {
            adjustTransferCounts(_to, CommonUtils.IncDec.Increase);
        }

//@audit => `from` investor is not a us investor
//@audit-info => The usInvestorLimit won't be decremented here, it will be decremented the counter of the from investor's region
        if (compareInvestorBalance(_from, _value, _value)) {
            adjustTotalInvestorsCounts(_from, CommonUtils.IncDec.Decrease);
        }

        cleanupInvestorIssuances(_from);
        cleanupInvestorIssuances(_to);
        return true;
    }

```

**Impact:** * 엄격한 투자자 제한을 요구하는 미국 증권 규정을 우회할 수 있습니다.
* 이는 국가별 전송 제한 위반으로 이어질 수 있습니다.

**Proof of Concept:** `dstoken-regulated.test.ts`에서 다음 개념 증명을 실행하세요:
```typescript
describe('US Investor Limit Bypass Vulnerability POC', function() {
    it('Should demonstrate that US investor limit can be bypassed with full transfer from any country', async function() {
      const [usInvestor1, usInvestor2, nonUsInvestor] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceConfigurationService, complianceService } = await loadFixture(deployDSTokenRegulatedWithRebasingAndEighteenDecimal);

      // Setup: US limit = 1, disable other limits
      await complianceConfigurationService.setAll(
        [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 150, 1, 1, 0], // No other limits, short lock period
        [false, false, false, false, false] // Disable all compliance checks initially
      );
      await complianceConfigurationService.setUSInvestorsLimit(1);// set one investor limit for us investor
      await complianceConfigurationService.setBlockFlowbackEndTime(1); // Disable flowback restriction
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.GERMANY, INVESTORS.Compliance.EU);

      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, usInvestor1.address, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, usInvestor2.address, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.GERMANY_INVESTOR_ID, nonUsInvestor.address, registryService);

      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.GERMANY_INVESTOR_ID, INVESTORS.Country.GERMANY);

      // Issue tokens to non-US investor first
      await dsToken.issueTokens(nonUsInvestor.address, 1000);
      console.log('Non-US investor count:', await complianceService.getUSInvestorsCount());

      // Issue tokens to first US investor (reaches limit)
      await dsToken.issueTokens(usInvestor1.address, 1000);
      console.log('After US investor count:', await complianceService.getUSInvestorsCount());
      expect(await complianceService.getUSInvestorsCount()).to.equal(1);

      // Direct issuance to second US investor should fail
      await expect(dsToken.issueTokens(usInvestor2.address, 100))
        .to.be.revertedWith('Max investors in category');

      // VULNERABILITY: Non-US investor can bypass US limit with full transfer
      // This should be blocked but isn't due to missing country check in US logic
      await dsToken.connect(nonUsInvestor).transfer(usInvestor2.address, 1000);

      // Verify bypass succeeded - usInvestor2 now has tokens despite limit being reached
      expect(await dsToken.balanceOf(usInvestor2.address)).to.equal(1000);
      expect(await dsToken.balanceOf(nonUsInvestor.address)).to.equal(0);
      expect(await complianceService.getUSInvestorsCount()).to.equal(2);
    });
  });
```

**Recommended Mitigation:** 발신자(`from`) 투자자가 미국 투자자가 아닌 경우, 또는 미국 투자자이지만 잔액 전액을 전송하지 않는 경우 true로 평가되도록 조건문을 업데이트하는 것을 고려하세요; 만약 잔액 전액을 전송하는 경우라면 `from` 투자자는 `usInvestorLimit`에서 감소될 것입니다.
```diff
        } else if (toRegion == US) {
            ...

            uint256 usInvestorsLimit = getUSInvestorsLimit(_services);
            if (
                usInvestorsLimit != 0 &&
-               _args.fromInvestorBalance > _args.value &&
+               (_args.fromRegion != US || _args.fromInvestorBalance > _args.value) &&
                ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE]).getUSInvestorsCount() >= usInvestorsLimit &&
                isNewInvestor(toInvestorBalance)
            ) {
                return (40, MAX_INVESTORS_IN_CATEGORY);
            }

            ...
        }
```

**Securitize:** 커밋 [5d52ceb](https://github.com/securitize-io/dstoken/commit/5d52ceb4434a89b9547cf22b8d9c8404fa4f5116)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TrustService::changeEntityOwner` can overwrite existing `_newOwner` record, breaking 1-1 relationship between owners and addresses

**Description:** `TrustService::changeEntityOwner`는 기존 `_newOwner` 레코드를 덮어써서 소유자와 주소 간의 1:1 관계를 깨뜨릴 수 있습니다.

**Proof of Concept:** `test/trust-service.test.ts`에 PoC를 추가하세요:
```typescript
    it('overwrite existing owner breaks 1-1 relationship', async function() {
      const [owner, firstOwner, secondOwner] = await hre.ethers.getSigners();
      const { trustService } = await loadFixture(deployDSTokenRegulated);

      // Setup: Create two entities with different owners
      await trustService.setRole(firstOwner, DSConstants.roles.ISSUER);
      await trustService.setRole(secondOwner, DSConstants.roles.ISSUER);

      const entity1 = "Entity1";
      const entity2 = "Entity2";

      const trustServiceFromFirst = await trustService.connect(firstOwner);
      await trustServiceFromFirst.addEntity(entity1, firstOwner);

      const trustServiceFromSecond = await trustService.connect(secondOwner);
      await trustServiceFromSecond.addEntity(entity2, secondOwner);

      // Verify initial state
      expect(await trustService.getEntityByOwner(firstOwner)).equal(entity1);
      expect(await trustService.getEntityByOwner(secondOwner)).equal(entity2);

      // Change entity1 owner from firstOwner to secondOwner
      await trustService.changeEntityOwner(entity1, firstOwner, secondOwner);

      // Bug: secondOwner now owns both entities in the forward mapping
      // but reverse mapping shows only entity1
      expect(await trustService.getEntityByOwner(secondOwner)).equal(entity1);
      // entity2 is now orphaned - no way to find its owner through getEntityByOwner

      // firstOwner has no entity in reverse mapping
      expect(await trustService.getEntityByOwner(firstOwner)).equal("");
    });
```

실행: `npx hardhat test --grep "overwrite existing owner"`.

**Recommended Mitigation:** `changeEntityOwner` 함수에 `onlyNewEntityOwner(_newOwner)` 제어자를 추가하세요.

**Securitize:** 커밋 [6cd6cca](https://github.com/securitize-io/dstoken/commit/6cd6ccae7201082e53befd6364aff8a1f57397f7)에서 수정되었습니다; 관련 스토리지 슬롯은 더 이상 사용되지 않으며 관련 함수는 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### `TrustService::removeRole` doesn't delete already owned entities so address which lost role can still manage existing entities

**Description:** `TrustService::removeRole`은 엔티티를 삭제하지 않으므로 역할을 잃은 주소가 기존 엔티티를 여전히 관리할 수 있습니다.

**Proof of Concept:** `test/trust-service.test.ts`에 PoC를 추가하세요:
```typescript
    it('Should demonstrate orphaned entity relationships after role removal', async function() {
      const [owner, entityOwner, operator, resource] = await hre.ethers.getSigners();
      const { trustService } = await loadFixture(deployDSTokenRegulated);

      // Step 1: Give entityOwner ISSUER role
      await trustService.setRole(entityOwner, DSConstants.roles.ISSUER);
      expect(await trustService.getRole(entityOwner)).equal(DSConstants.roles.ISSUER);

      // Step 2: Create an entity owned by entityOwner
      const entityName = "TestEntity";
      const trustServiceFromEntityOwner = await trustService.connect(entityOwner);
      await trustServiceFromEntityOwner.addEntity(entityName, entityOwner);

      // Verify entity ownership
      expect(await trustService.getEntityByOwner(entityOwner)).equal(entityName);

      // Step 3: Add operator and resource to the entity
      await trustServiceFromEntityOwner.addOperator(entityName, operator);
      await trustServiceFromEntityOwner.addResource(entityName, resource);

      // Verify operator and resource are linked to entity
      expect(await trustService.getEntityByOperator(operator)).equal(entityName);
      expect(await trustService.getEntityByResource(resource)).equal(entityName);

      // Step 4: Remove entityOwner's ISSUER role
      await trustService.removeRole(entityOwner);
      expect(await trustService.getRole(entityOwner)).equal(DSConstants.roles.NONE);

      // Step 5: Demonstrate the bug - entity relationships still exist
      // These should ideally be cleaned up but they're not:
      expect(await trustService.getEntityByOwner(entityOwner)).equal(entityName);  // Still owns entity!
      expect(await trustService.getEntityByOperator(operator)).equal(entityName);   // Still linked!
      expect(await trustService.getEntityByResource(resource)).equal(entityName);  // Still linked!

      // Step 6: Show the security issue - entityOwner can still manage the entity
      // even without any role
      await expect(
        trustServiceFromEntityOwner.addOperator(entityName, hre.ethers.Wallet.createRandom())
      ).to.not.be.reverted;  // This should fail but doesn't!

      // The onlyEntityOwnerOrAbove modifier still passes because:
      // - entityOwner has NONE role (not MASTER/ISSUER)
      // - But ownersEntities[entityOwner] still equals entityName
      // - So the check passes even though they shouldn't have access
    });
```

**Recommended Mitigation:** 주소가 역할을 잃으면 엔티티 소유권 매핑을 지워 이전에 소유한 엔티티를 삭제하세요.

**Securitize:** 커밋 [6cd6cca](https://github.com/securitize-io/dstoken/commit/6cd6ccae7201082e53befd6364aff8a1f57397f7)에서 수정되었습니다; 관련 스토리지 슬롯은 더 이상 사용되지 않으며 관련 함수는 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### No way for users to invalidate nonce used to sign

**Description:** 트랜잭션 서명 후, 사용자는 마음을 바꿔 서명을 무효화하고 싶을 수 있습니다. 이 경우 사용자는 자신의 논스(nonce)를 무효화하는 함수를 호출할 수 있어야 합니다.

`SecuritizeSwap::executePreApprovedTransaction`과 같은 곳에서는 사용자가 현재 논스를 증가시켜(따라서 무효화) 서명을 취소할 수 있는 함수가 없습니다.

**Impact:** 사용자는 현재 논스를 무효화할 수 없으므로 서명이 사용되기 전에 취소할 수 없습니다.

**Recommended Mitigation:** 사용자가 현재 논스를 단순히 증가시킬 수 있는 함수를 호출할 수 있도록 만드세요. 또한 서명을 사용하는 `MultiSigWallet`, `TransactionRelayer` 및 기타 장소를 확인하세요.

**Securitize:** 인정함(Acknowledged).


### No deadline for user signatures

**Description:** 사용자가 서명한 서명에는 항상 [만료 또는 타임스탬프 기한(deadline)](https://dacian.me/signature-replay-attacks#heading-no-expiration)이 있어야 하며, 해당 시간이 지나면 서명이 더 이상 유효하지 않아야 합니다.

사용자가 트랜잭션에 서명할 때 미래의 만료 기한을 사용하여 서명해야 하며, 해당 서명을 처리하는 함수는 기한이 경과한 경우 되돌려야(revert) 합니다.

**Recommended Mitigation:** `SecuritizeSwap::executePreApprovedTransaction`과 같은 함수는 사용자 제공 기한을 입력으로 받아야 하며, `doExecuteByInvestor`는 만료된 경우 되돌려야 하고 그렇지 않으면 다른 매개변수와 함께 해시를 계산할 때 포함해야 합니다. 또한 서명을 사용하는 `MultiSigWallet`, `TransactionRelayer` 및 기타 장소를 확인하세요.

`SecuritizeSwap::buy`와 같은 함수는 서명을 사용하지 않더라도 기한을 추가하는 것을 고려할 수 있지만, 해당 함수에는 슬리피지(slippage) 매개변수가 있으므로 이점은 크지 않습니다.

**Securitize:** 영향을 받는 대부분의 계약은 구식이므로 삭제되었으며, 남아 있는 유일한 계약인 `TransactionRelayer`는 그대로 둡니다.

**Cyfrin:** 검증되었습니다.


### Stale Issuance Records After `Burn` and `Seize` Operations

**Description:** `ComplianceServiceRegulated.sol`의 `recordBurn()` 및 `recordSeize()` 함수는 토큰이 소각되거나 압류될 때 발행 레코드를 정리하지 않습니다. 이로 인해 시스템에 오래된(stale) 발행 레코드가 유지되어 새로 발행된 토큰에 대한 잘못된 잠금 계산으로 이어질 수 있습니다. `cleanupInvestorIssuances()` 함수는 잠금 기간을 기준으로 만료된 발행 레코드만 제거하며, 완전히 소각되거나 압류된 토큰에 대한 레코드는 제거하지 않습니다.

```solidity
   function recordBurn(address _who, uint256 _value) internal override returns (bool) {

        if (compareInvestorBalance(_who, _value, _value)) {
            adjustTotalInvestorsCounts(_who, CommonUtils.IncDec.Decrease);
        }
        return true;
    }
```

**Impact:** 더 이상 존재하지 않는 토큰에 대한 발행 레코드가 시스템에 남아 있으며, 토큰 소각/압류 작업 후 투자자가 새 토큰을 받을 때 `getComplianceTransferableTokens()`의 잠금 계산에 이전에 소각/압류된 토큰의 오래된 발행 레코드가 포함됩니다.

**Proof of Concept:**
 1. 투자자가 타임스탬프 T1에 100 토큰을 받음 - 발행 레코드 생성: {value: 100, timestamp: T1}
 2. 100 토큰 모두 burn() 함수를 통해 소각됨
       - recordBurn()이 호출되지만 발행 레코드를 정리하지 않음
       - 오래된 발행 레코드 지속됨: {value: 100, timestamp: T1}
3. 투자자가 타임스탬프 T2(잠금 기간 이후)에 50개의 새 토큰을 받음
    - 새 발행 레코드 생성: {value: 50, timestamp: T2}
 4. 양도 가능한 토큰을 계산할 때, `getComplianceTransferableTokens()`는 두 레코드를 모두 사용합니다:
   - 오래된 레코드: T1의 100 토큰 (존재하지 않아야 함)
   - 새 레코드: T2의 50 토큰
5. T1 이후 잠금 기간이 만료되지 않은 경우, 150개의 모든 토큰이 잠긴 것으로 간주됩니다.
6. 결과: 오래된 잠금 계산으로 인해 투자자는 50개의 새 토큰 중 아무것도 전송할 수 없습니다.


**Recommended Mitigation:** 투자자가 완전히 소각하거나 압류할 때 이전 발행을 삭제하세요.

**Securitize:** 인정함(Acknowledged); 일반적으로 1개의 발행 레코드만 있고 소각 또는 압류가 모든 토큰에 대한 것이 아닌 한, 소각 또는 압류가 발생할 때 어떤 발행 레코드를 삭제해야 하는지 결정하는 것은 불가능합니다. 또한 설명된 시나리오가 처음부터 발생하는 것을 효과적으로 방지하는 몇 가지 규정 준수 규칙이 있습니다.


### Malicious investor can register wallets that belong to other investors

**Description:** `RegistryService.sol`의 `addWalletByInvestor` 함수는 등록된 모든 투자자가 현재 `investorsWallets` 매핑에 없는 지갑 주소의 소유권을 주장할 수 있도록 허용합니다. 이는 공격자가 합법적인 지갑 등록 트랜잭션을 선행 실행(front-run)하여 지갑 소유권을 가로챌 수 있는 치명적인 취약점을 생성합니다:

```solidity
function addWalletByInvestor(address _address) public override newWallet(_address) returns (bool) {
        require(!getWalletManager().isSpecialWallet(_address), "Wallet has special role");

        string memory owner = getInvestor(msg.sender);
        require(isInvestor(owner), "Unknown investor");

        investorsWallets[_address] = Wallet(owner, msg.sender, msg.sender);
        investors[owner].walletCount++;

        emit DSRegistryServiceWalletAdded(_address, owner, msg.sender);

        return true;
    }

```
이 함수는 호출자가 등록된 투자자이고 지갑이 특수 지갑이 아닌지만 확인하지만, 호출자가 등록하려는 지갑 주소를 실제로 제어하는지는 확인하지 않습니다.

이로 인해 악의적인 등록 투자자는 모든 `addWallet`, `updateInvestor`, `addWalletByInvestor` 호출을 선행 실행(또는 등록될 지갑을 이미 알고 있는 경우)하여 지갑을 자신으로 설정할 수 있으며, 해당 함수를 DoS 공격하고 다른 투자자를 위한 토큰 발행을 자신에게 리디렉션할 수 있습니다.


**Impact:** * 레지스트리의 `addWallet`, `updateInvestor`, `addWalletByInvestor` 및 `TokenIssuer:issueTokens`, `SecuritySwap:swap` 함수가 DoS될 수 있습니다.

* 토큰 발행 프로세스가 해당 지갑을 사용하여 토큰을 발행하기 때문입니다(`issueTokensCustom` 참조):

```solidity
 function issueTokensCustom(address _to, uint256 _value, uint256 _issuanceTime, uint256 _valueLocked, string memory _reason, uint64 _releaseTime) // _to could be the wallet that malicious investor just take
    public
    virtual
    override
```solidity
    returns (
    /*onlyIssuerOrAbove*/
        bool
    )
    {...}
```

**Proof of Concept:** `registry-service.test.ts`의 `Wallet By Investor` describe 블록에서 다음 개념 증명을 실행하세요:

```typescript
 it('Steal wallet', async function() {
        // victim wallet will be another wallet that the investor2 want to register
        const [owner, investor1, investor2, victimWallet] = await hre.ethers.getSigners();
        const { registryService, dsToken } = await loadFixture(deployDSTokenRegulated);

        // Setup: Register two investors
        await registryService.registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.INVESTOR_ID.INVESTOR_COLLISION_HASH_1);
        await registryService.registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.INVESTOR_ID.INVESTOR_COLLISION_HASH_2);

        // Setup: Add wallets to investors
        await registryService.addWallet(investor1, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);
        await registryService.addWallet(investor2, INVESTORS.INVESTOR_ID.INVESTOR_ID_2);

        // Verify initial state
        expect(await registryService.getInvestor(investor1.address)).to.equal(INVESTORS.INVESTOR_ID.INVESTOR_ID_1);
        expect(await registryService.getInvestor(investor2.address)).to.equal(INVESTORS.INVESTOR_ID.INVESTOR_ID_2);
        expect(await registryService.isWallet(victimWallet.address)).to.be.false;

        // ATTACK: Investor1 front-runs and steals victimWallet before Investor2 can register it
        const registryServiceFromInvestor1 = await registryService.connect(investor1);
        await registryServiceFromInvestor1.addWalletByInvestor(victimWallet.address);

        // Verify attack succeeded - victimWallet now belongs to Investor1
        expect(await registryService.getInvestor(victimWallet.address)).to.equal(INVESTORS.INVESTOR_ID.INVESTOR_ID_1);
        expect(await registryService.isWallet(victimWallet.address)).to.be.true;

        // Now if Investor2 tries to register the same wallet, it will fail
        const registryServiceFromInvestor2 = await registryService.connect(investor2);
        await expect(
          registryServiceFromInvestor2.addWalletByInvestor(victimWallet.address)
        ).to.be.revertedWith("Wallet already exists");

        // Demonstrate token hijacking - issue tokens to victimWallet
        // The tokens will go to Investor1 instead of Investor2
        await dsToken.setCap(1000);
        await dsToken.issueTokens(victimWallet.address, 100);

        // Verify Investor1 received the tokens (through their wallet)
        expect(await dsToken.balanceOf(victimWallet.address)).to.equal(100);
        expect(await registryService.getInvestor(victimWallet.address)).to.equal(INVESTORS.INVESTOR_ID.INVESTOR_ID_1);

      });
```

**Recommended Mitigation:** `addWalletByInvestor`를 제거하세요.

**Securitize:** 커밋 [05c5bad](https://github.com/securitize-io/dstoken/commit/05c5bada3c2801b1333fc96f4abc5226a84471f0)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `SecuritizeSwap::executeStableCoinTransfer` should use `SafeERC20` approval and transfer functions instead of standard `IERC20` functions

**Description:** `SecuritizeSwap::executeStableCoinTransfer`에서 표준 `IERC20` 함수 대신 [`SafeERC20::forceApprove`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L101-L108) 및 [`SafeERC20::safeTransferFrom`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L43-L47)을 사용하세요:
```diff
    function executeStableCoinTransfer(address from, uint256 value) private {
        if (bridgeChainId != 0 && address(USDCBridge) != address(0)) {
-           stableCoinToken.transferFrom(from, address(this), value);
-           stableCoinToken.approve(address(USDCBridge), value);
+           SafeERC20.safeTransferFrom(stableCoinToken, from, address(this), value);
+           SafeERC20.forceApprove(stableCoinToken, address(USDCBridge), value);
            USDCBridge.sendUSDCCrossChainDeposit(bridgeChainId, issuerWallet, value);
        } else {
-           stableCoinToken.transferFrom(from, issuerWallet, value);
+           SafeERC20.safeTransferFrom(stableCoinToken, from, issuerWallet, value);
        }
    }
```

**Securitize:** `SecuritizeSwap`은 더 이상 사용되지 않으므로 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### `MulticallProxy::_callTarget` doesn't check if `target` has code

**Description:** `IssuerMulticall::multicall`은 허가된 사용자가 임의의 대상을 상대로 여러 호출을 수행할 수 있도록 하며, 각 대상에 대해 `MulticallProxy::_callTarget`을 호출합니다.

`MulticallProxy::_callTarget`은 각 `target`에 코드가 있는지 확인하지 않습니다:
```solidity
function _callTarget(address target, bytes memory data, uint256 i) internal returns (bytes memory) {
    // @audit returns true if `target` has no code
    (bool success, bytes memory returndata) = target.call(data);
    if (!success) {
        if (returndata.length > 0) {
            // Assumes revert reason is encoded as string
            revert MulticallFailed(i, _getRevertReason(returndata));
        } else {
            revert MulticallFailed(i, "Call failed without revert reason");
        }
    }
    return returndata;
}
```

**Impact:** 코드가 없는 주소에서 `call`을 사용할 때, `call`은 `true`를 반환합니다. 이는 멀티콜 내의 대상 중 하나에 코드가 없는 경우 실제로 성공하지 않았음에도 불구하고 호출자가 멀티콜이 성공했다고 생각하게 만들 수 있습니다.

**Recommended Mitigation:** `target`에 코드가 있는지 확인하세요:
```diff
    function _callTarget(address target, bytes memory data, uint256 i) internal returns (bytes memory) {
+        require(target.code.length > 0, "Target is not a contract");
```

**Securitize:** 영향을 받는 계약은 더 이상 사용되지 않으므로 삭제되었습니다.

**Cyfrin:** 검증되었습니다.


### Seized tokens will be stuck on the receiver of the seized funds because it is not an investor and never receives investorsBalance

**Description:** DSToken을 압류(seize)할 때, 몰수된 자산의 수신자는 발행자(issuer) 지갑이어야 합니다. 발행자 지갑인 지갑은 투자자 지갑이 아니어야 합니다. 즉, 수신자 지갑은 투자자를 가리키지 않으므로 `getRegistryService().getInvestor(_wallet);`을 읽으면 빈 문자열이 반환됩니다.

이로 인해 자금 압류 중 몰수된 자금을 받는 지갑의 `investorsBalance`를 업데이트할 때 해당 지갑과 관련된 투자자가 없기 때문에 investorsBalance가 등록되지 않는 부작용이 발생합니다.
```solidity
    function seize(
        TokenData storage _tokenData,
        address[] memory _services,
        address _from,
        address _to,
        uint256 _value,
        uint256 _shares
)
    public
    validSeizeParameters(_tokenData, _from, _to, _shares)
    {
       ...
@>      updateInvestorBalance(_tokenData, registryService, _to, _shares, CommonUtils.IncDec.Increase);
    }

    function updateInvestorBalance(TokenData storage _tokenData, IDSRegistryService _registryService, address _wallet, uint256 _shares, CommonUtils.IncDec _increase) internal returns (bool) {
//@audit => investor for the wallet receiving the seized funds would return an empty string
        string memory investor = _registryService.getInvestor(_wallet);
//@audit => An empty string would skip the code to register the investorsBalance for the seized funds
        if (!CommonUtils.isEmptyString(investor)) {
            uint256 balance = _tokenData.investorsBalances[investor];
            if (_increase == CommonUtils.IncDec.Increase) {
                balance += _shares;
            } else {
                balance -= _shares;
            }
            _tokenData.investorsBalances[investor] = balance;
        }
        return true;
    }
```

압류된 자금에 대한 `investorsBalance`가 없다는 것은 `InvestorLockManager.getTransferableTokens()`가 압류된 자금을 이동하려고 할 때 전송 가능한 토큰이 0개를 반환한다는 것을 의미합니다.
- `sender`가 플랫폼 지갑이 아니고 발행자 지갑이며 전송을 처리할 투자자 잔액이 0이기 때문에, 전송 가능한 토큰이 0개이면 `ComplianceServiceRegulated.completeTransferCheck()`는 `TOKENS_LOCKED` 오류 메시지와 함께 코드 16을 반환합니다.

```solidity
    function completeTransferCheck(
        address[] memory _services,
        CompletePreTransferCheckArgs memory _args
    ) internal view returns (uint256 code, string memory reason) {
        ...
        if (
            !isPlatformWalletFrom &&
@>      IDSLockManager(_services[LOCK_MANAGER]).getTransferableTokens(_args.from, block.timestamp) < _args.value
        ) {
@>          return (16, TOKENS_LOCKED);
        }

}

```

**Impact:** 몰수된 자금을 수신하는 주소에서 압류된 토큰을 전송할 수 없게 됩니다.

**Proof of Concept:** `dstoken-regulated.test.ts`에 다음 PoC를 추가하세요:
```js
    it.only('seized tokens are stuck on the receiver', async function () {
      const [owner, investor1, investor2, seizedReceiver, investor3] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceService, complianceConfigurationService, walletManager } = await loadFixture(deployDSTokenRegulated);
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.FRANCE, INVESTORS.Compliance.EU);
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, INVESTORS.Compliance.US);
      await complianceConfigurationService.setEURetailInvestorsLimit(10);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, investor1, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, investor2, registryService);
      // await registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, seizedReceiver, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, investor3, registryService);

      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.FRANCE);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.Country.USA);
      // await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID, INVESTORS.Country.USA);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, INVESTORS.Country.USA);

      const issuedTokens = 500;
      const seizedTokens = issuedTokens;
      await dsToken.issueTokens(investor1, issuedTokens);
      await dsToken.issueTokens(investor2, issuedTokens);

      expect(await complianceService.getEURetailInvestorsCount(INVESTORS.Country.FRANCE)).equal(1);
      expect(await complianceService.getUSInvestorsCount()).equal(1);

      await walletManager.addIssuerWallet(seizedReceiver)

      await dsToken.seize(investor1, seizedReceiver, await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1), "");
      expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1)).equal(0);
      expect(await dsToken.balanceOf(investor1)).equal(0);
      expect(await complianceService.getEURetailInvestorsCount(INVESTORS.Country.FRANCE)).equal(0);
      expect(await complianceService.getUSInvestorsCount()).equal(1);

      expect(await dsToken.balanceOf(seizedReceiver)).equal(500);

      time.increase(10_000);

      const dsTokenFromSeizedReceiver = await dsToken.connect(seizedReceiver);
      //@audit-issue => Seized tokens will be stuck on the receiver of the seized funds because it is not an investor and never receives investorsBalance
      await expect(dsTokenFromSeizedReceiver.transfer(investor3, seizedTokens)).to.be.reverted;
    });
```

**Securitize:** 인정함(Acknowledged); 운영 팀에 따르면 워크플로는 다음과 같습니다: 압류된 자금은 재할당될 때까지 지갑에 머물며, 지갑은 미리 잠겨 있습니다. 그런 다음 토큰을 소각하고 올바른 지갑/보유자에게 다시 발행합니다.


### `ComplianceServiceRegulated::preIssuanceCheck` allows issuance to non-accredited investors when `forceAccredited` or `forceAccreditedUS` is set, and allows issuance below regional minimum thresholds, violating compliance requirements

**Description:** `ComplianceServiceRegulated::completeTransferCheck`에는 `preIssuanceCheck`가 누락된 여러 확인 사항이 있습니다:

1) Force Accredited (적격 투자자 강제)

`completeTransferCheck`는 force accredited가 활성화되었는지 확인하고 그렇다면 적격(accredited) 투자자에게만 전송을 허용합니다:
```solidity
bool isAccreditedTo = isAccredited(_services, _args.to);
if (
    IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getForceAccredited() && !isAccreditedTo
) {
    return (61, ONLY_ACCREDITED);
}

} else if (toRegion == US) {
    if (
        IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getForceAccreditedUS() &&
        !isAccreditedTo
    ) {
        return (62, ONLY_US_ACCREDITED);
    }
```

그러나 `preIssuanceCheck`는 이것을 강제하지 않으므로, `ForceAccredited` 또는 `ForceAccreditedUS`가 활성화된 경우에도 비적격 투자자에게 발행을 허용할 수 있습니다.

2) Regional Minimal Token Holdings (지역별 최소 토큰 보유량)

`completeTransferCheck`는 지역별 최소 토큰 보유량을 확인합니다. 예:
```solidity
if (toInvestorBalance + _value < IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getMinUSTokens()) {
    return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
}
```
하지만 preIssuanceCheck는 일반적인 최소 보유량만 확인합니다:
```solidity
        if (
            !walletManager.isPlatformWallet(_to) &&
        balanceOfInvestorTo + _value < complianceConfigurationService.getMinimumHoldingsPerInvestor()
        ) {
            return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
        }
```

따라서 `preIssuanceCheck`는 사용자가 지역별 최소 토큰 보유량을 위반하는 양의 토큰을 발행받도록 초래할 수 있습니다.

**Impact:** `ComplianceServiceRegulated::preIssuanceCheck`는 `forceAccredited` 또는 `forceAccreditedUS`가 설정된 경우 비적격 투자자에게 발행을 허용하고, 지역별 최소 임계값 미만의 발행을 허용하여 규정 준수 요구 사항을 위반합니다.

**Recommended Mitigation:** `ComplianceServiceRegulated::preIssuanceCheck`에서 위의 확인 사항을 강제하세요.

**Securitize:** 커밋 [4991826](https://github.com/securitize-io/dstoken/commit/499182670e4b05c0a8f5eefc639403e5dbaf15bd), [0656ccd](https://github.com/securitize-io/dstoken/commit/0656ccd7e48af296770bcf780a47c3a14fcc9eba)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Burn and seize functions can be DoS when investor has several wallets that they control

**Description:** `TokenLibrary.sol`의 `burn` 함수는 `investorsBalances[investorId]`(모든 지갑에 걸친 총 투자자 잔액) 대신 `walletsBalances[_who]`(개별 지갑 잔액, 위 화살표 참조)를 확인합니다. 이를 통해 등록된 지갑이 여러 개인 투자자는 자신의 지갑 간에 토큰을 전송하여 토큰을 소각할 수 없게 만들 수 있습니다.

```solidity
function burn(
        TokenData storage _tokenData,
        address[] memory _services,
        address _who,
        uint256 _value,
        ISecuritizeRebasingProvider _rebasingProvider
    ) public returns (uint256) {
        uint256 sharesToBurn = _rebasingProvider.convertTokensToShares(_value);

        require(sharesToBurn <= _tokenData.walletsBalances[_who], "Not enough balance"); <---------

        IDSComplianceService(_services[COMPLIANCE_SERVICE]).validateBurn(_who, _value);

        _tokenData.walletsBalances[_who] -= sharesToBurn;
        updateInvestorBalance(
            _tokenData,
            IDSRegistryService(_services[REGISTRY_SERVICE]),
            _who,
            sharesToBurn,
            CommonUtils.IncDec.Decrease
        );

        _tokenData.totalSupply -= sharesToBurn;
        return sharesToBurn;
    }

```

투자자가 원래 지갑에서 자신이 제어하는 다른 지갑으로 토큰을 전송하면, 투자자가 총 잔액에 토큰을 여전히 소유하고 있더라도 "Not enough balance"와 함께 소각 함수가 실패합니다.

**Impact:** 관리자가 소각 또는 압류와 같은 민감한 작업을 수행할 때 프라이빗 멤풀(private mempools)을 사용하지 않으면, 투자자는 자신의 지갑 간에 토큰을 전송하여 토큰 소각을 일시적으로 방지하기 위해 선행 실행(front-run)할 수 있습니다.

**Proof of Concept:** `dstoken-regulated.test.ts`에서 다음 개념 증명을 실행하세요:

```typescript
describe('Burn DoS Vulnerability POC', function() {
        it('Should demonstrate that burn can be DoS by transferring between investor wallets', async function() {
          const [investor, wallet2, wallet3] = await hre.ethers.getSigners();
          const { dsToken, registryService } = await loadFixture(deployDSTokenRegulatedWithRebasingAndEighteenDecimal);

          // Register investor with multiple wallets
          await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, investor.address, registryService);
          await registryService.addWallet(wallet2.address, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);
          await registryService.addWallet(wallet3.address, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);

          // Issue tokens to investor
          await dsToken.issueTokens(investor.address, 1000);

          // Transfer tokens between investor's own wallets
          await dsToken.connect(investor).transfer(wallet2.address, 1000);

          // Now investor has 0 balance in original wallet but 1000 total
          expect(await dsToken.balanceOf(investor.address)).to.equal(0);
          expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1)).to.equal(1000);

          // VULNERABILITY: Burn fails because wallet balance is 0, even though investor has tokens
          await expect(dsToken.burn(investor.address, 100, 'DoS test'))
            .to.be.revertedWith('Not enough balance');
        });
      });

```

**Recommended Mitigation:** 가능한 완화 옵션은 다음과 같습니다:
* 소각 및 압류와 같은 민감한 관리자 트랜잭션은 [flashbots](https://docs.flashbots.net/flashbots-protect/overview)와 같은 프라이빗 멤풀 서비스를 통해 수행하여 선행 실행되지 않도록 합니다.
* 투자자가 계속해서 지갑을 추가하고 토큰을 분산시키는 것을 방지하기 위해 `addWalletByInvestor` 함수를 제거합니다.
* 투자자에게 속한 모든 지갑을 반복하고 토큰을 소각/압류하는 `burnAll` 및 `seizeAll` 함수를 `DSToken`에 추가합니다.

**Securitize:** 커밋 [05c5bad](https://github.com/securitize-io/dstoken/commit/05c5bada3c2801b1333fc96f4abc5226a84471f0)에서 `addWalletByInvestor`를 제거하여 수정되었습니다. 운영 팀은 프라이빗 멤풀을 통한 민감한 관리자 트랜잭션 실행에 관한 조언을 고려할 것입니다.


### Not maximum amount of issuances enforce

**Description:** `ComplianceServiceRegulated.sol`의 `createIssuanceInformation` 함수는 최대 제한 없이 투자자당 무제한 발행 레코드를 생성할 수 있도록 허용합니다. 가스 부족을 방지하기 위해 `MAX_LOCKS_PER_INVESTOR = 30` 상수가 있는 잠금 시스템과 달리, 발행 시스템에는 그러한 보호 장치가 없습니다. 토큰이 투자자에게 발행될 때마다 새 발행 레코드가 생성되어 제한 없는 매핑(`issuancesValues`, `issuancesTimestamps`, `issuancesCounters`)에 저장됩니다. 이러한 레코드는 규정 준수 확인(`getComplianceTransferableTokens` 및 `cleanupExpiredIssuances`) 중에 루프에서 처리되므로 투자자가 너무 많은 발행 레코드를 축적하면 잠재적으로 가스 제한 문제가 발생할 수 있습니다.

```solidity
function createIssuanceInformation(
        string memory _investor,
        uint256 _shares,
        uint256 _issuanceTime
    ) internal returns (bool) {
        uint256 issuancesCount = issuancesCounters[_investor];//@audit-ok (low) there is not max limit for the issuancesCounters?

        issuancesValues[_investor][issuancesCount] = _shares;
        issuancesTimestamps[_investor][issuancesCount] = _issuanceTime;
        issuancesCounters[_investor] = issuancesCount + 1;

        return true;
    }

```

**Impact:** 투자자가 많은 발행 레코드를 축적하면 규정 준수 확인이 가스 제한에 도달할 수 있습니다. 발행은 접근 제어 작업이므로 공격 벡터는 제한적이지만, 잠금 관리자도 접근 제어 작업이며 최대 잠금이 있습니다.

**Recommended Mitigation:** 기존 잠금 시스템과 유사하게 투자자당 발행 레코드에 대한 최대 한도를 구현하세요.

**Securitize:** 인정함(Acknowledged); 정리(cleanup) 메서드가 바로 이러한 이유로 생성되었으며, 잠금 기간이 매우 오래된 발행 레코드를 유지해야 할 만큼 길지 않기 때문에 이러한 시나리오를 피하기 위해 충분한 레코드를 정리할 것이라고 가정합니다.


### Platform Wallet not exception for Maximum Holdings Per Investor Limits

**Description:** `ComplianceServiceRegulated.sol`의 `preIssuanceCheck` 함수는 투자자 보유 한도에 대한 플랫폼 지갑 면제와 관련하여 일관성 없는 동작을 보여줍니다. 플랫폼 지갑은 투자자당 최소 보유량 확인에서 적절하게 면제되지만, 투자자당 최대 보유량 확인에서는 면제되지 않습니다.

```solidity
if (
    !_args.isPlatformWalletTo &&
    toInvestorBalance + _args.value < IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getMinimumHoldingsPerInvestor()
) {
    return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
}
```
최소 보유량 확인에는 플랫폼 지갑을 면제하기 위해 `!_args.isPlatformWalletTo &&`가 올바르게 포함되어 있지만, 최대 보유량 확인에는 이 면제사항이 없어 플랫폼 지갑에도 일반 투자자와 동일한 최대 보유량 제한이 적용됩니다.

```solidity
if (
            isMaximumHoldingsPerInvestorOk(
                IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getMaximumHoldingsPerInvestor(),
                toInvestorBalance, _args.value)
        ) {
            return (52, AMOUNT_OF_TOKENS_ABOVE_MAX);
        }

```

**Impact:** 플랫폼 지갑은 최대 보유량 제한으로 인해 운영 목적으로 충분한 토큰을 보유하지 못할 수 있습니다.

**Recommended Mitigation:** 최대 보유량 확인에 플랫폼 지갑 면제를 추가하세요:
```diff
if (
+    !_args.isPlatformWalletTo &&
    isMaximumHoldingsPerInvestorOk(
        complianceConfigurationService.getMaximumHoldingsPerInvestor(),
        balanceOfInvestorTo,
        _value)
) {
    return (52, AMOUNT_OF_TOKENS_ABOVE_MAX);
}
```

**Securitize:** 커밋 [5b96460](https://github.com/securitize-io/dstoken/commit/5b964605cd92bdc1307f975355ec7ca402265119), [2cab0c2](https://github.com/securitize-io/dstoken/commit/2cab0c298cbeee8951c76e63947a69dc48a2698b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Attribute changes via `setAttribute` or `updateInvestor` do NOT trigger compliance count updates.

**Description:** 규정 준수 시스템은 투자자 제한 및 규제 준수를 시행하기 위해 여러 카운트 변수(`accreditedInvestorsCount`, `usAccreditedInvestorsCount`, `euRetailInvestorsCount` 등)를 유지 관리합니다. 그러나 `setAttribute()` 또는 `updateInvestor()` 함수를 통해 투자자 속성이 변경되면, 이러한 카운트 변수는 변경 사항을 반영하도록 업데이트되지 않습니다.

시스템은 `adjustInvestorCountsAfterCountryChange()`를 통해 투자자가 추가되거나 제거될 때만 카운트를 업데이트하지만, 투자자 분류(`ACCREDITED`, `QUALIFIED`)에 영향을 미치는 속성 변경은 이 메커니즘을 완전히 우회합니다. 이는 실제 투자자 상태와 규정 준수 추적 시스템 간의 근본적인 단절을 초래합니다.

이는 특히 `accreditedInvestorsCount`, `usAccreditedInvestorsCount`, `euRetailInvestorsCount`에 영향을 미칩니다. 예:

```solidity
} else if (countryCompliance == EU && !getRegistryService().isQualifiedInvestor(_id)) {//@audit  if there are an investor that become qualified after he enter the system counting  it will not be discounting the investor when he is leaving
            if(_increase == CommonUtils.IncDec.Increase) {
                euRetailInvestorsCount[_country]++;
            }
            else {
                euRetailInvestorsCount[_country]--;
            }
```

이 경우 EU 소매 투자자가 시스템에 들어오면 `euRetailInvestorsCount`에 계산되지만, 그가 `QUALIFIED`가 된 후 시스템을 떠나면 `euRetailInvestorsCount`는 부풀려진 상태로 유지됩니다.

**Impact:** 카운트가 실제 수를 반영하지 않으므로 투자자 제한 시행을 신뢰할 수 없게 됩니다.

**Proof of Concept:** `compliance-service-regulated.test.ts`에서 다음 개념 증명을 실행하세요:

```typescript
describe('Proof of Concept: Attribute Change Bug', function () {
    it('should demonstrate euRetailInvestorsCount inflation when investor becomes qualified', async function () {
      const [wallet, wallet2, transferAgent] = await hre.ethers.getSigners();
      const { dsToken, registryService, complianceService, complianceConfigurationService, trustService } = await loadFixture(deployDSTokenRegulated);

      // Set up EU compliance
      await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.FRANCE, INVESTORS.Compliance.EU);
      await complianceConfigurationService.setEURetailInvestorsLimit(1);

      // Register two investors in France
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, wallet, registryService);
      await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, wallet2, registryService);

      // Set both investors to France (EU)
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.FRANCE);
      await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.Country.FRANCE);

      // Issue tokens to first investor (non-qualified) - should be counted as EU retail
      await dsToken.issueTokens(wallet, 100);

      // Check initial EU retail count
      const initialEuRetailCount = await complianceService.getEURetailInvestorsCount(INVESTORS.Country.FRANCE);
      expect(initialEuRetailCount).to.equal(1, "Initial EU retail count should be 1");

      // Try to issue tokens to second investor - should fail due to EU retail limit
      await expect(dsToken.issueTokens(wallet2, 100)).revertedWith('Max investors in category');

      // Now make the first investor qualified via setAttribute
      // QUALIFIED = 4, APPROVED = 1
      await registryService.setAttribute(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, 4, 1, 0, "");


      // Check EU retail count - it should still be 1 (BUG: should be 0)
      const euRetailCountAfterQualification = await complianceService.getEURetailInvestorsCount(INVESTORS.Country.FRANCE);
      expect(euRetailCountAfterQualification).to.equal(1, "BUG: EU retail count should be 0 after qualification, but it's still 1");

      // Now try to issue tokens to second investor - this should still fail due to inflated count
      await expect(dsToken.issueTokens(wallet2, 100)).revertedWith('Max investors in category');

    });
  });
```

**Recommended Mitigation:** 투자자 분류에 영향을 미치는 속성 변경이 즉시 해당 규정 준수 카운트 변수를 업데이트하여 시스템 무결성을 유지하도록 하세요.

**Securitize:** 인정함(Acknowledged); 이러한 함수는 현재 호출되지 않습니다.


### Investor can prevent themselves from being removed by making `removeInvestor` revert

**Description:** `RegistryService.sol`의 `removeInvestor` 함수에는 투자자가 시스템에서 자신의 제거를 영구적으로 방지할 수 있는 결함이 포함되어 있습니다. 이 함수는 투자자 제거를 허용하기 전에 `investors[_id].walletCount == 0`을 요구하지만, 투자자는 `addWalletByInvestor`를 통해 제한 없이 지갑을 무제한으로 추가할 수 있는 반면, `removeWallet`을 통해 지갑을 제거하는 것은 `EXCHANGE` 역할만 가능합니다.

```solidity
function removeInvestor(string calldata _id) public override onlyExchangeOrAbove investorExists(_id) returns (bool) {
        require(getTrustService().getRole(msg.sender) != EXCHANGE || investors[_id].creator == msg.sender, "Insufficient permissions");
        require(investors[_id].walletCount == 0, "Investor has wallets"); <----------

        for (uint8 index = 0; index < 16; index++) {
            delete attributes[_id][index];
        }

        delete investors[_id];

        emit DSRegistryServiceInvestorRemoved(_id, msg.sender);

        return true;
    }
```

이는 악의적인 투자자가 자신의 제거를 방지하기 위해 지갑을 추가할 수 있는 영구적인 DoS 조건을 생성합니다.

**Impact:** `removeInvestor`가 DoS되어 투자자를 제거할 수 없게 될 수 있습니다.

**Recommended Mitigation:** `addWalletByInvestor` 제거를 고려하세요.

**Securitize:** 커밋 [05c5bad](https://github.com/securitize-io/dstoken/commit/05c5bada3c2801b1333fc96f4abc5226a84471f0)에서 `addWalletByInvestor`를 제거하여 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Resolve inconsistency between `DSToken::checkWalletsForList` and `RegistryService::removeWallet`

**Description:** `DSToken::checkWalletsForList`는 잔액이 0인 경우에만 지갑을 제거합니다:
```solidity
function checkWalletsForList(address _from, address _to) private {
    if (super.balanceOf(_from) == 0) {
        removeWalletFromList(_from);
    }
```

그러나 `RegistryService::removeWallet`은 양수 잔액이 있는 지갑 제거를 허용합니다:
```solidity
function removeWallet(address _address, string memory _id) public override onlyExchangeOrAbove walletExists(_address) walletBelongsToInvestor(_address, _id) returns (bool) {
    require(getTrustService().getRole(msg.sender) != EXCHANGE || investorsWallets[_address].creator == msg.sender, "Insufficient permissions");

    delete investorsWallets[_address];
    investors[_id].walletCount--;

    emit DSRegistryServiceWalletRemoved(_address, _id, msg.sender);

    return true;
}
```

**Impact:** 양수 잔액이 있는 지갑을 `RegistryService::removeWallet`을 통해 제거할 수 있으며, 이는 주어진 지갑 주소에 대한 투자자 ID를 반환하는 `RegistryService`의 함수에 의존하는 토큰 전송 및 기타 관련 활동을 방지합니다.

**Recommended Mitigation:** `RegistryService::removeWallet`은 양수 잔액이 있는 지갑 제거를 허용해서는 안 됩니다.

**Securitize:** 커밋 [1eaec18](https://github.com/securitize-io/dstoken/commit/1eaec18ce4a9dc6be24c43d02950839341b6282d#diff-8abf57cc60f7fc1ff193a0912746e8539f56be2d652f1bb59fa5ea8ef3c43d97R177-R178)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `RegistryService::addWallet` should revert if the wallet being added has positive balance of `DSToken`

**Description:** `RegistryService::addWallet`은 이미 긍정적인 `DSToken` 잔액이 있는 지갑을 추가하는 데 사용되는 경우 투자자 지갑이나 총 잔액을 업데이트하지 않습니다.

이를 사용하여 양수 잔액이 있는 지갑을 추가하면 여러 가지 잘못된 상태가 발생합니다:

#### Compliance Validation Problems ####
* 투자자가 총 투자자에 포함되지 않을 수 있음
* 투자자 제한 확인을 트리거하지 않음
* 미국/EU 투자자 제한을 우회할 수 있음

#### Transfer Validation Problems ####
* 규정 준수 확인은 투자자 잔액이 지갑 잔액과 일치할 것으로 예상함
* 일부 확인은 투자자 잔액을 전송 금액과 비교함
* 잘못된 "새로운 투자자" 로직을 트리거할 수 있음
* 기존 내부 지갑 / 투자자 잔액에서 뺄 때 언더플로우로 인해 전송이 되돌려져 토큰이 묶일 수 있음

**Recommended Mitigation:** 추가되는 지갑에 `DSToken`의 양수 잔액이 있는 경우 `RegistryService::addWallet`은 되돌려야(revert) 합니다. `addWalletByInvestor`에도 동일하게 적용되지만 해당 함수는 제거되고 있습니다.

대안으로 다른 옵션은 현재 프라이빗인 `DSToken::updateInvestorBalance` 및 `addWalletToList`를 호출하여 투자자에 대한 기존 토큰을 등록하는 것이지만, 이 함수들은 현재 프라이빗입니다.

**Securitize:** 커밋 [3e9c754](https://github.com/securitize-io/dstoken/commit/3e9c754c6e11866884457cedfa46dd55d5b6bc2a)에서 양수 잔액이 있는 지갑 추가를 방지하여 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Informational


### Use named imports

**Description:** 코드베이스의 일부 위치에서는 명명된 가져오기(named imports)를 사용했지만 다른 곳에서는 사용하지 않았습니다; 모든 곳에서 명명된 가져오기를 사용하는 것을 권장합니다.

**Securitize:** 커밋 [7fc22e2](https://github.com/securitize-io/dstoken/commit/7fc22e2bb994e071fae8f241b8558d10d199529a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Upgradeable contracts should call `_disableInitializers` in constructor

**Description:** 업그레이드 가능한 계약은 생성자에서 `_disableInitializers`를 [호출](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing_the_implementation_contract)해야 합니다:
```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

영향을 받는 계약:
* `contracts/bulk/BulkOperator.sol`
* `contracts/compliance/ComplianceConfigurationService.sol`
* `contracts/compliance/ComplianceServiceNotRegulated.sol`
* `contracts/compliance/ComplianceServiceWhitelisted.sol`
* `contracts/compliance/InvestorLockManager.sol`
* `contracts/compliance/LockManager.sol`
* `contracts/compliance/WalletManager.sol`
* `contracts/issuance/TokenIssuer.sol`
* `contracts/multicall/IssuerMulticall.sol`
* `contracts/rebasing/SecuritizeRebasingProvider.sol`
* `contracts/registry/RegistryService.sol`
* `contracts/registry/WalletRegistrar.sol`
* `contracts/swap/SecuritizeSwap.sol`
* `contracts/token/DSToken.sol`
* `contracts/trust/TrustService.sol`
* `contracts/utils/TransactionRelayer.sol`

**Securitize:** 커밋 [094baaf](https://github.com/securitize-io/dstoken/commit/094baaf8562cc2eaae1a89769188ad4e8e7476cb)에서 수정되었습니다; 나열된 계약 중 일부는 더 이상 사용되지 않아 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### Prefer explicit unsigned integer sizes

**Description:** 명시적인 부호 없는 정수 크기(explicit unsigned integer sizes)를 선호하세요:
```solidity
trust/TrustService.sol
218:        for (uint i = 0; i < _addresses.length; i++) {

compliance/ComplianceConfigurationService.sol
34:        for (uint i = 0; i < _countries.length; i++) {

compliance/WalletManager.sol
75:        for (uint i = 0; i < _wallets.length; i++) {
97:        for (uint i = 0; i < _wallets.length; i++) {

```

**Securitize:** 커밋 [26c5bb0](https://github.com/securitize-io/dstoken/commit/26c5bb05676233ca0c5caa1d9b02fd98360ad4e7)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use named mapping parameters to explicitly note the purpose of keys and values

**Description:** 키와 값의 목적을 명시적으로 기록하기 위해 명명된 매핑 매개변수(named mapping parameters)를 사용하세요:
```solidity
token/TokenLibrary.sol
36:        mapping(address => uint256) walletsBalances;
37:        mapping(string => uint256) investorsBalances;

mocks/TestToken.sol
39:    mapping(address => uint256) balances;
40:    mapping(address => mapping(address => uint256)) allowed;

data-stores/TokenDataStore.sol
27:    mapping(address => mapping(address => uint256)) internal allowances;
28:    mapping(uint256 => address) internal walletsList;
30:    mapping(address => uint256) internal walletsToIndexes;

swap/SecuritizeSwap.sol
43:    mapping(string => uint256) internal noncePerInvestor;

utils/TransactionRelayer.sol
48:    mapping(bytes32 => uint256) internal noncePerInvestor;

utils/MultiSigWallet.sol
46:    mapping(address => bool) isOwner; // immutable state

data-stores/RegistryServiceDataStore.sol
44:        // Ref: https://docs.soliditylang.org/en/v0.7.1/070-breaking-changes.html#mappings-outside-storage
45:        // mapping(uint8 => Attribute) attributes;
48:    mapping(string => Investor) internal investors;
49:    mapping(address => Wallet) internal investorsWallets;
52:     * @dev DEPRECATED: This mapping is no longer used but must be kept for storage layout compatibility in the proxy.
53:     * Do not use this mapping in new code. It will be removed in future non-proxy implementations.
56:    mapping(address => address) internal DEPRECATED_omnibusWalletsControllers;
58:    mapping(string => mapping(uint8 => Attribute)) public attributes;

data-stores/InvestorLockManagerDataStore.sol
24:    mapping(string => mapping(uint256 => Lock)) internal investorsLocks;
25:    mapping(string => uint256) internal investorsLocksCounts;
26:    mapping(string => bool) internal investorsLocked;
27:    mapping(string => mapping(bytes32 => mapping(uint256 => Lock))) internal investorsPartitionsLocks;
28:    mapping(string => mapping(bytes32 => uint256)) internal investorsPartitionsLocksCounts;
29:    mapping(string => bool) internal investorsLiquidateOnly;

data-stores/TrustServiceDataStore.sol
23:    mapping(address => uint8) internal roles;
24:    mapping(string => address) internal entitiesOwners;
25:    mapping(address => string) internal ownersEntities;
26:    mapping(address => string) internal operatorsEntities;
27:    mapping(address => string) internal resourcesEntities;

data-stores/ComplianceConfigurationDataStore.sol
24:    mapping(string => uint256) public countriesCompliances;

data-stores/WalletManagerDataStore.sol
24:    mapping(address => uint8) internal walletsTypes;
25:    mapping(address => mapping(string => mapping(uint8 => uint256))) internal walletsSlots;

data-stores/ServiceConsumerDataStore.sol
23:    mapping(uint256 => address) internal services;

data-stores/LockManagerDataStore.sol
24:    mapping(address => uint256) internal locksCounts;
25:    mapping(address => mapping(uint256 => Lock)) internal locks;

data-stores/ComplianceServiceDataStore.sol
29:    mapping(string => uint256) internal euRetailInvestorsCount;
30:    mapping(string => uint256) internal issuancesCounters;
31:    mapping(string => mapping(uint256 => uint256)) issuancesValues;
32:    mapping(string => mapping(uint256 => uint256)) issuancesTimestamps;
```

**Securitize:** 커밋 [6c7bc52](https://github.com/securitize-io/dstoken/commit/6c7bc52d2c0eaacc06c8c6e26a7abfbd69d2edae)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Emit missing events

**Description:** 누락된 이벤트를 내보내세요(Emit missing events):
* `DSToken::setFeature`, `setFeatures`, `setCap`
* `TrustService::addEntity`, `changeEntityOwner`, `addOperator`, `removeOperator`, `addResource`, `removeResource`
* `SecuritizeSwap::updateNavProvider`

**Securitize:** 이들 대부분은 더 이상 사용되지 않아 제거되었으며, `setFeature`는 현재 사용되지 않아 당장은 그대로 둡니다.

**Cyfrin:** 검증되었습니다.


### Consider reverting in `RebasingLibrary` functions if rounding down to zero occurs

**Description:** `RebasingLibrary`에는 `convertTokensToShares`와 `convertSharesToTokens`라는 두 가지 함수가 있습니다. 입력이 0이 아닌데 출력이 0이 되는 0으로 내림(rounding down to zero)이 발생하는 경우, 그 시점에서 처리를 계속하는 것은 의미가 없으므로 되돌리는(revert) 것을 고려하세요. 예를 들어:

* `convertTokensToShares`는 `_tokens > 0 && shares == 0`인 경우 되돌려야 합니다.
* `convertSharesToTokens`는 `_shares > 0 && tokens == 0`인 경우 되돌려야 합니다.

**Securitize:** 커밋 [60c5f92](https://github.com/securitize-io/dstoken/commit/60c5f92e4312b069d351dd08e745875fd8f60aa5)에서 `convertTokensToShares`에 이 확인을 추가하여 수정되었습니다. `StandardToken::balanceOf`와 같은 함수에서 사용되므로 `convertSharesToTokens`에는 추가하지 않았습니다.

**Cyfrin:** 검증되었습니다.


### Use named constants instead of hard-coded literals for important values

**Description:** 중요한 값에는 하드 코딩된 리터럴 대신 명명된 상수를 사용하세요:
```solidity
contracts/compliance/ComplianceServiceRegulated.sol
// error codes in `ComplianceServiceRegulated::completeTransferCheck`
217:            return (0, VALID);
221:            return (90, INVESTOR_LIQUIDATE_ONLY);
225:            return (20, WALLET_NOT_IN_REGISTRY_SERVICE);
230:            return (26, DESTINATION_RESTRICTED);
238:            return (16, TOKENS_LOCKED);
243:                return (32, HOLD_UP);
250:                return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
257:                return (50, ONLY_FULL_TRANSFER);
261:                return (33, HOLD_UP);
269:                return (25, FLOWBACK);
276:                return (50, ONLY_FULL_TRANSFER);
286:                return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
294:            return (61, ONLY_ACCREDITED);
305:                return (40, MAX_INVESTORS_IN_CATEGORY);
316:                return (40, MAX_INVESTORS_IN_CATEGORY);
322:                return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
329:                return (62, ONLY_US_ACCREDITED);
339:                return (40, MAX_INVESTORS_IN_CATEGORY);
350:                return (40, MAX_INVESTORS_IN_CATEGORY);
356:                return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
362:                return (40, MAX_INVESTORS_IN_CATEGORY);
373:            return (40, MAX_INVESTORS_IN_CATEGORY);
382:            return (71, NOT_ENOUGH_INVESTORS);
390:            return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
397:            return (51, AMOUNT_OF_TOKENS_UNDER_MIN);
405:            return (52, AMOUNT_OF_TOKENS_ABOVE_MAX);
408:        return (0, VALID);

// denominator representing 100%
108:            return compConfService.getMaxUSInvestorsPercentage() * (complianceService.getTotalInvestorsCount()) / 100;
111:        return Math.min(compConfService.getUSInvestorsLimit(), compConfService.getMaxUSInvestorsPercentage() * (complianceService.getTotalInvestorsCount()) / 100);

compliance/ComplianceConfigurationService.sol
```solidity
238:        require(_uint_values.length == 16, "Wrong length of parameters");

registry/RegistryService.sol
43:        for (uint8 index = 0; index < 16; index++) {
135:        require(_attributeId < 16, "Unknown attribute");

trust/TrustService.sol
216:        require(_addresses.length <= 30, "Exceeded the maximum number of addresses");

compliance/WalletManager.sol
74:        require(_wallets.length <= 30, "Exceeded the maximum number of wallets");
96:        require(_wallets.length <= 30, "Exceeded the maximum number of wallets");
```

**Securitize:** 인정함(Acknowledged).


### Remove obsolete `return` statements when already using named return variables

**Description:** 이미 명명된 반환 변수(named return variables)를 사용하고 있는 경우, 오래된 `return` 문을 제거하세요.

* `contracts/utils/BulkBalanceChecker.sol`
```solidity
43:        return balances;
```

* `contracts/swap/SecuritizeSwap.sol`
```solidity
236:       return (dsTokenAmount, currentNavRate);
```

**Securitize:** `BulkBalanceChecker`는 커밋 [f3daea2](https://github.com/securitize-io/dstoken/commit/f3daea22479e95886f25e2c3a70b9618fed97a1c)에서 수정되었습니다; `SecuritizeSwap`은 더 이상 사용되지 않으므로 삭제되었습니다.

**Cyfrin:** 검증되었습니다.


### Prefer `ECDSA::tryRecover` to using `ecrecover` directly

**Description:** `ecrecover`는 [서명 변형 가능성(signature malleability)](https://dacian.me/signature-replay-attacks#heading-signature-malleability)에 취약하므로 직접 사용하는 것은 권장되지 않습니다.

`MultiSigWallet` 및 `TransactionRelayer`에서 [ECDSA::tryRecover](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol#L169-L174)를 사용하는 것을 선호하세요.

**Securitize:** `TransactionRelayer`가 크게 리팩터링되어 이제 OZ `ECDSA`를 사용합니다. `MultiSigWallet`은 더 이상 사용되지 않아 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### `TransactionRelayer` and `SecuritizeSwap` should use `CommonUtils::encodeString`

**Description:** `TransactionRelayer::toBytes32`는 `CommonUtils::encodeString`에서 이미 사용 가능한 기능을 중복합니다; 중복 코드를 제거하고 대신 `CommonUtils::encodeString`을 사용하세요.

또한 이 라인은 다시 구현하는 대신 `CommonUtils::encodeString`을 사용해야 합니다:
```solidity
L178:                        keccak256(abi.encodePacked(senderInvestor)),
```

`SecuritizeSwap`도 이 기능을 중복합니다:
```solidity
191:                keccak256(abi.encodePacked(_senderInvestorId))
```

**Securitize:** `TransactionRelayer`에 대해 커밋 [1f69125](https://github.com/securitize-io/dstoken/commit/1f691255378a0deba62281755feb3a28339b194e)에서 수정되었습니다. `SecurtizeSwap`은 더 이상 사용되지 않아 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### Not mechanism to automatically cleanup locks that are already unlock-able on LockManagers

**Description:** `LockManager` 계약뿐만 아니라 `InvestorLockManager`도 이미 잠금 해제 가능한(unlock-able) 잠금을 자동으로 정리하는 메커니즘이 없습니다; `removeLockRecord` 또는 `removeLockRecordForInvestor` 함수를 사용한 수동 개입을 통해서만 잠금을 제거할 수 있으며, 이들 중 어느 것도 잠금 기간이 이미 지났는지 여부를 검증하지 않습니다.

**Impact:** 이미 잠금 해제된 잠금을 자동으로 정리하는 메커니즘의 부재는 다음과 같은 여러 문제/비효율성을 초래할 수 있습니다:
- 이미 잠금 해제된 잠금을 반복해서 반복(iterating)하여 토큰을 전송할 때 가스 낭비
- 실제로 잠겨 있는 상태여서 그대로 유지되어야 하는 `lockIndex`에 대한 잠금을 실수로 제거할 수 있는 관리자 오류

**Recommended Mitigation:** `ComplianceServiceRegulated::cleanUpInvestorIssuances`에 의해 투자자 발행(Investors Issuances)이 자동으로 정리되는 것과 유사하게, 이미 잠금 해제된 잠금을 자동으로 정리하는 메커니즘 구현을 고려하세요.

**Securitize:** 인정함(Acknowledged).


### `SecuritizeSwap::buy` should revert if `stableCoinAmount` is zero

**Description:** `SecuritizeSwap::buy`는 `stableCoinAmount`가 0인 경우 되돌려야(revert) 합니다. 그렇지 않으면 투자자가 0 스테이블 코인을 지불하고 `dsToken`을 구매할 수 있음을 의미합니다; 이는 투자자가 `calculateStableCoinAmount` 내부에서 0으로 내림(rounding down to zero)을 트리거할 만큼 충분히 작은 `_dsTokenAmount`로 `buy`를 호출하는 경우 가능할 수 있습니다.

`DSToken`이 소수점 2자리로 설정되면 `calculateStableCoinAmount`에서 0으로 내림이 발생하지 않지만, `DSToken`이 표준 18자리 소수점으로 설정된다면 이 코드는 토큰의 무료 발행(minting)에 취약해질 수 있습니다.

**Recommended Mitigation:**
```diff
    function buy(uint256 _dsTokenAmount, uint256 _maxStableCoinAmount) public override whenNotPaused {
        require(IDSRegistryService(getDSService(REGISTRY_SERVICE)).isWallet(msg.sender), "Investor not registered");
        require(_dsTokenAmount > 0, "DSToken amount must be greater than 0");
        require(navProvider.rate() > 0, "NAV Rate must be greater than 0");

        uint256 stableCoinAmount = calculateStableCoinAmount(_dsTokenAmount);
+       require(stableCoinAmount != 0, "Paying zero not allowed");
```

**Securitize:** `SecuritizeSwap`은 더 이상 사용되지 않아 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### Possible to escape burning of DSTokens by removing the wallet from the current investor and adding it as the wallet of another investor

**Description:** 여러 투자자 계정을 소유한 투자자는 토큰을 발행할 때 투자자로부터 지갑을 제거한 다음 다른 투자자 계정을 사용하여 해당 지갑을 자신의 지갑으로 추가함으로써 계정의 `balanceOfInvestor()`와 `balanceOf()`를 분리할 수 있습니다.
- 이로 인해 발행된 토큰에 대한 `balanceOfInvestor()`는 investor1에 대해 설명된 상태로 유지되고, `balanceOf()`는 실제 지갑을 가리키게 됩니다.

계약을 해당 상태로 만들면 투자자가 소각 작업을 피할 수 있는데, 이는 토큰을 소각해야 하는 지갑을 소유한 현재 투자자의 `investorsBalance`를 소각하려고 할 때 언더플로우가 발생하여 소각 실행이 실패하기 때문입니다.
```solidity
    function burn(
        ...
    ) public returns (uint256) {
        ...
        //@audit-info => here will occur the underflow because the current investor owning the `who` wallet has not the investorBalance required to decrement `sharesToBurn` from it
        updateInvestorBalance(
            _tokenData,
            IDSRegistryService(_services[REGISTRY_SERVICE]),
            _who,
            sharesToBurn,
            CommonUtils.IncDec.Decrease
        );

        ...
    }
```

**Impact:** 투자자는 DSToken의 소각을 피할 수 있습니다.

**Proof of Concept:** `dstoken-regulated.test.ts`에 다음 PoC를 추가하세요:

```js
    it.only('escaping burning by forcing an underflow', async function () {
        const [owner, investor1, investor2, secondWalletInvestor1] = await hre.ethers.getSigners();
        const { registryService, dsToken } = await loadFixture(deployDSTokenRegulated);

        // Setup: Register two investors
        await registryService.registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.INVESTOR_ID.INVESTOR_COLLISION_HASH_1);
        await registryService.registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.INVESTOR_ID.INVESTOR_COLLISION_HASH_2);

        // Setup: Add wallets to investors
        await registryService.addWallet(investor1, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);
        await registryService.addWallet(investor2, INVESTORS.INVESTOR_ID.INVESTOR_ID_2);

        const registryServiceFromInvestor1 = await registryService.connect(investor1);
        await registryServiceFromInvestor1.addWalletByInvestor(secondWalletInvestor1.address);

        // Setup: Issue tokens to secondWalletInvestor1
        const issueTokens = 500;
        await dsToken.setCap(1000);
        await dsToken.issueTokens(secondWalletInvestor1.address, issueTokens);

        expect(await dsToken.balanceOf(secondWalletInvestor1.address)).to.equal(issueTokens);

        // secondWalletInvestor1 is removed as a wallet of investor1
        await registryService.removeWallet(secondWalletInvestor1.address, INVESTORS.INVESTOR_ID.INVESTOR_ID_1);

        // investor2 adds secondWalletInvestor1 wallet as its own
        const registryServiceFromInvestor2 = await registryService.connect(investor2);
        await registryServiceFromInvestor2.addWalletByInvestor(secondWalletInvestor1.address);

        //@audit-info => attempting to burn tokens from secondWalletInvestor1 fails because of underflow when reducing the investorsBalance of the investor2
        //@audit-info => reverts because of underflow
        await expect(dsToken.burn(secondWalletInvestor1.address, issueTokens, "")).to.be.reverted;

        expect(await dsToken.balanceOfInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1)).to.equal(issueTokens);
        expect(await dsToken.balanceOf(secondWalletInvestor1.address)).to.equal(issueTokens);
    });
```

**Securitize:** 인정하지만(Acknowledged) 실제로는 동일한 엔티티가 여러 투자자 계정을 제어할 수 없으므로 불가능합니다; 우리는 KYC/온보딩 프로세스 중에 이를 방지합니다.


### Refactor duplicated checks into modifiers

**Description:** 중복된 확인을 수정자(modifiers)로 리팩터링하세요:
* `contracts/compliance/LockManager.sol`
```solidity
// 1) invalid address
compliance/InvestorLockManager.sol
60:        require(_to != address(0), "Invalid address");
108:        require(_to != address(0), "Invalid address");
132:        require(_who != address(0), "Invalid address");
152:        require(_who != address(0), "Invalid address");

compliance/LockManager.sol
96:        require(_to != address(0), "Invalid address");
109:        require(_to != address(0), "Invalid address");
144:        require(_who != address(0), "Invalid address");
159:        require(_who != address(0), "Invalid address");

token/TokenLibrary.sol
76:        require(_params._to != address(0), "Invalid address");
142:        require(_from != address(0), "Invalid address");
143:        require(_to != address(0), "Invalid address");

// 2) invalid time
compliance/ComplianceServiceNotRegulated.sol
65:        require(_time > 0, "Time must be greater than zero");

compliance/InvestorLockManager.sol
177:        require(_time > 0, "Time must be greater than zero");

compliance/ComplianceServiceWhitelisted.sol
116:        require(_time > 0, "Time must be greater than zero");

compliance/LockManager.sol
169:        require(_time > 0, "Time must be greater than zero");

compliance/ComplianceServiceRegulated.sol
706:        require(_time != 0, "Time must be greater than zero");

// 3) max number of wallets
compliance/WalletManager.sol
74:        require(_wallets.length <= 30, "Exceeded the maximum number of wallets");
96:        require(_wallets.length <= 30, "Exceeded the maximum number of wallets");
```

**Securitize:** **Cyfrin:**


### `InvestorLockManager::createLockForInvestor`, `removeLockRecordForInvestor` should revert for invalid investor id

**Description:** `InvestorLockManager::createLockForInvestor`는 잘못된 투자자 ID에 대해 되돌려야 합니다:
```diff
    function createLockForInvestor(string memory _investor, uint256 _valueLocked, uint256 _reasonCode, string calldata _reasonString, uint256 _releaseTime)
        public
        override
        validLock(_valueLocked, _releaseTime)
        onlyTransferAgentOrAboveOrToken
    {
+    require(!CommonUtils.isEmptyString(_investor), "Unknown investor");
```

`removeLockRecordForInvestor`에도 동일하게 적용됩니다 - 아마도 수정자를 만들고 두 함수 모두에서 해당 수정자를 사용할 수 있습니다.

**Securitize:** 인정함(Acknowledged); 이것은 사실이지만 체인에서 실제로 생성되기 전에 investorId를 완전히 잠글 수 있게 해줍니다. 투자자 ID를 미리 알고 있는 경우가 있으며 그런 경우 이를 사용할 수 있습니다.


### Refactor away duplicated code between `ComplianceService::newPreTransferCheck` and `preTransferCheck`

**Description:** `ComplianceService::newPreTransferCheck`와 `preTransferCheck`는 정확히 같은 일을 수행하며, 유일한 차이점은 다음과 같습니다:
* `newPreTransferCheck`는 `balanceFrom`과 `paused`를 입력 매개변수로 받습니다.
* `preTransferCheck`는 그렇지 않으므로 직접 조회해야 합니다.

따라서 코드 중복을 제거하기 위해 `preTransferCheck`는 조회한 매개변수로 `newPreTransferCheck`를 호출해야 합니다. 예:
```solidity
    function preTransferCheck(
        address _from,
        address _to,
        uint256 _value
    ) public view virtual override returns (uint256 code, string memory reason) {
        IDSToken token = getToken();
        return newPreTransferCheck(_from, _to, _value, token.balanceOf(_from), token.isPaused());
    }
```

**Securitize:** 커밋 [3b1894a](https://github.com/securitize-io/dstoken/commit/3b1894af9b02fc10f41dd697122f6339db518d2f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Cache compliance service and compliance configuration service for much cleaner code in `ComplianceServiceRegulated::completeTransferCheck`

**Description:** `ComplianceServiceRegulated::completeTransferCheck`에는 `ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE])` 및 `IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE])`로 가득 차 있어서 매우 크고 이해하기 어려운 `if` 문이 많이 있습니다.

이 두 값을 로컬 변수에 캐싱하면 쉽게 개선할 수 있습니다:
```solidity
ComplianceServiceRegulated complServiceReg = ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE]);
IDSComplianceConfigurationService complConfigService
    = IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]);
```

그런 다음 `if` 조건에서 `complServiceReg`와 `complConfigService`를 사용하면 조건이 대폭 단순화되어 가독성이 크게 향상됩니다.

**Securitize:** 인정함(Acknowledged); 이것은 과거에 분석되었으며 로컬 변수를 사용할 때 의도적인 균형이 있습니다. 역사적으로, 우리는 너무 많은 로컬 변수를 사용할 때 "Stack Too Deep"과 같은 오류로 인한 많은 문제를 겪었으며, 가독성이나 가스 사용량을 크게 개선하지 못했습니다. 현재 의도한 대로 작동하고 있으므로 이 프로세스를 다시 진행하지 않기를 선호합니다.


### `ComplianceServiceRegulated::getServices` should use constants from `ComplianceServiceLibrary` when setting array indexes

**Description:** `ComplianceServiceRegulated::getServices`는 배열 인덱스를 설정할 때 `ComplianceServiceLibrary`의 상수를 사용해야 합니다:
```solidity
function getServices() internal view returns (address[] memory services) {
    services = new address[](6);
    services[ComplianceServiceLibrary.DS_TOKEN] = getDSService(DS_TOKEN);
    services[ComplianceServiceLibrary.REGISTRY_SERVICE] = getDSService(REGISTRY_SERVICE);
    services[ComplianceServiceLibrary.WALLET_MANAGER] = getDSService(WALLET_MANAGER);
    services[ComplianceServiceLibrary.COMPLIANCE_CONFIGURATION_SERVICE] = getDSService(COMPLIANCE_CONFIGURATION_SERVICE);
    services[ComplianceServiceLibrary.LOCK_MANAGER] = getDSService(LOCK_MANAGER);
    services[ComplianceServiceLibrary.COMPLIANCE_SERVICE] = address(this);
}
```

**Securitize:** 인정함(Acknowledged).


### Using `block.number` instead of `block.timestamp` as an expiration check to execute signature

**Description:**
`TransactionRelayer::executeByInvestorWithBlockLimit()`에서, `params` 배열의 3번째 값은 `block.number`와 비교되며, 목적은 오래된 서명을 실행하는 것을 방지하는 것입니다. 시간 의존적 작업을 시행할 때 `block.number`를 사용하는 것은 권장되지 않습니다.

```solidity
    function executeByInvestorWithBlockLimit(
        ...
        uint256[] memory params
    ) public {
        ...
        require(params[2] >= block.number, "Transaction too old");
        ...
    }

```

`block.number` 시간 길이는 체인마다 다르기 때문에 `block.timestamp`를 사용하는 것이 가장 좋습니다.

**Securitize:**

**Cyfrin:**




### Remove useless function `ComplianceServiceRegulated::adjustTransferCounts`

**Description:** `ComplianceServiceRegulated::adjustTransferCounts` 함수는 항상 두 매개변수로 `adjustTotalInvestorsCounts`를 호출하므로 쓸모가 없습니다; 이를 제거하고 `adjustTotalInvestorsCounts`를 직접 호출하세요:
```diff
    function recordTransfer(
        address _from,
        address _to,
        uint256 _value
    ) internal override returns (bool) {
        if (compareInvestorBalance(_to, _value, 0)) {
-            adjustTransferCounts(_to, CommonUtils.IncDec.Increase);
+            adjustTotalInvestorsCounts(_to, CommonUtils.IncDec.Increase);
        }

        if (compareInvestorBalance(_from, _value, _value)) {
            adjustTotalInvestorsCounts(_from, CommonUtils.IncDec.Decrease);
        }

        cleanupInvestorIssuances(_from);
        cleanupInvestorIssuances(_to);
        return true;
    }

-    function adjustTransferCounts(
-        address _from,
-        CommonUtils.IncDec _increase
-    ) internal {
-        adjustTotalInvestorsCounts(_from, _increase);
-    }
```

**Securitize:** 커밋 [5e524cd](https://github.com/securitize-io/dstoken/commit/5e524cdb10bffc7c118ad0f19bfed315b098f795)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Protocol classifies retail investors based solely on whether they are qualified investors, without checking if they are accredited investors

**Description:** 프로토콜은 "소매 투자자(retail investors)"를 적격(accredited) 투자자인지 확인하지 않고 오로지 자격 있는(qualified) 투자자인지 여부에 따라 분류합니다. 코드베이스 전반에 걸쳐 소매 투자자는 `!RegistryService::isQualifiedInvestor()`를 사용하여 식별되지만 미국 투자자의 경우 올바른 정의는 `!isQualifiedInvestor() && !isAccreditedInvestor()`여야 합니다.

**Impact:** 현재 구성에서:
* `isRetail` 및 기타 관련 확인은 EU 투자자에 대해서만 수행됩니다.
* 비소매 EU 투자자는 `Qualified` 속성을 사용하여 표시됩니다.

따라서 현재 코드는 EU 투자자에게만 사용되므로 괜찮아 보이지만, 개발자가 미국 투자자와 관련하여 `isRetail`을 호출하거나 다른 곳에서 미국 투자자에 대해 이 패턴을 채택하면 미래에 버그가 쉽게 도입될 수 있습니다.

**Recommended Mitigation:** `isRetail`의 이름을 바꾸거나 바로 위에 설명 주석을 추가하여 이 함수가 EU 투자자에게만 사용되어야 함을 나타내는 것을 고려하세요.

**Securitize:** 이 함수는 EU 투자자에게만 사용되어야 하므로 이는 올바른 동작입니다; [6aef381](https://github.com/securitize-io/dstoken/commit/6aef381b1c88df0a376557f41bd900af8f1a0fe1) 커밋에서 이를 설명하는 주석을 추가했습니다.

**Cyfrin:** 검증되었습니다.


### Not possible to rehash `DOMAIN_SEPARATOR` in `MultiSigWallet` to update chainId if is ever required

**Description:** `MultiSigWallet`에는 계약이 배포될 때 초기 DOMAIN_SEPARATOR에 사용되었던 chainId를 업데이트해야 하는 경우 `DOMAIN_SEPARATOR`를 다시 해시(rehash)하는 함수가 없습니다.

**Recommended Mitigation:** MultiSigWallet에서 `DOMAIN_SEPARATOR`를 업데이트할 수 있도록 `TransactionRelayer::updatedomainSeparator()`와 유사한 함수 추가를 고려하세요.

**Securitize:** `MultiSigWallet`은 더 이상 사용되지 않아 제거되었습니다.

**Cyfrin:** 검증되었습니다.


### `ComplianceConfigurationService::getWorldWideForceFullTransfer` is not applied to US investors

**Description:** `ComplianceConfigurationService::getWorldWideForceFullTransfer` 함수 이름은 모든 지역에 걸쳐 전 세계적으로 적용됨을 암시하지만, 구현은 이 설정을 미국 이외의 투자자에게만 적용합니다. `completeTransferCheck` 함수에서 전 세계 설정은 비 미국 투자자 코드 경로에서만 확인되는 반면, 미국 투자자는 지역별 `getForceFullTransfer()` 설정만 적용됩니다:
```solidity
 if (_args.fromRegion == US) {
           ...
            }
        } else {
           ...
                IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getWorldWideForceFullTransfer() &&
                _args.fromInvestorBalance > _args.value
            ) { //@audit the getgetWorldWideForceFullTransfer is not being checked for us investor?
                return (50, ONLY_FULL_TRANSFER);
            }
        }

```

이는 "전 세계(worldwide)" 규정 준수 설정이 실제로 전 세계에 적용되지 않는 근본적인 불일치를 생성합니다.

**Impact:** 미국 투자자는 `ComplianceConfigurationService::getWorldWideForceFullTransfer` 제한을 우회할 수 있습니다.

**Proof of Concept:** `dstoken-regulated.test.ts`에서 다음 개념 증명을 실행하세요:
```typescript
describe('Worldwide Force Full Transfer Bug POC', function () {
        it('Should demonstrate that getWorldWideForceFullTransfer is not applied to US investors', async function () {
          const [usInvestor, nonUsInvestor, differentInvestor] = await hre.ethers.getSigners();
          const { dsToken, registryService, complianceService, complianceConfigurationService } = await loadFixture(deployDSTokenRegulatedWithRebasingAndEighteenDecimal);

          // Setup: Register investors
          await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, usInvestor, registryService);
          await registerInvestor(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, nonUsInvestor, registryService);
          await registerInvestor(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, differentInvestor, registryService);

          // Set countries
          await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_1, INVESTORS.Country.USA);
          await registryService.setCountry(INVESTORS.INVESTOR_ID.INVESTOR_ID_2, INVESTORS.Country.GERMANY);
          await registryService.setCountry(INVESTORS.INVESTOR_ID.US_INVESTOR_ID_2, INVESTORS.Country.FRANCE);

          // Set country compliance
          await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.USA, 1); // US = compliant
          await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.GERMANY, 2); // Germany = EU
          await complianceConfigurationService.setCountryCompliance(INVESTORS.Country.FRANCE, 2); // France = EU

          // Set all compliance rules - no limits and short lock period for testing
          await complianceConfigurationService.setAll(
            [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 150, 1, 1, 0], // No investor limits, 1 second lock period
            [false, false, false, false, false] // Disable all compliance checks initially
          );

          // Issue tokens to both investors
          await dsToken.issueTokens(usInvestor.address, 200);
          await dsToken.issueTokens(nonUsInvestor.address, 200);

          // Wait for lock period to expire
          await time.increase(2); // Wait 2 seconds to ensure lock period expires

          // Configure compliance settings
          await complianceConfigurationService.setForceFullTransfer(false); // Disable US-specific force full transfer
          await complianceConfigurationService.setWorldWideForceFullTransfer(true); // Enable worldwide force full transfer

          // Verify balances
          expect(await dsToken.balanceOf(usInvestor.address)).to.equal(200);
          expect(await dsToken.balanceOf(nonUsInvestor.address)).to.equal(200);

          // Non-US investor partial transfer - should fail (correct behavior)
          await expect(
            dsToken.connect(nonUsInvestor).transfer(differentInvestor.address, 100)
          ).to.be.revertedWith('Only full transfer');

           // US investor partial transfer - should succeed due to bug
           await dsToken.connect(usInvestor).transfer(differentInvestor.address, 100);
           expect(await dsToken.balanceOf(usInvestor.address)).to.equal(100);
           expect(await dsToken.balanceOf(differentInvestor.address)).to.equal(100);

        });
      });
```


**Recommended Mitigation:** 미국 투자자 코드 경로도 전 세계 설정을 확인하도록 수정하여 `ComplianceConfigurationService::getWorldWideForceFullTransfer`가 실제로 전 세계적으로 적용되도록 하세요.

**Securitize:** 인정함(Acknowledged); 의도된 설계(by design)입니다.


### `ComplianceServiceRegulated` and its parent `ComplianceServiceWhitelisted` uses a chain of `initializer` modifiers when calling the `initialize`

**Description:** `ComplianceServiceRegulated`는 `ComplianceServiceWhitelisted`를 상속받은 자식 계약입니다.
`ComplianceServiceRegulated::initialize()`는 `initializer modifier`를 사용하며 `ComplianceServiceWhitelisted:initialize()`도 마찬가지입니다.

```solidity
contract ComplianceServiceRegulated is ComplianceServiceWhitelisted {
    function initialize() public virtual override onlyProxy initializer {
        super.initialize();
    }
}

contract ComplianceServiceWhitelisted is ComplianceService {
    function initialize() public virtual override onlyProxy initializer {
        ComplianceService.initialize();
    }
}
```

[OpenZeppelin의 문서](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-initializer--) 및 모범 사례에 따르면, initializer 수정자는 상속 체인의 최종 초기화 함수에서만 사용해야 하며, 부모 계약의 초기화 함수는 [onlyInitializing](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-onlyInitializing--) 수정자를 사용해야 합니다. 이는 상속을 사용할 때 적절한 초기화를 보장합니다.


**Recommended Mitigation:** * `ComplianceServiceWhitelisted`를 다음과 같이 변경하세요:
```solidity
contract ComplianceServiceWhitelisted is ComplianceService {

    function initialize() public virtual override onlyProxy initializer {
        _initialize();
    }

    function _initialize() internal onlyInitializing {
        ComplianceService.initialize();
    }
```

* `ComplianceServiceRegulated`를 다음과 같이 변경하세요:
```solidity
contract ComplianceServiceRegulated is ComplianceServiceWhitelisted {

    function initialize() public virtual override onlyProxy initializer {
        _initialize();
    }
```

**Securitize:** 커밋 [b24ecd5](https://github.com/securitize-io/dstoken/commit/b24ecd57bc82d0fd473d9de1317046300faab984)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Feature flags are not being used in the `TokenLibrary`

**Description:** `TokenLibrary::setFeature`는 현재 코드베이스에서 완전히 작동하지 않는 기능 플래그 시스템을 구현합니다. 기능을 활성화/비활성화하는 인프라는 존재하지만, 실제로 이러한 기능 플래그를 확인하는 비즈니스 로직은 없습니다; 이 코드가 더 이상 사용되지 않거나 쓸모없는 경우 제거를 고려하세요.

```solidity
 function setFeature(SupportedFeatures storage supportedFeatures, uint8 featureIndex, bool enable) public {
        uint256 base = 2;
        uint256 mask = base**featureIndex;

        // Enable only if the feature is turned off and disable only if the feature is turned on
        if (enable && (supportedFeatures.value & mask == 0)) {
            supportedFeatures.value = supportedFeatures.value ^ mask;
        } else if (!enable && (supportedFeatures.value & mask >= 1)) {
            supportedFeatures.value = supportedFeatures.value ^ mask;
        }
    }
```

**Securitize:** 인정함(Acknowledged).

\clearpage
## Gas Optimization


### Don't perform storage reads unless necessary

**Description:** 스토리지에서 읽는 것은 비용이 많이 듭니다; 필요하지 않은 경우 스토리지 읽기를 수행하지 마세요.

* `contracts/service/ServiceConsumer.sol`
```diff
// in `onlyMaster` don't load trust manager unless required
    modifier onlyMaster {
-        IDSTrustService trustManager = getTrustService();
-        require(owner() == msg.sender || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
+        if(owner() != msg.sender) require(getTrustService().getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
        _;
    }
```

**Securitize:** 커밋 [1ab6fd9](https://github.com/securitize-io/dstoken/commit/1ab6fd99a2a6814b23ec7e51359ced72159bfe80)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Cache result of identical external calls when the result can't change

**Description:** 결과가 변경될 수 없는 경우 동일한 외부 호출의 결과를 캐시하세요.

* `contracts/service/ServiceConsumer.sol`
```solidity
// in the modifiers, cache `trustManager.getRole(msg.sender)` and
// use the cached value inside the `require` statements
59:        require(trustManager.getRole(msg.sender) == ROLE_TRANSFER_AGENT || trustManager.getRole(msg.sender) == ROLE_ISSUER || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
65:        require(trustManager.getRole(msg.sender) == ROLE_ISSUER || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
71:        require(trustManager.getRole(msg.sender) == ROLE_TRANSFER_AGENT || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
77:        require(
78:            trustManager.getRole(msg.sender) == ROLE_EXCHANGE
79:            || trustManager.getRole(msg.sender) == ROLE_ISSUER
80:            || trustManager.getRole(msg.sender) == ROLE_TRANSFER_AGENT
81:            || trustManager.getRole(msg.sender) == ROLE_MASTER,
100:            require(trustManager.getRole(msg.sender) == ROLE_ISSUER || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
108:            require(trustManager.getRole(msg.sender) == ROLE_TRANSFER_AGENT || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
116:            require(trustManager.getRole(msg.sender) == ROLE_ISSUER || trustManager.getRole(msg.sender) == ROLE_MASTER, "Insufficient trust level");
```

* `contracts/swap/SecuritizeSwap.sol`
```solidity
// cache `navProvider.rate` in `buy`, also change `calculateStableCoinAmount` to take the rate
// as an input to save another call inside it
L132:        require(navProvider.rate() > 0, "NAV Rate must be greater than 0");
L141:        emit Buy(msg.sender, _dsTokenAmount, stableCoinAmount, navProvider.rate());
L240:        return _dsTokenAmount * navProvider.rate() / (10 ** ERC20(address(dsToken)).decimals());
```

* `contracts/compliance/ComplianceServiceRegulated.sol`
```solidity
// cache these two external calls in `getUSInvestorsLimit`
103:        if (compConfService.getMaxUSInvestorsPercentage() == 0) {
107:        if (compConfService.getUSInvestorsLimit() == 0) {

// cache `getInvestor(_to)` in `preIssuanceCheck`
420:        string memory toCountry = IDSRegistryService(_services[REGISTRY_SERVICE]).getCountry(IDSRegistryService(_services[REGISTRY_SERVICE]).getInvestor(_to));
431:        if (IDSLockManager(_services[LOCK_MANAGER]).isInvestorLiquidateOnly(IDSRegistryService(_services[REGISTRY_SERVICE]).getInvestor(_to))) {
```

**Securitize:** 인정함(Acknowledged).


### Use `calldata` for read-only external inputs

**Description:** 읽기 전용 외부 입력에는 `calldata`를 사용하세요:
* `BulkOperator::bulkIssuance`, `bulkRegisterAndIssuance`, `bulkBurn`
* `TokenIssuer::issueTokens`, `registerInvestor`
* `SecuritizeSwap::nonceByInvestor`, `swap`, `executePreApprovedTransaction`, `doExecuteByInvestor`, `_registerNewInvestor`
* `WalletManager::addIssuerWallets`, `addPlatformWallets`
* `InvestorLockManagerBase::lockInvestor`, `unlockInvestor`, `isInvestorLocked`, `setInvestorLiquidateOnly`, `isInvestorLiquidateOnly` (및 관련 클래스 `IDSLockManager`, `LockManager`의 유사한 함수들)
* `ComplianceConfigurationService::getCountryCompliance`
* `ComplianceServiceRegulated::doPreTransferCheckRegulated`, `completeTransferCheck`의 `_services`

**Securitize:** 우리는:
* 구식이므로 이러한 계약 중 다수를 삭제했습니다.
* "stack-too-deep" 오류가 발생하지 않는 일부 위치에서 `calldata`를 사용했습니다.

**Cyfrin:** 검증되었습니다.


### In Solidity don't initialize to default values

**Description:** Solidity에서는 기본값으로 초기화하지 마세요:
```solidity
multicall/IssuerMulticall.sol
31:        for (uint256 i = 0; i < data.length; i++) {

service/ServiceConsumer.sol
38:    uint8 public constant ROLE_NONE = 0;

compliance/ComplianceConfigurationService.sol
34:        for (uint i = 0; i < _countries.length; i++) {

compliance/ComplianceServiceRegulated.sol
27:    uint256 internal constant DS_TOKEN = 0;
34:    uint256 internal constant NONE = 0;
718:        uint256 totalLockedTokens = 0;
720:        for (uint256 i = 0; i < investorIssuancesCount; i++) {
815:        uint256 currentIndex = 0;

trust/TrustService.sol
218:        for (uint i = 0; i < _addresses.length; i++) {

compliance/IDSComplianceService.sol
23:    uint256 internal constant NONE = 0;

trust/IDSTrustService.sol
40:    uint8 public constant NONE = 0;

compliance/WalletManager.sol
75:        for (uint i = 0; i < _wallets.length; i++) {
97:        for (uint i = 0; i < _wallets.length; i++) {

compliance/InvestorLockManager.sol
190:        uint256 totalLockedTokens = 0;
191:        for (uint256 i = 0; i < investorLockCount; i++) {

registry/WalletRegistrar.sol
48:        for (uint256 i = 0; i < _wallets.length; i++) {
56:        for (uint256 i = 0; i < _attributeIds.length; i++) {

registry/IDSRegistryService.sol
34:    uint8 public constant NONE = 0;
40:    uint8 public constant PENDING = 0;

registry/RegistryService.sol
43:        for (uint8 index = 0; index < 16; index++) {
74:        for (uint256 i = 0; i < _wallets.length; i++) {
82:        for (uint256 i = 0; i < _attributeIds.length; i++) {
99:        for (uint8 i = 0; i < 4; i++) {

token/DSToken.sol
29:    uint256 internal constant DEPRECATED_OMNIBUS_NO_ACTION = 0;  // Deprecated, kept for backward compatibility

token/TokenLibrary.sol
29:    uint256 internal constant COMPLIANCE_SERVICE = 0;
31:    uint256 internal constant DEPRECATED_OMNIBUS_NO_ACTION = 0; // Deprecated, keep for backwards compatibility
96:        uint256 totalLocked = 0;
97:        for (uint256 i = 0; i < _params._valuesLocked.length; i++) {

swap/SecuritizeSwap.sol
222:        for (uint256 i = 0; i < _investorAttributeIds.length; i++) {

compliance/ComplianceServiceNotRegulated.sol
48:        code = 0;

utils/BulkBalanceChecker.sol
39:        for (uint256 i = 0; i < length; i++) {

utils/MultiSigWallet.sol
61:        for (uint256 i = 0; i < owners_.length; i++) {
113:        for (uint256 i = 0; i < threshold; i++) {
123:        bool success = false;

compliance/LockManager.sol
180:        uint256 totalLockedTokens = 0;
181:        for (uint256 i = 0; i < investorLockCount; i++) {

compliance/IDSWalletManager.sol
26:    uint8 public constant NONE = 0;

bulk/BulkOperator.sol
54:        for (uint256 i = 0; i < addresses.length; i++) {
61:        for (uint256 i = 0; i < data.length; i++) {
80:        for (uint256 i = 0; i < addresses.length; i++) {

utils/TransactionRelayer.sol
191:        bool success = false;
```

**Securitize:** 인정함(Acknowledged).


### Cache storage to prevent identical storage reads

**Description:** 스토리지에서 읽는 것은 비용이 많이 듭니다; 동일한 스토리지 읽기를 방지하기 위해 스토리지를 캐시하세요:

* `contracts/registry/RegistryService.sol`:
```solidity
// cache `getInvestor(_address)` in `getInvestorDetails`
206:        return (getInvestor(_address), getCountry(getInvestor(_address)));
```

* `contracts/issuance/TokenIssuer.sol`
```solidity
// cache `getRegistryService()` in `issueTokens` and `registerInvestor`
44:        if (getRegistryService().isWallet(_to)) {
45:            require(CommonUtils.isEqualString(getRegistryService().getInvestor(_to), _id), "Wallet does not belong to investor");
63:        if (!getRegistryService().isInvestor(_id)) {
64:            getRegistryService().registerInvestor(_id, _collisionHash);
65:            getRegistryService().setCountry(_id, _country);
69:                getRegistryService().setAttribute(_id, KYC_APPROVED, _attributeValues[0], _attributeExpirations[0], "");
70:                getRegistryService().setAttribute(_id, ACCREDITED, _attributeValues[1], _attributeExpirations[1], "");
71:                getRegistryService().setAttribute(_id, QUALIFIED, _attributeValues[2], _attributeExpirations[2], "");
```

* `contracts/token/TokenLibrary.sol`
```solidity
// cache `supportedFeatures.value` in `setFeature`
62:        if (enable && (supportedFeatures.value & mask == 0)) {
63:            supportedFeatures.value = supportedFeatures.value ^ mask;
64:        } else if (!enable && (supportedFeatures.value & mask >= 1)) {
65:            supportedFeatures.value = supportedFeatures.value ^ mask;

// This function can also delete the `base` variable since it is hard-coded and
// only used once so no point to it.
```

* `contracts/token/StandardToken.sol`
```solidity
// cache `allowances[msg.sender][_spender] + _addedValue` then use it
// when writing storage and emitting event. Do something similar in `decreaseApproval`
// to avoid re-reading storage when emitting the event
128:        allowances[msg.sender][_spender] = allowances[msg.sender][_spender] + _addedValue;
129:        emit Approval(msg.sender, _spender, allowances[msg.sender][_spender]);
```

* `contracts/trust/TrustService.sol`
```solidity
// cache `roles[msg.sender]` in modifiers
70:        require(roles[msg.sender] == MASTER || roles[msg.sender] == ISSUER, "Not enough permissions");
86:        if (roles[msg.sender] != MASTER) {
87:            if (roles[msg.sender] == ISSUER) {
90:                require(roles[msg.sender] == _role, "Not enough permissions. Only same role allowed");
100:        if (roles[msg.sender] != MASTER) {
102:            if (roles[msg.sender] == ISSUER) {
105:                require(roles[msg.sender] == role, "Not enough permissions. Only same role allowed");

// cache `roles[msg.sender]` and `ownersEntities[msg.sender]` in `onlyEntityOwnerOrAbove`
// ideally here perform the `roles[msg.sender]` checks first and only perform the `ownersEntities[msg.sender]`
// afterwards if required
113:            roles[msg.sender] == MASTER ||
114:                roles[msg.sender] == ISSUER ||
115:                (!CommonUtils.isEmptyString(ownersEntities[msg.sender]) &&
116:                  CommonUtils.isEqualString(ownersEntities[msg.sender], _name)),

// cache `ownersEntities[_owner]` in `onlyExistingEntityOwner`
140:            !CommonUtils.isEmptyString(ownersEntities[_owner]) &&
141:            CommonUtils.isEqualString(ownersEntities[_owner], _name),

// cache `operatorsEntities[_operator]` in `onlyExistingOperator`
154:            !CommonUtils.isEmptyString(operatorsEntities[_operator]) &&
155:            CommonUtils.isEqualString(operatorsEntities[_operator], _name),

// cache `resourcesEntities[_resource]` in `onlyExistingResource`
168:            !CommonUtils.isEmptyString(resourcesEntities[_resource]) &&
169:           CommonUtils.isEqualString(resourcesEntities[_resource], _name),
```

* `contracts/swap/SecuritizeSwap.sol`
```solidity
// cache `IDSRegistryService(getDSService(REGISTRY_SERVICE))` in `swap`
// pass it in as a parameter to `_registerNewInvestor` to save another identical storage read
103:        if (!IDSRegistryService(getDSService(REGISTRY_SERVICE)).isInvestor(_senderInvestorId)) {
114:        string memory investorWithNewWallet = IDSRegistryService(getDSService(REGISTRY_SERVICE)).getInvestor(_newInvestorWallet);
116:            IDSRegistryService(getDSService(REGISTRY_SERVICE)).addWallet(_newInvestorWallet, _senderInvestorId);
```solidity
214:        IDSRegistryService registryService = IDSRegistryService(getDSService(REGISTRY_SERVICE));

// cache `USDCBridge` and `bridgeChainId` in `executeStableCoinTransfer`
// a more optimized implementation looks like this:
function executeStableCoinTransfer(address from, uint256 value) private {
    // 1 SLOAD since both stored in the same slot
    (IUSDCBridge USDCBridgeCache, uint16 bridgeChainIdCache) = (USDCBridge, bridgeChainId);

    if (bridgeChainIdCache != 0 && address(USDCBridgeCache) != address(0)) {
        stableCoinToken.transferFrom(from, address(this), value);
        stableCoinToken.approve(address(USDCBridgeCache), value);
        USDCBridgeCache.sendUSDCCrossChainDeposit(bridgeChainIdCache, issuerWallet, value);
    } else {
        stableCoinToken.transferFrom(from, issuerWallet, value);
    }
}
```

* `contracts/compliance/ComplianceServiceWhitelisted.sol`
```solidity
// cache `getToken()` in `preTransferCheck`
50:        return doPreTransferCheckWhitelisted(_from, _to, _value, getToken().balanceOf(_from), getToken().isPaused());
```

* `contracts/compliance/ComplianceServiceRegulated.sol`
```solidity
// cache `getRegistryService()` and `getComplianceConfigurationService()` in `recordTransfer`
// ideally these would be cached upstream and passed down to functions such as `recordTransfer`
// that need them
800:        string memory investor = getRegistryService().getInvestor(_who);
801:        string memory country = getRegistryService().getCountry(investor);

803:        uint256 region = getComplianceConfigurationService().getCountryCompliance(country);
807:            lockTime = getComplianceConfigurationService().getUSLockPeriod();
809:            lockTime = getComplianceConfigurationService().getNonUSLockPeriod();
```

**Securitize:** 인정함(Acknowledged).


### Use named return variables where this optimizes away a local variable definition

**Description:** 로컬 변수 정의를 최적화하여 없앨 수 있는 경우 명명된 반환 변수(named return variables)를 사용하고, 마지막에 있는 쓸모없는 `return` 문도 제거하세요:
* `StandardToken::balanceOf`, `totalSupply`
* `DSToken::totalIssued`, `balanceOfInvestor`, `getCommonServices`
* `TokenLibrary::issueTokensCustom`, `issueTokensWithNoCompliance`, `burn`
* `RegistryService::getInvestorDetailsFull`
* `SecuritizeSwap::calculateDsTokenAmount`
* `MulticallProxy::_slice`, `_callTarget`
* `LockManager::getTransferableTokens`
* `InvestorLockManager::getTransferableTokens`
* `ComplianceConfigurationService::getAll`

**Securitize:** 인정함(Acknowledged).


### Since attribute expiration is deprecated, remove as input parameters, don't write it to storage and put comment explaining this

**Description:** `RegistryService::setAttribute`는 각 속성이 만료되어야 하는 시점을 나타내는 `_expiry` 필드를 설정합니다.

그러나 이것은 어디에서도 확인되지 않으려, 예를 들어 사용자가 적격(accredited) 또는 자격(qualified)이 있는지 여부를 결정하는 이러한 함수들은 만료를 확인하지 않습니다:
```solidity
    function isAccreditedInvestor(string calldata _id) external view override returns (bool) {
        return getAttributeValue(_id, ACCREDITED) == APPROVED;
    }

    function isAccreditedInvestor(address _wallet) external view override returns (bool) {
        string memory investor = investorsWallets[_wallet].owner;
        return getAttributeValue(investor, ACCREDITED) == APPROVED;
    }

    function isQualifiedInvestor(address _wallet) external view override returns (bool) {
        string memory investor = investorsWallets[_wallet].owner;
        return getAttributeValue(investor, QUALIFIED) == APPROVED;
    }

    function isQualifiedInvestor(string calldata _id) external view override returns (bool) {
        return getAttributeValue(_id, QUALIFIED) == APPROVED;
    }
```

클라이언트에 문의한 결과 속성 만료는 더 이상 사용되지 않으며 실제로 어디에서도 사용되지 않으며, 실제로는 만료 입력에 대해 0을 전달하고 있다고 합니다.

**Recommended Mitigation:** 이상적으로는 함수에서 모든 속성 입력을 제거해야 하지만, 이는 기존 인터페이스를 깨트리므로 더 침습적입니다.

최소한 `RegistryService::setAttribute`가 `attributes[_id][_attributeId].expiry`를 설정하지 않도록 변경하고(기본값 0으로 남겨둠), 관련 데이터 저장소에 사용 중단을 알리는 주석을 추가하세요:
```solidity
data-stores/RegistryServiceDataStore.sol
26:        uint256 expiry;
```

**Securitize:** 인정함(Acknowledged).


### Remove setting deprecated `lastUpdatedBy` in RegistryService

**Description:** 클라이언트는 `lastUpdatedBy`가 더 이상 사용되지 않으므로 `RegistryService`에서 업데이트해서는 안 된다고 알렸습니다:
```solidity
registry/RegistryService.sol
113:        investors[_id].lastUpdatedBy = msg.sender;
140:        investors[_id].lastUpdatedBy = msg.sender;
```

또한 관련 데이터 저장소에 이를 나타내는 주석을 추가해야 합니다:
```solidity
data-stores/RegistryServiceDataStore.sol
33:        address lastUpdatedBy;
40:        address lastUpdatedBy;
```

또는 다른 곳에서 더 이상 사용되지 않는 스토리지 슬롯에 대해 수행된 것처럼 스토리지 슬롯의 이름을 `DEPRECATED_lastUpdatedBy`로 변경해야 합니다.

**Securitize:** 커밋 [9a80a47](https://github.com/securitize-io/dstoken/commit/9a80a478fffcf7ef81e5c0ca229ab2ff4efc7b9e)에서 `lastUpdatedBy`에 더 이상 쓰지 않음으로써 수정되었고, 커밋 [e6165e4](https://github.com/securitize-io/dstoken/commit/e6165e4ae5c29ca1787bbf90dd62c83a6e915ba6)에서 변수 이름을 변경하여 더 이상 사용되지 않음을 명시적으로 나타내어 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Cheaper not to cache `calldata` array length

**Description:** `calldata` 배열 길이를 캐시하지 않는 것이 [더 저렴합니다(cheaper)](https://github.com/devdacian/solidity-gas-optimization?tab=readme-ov-file#6-dont-cache-calldata-length-effective-009-cheaper):
* `BulkBalanceChecker::getTokenBalances`

**Securitize:** 커밋 [fd6eb3b](https://github.com/securitize-io/dstoken/commit/fd6eb3bc4b075ec5975bc8d05305bfcdda054847)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### More efficient way of checking for empty string in `CommonUtils::isEmptyString`

**Description:** `CommonUtils::isEmptyString`에서 빈 문자열을 확인하는 더 효율적인 방법:
```solidity
function isEmptyString(string memory _str) internal pure returns (bool) {
    return bytes(_str).length == 0;
}
```

**Securitize:** 커밋 [22b117a](https://github.com/securitize-io/dstoken/commit/22b117a3514c04b766aa7be6c855683865549e82)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### More efficient way of comparing two strings for equality in `CommonUtils::isEqualString`

**Description:** Solady의 `CommonUtils::isEqualString`에서 두 문자열의 동등성을 비교하는 더 효율적인 [방법](https://github.com/Vectorized/solady/blob/main/src/utils/g/LibString.sol#L858-L863):
```solidity
  function isEqualString(string memory a, string memory b) internal pure returns (bool result) {
      /// @solidity memory-safe-assembly
      assembly {
          result := eq(keccak256(add(a, 0x20), mload(a)), keccak256(add(b, 0x20), mload(b)))
      }
  }
```

**Securitize:** 인정함(Acknowledged).


### Cache computation results instead of repeatedly performing the same computation

**Description:** 동일한 계산을 반복적으로 수행하는 대신 계산 결과를 캐시하세요.

* `contracts/utils/TransactionRelayer.sol`
```solidity
// cache `toBytes32(investorId)` in `setInvestorNonce`
142:        uint256 investorNonce = noncePerInvestor[toBytes32(investorId)];
144:        noncePerInvestor[toBytes32(investorId)] = newNonce;

// cache `toBytes32(senderInvestor)` in `doExecuteByInvestor`
175:                        noncePerInvestor[toBytes32(senderInvestor)],
178:                        keccak256(abi.encodePacked(senderInvestor)),
190:        noncePerInvestor[toBytes32(senderInvestor)]++;
```

**Securitize:** 인정함(Acknowledged).


### Remove deprecated collision hash and proof hash from function calls

**Description:** 사용자 등록 시 충돌(collision) 해시와 속성 등록 시 증명(proof) 해시는 더 이상 사용되지 않습니다; 함수 인터페이스에서 제거하세요.

또한 `RegistryServiceDataStore`에서 `Attribute::proofHash` 및 `Investor::collisionHash` 옆에 주석을 추가하여 더 이상 사용되지 않으며 더 이상 사용되지 않음을 알립니다.

**Securitize:** 인정함(Acknowledged).


### Return fast in `ComplianceServiceRegulated::checkHoldUp` if platform wallet

**Description:** `ComplianceServiceRegulated::checkHoldUp`은 `_isPlatformWalletFrom == true`인 경우 빠르게 반환해야 합니다; 그 경우에는 모든 처리를 수행할 이유가 없습니다:
```solidity
    function checkHoldUp(
        address[] memory _services,
        address _from,
        uint256 _value,
        bool _isUSLockPeriod,
        bool _isPlatformWalletFrom
    ) internal view returns (bool hasHoldUp) {
        // platform wallets have no lock period so return false (default)
        // and skip all processing if it is a platform wallet
        if(!_isPlatformWalletFrom) {
            ComplianceServiceRegulated complianceService
                = ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE]);
            uint256 lockPeriod;
            if (_isUSLockPeriod) {
                lockPeriod = IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getUSLockPeriod();
            } else {
                lockPeriod = IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]).getNonUSLockPeriod();
            }

            hasHoldUp =
                complianceService.getComplianceTransferableTokens(
                    _from, block.timestamp, uint64(lockPeriod)) < _value;
        }
    }
```

**Securitize:** 커밋 [c407d0c](https://github.com/securitize-io/dstoken/commit/c407d0c5a7d702629db5a1fed1b65347078cea9d)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Perform local variable checks first prior to external calls in composite `if` statement conditions

**Description:** `if` 문 조건이 여러 복합 부분에서 `&&` 연산자를 사용하여 함께 결합된 경우, 외부 호출 전에 로컬 변수 확인을 먼저 수행해야 합니다. 로컬 변수 확인이 `false`로 평가되면 외부 호출을 수행할 필요가 없기 때문입니다.

`ComplianceServiceRegulated::completeTransferCheck`에서 이것이 발생하는 세 곳이 있습니다:
```solidity
253:            if (IDSComplianceConfigurationService(
254:                   _services[COMPLIANCE_CONFIGURATION_SERVICE]).getForceFullTransfer() &&
255:                _args.fromInvestorBalance > _args.value
256:            ) {

272:            if (IDSComplianceConfigurationService(.
273:                   _services[COMPLIANCE_CONFIGURATION_SERVICE]).getWorldWideForceFullTransfer() &&
274:                _args.fromInvestorBalance > _args.value
275:            ) {
```

대신 로컬 변수 확인을 먼저 수행하세요:
```solidity
            if (_args.fromInvestorBalance > _args.value &&
                IDSComplianceConfigurationService(
                   _services[COMPLIANCE_CONFIGURATION_SERVICE]).getForceFullTransfer()
            ) {

            if (_args.fromInvestorBalance > _args.value &&
                IDSComplianceConfigurationService(
                   _services[COMPLIANCE_CONFIGURATION_SERVICE]).getWorldWideForceFullTransfer()
            ) {
```

유사한 최적화를 다음에 적용할 수 있습니다:
* L284->287 EU 확인
* L290->292 적격성(accreditation) 확인

**Securitize:** **Cyfrin:**


### Fast fail without performing unnecessary storage reads or external calls

**Description:** 불필요한 스토리지 읽기나 외부 호출을 수행하지 않고 빠르게 실패하세요. 예를 들어 `ComplianceServiceRegulated::preIssuanceCheck`에서 함수의 시작 부분은 다음과 같습니다:
```solidity
function preIssuanceCheck(
    address[] calldata _services,
    address _to,
    uint256 _value
) public view returns (uint256 code, string memory reason) {
    ComplianceServiceRegulated complianceService = ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE]);
    IDSComplianceConfigurationService complianceConfigurationService = IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]);
    IDSWalletManager walletManager = IDSWalletManager(_services[WALLET_MANAGER]);
    string memory toCountry = IDSRegistryService(_services[REGISTRY_SERVICE]).getCountry(IDSRegistryService(_services[REGISTRY_SERVICE]).getInvestor(_to));
    uint256 toRegion = complianceConfigurationService.getCountryCompliance(toCountry);

    if (toRegion == FORBIDDEN) {
        return (26, DESTINATION_RESTRICTED);
    }

    if (!complianceService.checkWhitelisted(_to)) {
        return (20, WALLET_NOT_IN_REGISTRY_SERVICE);
    }
```

그러나 `_to`가 허용 목록(whitelisted)에 없어 함수가 반환될 경우, 중간에 관계없는 모든 스토리지 읽기 및 외부 호출을 수행하여 가스를 소비하는 것은 의미가 없습니다. 대신 스토리지 읽기 및 외부 호출은 필요한 경우에만 수행해야 합니다. 예:
```solidity
function preIssuanceCheck(
    address[] calldata _services,
    address _to,
    uint256 _value
) public view returns (uint256 code, string memory reason) {
    ComplianceServiceRegulated complianceService = ComplianceServiceRegulated(_services[COMPLIANCE_SERVICE]);
    // fail fast if not whitelisted
    if (!complianceService.checkWhitelisted(_to)) {
        return (20, WALLET_NOT_IN_REGISTRY_SERVICE);
    }

    IDSComplianceConfigurationService complianceConfigurationService = IDSComplianceConfigurationService(_services[COMPLIANCE_CONFIGURATION_SERVICE]);

     // don't need this until much later in the function so no point doing it here
    // IDSWalletManager walletManager = IDSWalletManager(_services[WALLET_MANAGER]);

    // add this to improve readability as this is used multiple times
    IDSRegistryService regService = IDSRegistryService(_services[REGISTRY_SERVICE]);

    string memory toCountry = regService.getCountry(regService.getInvestor(_to));
    uint256 toRegion = complianceConfigurationService.getCountryCompliance(toCountry);

    if (toRegion == FORBIDDEN) {
        return (26, DESTINATION_RESTRICTED);
    }

    // continue remaining processing, following the principles of failing fast by only
    // perform storage reads and external calls as they are needed
```

**Securitize:** 커밋 [80d536e](https://github.com/securitize-io/dstoken/commit/80d536ec48d316b08226fbe53ee8a0e793ec074a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Not possible to send native via the `TransactionRelayer` to the target contract when executing `executeByInvestorWithBlockLimit`

**Description:** `TransactionRelayer::executeByInvestorWithBlockLimit`은 `destination` 주소로 외부 호출을 수행합니다. 이 함수의 입력 매개변수 중 하나는 `value`이며, 이는 `params[0]`에 인코딩되어 있고, 이 매개변수는 `doExecuteByInvestor`에서 수행된 외부 호출에서 `destination` 주소로 전송될 기본(native) 잔액의 양을 지정하는 데 사용됩니다.

문제는 `TransactionRelayer::executeByInvestorWithBlockLimit`이 payable이 아니므로 txn의 일부로 native를 보낼 수 없다는 것입니다. 또한 `TransactionRelayer`는 한 계정에서 다른 계정으로 native를 전송하여 자금을 조달하려고 할 때 되돌립니다(reverts).
```solidity
    function doExecuteByInvestor(
        ...
        uint256[] memory params
    ) private {
       ...
        bool success = false;
        uint256 value = params[0];
        uint256 gasLimit = params[1];
        assembly {
            success := call(
            gasLimit,
            destination,
//@audit => Amount of native to transfer on the external call
            value,
            add(data, 0x20),
            mload(data),
            0,
            0
            )
        }
        require(success, "transaction was not executed");
    }
```

**Impact:** 대상 계약으로 보낼 기본 잔액을 포함하는 서명은 되돌려집니다.

**Proof of Concept:** 테스트 스위트에 다음 foundry PoC를 추가하세요.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

import "forge-std/Test.sol";
import "../contracts/utils/TransactionRelayer.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract MockTransactionRelayer is TransactionRelayer {
    function entryPoint(address target, uint256 _value) public {
        internalCall(target, _value);
    }

    function internalCall(address target, uint256 _value) internal {
        bool success = false;
        assembly {
            success := call(
            gas(),
            target,
            _value,
            0,
            0,
            0,
            0
            )
        }
        require(success, "call to target reverted");
    }
}

contract TransactionRelayerTest is Test {
    address implementation;
    MockTransactionRelayer transactionRelayer;

    function setUp() public {
        implementation = address(new MockTransactionRelayer());
        // bytes memory data = abi.encodeCall(TransactionRelayer.initialize, "");
        address proxy = address(new ERC1967Proxy(implementation, ""));
        transactionRelayer = MockTransactionRelayer(proxy);
    }

    function test_transactionRelayer() public {
        transactionRelayer.initialize();

        //@audit-info => Not possible to fund native to the TransactionRelayer
        vm.expectRevert();
        (bool success, ) = address(transactionRelayer).call{value: 1 ether}("");
        require(success);

        assertEq(address(transactionRelayer).balance, 0);

        //@audit-info => Not possible to transfer out native from the TransactionRelayer
        address user1 = makeAddr("user1");
        vm.expectRevert();
        transactionRelayer.entryPoint(user1, 1 ether);

        //@audit-info => Not possible to send native in the call because not payable modifier
        // transactionRelayer.entryPoint{value: 1 ether}(user1, 1 ether);
    }
}
```

**Recommended Mitigation:** `executeByInvestorWithBlockLimit` 함수를 payable로 만들거나, Relayer가 별도로 자금을 조달받고 `executeByInvestorWithBlockLimit`이 해당 잔액에서 지출할 수 있도록 `receive`를 추가하세요.

또는, 서명과 `params[]`에서 `value` 매개변수를 제거하여 외부 호출이 대상 주소로 기본적으로 전송되지 않도록 하세요.

**Securitize:** `TransactionRelayer`가 크게 변경되어 `executeByInvestorWithBlockLimit` 및 대부분의 다른 함수는 더 이상 사용되지 않으므로 이제 항상 되돌립니다(revert). `value` 매개변수를 받지 않거나 native를 보내지 않는 `executePreApprovedTransaction`만이 유일하게 작동하는 함수로 남아 있습니다.

**Cyfrin:** 검증되었습니다.


### Don't write to the same storage slot multiple times

**Description:** EVM에서 스토리지에 쓰는 것은 비용이 많이 듭니다; 이상적으로는 동일한 스토리지 슬롯에 한 번만 씁니다. 예를 들어 `ComplianceServiceRegulated::cleanupInvestorIssuances`는 다음과 같이 수행합니다:
```solidity
        uint256 time = block.timestamp;

        uint256 currentIssuancesCount = issuancesCounters[investor];
        uint256 currentIndex = 0;

        if (currentIssuancesCount == 0) {
            return;
        }

        while (currentIndex < currentIssuancesCount) {
            uint256 issuanceTimestamp = issuancesTimestamps[investor][currentIndex];

            bool isNoLongerLocked = issuanceTimestamp <= (time - lockTime);

            if (isNoLongerLocked) {
                if (currentIndex != currentIssuancesCount - 1) {
                    issuancesTimestamps[investor][currentIndex] = issuancesTimestamps[investor][currentIssuancesCount - 1];
                    issuancesValues[investor][currentIndex] = issuancesValues[investor][currentIssuancesCount - 1];
                }

                delete issuancesTimestamps[investor][currentIssuancesCount - 1];
                delete issuancesValues[investor][currentIssuancesCount - 1];

                // @audit storage write to decrement
                issuancesCounters[investor]--;
                // @audit storage read of value just written
                currentIssuancesCount = issuancesCounters[investor];
            } else {
                currentIndex++;
            }
        }
```

이는 `isNoLongerLocked == true`일 때마다 모든 루프 반복 중에 추가 스토리지 쓰기 및 스토리지 읽기를 초래하므로 매우 비효율적입니다. 대신 루프 후 `currentIssuancesCount` 변수를 감소시킨 다음 `issuancesCounters[investor]`에 한 번만 쓰세요:

```solidity
        while (currentIndex < currentIssuancesCount) {
            uint256 issuanceTimestamp = issuancesTimestamps[investor][currentIndex];

            bool isNoLongerLocked = issuanceTimestamp <= (time - lockTime);

            if (isNoLongerLocked) {
                if (currentIndex != currentIssuancesCount - 1) {
                    issuancesTimestamps[investor][currentIndex] = issuancesTimestamps[investor][currentIssuancesCount - 1];
                    issuancesValues[investor][currentIndex] = issuancesValues[investor][currentIssuancesCount - 1];
                }

                delete issuancesTimestamps[investor][currentIssuancesCount - 1];
                delete issuancesValues[investor][currentIssuancesCount - 1];

                currentIssuancesCount--;
            } else {
                currentIndex++;
            }
        }

        issuancesCounters[investor] = currentIssuancesCount;
```

또한 이 영역 주변에는 단위 테스트가 없는 것으로 보입니다; `while` 루프를 주석 처리하고 테스트 스위트를 다시 실행했는데 실패한 테스트가 없었습니다! 따라서 이상적으로는 변경하기 전에 최적화된 버전이 아무것도 깨뜨리지 않도록 단위 테스트를 먼저 작성하세요.

**Securitize:** 커밋 [10ac116](https://github.com/securitize-io/dstoken/commit/10ac116901001ce804c39aecdadad16b4fc3251c)에서 수정되었으며, 여기서 `while` 루프의 내용 주변에 추가 단위 테스트도 추가했습니다.

**Cyfrin:** 검증되었습니다.


### `ComplianceServiceRegulated::getComplianceTransferableTokens` should call `IDSLockManager::getTransferableTokensForInvestor`

**Description:** `ComplianceServiceRegulated::getComplianceTransferableTokens`는 이미 레지스트리를 로드하고 투자자 ID를 가져오므로 레지스트리를 다시 로드하고 투자자 ID를 다시 가져오는 것을 저장하기 위해 `getTransferableTokens` 대신 `IDSLockManager::getTransferableTokensForInvestor`를 호출해야 합니다:
```diff
    function getComplianceTransferableTokens(
        address _who,
        uint256 _time,
        uint64 _lockTime
    ) public view override returns (uint256) {
        require(_time != 0, "Time must be greater than zero");
        string memory investor = getRegistryService().getInvestor(_who);

-       uint256 balanceOfInvestor = getLockManager().getTransferableTokens(_who, _time);
+       uint256 balanceOfInvestor = getLockManager().getTransferableTokensForInvestor(investor, _time);

```

**Securitize:** 커밋 [382eaae](https://github.com/securitize-io/dstoken/commit/382eaae50dbf5a9a33ce343268e6dc9d257428c2)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove return value from `DSToken::updateInvestorBalance` as it is never checked

**Description:** `DSToken::updateInvestorBalance`는 `bool`을 반환하는 `internal` 함수이지만 이 반환 값은 어디에서도 확인되지 않습니다; 제거하세요:
```solidity
token/DSToken.sol
304:        updateInvestorBalance(_from, _value, CommonUtils.IncDec.Decrease);
305:        updateInvestorBalance(_to, _value, CommonUtils.IncDec.Increase);
308:    function updateInvestorBalance(address _wallet, uint256 _value, CommonUtils.IncDec _increase) internal override returns (bool) {

mocks/StandardTokenMock.sol
72:    function updateInvestorBalance(address, uint256, CommonUtils.IncDec) internal pure override returns (bool) {

token/TokenLibrary.sol
94:        updateInvestorBalance(_tokenData, IDSRegistryService(_services[REGISTRY_SERVICE]), _params._to, shares, CommonUtils.IncDec.Increase);
129:        updateInvestorBalance(
165:        updateInvestorBalance(
193:        updateInvestorBalance(_tokenData, registryService, _from, _shares, CommonUtils.IncDec.Decrease);
194:        updateInvestorBalance(_tokenData, registryService, _to, _shares, CommonUtils.IncDec.Increase);
197:    function updateInvestorBalance(TokenData storage _tokenData, IDSRegistryService _registryService, address _wallet, uint256 _shares, CommonUtils.IncDec _increase) internal returns (bool) {

token/IDSToken.sol
131:    function updateInvestorBalance(address _wallet, uint256 _value, CommonUtils.IncDec _increase) internal virtual returns (bool);
```

현재 반환 값은 오해의 소지가 있을 수 있습니다. 예를 들어 `_wallet`이 투자자에 속하지 않는 경우 업데이트가 발생하지 않지만 여전히 `true`를 반환합니다:
```solidity
function updateInvestorBalance(address _wallet, uint256 _value, CommonUtils.IncDec _increase) internal override returns (bool) {
    string memory investor = getRegistryService().getInvestor(_wallet);
    // @audit if `_wallet` doesn't belong to an investor, no update occurs
    if (!CommonUtils.isEmptyString(investor)) {
        uint256 balance = balanceOfInvestor(investor);
        if (_increase == CommonUtils.IncDec.Increase) {
            balance += _value;
        } else {
            balance -= _value;
        }

        ISecuritizeRebasingProvider rebasingProvider = getRebasingProvider();

        uint256 sharesBalance = rebasingProvider.convertTokensToShares(balance);

        tokenData.investorsBalances[investor] = sharesBalance;
    }
    // @audit but the function still returns `true` which is misleading
    return true;
}
```

따라서 어차피 읽히지도 않고 공개 인터페이스에 영향을 미치지 않는 내부 함수이므로 `bool` 반환 값을 제거하는 것이 더 간단해 보입니다.

**Securitize:** 커밋 [2219e9a](https://github.com/securitize-io/dstoken/commit/2219e9a14207b4b1faf3f1c35409771cc23251b6)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
