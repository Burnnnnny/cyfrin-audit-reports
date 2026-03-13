**Lead Auditors**

[Kage](https://x.com/0kage_eth)

[Farouk](https://x.com/Ubermensh3dot0)
**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 인라인 어셈블리에서 free memory pointer를 손상시켜 메모리 할당 panic 및 서비스 거부를 유발함

**설명:** `AgreementFactory` 컨트랙트는 중간 값을 메모리에 직접 써서 CREATE2 salt를 파생하는 인라인 어셈블리를 사용합니다. `AgreementFactory::create`와 `AgreementFactory::computeAddress` 모두에서, 어셈블리 블록이 `mstore(0x40, salt)`를 통해 메모리 슬롯 `0x40`에 값을 씁니다.

```Solidity
// Include chainid in salt to prevent cross-chain address collisions
bytes32 finalSalt;
assembly {
    mstore(0x00, chainid())
    mstore(0x20, caller())
    mstore(0x40, salt)
    finalSalt := keccak256(0x00, 0x60)
}
```

Solidity에서 메모리 슬롯 `0x40`은 free-memory pointer로 예약되어 있으며, 동적 메모리 할당 시 다음 사용 가능한 메모리 위치를 가리킵니다. 이 슬롯을 덮어쓰면 메모리 할당기 상태가 손상됩니다.

이 어셈블리 블록 직후 컨트랙트는 `bytes.concat(...)`와 `abi.encode(...)`를 통해 동적 메모리 할당을 수행합니다. 하지만 free-memory pointer가 임의의 값(`salt`)으로 덮여 있으므로, 할당기는 잘못되었거나 지나치게 큰 오프셋에 메모리를 할당하려고 시도하게 되고, 결국 Solidity panic error code `0x41`(memory allocation error)을 발생시킵니다.

이 문제는 결정적이며, 손상된 포인터 이후에 이러한 메모리 할당에 도달하는 모든 실행 경로에 영향을 주므로, 해당 함수들은 사실상 사용할 수 없게 됩니다.

**영향:** `AgreementFactory::create`와 `AgreementFactory::computeAddress`는 모두 일관되게 `panic: memory allocation error (0x41)`로 revert하며, 그 결과 핵심 factory 기능 전체가 DoS됩니다.


**개념 증명 (Proof of Concept):**
- `AgreementFactoryTest::test_createWithFreeMemoryPointerIssue`:
```Solidity
function test_createWithFreeMemoryPointerIssue() public {
    // Prepare a fully populated AgreementDetails struct
    // The exact contents are not important; the bug is independent of input values
    AgreementDetails memory agreementDetails = getMockAgreementDetails("0xAABB");
    bytes32 salt = keccak256("test-salt");

    // Impersonate the protocol address (msg.sender)
    vm.prank(protocol);

    // Expect a Solidity panic due to memory allocation failure (Panic(0x41))
    // This happens because mstore(0x40, salt) corrupts the free-memory pointer,
    // and the subsequent Agreement constructor call performs dynamic memory allocation.
    vm.expectRevert(stdError.memOverflowError);

    // This call deterministically reverts due to free memory pointer corruption
    factory.create(agreementDetails, address(chainValidator), protocol, salt);
}
```
- 출력:
```Python
Ran 1 test for test/unit/AgreementFactoryTest.t.sol:AgreementFactoryTest
[PASS] test_createWithFreeMemoryPointerIssue() (gas: 25813)
Logs:
  Getting network config for chain ID: 31337
  Getting network config for chain ID: 31337
  ChainValidator implementation deployed at: 0x42Ad6372A7676878a95Ae9993D6eB88543A7D47a

Traces:
  [25813] AgreementFactoryTest::test_createWithFreeMemoryPointerIssue()
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000AB)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  $NH{qA)
    │   └─ ← [Return]
    ├─ [6024] AgreementFactory::create(AgreementDetails({ protocolName: "testProtocolV2", contactDetails: [Contact({ name: "Test Name V2", contact: "test@mail.com" })], chains: [Chain({ assetRecoveryAddress: "0x0000000000000000000000000000000000000022", accounts: [Account({ accountAddress: "0xAABB", childContractScope: 2 })], caip2ChainId: "eip155:1" })], bountyTerms: BountyTerms({ bountyPercentage: 10, bountyCapUSD: 100, retainable: false, identity: 0, diligenceRequirements: "none", aggregateBountyCapUSD: 1000 }), agreementURI: "ipfs://testHash" }), ERC1967Proxy: [0x2FF6CD634dfa4B2105dc8f53ca262e64Ed089049], 0x00000000000000000000000000000000000000AB, 0x8bcfa1e0aed22543ed44d41a95e315383294a18f9fb6e67ee082afcd585a6ff1)
    │   └─ ← [Revert] panic: memory allocation error (0x41)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.22ms (170.75µs CPU time)
```
- `AgreementFactoryTest::test_computeAddressWithFreeMemoryPointerIssue`:
```Solidity
function test_computeAddressWithFreeMemoryPointerIssue() public {
    // Prepare inputs for address computation
    AgreementDetails memory agreementDetails = getMockAgreementDetails("0xAABB");
    bytes32 salt = keccak256("test-salt");

    // Expect the same memory allocation panic (Panic(0x41))
    // In this case, the revert occurs during:
    // bytes.concat(type(Agreement).creationCode, abi.encode(...))
    // because abi.encode reads the corrupted free-memory pointer.
    vm.expectRevert(stdError.memOverflowError);

    // Address computation reverts before returning a value
    factory.computeAddress(
        agreementDetails,
        address(chainValidator),
        protocol,
        salt,
        protocol
    );
}
```
- 출력:
```Python
Ran 1 test for test/unit/AgreementFactoryTest.t.sol:AgreementFactoryTest
[PASS] test_computeAddressWithFreeMemoryPointerIssue() (gas: 25485)
Logs:
  Getting network config for chain ID: 31337
  Getting network config for chain ID: 31337
  ChainValidator implementation deployed at: 0x42Ad6372A7676878a95Ae9993D6eB88543A7D47a

Traces:
  [25485] AgreementFactoryTest::test_computeAddressWithFreeMemoryPointerIssue()
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  $NH{qA)
    │   └─ ← [Return]
    ├─ [6210] AgreementFactory::computeAddress(AgreementDetails({ protocolName: "testProtocolV2", contactDetails: [Contact({ name: "Test Name V2", contact: "test@mail.com" })], chains: [Chain({ assetRecoveryAddress: "0x0000000000000000000000000000000000000022", accounts: [Account({ accountAddress: "0xAABB", childContractScope: 2 })], caip2ChainId: "eip155:1" })], bountyTerms: BountyTerms({ bountyPercentage: 10, bountyCapUSD: 100, retainable: false, identity: 0, diligenceRequirements: "none", aggregateBountyCapUSD: 1000 }), agreementURI: "ipfs://testHash" }), ERC1967Proxy: [0x2FF6CD634dfa4B2105dc8f53ca262e64Ed089049], 0x00000000000000000000000000000000000000AB, 0x8bcfa1e0aed22543ed44d41a95e315383294a18f9fb6e67ee082afcd585a6ff1, 0x00000000000000000000000000000000000000AB) [staticcall]
    │   └─ ← [Revert] panic: memory allocation error (0x41)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.63ms (410.21µs CPU time)

Ran 1 test suite in 146.15ms (3.63ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
**권장 완화 조치:** free-memory pointer를 안전하게 사용해 메모리를 할당하도록 수정하는 것을 고려하십시오.
```diff
    function create(
        AgreementDetails memory details,
        address chainValidator,
        address owner,
        bytes32 salt
    )
        external
        returns (address agreementAddress)
    {
        // Include chainid in salt to prevent cross-chain address collisions
        bytes32 finalSalt;
        assembly {
-            mstore(0x00, chainid())
-            mstore(0x20, caller())
-            mstore(0x40, salt)
-            finalSalt := keccak256(0x00, 0x60)
+            let ptr := mload(0x40)
+            mstore(ptr, chainid())
+            mstore(add(ptr, 0x20), caller())
+            mstore(add(ptr, 0x40), salt)
+            finalSalt := keccak256(ptr, 0x60)
+            mstore(0x40, add(ptr, 0x60))
        }
```

```diff
    function computeAddress(
        AgreementDetails memory details,
        address chainValidator,
        address owner,
        bytes32 salt,
        address deployer
    )
        external
        view
        returns (address)
    {
        bytes32 finalSalt;
        assembly {
-            mstore(0x00, chainid())
-            mstore(0x20, deployer)
-            mstore(0x40, salt)
-            finalSalt := keccak256(0x00, 0x60)
+            let ptr := mload(0x40)
+            mstore(ptr, chainid())
+            mstore(add(ptr, 0x20), deployer)
+            mstore(add(ptr, 0x40), salt)
+            finalSalt := keccak256(ptr, 0x60)
+            mstore(0x40, add(ptr, 0x60))
        }
```


**SafeHarbor:**
[57434a1](https://github.com/PatrickAlphaC/safe-harbor/commit/57434a1299e725fb33ef3895c054b2349baa2550)에서 수정됨.

**Cyfrin:** 확인함.



\clearpage
## 낮은 위험 (Low Risk)


### bounty percentage와 cap 검증이 누락됨

**설명:** `Agreement::_validateBountyTerms`에는 다음 두 가지 중요한 검증이 빠져 있습니다.

1. `bountyPercentage`에 대한 상한 검증이 없음
2. `aggregateBountyCapUSD >= bountyCapUSD`에 대한 검증이 없음


**영향:** 소유자가 실수로 `bountyPercentage > 100`을 설정하면, `retainable = false`인 경우 프로토콜이 회수 금액보다 더 큰 금액을 지급해야 하는 법적 책임을 질 수 있습니다.

또한 `aggregateBountyCapUSD >= bountyCapUSD`가 보장되지 않으면 해석상 모호성이 생겨 분쟁으로 이어질 수 있습니다. 예를 들어 `bountyCapUSD = $1M`인데 `aggregateBountyCapUSD = $500K`라면, 단일 whitehat이 $1M까지 받을 수 있는지 아니면 총 $500K로 제한되는지가 불명확해집니다.

**권장 완화 조치:** `Agreement::_validateBountyTerms`에 추가 검증을 넣는 것을 고려하십시오.

**SafeHarbor:**
[3c0b5a95](https://github.com/PatrickAlphaC/safe-harbor/commit/3c0b5a95c926e716fae2aada8ef6070dd6da019a)에서 수정됨.

**Cyfrin:** 확인함.


### 빈 account address를 허용하는 검증 누락으로 agreement scope가 모호해짐

**설명:** `Agreement::_validateChains`는 accounts 배열이 비어 있지 않은지만 검증하고, 개별 `Account` 구조체의 내용은 검증하지 않습니다. 그 결과 `accountAddress`가 빈 문자열인 계정도 agreement에 포함될 수 있습니다.

참고로 `assetRecoveryAddress`는 빈 문자열이 아니어야 한다고 검증하지만, 각 account의 `accountAddress`에는 동일한 검증이 적용되지 않습니다.

**영향:** 프로토콜이 겉보기에는 유효하지만 실제 scope 정의가 없는 agreement를 실수로 배포할 수 있어, Safe Harbor 도입이 무의미해질 수 있습니다. 또한 Safe Harbor Registry를 조회하는 whitehat 입장에서는 빈 account address가 "모든 주소"를 의미하는지, 아니면 "어떤 주소도 아님"을 의미하는지 판단할 수 없습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `AgreementTest.t.sol`에 추가하십시오.

```solidity
   function test_POC_emptyAccountAddressAllowed() public {
        // Create account with empty address
        SHAccount[] memory accounts = new SHAccount[](1);
        accounts[0] = SHAccount({
            accountAddress: "",  // Empty!
            childContractScope: ChildContractScope.All
        });

        SHChain[] memory chains = new SHChain[](1);
        chains[0] = SHChain({
            assetRecoveryAddress: "0x742d35Cc6634C0532925a3b844Bc454e",
            accounts: accounts,
            caip2ChainId: "eip155:56"
        });

        vm.prank(owner);
        agreement.addChains(chains); //@note this does not revert

        // Verify empty account was added
        AgreementDetails memory details = agreement.getDetails();

        bool foundBscChain = false;
        for (uint256 i = 0; i < details.chains.length; i++) {
            if (keccak256(bytes(details.chains[i].caip2ChainId)) == keccak256(bytes("eip155:56"))) {
                foundBscChain = true;
                assertEq(details.chains[i].accounts.length, 1);
                assertEq(details.chains[i].accounts[0].accountAddress, "");
                break;
            }
        }
        assertTrue(foundBscChain, "BSC chain not found");
    }
```


**권장 완화 조치:** `assetRecoveryAddress`에 적용된 기존 검증과 일관되게, `_validateChains`에서 개별 account address에 대한 검증도 추가하는 것을 고려하십시오.

**SafeHarbor:**
[e32dd76](https://github.com/PatrickAlphaC/safe-harbor/commit/e32dd768c48b72b3cfe72a17676e6348c6da1837), [a881fdc](https://github.com/PatrickAlphaC/safe-harbor/commit/a881fdc10a19364c36c92b15e10d74e0143a291b)에서 수정됨.

**Cyfrin:** 확인함.


### `Agreement::removeAccounts`는 chain을 0 accounts 상태로 남길 수 있음

**설명:** `Agreement::addChains`, `Agreement::addOrSetChains`를 통해 chain을 생성하거나 수정할 때, `Agreement::_validateChains`는 각 chain에 최소 1개 이상의 account가 있어야 한다고 강제합니다.

```solidity
function _validateChains(Chain[] memory _chains) internal {
    for (uint256 i = 0; i < _chains.length; i++) {
        // ...
        if (_chains[i].accounts.length == 0) { //@audit does not allow chain with zero accounts
            revert Agreement__ZeroAccountsForChainId(_chains[i].caip2ChainId);
        }
        // ...
    }
}
```

하지만 `Agreement::removeAccounts`는 account를 제거한 뒤 0개가 남는지 검사하지 않습니다.

```solidity
function removeAccounts(string memory _caip2ChainId, string[] memory _accountAddresses) external onlyOwner {
    if (!_chainExists(_caip2ChainId)) {
        revert Agreement__ChainNotFoundByCaip2Id(_caip2ChainId);
    }
    // @audit No check for remaining accounts after removal
    for (uint256 i = 0; i < _accountAddresses.length; i++) {
        uint256 accountIndex = _findAccountIndex(_caip2ChainId, _accountAddresses[i]);
        emit AccountRemoved(_caip2ChainId, _accountAddresses[i]);

        uint256 lastAccountId = accounts[_caip2ChainId].length - 1;
        accounts[_caip2ChainId][accountIndex] = accounts[_caip2ChainId][lastAccountId];
        accounts[_caip2ChainId].pop();
    }
}
```


**영향:** 생성 시에는 허용되지 않는 "account가 0개인 chain" 상태가, 이후 account 제거 과정을 통해 생길 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 내용을 `Agreement.t.sol`에 추가하십시오.

```solidity
 function test_POC_removeAllAccountsFromChain() public {
        AgreementDetails memory detailsBefore = agreement.getDetails();
        assertEq(detailsBefore.chains[0].accounts.length, 1);

        string memory chainId = detailsBefore.chains[0].caip2ChainId;
        string memory accountAddr = detailsBefore.chains[0].accounts[0].accountAddress;

        // Remove the account
        string[] memory toRemove = new string[](1);
        toRemove[0] = accountAddr;

        vm.prank(owner);
        agreement.removeAccounts(chainId, toRemove);

        // Verify chain now has zero accounts
        AgreementDetails memory detailsAfter = agreement.getDetails();
        assertEq(detailsAfter.chains[0].accounts.length, 0);  // @audit chain now has zero accounts

    }
```

**권장 완화 조치:** `Agreement::removeAccounts`에 최소 1개의 account는 남아 있어야 한다는 검사를 추가하는 것을 고려하십시오.

**SafeHarbor:**
[bf411ed](https://github.com/PatrickAlphaC/safe-harbor/commit/bf411edbace91ad93c1285ac4ac8c391fa338404)에서 수정됨.

**Cyfrin:** 확인함.


### 법적 합의서에 명시된 선택적 `signature` 필드가 `Chain` 구조체에 구현되어 있지 않음

**설명:** SEAL Whitehat Safe Harbor Agreement 법적 문서는 `Chain` 구조체 안에 선택적 `signature` 필드를 명시하고 있지만, 스마트 컨트랙트 코드에는 구현되어 있지 않습니다.

>Legal Agreement Reference (Section 1.1(c)(ii)):
"D. optionally, the signature for such account, which may be used as additional evidence that such account has affirmatively accepted being subject to this Agreement."

이 필드는 특정 체인의 특정 account가 Safe Harbor scope에 포함되는 데 명시적으로 동의했다는 암호학적 증거를 프로토콜이 제공할 수 있도록 설계된 것입니다.

하지만 현재 구현은 다음과 같습니다.

```solidity
struct Chain {
    // The address to which recovered assets will be sent.
    string assetRecoveryAddress;
    // The accounts in scope for the agreement.
    Account[] accounts;
    // The CAIP-2 chain ID.
    string caip2ChainId;
    // @audit Missing: signature field as specified in agreement Section 1.1(c)(ii)(D)
}
```
**영향:** 법적 문서에서 `signature` 필드는 "optional"로 설명되어 있지만, 구현에서 완전히 빠져 있기 때문에 프로토콜이 원하더라도 특정 체인에 대해 account-level consent의 추가 증거를 제공하는 이 기능을 사용할 수 없습니다.

**권장 완화 조치:** 법적 합의서와 기능 parity를 맞추기 위해 `Chain` 구조체에 `signature` 필드를 추가하는 것을 고려하십시오. 또는 법적 문서에서 이 필드를 제거하는 것도 한 방법입니다.


**SafeHarbor:**
[ebfcb1a](https://github.com/PatrickAlphaC/safe-harbor/commit/ebfcb1aa649b67afc758a6b252124b60f99f96c0)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### contact details 검증 누락

**설명:** `Agreement::_setDetails`는 contact details를 아무 검증 없이 복사합니다. `Contact` 구조체의 필드들은 빈 문자열일 수 있습니다.

생성자도 `setContactDetails`도 contact 필드가 비어 있지 않은지 검사하지 않습니다.

```solidity
function _setDetails(AgreementDetails memory _details) internal {
     // code...
     delete contactDetails;
     for (uint256 i = 0; i < _details.contactDetails.length; ++i) {
         contactDetails.push(_details.contactDetails[i]); //@audit no check for empty name or contact
      }
}

function setContactDetails(Contact[] memory _contactDetails) external onlyOwner {
    emit ContactDetailsSet(_contactDetails);
    delete contactDetails;
    for (uint256 i = 0; i < _contactDetails.length; i++) {
        contactDetails.push(_contactDetails[i]); //@audit no check for empty name or contact
    }
}
```

**영향:** whitehat은 구조 활동 전에 프로토콜 팀에 사전 통지하기 위해 contact details에 의존합니다. 비어 있거나 잘못된 연락처 정보는 whitehat이 프로토콜 팀에 효과적으로 연락하지 못하게 만들 수 있습니다.

**권장 완화 조치:** contact details에 대한 검증을 추가하는 것을 고려하십시오. 또는 빈 연락처도 허용된다고 명시적으로 문서화하십시오.

**SafeHarbor:**
[0f492fc](https://github.com/PatrickAlphaC/safe-harbor/commit/0f492fc78088a5dca4adc7201b669f92fab17b7f)에서 수정됨.

**Cyfrin:** 확인함.


### Solidity에서는 기본값으로의 초기화를 피할 것

**설명:** Solidity에서는 기본값으로의 초기화가 불필요합니다.
```solidity
Agreement.sol
95:        for (uint256 i = 0; i < _contactDetails.length; i++) {
105:        for (uint256 i = 0; i < _chains.length; i++) {
114:            for (uint256 j = 0; j < _chains[i].accounts.length; j++) {
125:        for (uint256 i = 0; i < _chains.length; i++) {
137:            for (uint256 j = 0; j < _chains[i].accounts.length; j++) {
149:        for (uint256 i = 0; i < _chains.length; i++) {
158:            for (uint256 j = 0; j < _chains[i].accounts.length; j++) {
169:        for (uint256 i = 0; i < _caip2ChainIds.length; i++) {
192:        for (uint256 i = 0; i < _accounts.length; i++) {
207:        for (uint256 i = 0; i < _accountAddresses.length; i++) {
235:        for (uint256 i = 0; i < _details.contactDetails.length; ++i) {
241:        for (uint256 i = 0; i < _details.chains.length; ++i) {
247:            for (uint256 j = 0; j < _details.chains[i].accounts.length; ++j) {
257:        for (uint256 i = 0; i < _chains.length; i++) {
288:        for (uint256 i = 0; i < _chains.length; i++) {
307:        for (uint256 i = 0; i < contactsLength; ++i) {
314:        for (uint256 i = 0; i < chainsLength; ++i) {
321:            for (uint256 j = 0; j < accts.length; ++j) {
365:        for (uint256 i = 0; i < length; ++i) {
378:        for (uint256 i = 0; i < length; i++) {
398:        for (uint256 i = 0; i < chainAccounts.length; i++) {

SafeHarborRegistry.sol
34:        uint256 migratedCount = 0;
36:        for (uint256 i = 0; i < length; i++) {

ChainValidator.sol
39:        for (uint256 i = 0; i < length; i++) {
64:        for (uint256 i = 0; i < length; i++) {
80:        for (uint256 i = 0; i < length; i++) {
```

**SafeHarbor:**
커밋 [ed67312](https://github.com/PatrickAlphaC/safe-harbor/commit/ed67312d7679596fe554503406283bd9194430bb)에서 수정됨.

**Cyfrin:** 확인함.


### named return을 이미 사용 중이라면 마지막 `return`문은 제거할 것

**설명:** 이미 named return을 사용하고 있다면 마지막의 불필요한 `return`문은 제거하는 것이 좋습니다.

* `AgreementFactory::create`

**SafeHarbor:**
커밋 [220e5fb](https://github.com/PatrickAlphaC/safe-harbor/commit/220e5fb0af3f1750fd066a933f23f72124c37a6e)에서 수정됨.

**Cyfrin:** 확인함.


### 법적 합의서와 스마트 컨트랙트 사이의 이름 및 데이터 타입 불일치

**설명:** 법적 합의서(Section 1.1(c))는 `DAO Adoption Procedures`가 `0x1eaCD100B0546E433fbf4d773109cAD482c34686`에 배포된 `SafeHarborRegistryV2.sol`의 `adoptSafeHarbor`를 호출하고, `AgreementDetailsV2` 구조체를 설정해야 한다고 명시합니다.

현재 구현은 다음과 같은 차이를 보입니다.

Component | Legal Agreement | Implementation
-- | -- | --
Registry Contract | SafeHarborRegistryV2.sol | SafeHarborRegistry.sol
Version | V2 (implied) | VERSION = "3.0.0"
Details Struct | AgreementDetailsV2 | AgreementDetails


변수 이름도 다릅니다.
Legal Agreement (section 1.1.c)  | Implementation | Location
-- | -- | --
chainID|caip2ChainId| `Chain` struct
`identityRequirement`| `identity` | `BountyTerms` struct

타입 이름 차이도 있습니다.
Legal Agreement (section 1.1.c.iv)  | Implementation
-- | --
`identityRequirement`| `identityRequirements`


타입 자체가 다른 경우도 있습니다.
Field  | Legal agreement (section 1.1.c.iv) | Implementation
-- | -- | --
`bountPercentage`| `string`| `uint256`


**권장 완화 조치:** 구현을 법적 합의서 용어와 일치시키는 것을 고려하십시오.

**SafeHarbor:**
[0d70af3](https://github.com/PatrickAlphaC/safe-harbor/commit/0d70af3c37ff451e50bc2f2087d23d23aec7d08a)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 읽기 전용 external 입력에는 `memory` 대신 `calldata` 선호

**설명:** 읽기 전용 external 입력이며, 내부 함수에 `memory`로 넘길 필요도 없다면 `memory` 대신 `calldata`를 선호하는 것이 좋습니다.

* `Agreement.sol`
```solidity
84:    function setProtocolName(string memory _protocolName) external onlyOwner {
91:    function setContactDetails(Contact[] memory _contactDetails) external onlyOwner {
// for `_accounts` only
187:    function addAccounts(string memory _caip2ChainId, Account[] memory _accounts) external onlyOwner {
```

* `ChainValidator.sol`
```solidity
35:    function initialize(address _initialOwner, string[] memory _initialValidChains) external initializer {
```

**SafeHarbor:**
[d72fd17](https://github.com/PatrickAlphaC/safe-harbor/commit/d72fd17a1e57aa9060f456e2b0d47591de6b9a56), [754edb9](https://github.com/PatrickAlphaC/safe-harbor/commit/754edb9c10e5830a99cf6ada4c8792728b25e2fe)에서 수정됨.

**Cyfrin:** 확인함.


### 동일한 storage read를 막기 위해 storage 캐싱

**설명:** 동일한 storage read가 반복되지 않도록 캐싱하는 것이 좋습니다.

* `Agreement.sol`
```solidity
// cache `accts.length` in `getDetails`
320:            _details.chains[i].accounts = new Account[](accts.length);
321:            for (uint256 j = 0; j < accts.length; ++j) {

// cache `chainAccounts.length` in `_findAccountIndex`
398:        for (uint256 i = 0; i < chainAccounts.length; i++) {
```

**SafeHarbor:**
커밋 [eb51eb0](https://github.com/PatrickAlphaC/safe-harbor/commit/eb51eb04669c5c48ca6fa692b2215593f7eae11b)에서 수정됨.

**Cyfrin:** 확인함.


### `calldata` 배열 길이는 캐싱하지 말 것

**설명:** [`calldata` 배열 길이는 캐싱하지 않는 편이 더 저렴합니다](https://github.com/devdacian/solidity-gas-optimization?tab=readme-ov-file#6-dont-cache-calldata-length-effective-009-cheaper). 이는 입력 read-only 배열에 대해 `memory`보다 `calldata`를 사용하는 것이 더 좋은 이유이기도 합니다.
* `ChainValidator::setvalidChains, setInvalidChains`

**SafeHarbor:**
커밋 [af8cbc7](https://github.com/PatrickAlphaC/safe-harbor/commit/af8cbc70851ffcf1fb8d393e8fb91e7ad077ad70)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
