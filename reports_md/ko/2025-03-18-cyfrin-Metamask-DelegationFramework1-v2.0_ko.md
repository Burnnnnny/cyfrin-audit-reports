**Lead Auditors**

[0kage](https://x.com/0kage_eth)

[Al-qaqa](https://x.com/Al_Qa_qa)

**Assisting Auditors**


---

# Findings
## High Risk


### 사용자 작업 해시에 `EntryPoint`가 포함되지 않아 리플레이 공격의 가능성이 생성됨 (EntryPoint not included in user operation hash creates the possibility of Replay Attacks)

**설명:** `EIP-4337`에 따르면, 사용자 작업(user operation) 해시는 동일한 체인 또는 다른 체인에서의 리플레이 공격(replay attacks)으로부터 보호할 수 있는 방식으로 구성되어야 합니다.

[eip-4337#useroperation](https://eips.ethereum.org/EIPS/eip-4337#useroperation)
> 리플레이 공격(크로스 체인 및 여러 `EntryPoint` 구현 모두)을 방지하기 위해, `signature`는 `chainid`와 `EntryPoint` 주소에 의존해야 합니다.

> `userOpHash`는 userOp(서명 제외), entryPoint 및 chainId에 대한 해시입니다.

현재 `DeleGatorCore` 및 `EIP7702DeleGatorCore` 구현에서 사용자 작업 해시에는 `entryPoint` 주소가 포함되지 않습니다.

```solidity
    function validateUserOp(
        PackedUserOperation calldata _userOp,
        bytes32, //@audit ignores UserOpHash from the Entry Point
        uint256 _missingAccountFunds
    ) ... {
        validationData_ = _validateUserOpSignature(_userOp, getPackedUserOperationTypedDataHash(_userOp));
        _payPrefund(_missingAccountFunds);
    }
// ------------------
    function getPackedUserOperationTypedDataHash(PackedUserOperation calldata _userOp) public view returns (bytes32) {
        return MessageHashUtils.toTypedDataHash(_domainSeparatorV4(), getPackedUserOperationHash(_userOp));
    }
// ------------------
    function getPackedUserOperationHash(PackedUserOperation calldata _userOp) public pure returns (bytes32) {
        return keccak256(
            abi.encode(
                PACKED_USER_OP_TYPEHASH,
                _userOp.sender,
                _userOp.nonce,
                keccak256(_userOp.initCode),
                keccak256(_userOp.callData),
                _userOp.accountGasLimits,
                _userOp.preVerificationGas,
                _userOp.gasFees,
                keccak256(_userOp.paymasterAndData)
            )
        ); //@audeit does not include entry point address
    }
```

위에서 `getPackedUserOperationHash()`를 통해 생성된 해시에 `EntryPoint` 주소가 포함되지 않았음을 확인하세요. `_domainSeparatorV4()`는 `chainId`와 `address(this)`만 포함하고 `EntryPoint` 주소는 제외합니다.

**영향:** 위임자(delegator) 계약을 새로운 `EntryPoint` 주소를 포함하도록 업그레이드하면 리플레이 공격의 가능성이 열립니다.

**개념 증명 (Proof of Concept):** 다음 POC는 위임자 계약이 새로운 `EntryPoint`로 업그레이드될 때 이전 네이티브 전송을 리플레이할 수 있는 가능성을 보여줍니다.

```solidity
contract EIP7702EntryPointReplayAttackTest is BaseTest {
    using MessageHashUtils for bytes32;

    constructor() {
        IMPLEMENTATION = Implementation.EIP7702Stateless;
        SIGNATURE_TYPE = SignatureType.EOA;
    }

    // New EntryPoint to upgrade to
    EntryPoint newEntryPoint;
    // Implementation with the new EntryPoint
    EIP7702StatelessDeleGator newImpl;

    function setUp() public override {
        super.setUp();

        // Deploy a second EntryPoint
        newEntryPoint = new EntryPoint();
        vm.label(address(newEntryPoint), "New EntryPoint");

        // Deploy a new implementation connected to the new EntryPoint
        newImpl = new EIP7702StatelessDeleGator(delegationManager, newEntryPoint);
        vm.label(address(newImpl), "New EIP7702 StatelessDeleGator Impl");
    }

    function test_replayAttackAcrossEntryPoints() public {
        // 1. Create a UserOp that will be valid with the original EntryPoint
        address aliceDeleGatorAddr = address(users.alice.deleGator);

        // A simple operation to transfer ETH to Bob
        Execution memory execution = Execution({ target: users.bob.addr, value: 1 ether, callData: hex"" });

        // Create the UserOp with current EntryPoint
        bytes memory userOpCallData = abi.encodeWithSignature(EXECUTE_SINGULAR_SIGNATURE, execution);
        PackedUserOperation memory userOp = createUserOp(aliceDeleGatorAddr, userOpCallData);

        // Alice signs it with the current EntryPoint's context
        userOp.signature = signHash(users.alice, getPackedUserOperationTypedDataHash(userOp));

        // Bob's initial balance for verification
        uint256 bobInitialBalance = users.bob.addr.balance;

        // Execute the original UserOp through the first EntryPoint
        PackedUserOperation[] memory userOps = new PackedUserOperation[](1);
        userOps[0] = userOp;
        vm.prank(bundler);
        entryPoint.handleOps(userOps, bundler);

        // Verify first execution worked
        uint256 bobBalanceAfterExecution = users.bob.addr.balance;
        assertEq(bobBalanceAfterExecution, bobInitialBalance + 1 ether);

        // 2. Modify code storage
        // The code will be: 0xef0100 || address of new implementation
        vm.etch(aliceDeleGatorAddr, bytes.concat(hex"ef0100", abi.encodePacked(newImpl)));

        // Verify the implementation was updated
        assertEq(address(users.alice.deleGator.entryPoint()), address(newEntryPoint));

        // 3. Attempt to replay the original UserOp through the new EntryPoint
        vm.prank(bundler);
        newEntryPoint.handleOps(userOps, bundler);

        // 4. Verify if the attack succeeded - check if Bob received ETH again
        assertEq(users.bob.addr.balance, bobBalanceAfterExecution + 1 ether);

        console.log("Bob's initial balance was: %d", bobInitialBalance / 1 ether);
        console.log("Bob's balance after execution on old entry point was: %d", bobBalanceAfterExecution / 1 ether);
        console.log("Bob's balance after replaying user op on new entry point: %d", users.bob.addr.balance / 1 ether);
    }
}
```

**권장 완화 방법:** `DeleGatorCore`와 `EIP7702DeleGatorCore` 모두의 `getPackedUserOperationHash`의 해싱 로직에 `EntryPoint` 주소를 포함하는 것을 고려하세요.

**Metamask:** 커밋 [1f91637](https://github.com/MetaMask/delegation-framework/commit/1f91637e7f61d03e012b7c9d7fc5ee4dc86ce3f3)에서 수정됨.

**Cyfrin:** 해결됨. `EntryPoint`가 이제 해싱 로직에 포함됩니다.

\clearpage
## Medium Risk


### ERC20 및 네이티브 전송에 대한 전송 금액 집행자가 실제 전송을 확인하지 않고 지출 한도를 증가시킴 (Transfer Amount enforcer for ERC20 and Native transfers increase spend limit without checking actual transfers)

**설명:** 실패한 토큰/네이티브 전송은 `EXECTYPE_TRY` 실행 모드와 함께 사용될 때 위임의 지출 허용량(allowance)을 잠재적으로 고갈시킬 수 있습니다.

이 문제는 집행자가 실제 토큰 전송이 발생하기 전에 `beforeHook`에서 카운터를 증가시켜 지출 한도를 추적하기 때문에 발생합니다. `EXECTYPE_TRY` 모드에서는 토큰 전송이 실패하더라도 실행은 revert되지 않고 계속되지만 한도는 여전히 증가합니다.

```solidity
function _validateAndIncrease(
    bytes calldata _terms,
    bytes calldata _executionCallData,
    bytes32 _delegationHash
)
    internal
    returns (uint256 limit_, uint256 spent_)
{
    // ... validation code ...

    //@audit This line increases the spent amount BEFORE the actual transfer happens
    spent_ = spentMap[msg.sender][_delegationHash] += uint256(bytes32(callData_[36:68]));
    require(spent_ <= limit_, "ERC20TransferAmountEnforcer:allowance-exceeded");
}

```

즉, 악의적인 대리인(delegate)은 실패하도록 설계된 전송을 반복적으로 시도하여 실제로 토큰을 전송하지 않고 허용량을 고갈시킬 수 있습니다.

**영향:** 이 취약점을 통해 공격자는 실제로 토큰을 전송하지 않고도 위임자(delegator)의 전체 토큰 전송 허용량을 잠재적으로 고갈시킬 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_transferFailsButSpentLimitIncreases() public {
        // Create a delegation from Alice to Bob with spending limits
        Caveat[] memory caveats = new Caveat[](3);

        // Allowed Targets Enforcer - allow only the token
        caveats[0] = Caveat({ enforcer: address(allowedTargetsEnforcer), terms: abi.encodePacked(address(mockToken)), args: hex"" });

        // Allowed Methods Enforcer - allow only transfer
        caveats[1] =
            Caveat({ enforcer: address(allowedMethodsEnforcer), terms: abi.encodePacked(IERC20.transfer.selector), args: hex"" });

        // ERC20 Transfer Amount Enforcer - limit to TRANSFER_LIMIT tokens
        caveats[2] = Caveat({
            enforcer: address(transferAmountEnforcer),
            terms: abi.encodePacked(address(mockToken), uint256(TRANSFER_LIMIT)),
            args: hex""
        });

        Delegation memory delegation = Delegation({
            delegate: address(users.bob.deleGator),
            delegator: address(users.alice.deleGator),
            authority: ROOT_AUTHORITY,
            caveats: caveats,
            salt: 0,
            signature: hex""
        });

        // Sign the delegation
        delegation = signDelegation(users.alice, delegation);

        // First, verify the initial spent amount is 0
        bytes32 delegationHash = EncoderLib._getDelegationHash(delegation);
        uint256 initialSpent = transferAmountEnforcer.spentMap(address(delegationManager), delegationHash);
        assertEq(initialSpent, 0, "Initial spent should be 0");

        // Initial balances
        uint256 aliceInitialBalance = mockToken.balanceOf(address(users.alice.deleGator));
        uint256 bobInitialBalance = mockToken.balanceOf(address(users.bob.addr));
        console.log("Alice initial balance:", aliceInitialBalance / 1e18);
        console.log("Bob initial balance:", bobInitialBalance / 1e18);

        // Amount to transfer
        uint256 amountToTransfer = 500 ether;

        // Create the mode for try execution
        ModeCode tryExecuteMode = ModeLib.encode(CALLTYPE_SINGLE, EXECTYPE_TRY, MODE_DEFAULT, ModePayload.wrap(bytes22(0x00)));

        // First test successful transfer
        {
            // Make sure token transfers will succeed
            mockToken.setHaltTransfer(false);

            // Prepare transfer execution
            Execution memory execution = Execution({
                target: address(mockToken),
                value: 0,
                callData: abi.encodeWithSelector(
                    IERC20.transfer.selector,
                    address(users.bob.addr), // Transfer to Bob's EOA
                    amountToTransfer
                )
            });

            // Execute the delegation with try mode
            execute_UserOp(
                users.bob,
                abi.encodeWithSelector(
                    delegationManager.redeemDelegations.selector,
                    createPermissionContexts(delegation),
                    createModes(tryExecuteMode),
                    createExecutionCallDatas(execution)
                )
            );

            // Check balances after successful transfer
            uint256 aliceBalanceAfterSuccess = mockToken.balanceOf(address(users.alice.deleGator));
            uint256 bobBalanceAfterSuccess = mockToken.balanceOf(address(users.bob.addr));
            console.log("Alice balance after successful transfer:", aliceBalanceAfterSuccess / 1e18);
            console.log("Bob balance after successful transfer:", bobBalanceAfterSuccess / 1e18);

            // Check spent map was updated
            uint256 spentAfterSuccess = transferAmountEnforcer.spentMap(address(delegationManager), delegationHash);
            console.log("Spent amount after successful transfer:", spentAfterSuccess / 1e18);
            assertEq(spentAfterSuccess, amountToTransfer, "Spent amount should be updated after successful transfer");

            // Verify the transfer actually occurred
            assertEq(aliceBalanceAfterSuccess, aliceInitialBalance - amountToTransfer);
            assertEq(bobBalanceAfterSuccess, bobInitialBalance + amountToTransfer);
        }

        // Now test failing transfer
        {
            // Make token transfers fail
            mockToken.setHaltTransfer(true);

            // Prepare failing transfer execution
            Execution memory execution = Execution({
                target: address(mockToken),
                value: 0,
                callData: abi.encodeWithSelector(
                    IERC20.transfer.selector,
                    address(users.bob.addr), // Transfer to Bob's EOA
                    amountToTransfer
                )
            });

            // Execute the delegation with try mode
            execute_UserOp(
                users.bob,
                abi.encodeWithSelector(
                    delegationManager.redeemDelegations.selector,
                    createPermissionContexts(delegation),
                    createModes(tryExecuteMode),
                    createExecutionCallDatas(execution)
                )
            );

            // Check balances after failed transfer
            uint256 aliceBalanceAfterFailure = mockToken.balanceOf(address(users.alice.deleGator));
            uint256 bobBalanceAfterFailure = mockToken.balanceOf(address(users.bob.addr));
            console.log("Alice balance after failed transfer:", aliceBalanceAfterFailure / 1e18);
            console.log("Bob balance after failed transfer:", bobBalanceAfterFailure / 1e18);

            // Check spent map after failed transfer
            uint256 spentAfterFailure = transferAmountEnforcer.spentMap(address(delegationManager), delegationHash);
            console.log("Spent amount after failed transfer:", spentAfterFailure / 1e18);

            // THE KEY TEST: The spent amount increased even though the transfer failed!
            assertEq(spentAfterFailure, amountToTransfer * 2, "Spent amount should increase even with failed transfer");

            // Verify tokens weren't actually transferred
            assertEq(aliceBalanceAfterFailure, aliceInitialBalance - amountToTransfer);
            assertEq(bobBalanceAfterFailure, bobInitialBalance + amountToTransfer);
        }
    }
```

**권장 완화 방법:** 다음 옵션 중 하나를 고려하세요:
1. 다음 단계로 afterHook()에 사후 확인(post check)을 구현합니다:
    - beforeHook에서 초기 잔액 추적
    - afterHook에서 실제 잔액 추적
    - afterHook에서 (실제 - 초기)를 기반으로만 지출 한도 업데이트

2. 또는 `TransferAmountEnforcers`가 `EXECTYPE_DEFAULT` 실행 유형만 갖도록 강제합니다.

**Metamask:** 커밋 [cdd39c6](https://github.com/MetaMask/delegation-framework/commit/cdd39c62d65436da0d97bff53a7a5714a3505453)에서 수정됨.

**Cyfrin:** 해결됨. 실행 유형을 `EXECTYPE_DEFAULT`로만 제한함.


### `Allowed` 클래스 집행자의 중복 항목을 통한 가스 그리핑(Gas griefing) (Gas griefing via duplicate entries in `Allowed` class of enforcers)

**설명:** 여러 집행자 계약(`AllowedMethodsEnforcer` 및 `AllowedTargetsEnforcer`)은 조건 데이터에서 항목의 고유성을 검증하지 않아, 악의적인 사용자가 의도적으로 과도한 중복이 포함된 위임을 생성하여 검증 중 가스 비용을 크게 증가시킬 수 있습니다.

`AllowedMethodsEnforcer`: 중복 메서드 선택자(각 4바이트) 허용
`AllowedTargetsEnforcer`: 중복 대상 주소(각 20바이트) 허용

이러한 계약 중 어느 것도 조건 데이터에서 중복을 방지하거나 감지하지 않습니다. 이를 통해 공격자는 동일한 항목을 여러 번 포함하여 인위적으로 가스 비용을 부풀려 검증 중 비용이 많이 드는 선형 검색 작업을 수행할 수 있습니다.


```solidity
function getTermsInfo(bytes calldata _terms) public pure returns (bytes4[] memory allowedMethods_) {
    uint256 j = 0;
    uint256 termsLength_ = _terms.length;
    require(termsLength_ % 4 == 0, "AllowedMethodsEnforcer:invalid-terms-length");
    allowedMethods_ = new bytes4[](termsLength_ / 4);
    for (uint256 i = 0; i < termsLength_; i += 4) {
        allowedMethods_[j] = bytes4(_terms[i:i + 4]);
        j++;
    }
}

// In beforeHook:
for (uint256 i = 0; i < allowedSignaturesLength_; ++i) {
    if (targetSig_ == allowedSignatures_[i]) { //@audit linear search can be expensive
        return;
    }
}
```

**영향:** `AllowedMethodsEnforcer`의 테스트 케이스에서 입증된 바와 같이, 100개의 중복 메서드 서명을 포함하면 가스 소비가 50,881에서 155,417로 증가합니다(104,536 가스 차이). 이는 단 100개의 중복으로 가스 소비가 약 3배 증가한 것을 나타냅니다. 악의적인 행위자는 이를 사용하여 위임자의 실행 비용을 비싸게 만들 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 실행하세요:

```solidity
 function test_AllowedMethods_DuplicateMethodsGriefing() public {
        // Create terms with a high number of duplicated methods to increase gas costs
        bytes memory terms = createDuplicateMethodsTerms(INCREMENT_SELECTOR, 100);

        // Create execution to increment counter
        Execution memory execution =
            Execution({ target: address(aliceCounter), value: 0, callData: abi.encodeWithSelector(INCREMENT_SELECTOR) });

        // Create delegation with allowed methods caveat
        Caveat[] memory caveats = new Caveat[](1);
        caveats[0] = Caveat({ enforcer: address(allowedMethodsEnforcer), terms: terms, args: "" });

        // Create and sign the delegation
        Delegation memory delegation = Delegation({
            delegate: address(users.bob.deleGator),
            delegator: address(users.alice.deleGator),
            authority: ROOT_AUTHORITY,
            caveats: caveats,
            salt: 0,
            signature: hex""
        });

        delegation = signDelegation(users.alice, delegation);

        // Measure gas usage with many duplicate methods
        Delegation[] memory delegations = new Delegation[](1);
        delegations[0] = delegation;

        uint256 gasUsed = uint256(
            bytes32(
                gasReporter.measureGas(
                    address(users.bob.deleGator),
                    address(delegationManager),
                    abi.encodeWithSelector(
                        delegationManager.redeemDelegations.selector,
                        createPermissionContexts(delegation),
                        createModes(),
                        createExecutionCallDatas(execution)
                    )
                )
            )
        );

        console.log("Gas used with 100 duplicate methods:", gasUsed);

        // Now compare to normal case with just one method
        terms = abi.encodePacked(INCREMENT_SELECTOR);
        caveats[0].terms = terms;

        delegation.caveats = caveats;
        delegation = signDelegation(users.alice, delegation);

        delegations[0] = delegation;

        uint256 gasUsedNormal = uint256(
            bytes32(
                gasReporter.measureGas(
                    address(users.bob.deleGator),
                    address(delegationManager),
                    abi.encodeWithSelector(
                        delegationManager.redeemDelegations.selector,
                        createPermissionContexts(delegation),
                        createModes(),
                        createExecutionCallDatas(execution)
                    )
                )
            )
        );

        console.log("Gas used with 1 method:", gasUsedNormal);
        console.log("Gas diff:", gasUsed - gasUsedNormal);

        assertGt(gasUsed, gasUsedNormal, "Griefing with duplicate methods should use more gas");
    }

    function createDuplicateMethodsTerms(bytes4 selector, uint256 count) internal pure returns (bytes memory) {
        bytes memory terms = new bytes(count * 4);
        for (uint256 i = 0; i < count; i++) {
            bytes4 methodSig = selector;
            for (uint256 j = 0; j < 4; j++) {
                terms[i * 4 + j] = methodSig[j];
            }
        }
        return terms;
    }

    function createPermissionContexts(Delegation memory del) internal pure returns (bytes[] memory) {
        Delegation[] memory delegations = new Delegation[](1);
        delegations[0] = del;

        bytes[] memory permissionContexts = new bytes[](1);
        permissionContexts[0] = abi.encode(delegations);

        return permissionContexts;
    }

    function createExecutionCallDatas(Execution memory execution) internal pure returns (bytes[] memory) {
        bytes[] memory executionCallDatas = new bytes[](1);
        executionCallDatas[0] = ExecutionLib.encodeSingle(execution.target, execution.value, execution.callData);
        return executionCallDatas;
    }

    function createModes() internal view returns (ModeCode[] memory) {
        ModeCode[] memory modes = new ModeCode[](1);
        modes[0] = mode;
        return modes;
    }
```
**권장 완화 방법:** 조건의 항목이 엄격하게 증가하도록 강제하여 자연스럽게 중복을 방지하는 것을 고려하세요. 아래와 같은 유효성 검사는 이 경우 중복을 방지합니다. 또한, 선형 검색 대신 정렬된 배열에 대한 이진 검색을 구현하는 것을 고려하세요.

```solidity
function getTermsInfo(bytes calldata _terms) public pure returns (bytes4[] memory allowedMethods_) {
    uint256 termsLength_ = _terms.length;
    require(termsLength_ % 4 == 0, "AllowedMethodsEnforcer:invalid-terms-length");
    allowedMethods_ = new bytes4[](termsLength_ / 4);

    bytes4 previousSelector = bytes4(0);

    for (uint256 i = 0; i < termsLength_; i += 4) {
        bytes4 currentSelector = bytes4(_terms[i:i + 4]);

        // Ensure selectors are strictly increasing (prevents duplicates)
        require(uint32(currentSelector) > uint32(previousSelector),
                "AllowedMethodsEnforcer:selectors-must-be-strictly-increasing"); //@audit prevents duplicates

        allowedMethods_[i/4] = currentSelector;
        previousSelector = currentSelector;
    }
}
```

**Metamask:** 인지함. 그리핑 공격을 방지하도록 조건을 설정하는 것은 상환자(redeemer)의 책임입니다.

**Cyfrin:** 인지함.

\clearpage
## Low Risk


### `EIP7702StatelessDeleGator`가 `EIP4337` 서명 유효성 검사 표준을 위반함 (`EIP7702StatelessDeleGator` violates `EIP4337` signature validation standards)

**설명:** `EIP7702StatelessDeleGator` 구현은 서명 유효성 검사 동작과 관련하여 EIP4337 표준을 적절하게 준수하지 않습니다.

EIP4337에 따르면, `validateUserOp` 메서드는 다음을 수행해야 합니다:

1. 서명 불일치 경우에만 SIG_VALIDATION_FAILED 반환 (revert 없이)
2. 다른 오류(유효하지 않은 서명 형식, 잘못된 길이 포함)의 경우 Revert

`EIP7702StatelessDeleGator._isValidSignature()`의 현재 구현은 서명 불일치 및 유효하지 않은 서명 길이 모두에 대해 `SIG_VALIDATION_FAILED`를 반환하며, 이는 사양을 위반합니다:

```solidity
// EIP7702StatelessDeleGator.sol
    function _isValidSignature(bytes32 _hash, bytes calldata _signature) internal view override returns (bytes4) {
        if (_signature.length != SIGNATURE_LENGTH) return ERC1271Lib.SIG_VALIDATION_FAILED;

        if (ECDSA.recover(_hash, _signature) == address(this)) return ERC1271Lib.EIP1271_MAGIC_VALUE;

        return ERC1271Lib.SIG_VALIDATION_FAILED;
    }
```

이 구현은 `EntryPoint`가 `UserOperations`의 유효성을 검사할 때 사용되며, 잘못된 처리는 다른 EIP4337 호환 지갑과의 불일치로 이어질 수 있습니다.

**영향:** 다른 EIP4337 호환 지갑과의 잠재적인 불일치 생성

**권장 완화 방법:** `EIP7702StatelessDeleGator._isValidSignature()`에서 서명 길이 확인을 제거하는 것을 고려하세요. 이렇게 하면 EIP4337 사양과 일치하게 함수가 OpenZeppelin의 `ECDSA.recover()` 함수에 의존하여 유효하지 않은 서명 형식에 대해 revert하게 됩니다.

**Metamask**
[b52cf04](https://github.com/MetaMask/delegation-framework/commit/b52cf041f5a41914d9014d777da9a3db17080fa6)에서 수정됨.

**Cyfrin**
해결됨.


### `AllowedCalldataEnforcer`가 빈 calldata를 인증할 수 없어 `receive()` 함수 호출을 방지함 (`AllowedCalldataEnforcer` cannot authenticate empty calldata preventing `receive()` function calls)

**설명:** `AllowedCalldataEnforcer` 계약에는 빈 calldata로 호출을 인증하는 것을 방지하는 설계 결함이 있습니다. 이 문제는 `_terms.length >= 33`을 강제하는 `getTermsInfo` 함수의 요구 사항 확인에서 발생합니다:

```solidity
// AllowedCalldataEnforcer.sol
    function getTermsInfo(bytes calldata _terms) public pure returns (uint256 dataStart_, bytes memory value_) {
        require(_terms.length >= 33, "AllowedCalldataEnforcer:invalid-terms-size");
        dataStart_ = uint256(bytes32(_terms[0:32]));
        value_ = _terms[32:];
    }
```

처음 32바이트는 calldata의 시작 오프셋을 나타내고, 그 이후의 모든 것은 일치시킬 예상 값을 나타냅니다. 이 설계는 값에 대해 최소 1바이트를 요구하므로 집행자가 빈 calldata 시나리오를 처리하지 못하게 합니다.

이더리움 계약은 다음과 같이 빈 calldata가 있는 함수를 통해 ETH를 받을 수 있습니다:
- 빈 calldata가 필요한 계약의 `receive()` 함수를 호출할 때
- `receive()`를 구현하는 계약에 간단한 ETH 전송을 할 때

**영향:** 사용자는 `receive()`가 있는 계약으로의 간단한 ETH 전송만 허용해야 하는 위임을 승인하기 위해 `AllowedCalldataEnforcer`를 사용할 수 없습니다. ETH를 WETH(receive() 함수 사용)에 입금하는 것과 같은 일반적인 사용 사례는 이 조건을 통해 적절하게 강제될 수 없습니다.


**권장 완화 방법:** `getTermsInfo` 함수를 수정하여 정확히 32바이트 길이의 조건을 허용하고, 이를 값이 비어 있는 특수한 경우로 처리하는 것을 고려하세요:

```solidity
function getTermsInfo(bytes calldata _terms) public pure returns (uint256 dataStart_, bytes memory value_) {
    require(_terms.length >= 32, "AllowedCalldataEnforcer:invalid-terms-size");
    dataStart_ = uint256(bytes32(_terms[0:32]));
    if (_terms.length == 32) {
        value_ = new bytes(0); // @audit Empty bytes for empty calldata
    } else {
        value_ = _terms[32:];
    }
}
```

**Metamask:** 커밋 [db11bf7](https://github.com/MetaMask/delegation-framework/commit/db11bf74034a94bddaae189299cb757fd03cadeb)에서 처리됨.

**Cyfrin:** 해결됨. 문제를 해결하기 위해 새로운 조건 `ExactCalldataEnforcer`를 추가함.


### `ERC721TransferEnforcer`를 사용하면 NFT 안전 전송(safe transfer)이 revert됨 (NFT safe transfers will revert using `ERC721TransferEnforcer`)

**설명:** `ERC721TransferEnforcer` 집행자는 NFT 토큰 전송을 승인하도록 설계되었지만, 현재 `transferFrom` 선택자만 사용하도록 전송을 제한합니다. 문제는 이 코드 섹션에 있습니다:

```solidity
//ERC721TransferEnforcer.sol
bytes4 selector_ = bytes4(callData_[0:4]);

if (target_ != permittedContract_) {
    revert("ERC721TransferEnforcer:unauthorized-contract-target");
} else if (selector_ != IERC721.transferFrom.selector) {
    revert("ERC721TransferEnforcer:unauthorized-selector");
} else if (transferTokenId_ != permittedTokenId_) {
    revert("ERC721TransferEnforcer:unauthorized-token-id");
}
```

ERC721 표준에는 여러 전송 메서드가 포함되어 있습니다:

`transferFrom(address from, address to, uint256 tokenId)`
`safeTransferFrom(address from, address to, uint256 tokenId)`
`safeTransferFrom(address from, address to, uint256 tokenId, bytes data)`

집행자는 현재 수신자 기능 확인을 수행하지 않는 `transferFrom` 메서드만 지원합니다. `safeTransferFrom` 메서드는 수신 계약이 onERC721Received 콜백을 통해 ERC721 표준을 지원하는지 확인하므로 NFT를 계약으로 안전하게 전송하는 데 중요합니다.


**영향:** 사용자는 이 집행자를 사용할 때 더 안전한 NFT 전송 메서드를 활용할 수 없습니다.


**권장 완화 방법:** `ERC721TransferEnforcer`가 모든 유효한 ERC721 전송 선택자를 수락하도록 수정하는 것을 고려하세요.

**Metamask:** [1a0ff2d](https://github.com/MetaMask/delegation-framework/commit/1a0ff2d92162f61945154d0a56a52b5878a2c9d0)에서 수정됨.

**Cyfrin:** 해결됨.


### `IdEnforcer::beforeHook()` 이벤트 방출에서 매개변수 불일치 (Parameter mismatch in `IdEnforcer::beforeHook()` event emission)

**설명:** `IdEnforcer` 계약의 `beforeHook()` 함수에서 `UsedId` 이벤트를 방출할 때 매개변수 순서 불일치가 있습니다. 이벤트 선언과 실제 방출은 위임자(delegator)와 상환자(redeemer) 인수에 대해 다른 매개변수 순서를 사용합니다.

이벤트는 다음 순서의 매개변수로 선언됩니다:

```solidity
//IdEnforcer.sol
    event UsedId(address indexed sender, address indexed delegator, address indexed redeemer, uint256 id);
// ---------
    function beforeHook( ... ) ... {
        ...
        emit UsedId(msg.sender, _redeemer, _delegator, id_);
    }
```

**영향:** 잘못된 이벤트 데이터 인덱싱.

**권장 완화 방법:** 이벤트 선언에 맞추기 위해 이벤트 방출에서 `_redeemer`와 `_delegator`의 위치를 바꾸는 것을 고려하세요.

**Metamask:** [5f8bea9](https://github.com/MetaMask/delegation-framework/commit/5f8bea93ea2f16b48104c901c69df504355546bd)에서 수정됨.

**Cyfrin:** 해결됨.


### `TimestampEnforcer`에서 일관되지 않은 타임스탬프 범위 유효성 검사 (Inconsistent timestamp range validation in `TimestampEnforcer`)

**설명:** `TimestampEnforcer` 계약은 시간 기반 유효성 창이 있는 위임 생성을 허용합니다. 그러나 시간 범위의 논리적 일관성을 보장하는 유효성 검사가 부족합니다.

구체적으로, "after" 및 "before" 임계값이 모두 지정된 경우 `timestampBeforeThreshold_`가 `timestampAfterThreshold_`보다 큰지 확인하는 검사가 없습니다.

```solidity
//TimeStampEnforcer.sol

function getTermsInfo(bytes calldata _terms)
    public
    pure
    returns (uint128 timestampAfterThreshold_, uint128 timestampBeforeThreshold_)
{
    require(_terms.length == 32, "TimestampEnforcer:invalid-terms-length");
    timestampBeforeThreshold_ = uint128(bytes16(_terms[16:]));
    timestampAfterThreshold_ = uint128(bytes16(_terms[:16]));
}
```

두 타임스탬프가 모두 0이 아닌 경우 다음과 같은 독립적인 검사가 있습니다.

```solidity
//TimeStampEnforcer.sol

require(block.timestamp > timestampAfterThreshold_, "TimestampEnforcer:early-delegation");
require(block.timestamp < timestampBeforeThreshold_, "TimestampEnforcer:expired-delegation");
//@audit no check on validity of the range
```
이로 인해 "after" 임계값이 "before" 임계값보다 커서 영구적으로 사용할 수 없는 위임이 발생하는 일관성 없는 시간 범위의 가능성이 생성됩니다.

**영향:** 유효한 것처럼 보이지만 불가능한 시간 제약으로 인해 결코 행사할 수 없는 위임이 생성될 수 있습니다.

**권장 완화 방법:** 두 타임스탬프 임계값이 모두 0이 아닌 경우 "before" 임계값이 "after" 임계값보다 큰지 확인하기 위해 `getTermsInfo` 함수에 유효성 검사 확인을 추가하는 것을 고려하세요.

**Metamask:** 인지함. 위임에 대한 올바른 조건을 생성하는 것은 위임자에게 있습니다.

**Cyfrin:** 인지함.

\clearpage
## Informational


### `DelegationManager`는 승인된 해시가 있는 스마트 계약 지갑과 호환되지 않음 (DelegationManager is incompatible with smart contract wallets with Approved hashes)

**설명:** `DelegationManager` 계약에는 Safe(이전 Gnosis Safe) 지갑과 같이 사전 승인된 해시 기능을 구현하는 스마트 계약 지갑에 영향을 미치는 제한 사항이 있습니다. 현재 구현은 EOA 및 스마트 계약 지갑 모두에 대해 빈 서명이 있는 위임을 거부합니다.

```solidity
// DelegationManager.sol
    function redeemDelegations( ... ) ... {
        ...
        for (uint256 batchIndex_; batchIndex_ < batchSize_; ++batchIndex_) {
            ...
            if (delegations_.length == 0) { ... } else {
                ...
                for (uint256 delegationsIndex_; delegationsIndex_ < delegations_.length; ++delegationsIndex_) {
                    ...
                    if (delegation_.signature.length == 0) {
                        // Ensure that delegations without signatures revert
                        revert EmptySignature();
                    }

                    if (delegation_.delegator.code.length == 0) {
                        // Validate delegation if it's an EOA
                        ...
                    } else {
                        // Validate delegation if it's a contract
                        ...
                        bytes32 result_ = IERC1271(delegation_.delegator).isValidSignature(typedDataHash_, delegation_.signature);
                        ...
                    }
                }
```

이는 빈 서명이 사전 승인된 해시 확인을 트리거하는 패턴을 사용하는 Safe 지갑 및 유사한 스마트 계약 지갑과 호환성 문제를 나타냅니다. Safe의 구현에서:

[SignatureVerifierMuxer.sol#L163-L165](https://github.com/safe-global/safe-smart-account/blob/main/contracts/handler/extensible/SignatureVerifierMuxer.sol#L163-L165)
```solidity
    function isValidSignature(bytes32 _hash, bytes calldata signature) external view override returns (bytes4 magic) {
        (ISafe safe, address sender) = _getContext();

        // Check if the signature is for an `ISafeSignatureVerifier` and if it is valid for the domain.
        if (signature.length >= 4) {
            ...

            // Guard against short signatures that would cause abi.decode to revert.
            if (sigSelector == SAFE_SIGNATURE_MAGIC_VALUE && signature.length >= 68) { ... }
        }

        // domainVerifier doesn't exist or the signature is invalid for the domain - fall back to the default
        return defaultIsValidSignature(safe, _hash, signature);
    }
// -----------------
    function defaultIsValidSignature(ISafe safe, bytes32 _hash, bytes memory signature) internal view returns (bytes4 magic) {
        bytes memory messageData = EIP712.encodeMessageData( ... );
        bytes32 messageHash = keccak256(messageData);
        if (signature.length == 0) {
            // approved hashes
            require(safe.signedMessages(messageHash) != 0, "Hash not approved");
        } else {
            // threshold signatures
            safe.checkSignatures(address(0), messageHash, signature);
        }
        magic = ERC1271.isValidSignature.selector;
    }
```

DelegationManager는 `isValidSignature`를 호출하기 전에 빈 서명을 거부하므로 Safe 지갑이 위임에 사전 승인된 해시 메커니즘을 사용하는 것을 방지합니다.

**영향:** 이 제한은 Safe 지갑 및 유사한 스마트 계약 지갑이 위임과 함께 가스 효율적인 사전 승인된 해시 메커니즘을 사용하는 것을 방지합니다.

**권장 완화 방법:** Safe 지갑의 사전 승인된 해시 메커니즘과의 호환성을 활성화하려면 빈 서명 확인을 EOA에만 적용하는 것을 고려하세요. 또는 현재 위임 프레임워크에서 Safe 지갑 사전 승인된 해시가 지원되지 않음을 문서화하는 것을 고려하세요.

**Metamask:** 커밋 [155d20c](https://github.com/MetaMask/delegation-framework/commit/155d20c8bf173d556ef738ec808b3583da1a7c9d)에서 수정됨.

**Cyfrin:** 해결됨.


### `NotSelf()` 오류 선언이 사용되지 않음 (`NotSelf()` error declaration is unused)

**설명:** `DeleGatorCore` 계약과 `EIP7702DeleGatorCore` 계약은 모두 `NotSelf()`라는 사용자 정의 오류를 정의하지만, 이 오류는 두 계약이나 파생 구현 어디에서도 실제로 발생하지 않습니다.

```solidity
   // DeleGatorCore and EIP7702DeleGatorcore
    /// @dev Error thrown when the caller is not this contract.
    error NotSelf();
```

**권장 완화 방법:** 이 두 계약에서 오류를 제거하는 것을 고려하세요.

**Metamask:** 인지함. 상속하는 구현이 나중에 활용하려는 경우를 대비하여 유지할 것입니다.

**Cyfrin:** 인지함.


### `AllowedMethodsEnforcer::getTermsInfo()`에서 길이 0 확인 누락 (Missing zero length check in `AllowedMethodsEnforcer::getTermsInfo()`)

**설명:** `AllowedMethodsEnforcer` 계약의 `getTermsInfo()` 함수는 제공된 조건 길이가 4로 나누어 떨어지는지(각 메서드 선택자가 4바이트이므로) 올바르게 검증하지만, 빈 조건(길이 == 0)을 거부하지 못합니다. 빈 조건은 `0 % 4 == 0`이므로 기술적으로 모듈로 확인을 통과하지만, 허용된 메서드의 빈 배열을 초래합니다.

**영향:** 빈 조건이 있는 위임은 집행자가 허용된 메서드의 빈 배열을 반복한 다음 "AllowedMethodsEnforcer:invalid-terms-length"로 유효하지 않은 조건을 적절하게 거부하는 대신 "AllowedMethodsEnforcer:method-not-allowed"로 revert되므로 메서드 호출을 허용하지 않습니다.

**권장 완화 방법:** 조건 길이가 0보다 큰지 확인하는 특정 확인을 추가하는 것을 고려하세요.

**Metamask:** 커밋 [cb2d4d7](https://github.com/MetaMask/delegation-framework/commit/cb2d4d77a66643e541141b3c8291df52340d60ce)에서 수정됨.

**Cyfrin:** 해결됨.


### `NativeTokenPaymentEnforcer`에서 위임자 주소 유효성 검사 불충분 (Insufficient delegate address validation in `NativeTokenPaymentEnforcer`)

**설명:** `NativeTokenPaymentEnforcer` 계약의 `afterAllHook()` 함수는 허용량 위임을 사용하여 지불을 처리하기 위해 `delegationManager.redeemDelegations()`를 호출합니다. 그러나 허용량 위임의 대리인 주소가 집행자 계약 자체(`address(this)`)이거나 특수 `ANY_DELEGATE` 주소인지 제대로 검증하지 않습니다.

```solidity
   // NativeTokenPaymentEnforcer.sol
    function afterAllHook( ... ) ... {
        ...
        Delegation[] memory allowanceDelegations_ = abi.decode(_args, (Delegation[]));

        ...
        // Attempt to redeem the delegation and make the payment
        delegationManager.redeemDelegations(permissionContexts_, encodedModes_, executionCallDatas_);

        ...
    }
```

호출자가 지정된 대리인이거나 대리인이 `ANY_DELEGATE`로 설정된 경우에만 위임이 성공적으로 상환되므로 이 유효성 검사는 중요합니다. 이 확인이 없으면 호출자(`NativeTokenPaymentEnforcer` 계약)가 대리인으로 승인되지 않은 경우 지불 프로세스가 예기치 않게 실패할 수 있습니다.

**영향:** 지불 트랜잭션이 예기치 않게 revert될 수 있습니다.

**권장 완화 방법:** 허용량 위임의 대리인 주소가 `address(this)` 또는 특수 `ANY_DELEGATE` 주소인지 확인하는 검사를 추가하는 것을 고려하세요.

**Metamask:** 인지함. `DelegationManager`에서 revert될 것입니다.

**Cyfrin:** 인지함.

\clearpage
## Gas Optimization


### `WebAuthn::contains()` 최적화 (`WebAuthn::contains()` optimization)

**설명:** `WebAuthn::contains()` 함수에서 현재 구현은 각 반복마다 루프 내부에서 범위를 벗어난 조건을 확인합니다. 동일한 확인이 반복적으로 수행되므로 비효율적입니다.

```solidity
//WebAuthn.sol
function contains(string memory substr, string memory str, uint256 location) internal pure returns (bool) {
    bytes memory substrBytes = bytes(substr);
    bytes memory strBytes = bytes(str);

    uint256 substrLen = substrBytes.length;
    uint256 strLen = strBytes.length;

    for (uint256 i = 0; i < substrLen; i++) {
        if (location + i >= strLen) {
            return false;
        }
        ...
    }

    return true;
}
```

**권장 완화 방법:** 루프에 들어가기 전에 단일 경계 확인을 수행하는 것을 고려하세요. 이는 하위 문자열이 지정된 위치의 메인 문자열 내에 맞는지 확인합니다.

**Metamask:** 인지함. SmoothCyptoLib에서 가져옴 - 일관성을 위해 유지할 것입니다.

**Cyfrin:** 인지함.

\clearpage
