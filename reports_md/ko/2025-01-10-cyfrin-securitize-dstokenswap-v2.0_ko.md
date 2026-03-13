**Lead Auditors**

[Hans](https://twitter.com/hansfriese)
**Assisting Auditors**



---

# Findings
## Medium Risk


### `validateLockedTokens`에서 `msg.sender`를 직접 사용하여 메타 트랜잭션이 작동하지 않음 (Meta transactions will not work due to direct msg.sender usage in validateLockedTokens)

**설명:** 프로토콜은 여러 부분에서 `_msgSender()`를 사용하며, 릴레이어(relayers)가 투자자가 서명한 트랜잭션을 처리하는 메타 트랜잭션 지원 가능성을 고려하는 것으로 이해됩니다.
그러나 `validateLockedTokens` 함수는 `msg.sender`를 직접 사용하여 전송 가능한 잔액을 확인합니다. 이는 메타 트랜잭션 컨텍스트에서 실제 토큰 소유자의 주소가 릴레이어의 주소(msg.sender)와 다르기 때문에 계약이 메타 트랜잭션을 지원하는 것을 방지합니다.

```solidity
81  function validateLockedTokens(string memory investorId, uint256 value, IDSRegistryService registryService) private view {
82      IDSComplianceService complianceService = IDSComplianceService(sourceServiceConsumer.getDSService(sourceServiceConsumer.COMPLIANCE_SERVICE()));
83      IDSComplianceConfigurationService complianceConfigurationService = IDSComplianceConfigurationService(sourceServiceConsumer.getDSService(sourceServiceConsumer.COMPLIANCE_CONFIGURATION_SERVICE()));
84
85      string memory country = registryService.getCountry(investorId);
86      uint256 region = complianceConfigurationService.getCountryCompliance(country);
87
88      // lock/hold up validation
89      uint256 lockPeriod = (region == US) ? complianceConfigurationService.getUSLockPeriod() : complianceConfigurationService.getNonUSLockPeriod();
90      uint256 availableBalanceForTransfer = complianceService.getComplianceTransferableTokens(msg.sender, block.timestamp, uint64(lockPeriod));//@audit-issue msg.sender can be different from _msgSender
91      require(availableBalanceForTransfer >= value, "Not enough unlocked balance");
92  }
```
`ComplianceServiceRegulated::getComplianceTransferableTokens()` 함수에서 첫 번째 매개변수 `_who`는 `getRegistryService().getInvestor(_who);`를 통해 투자자 정보를 가져오는 데 사용됩니다.
```solidity
@securitize\digital_securities\contracts\compliance\ComplianceServiceRegulated.sol
658:     function getComplianceTransferableTokens(
659:         address _who,
660:         uint256 _time,
661:         uint64 _lockTime
662:     ) public view override returns (uint256) {
663:         require(_time != 0, "Time must be greater than zero");
664:         string memory investor = getRegistryService().getInvestor(_who);
665:
666:         uint256 balanceOfInvestor = getLockManager().getTransferableTokens(_who, _time);
667:
668:         uint256 investorIssuancesCount = issuancesCounters[investor];
669:
670:         //No locks, go to base class implementation
671:         if (investorIssuancesCount == 0) {
672:             return balanceOfInvestor;
673:         }
674:
675:         uint256 totalLockedTokens = 0;
676:         for (uint256 i = 0; i < investorIssuancesCount; i++) {
677:             uint256 issuanceTimestamp = issuancesTimestamps[investor][i];
678:
679:             if (uint256(_lockTime) > _time || issuanceTimestamp > (_time - uint256(_lockTime))) {
680:                 totalLockedTokens = totalLockedTokens + issuancesValues[investor][i];
681:             }
682:         }
683:
684:         //there may be more locked tokens than actual tokens, so the minimum between the two
685:         uint256 transferable = balanceOfInvestor - Math.min(totalLockedTokens, balanceOfInvestor);
686:
687:         return transferable;
688:     }
```
다른 부분에서는 `msg.sender`와 `_msgSender()`가 메타 트랜잭션을 처리하기 위해 올바르게 사용되고 있습니다.

**영향:** 메타 트랜잭션의 경우, `msg.sender`가 반드시 투자자인 것은 아니므로 `getComplianceTransferableTokens`가 잘못된 값을 반환합니다. 사용자는 가스 비용을 지불하기 위해 항상 ETH를 보유해야 하며, 이는 사용자가 다른 사람에 의해 트랜잭션을 중계받을 수 있는 메타 트랜잭션의 주요 이점 중 하나를 무효화합니다.

**권장 완화 방법:** 아래와 같이 특정 부분에서 `msg.sender`를 사용하는 대신 `_msgSender()`를 사용하세요.

```diff
    function validateLockedTokens(string memory investorId, uint256 value, IDSRegistryService registryService) private view {
        IDSComplianceService complianceService = IDSComplianceService(sourceServiceConsumer.getDSService(sourceServiceConsumer.COMPLIANCE_SERVICE()));
        IDSComplianceConfigurationService complianceConfigurationService = IDSComplianceConfigurationService(sourceServiceConsumer.getDSService(sourceServiceConsumer.COMPLIANCE_CONFIGURATION_SERVICE()));

        string memory country = registryService.getCountry(investorId);
        uint256 region = complianceConfigurationService.getCountryCompliance(country);

        // lock/hold up validation
        uint256 lockPeriod = (region == US) ? complianceConfigurationService.getUSLockPeriod() : complianceConfigurationService.getNonUSLockPeriod();//@audit-info assume these values are representing time duration in seconds
--        uint256 availableBalanceForTransfer = complianceService.getComplianceTransferableTokens(msg.sender, block.timestamp, uint64(lockPeriod));
++        uint256 availableBalanceForTransfer = complianceService.getComplianceTransferableTokens(_msgSender(), block.timestamp, uint64(lockPeriod));

        require(availableBalanceForTransfer >= value, "Not enough unlocked balance");
    }
```

**Securitize:** 커밋 [b26a16](https://bitbucket.org/securitize_dev/bc-dstoken-class-swap-sc/commits/b26a167524dfa96fc92dc18a863998a50e533bf2)에서 수정됨.

**Cyfrin:** 검증됨.


\clearpage
## Informational


### `initialize` 함수에서 0 주소 유효성 검사 누락 (Missing zero address validation in initialize function)

**설명:** `DSTokenClassSwap` 계약의 `initialize` 함수는 입력 주소 `_sourceDSToken` 및 `_targetDSToken`이 0이 아닌 주소인지 확인하지 않습니다.

```solidity
DSTokenClassSwap.sol
40:     function initialize(address _sourceDSToken, address _targetDSToken) public override onlyProxy initializer {
41:         __BaseDSContract_init();
42:         sourceDSToken = IDSToken(_sourceDSToken);//@audit-issue INFO check zero address
43:         sourceServiceConsumer = IDSServiceConsumer(_sourceDSToken);
44:         targetDSToken = IDSToken(_targetDSToken);
45:         targetServiceConsumer = IDSServiceConsumer(_targetDSToken);
46:     }
```

**권장 완화 방법:** 0 주소 유효성 검사 확인을 추가하세요.

**Securitize:** 커밋 [b26a16](https://bitbucket.org/securitize_dev/bc-dstoken-class-swap-sc/commits/b26a167524dfa96fc92dc18a863998a50e533bf2)에서 수정됨.

**Cyfrin:** 검증됨.


\clearpage
