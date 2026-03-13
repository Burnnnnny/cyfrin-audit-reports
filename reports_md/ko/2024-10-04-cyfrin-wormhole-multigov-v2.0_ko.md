**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Hans](https://twitter.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Low Risk


### Lack of access control in `HubProposalExtender::initialize`

**Description:** `HubProposalExtender::initialize`는 한 번만 호출될 수 있습니다. 이 함수에 접근 제어가 없으면 공격자가 이 함수를 프론트 러닝(front-run)하여 악성 거버너(Governor)를 설정할 수 있습니다. 이는 익스텐더(extender) 관리자가 제안의 기간을 연장하는 것을 방지할 수 있습니다.

**Impact:** initialize를 프론트 러닝하면 잠재적으로 제안 연장을 방지할 수 있습니다.

**Proof of Concept:** 다음 퍼즈 테스트를 실행하십시오:

```solidity
  function testFuzz_FrontRun_HubProposalExtender_Initialize(address attacker, address maliciousGovernor) public {
    vm.assume(attacker != address(0) && attacker != address(this) && attacker != address(timelock));
    vm.assume(maliciousGovernor != address(0) && maliciousGovernor != address(governor));

    // Create a new HubProposalExtender
    HubProposalExtenderHarness newExtender = new HubProposalExtenderHarness(
        whitelistedExtender,
        extensionDuration,
        address(timelock),
        minimumTime,
        voteWeightWindow,
        minimumTime
    );

    // Attacker front-runs the initialization
    vm.prank(attacker);
    newExtender.initialize(payable(maliciousGovernor));

    // Assert that the malicious governor was set
    assertEq(address(newExtender.governor()), maliciousGovernor);

    // Attempt to initialize again with the intended governor (this should fail)
    vm.prank(address(timelock));
    vm.expectRevert(HubProposalExtender.AlreadyInitialized.selector);
    newExtender.initialize(payable(address(governor)));

    // Assert that the governor is still the malicious one
    assertEq(address(newExtender.governor()), maliciousGovernor);
}
```

**Recommended Mitigation:** `initialize` 함수에 대한 접근을 컨트랙트 소유자로만 제한하는 것을 고려하십시오.

**Wormhole Foundation**
[PR 177](https://github.com/wormhole-foundation/example-multigov/pull/177/commits/7c74ecea48fe3379d1b8a5c7b2374aa3adeb997e)에서 수정됨.

**Cyfrin:** 확인됨.


### Lack of zero address checks at multiple places

**Description:** 일반적으로 코드베이스에 제로 주소 확인과 같은 입력 유효성 검사가 부족합니다.

`HubVotePool::constructor`

```solidity
  constructor(address _core, address _hubGovernor, address _owner) QueryResponse(_core) Ownable(_owner) { //@note initialized query response with womrhole core
    hubGovernor = IGovernor(_hubGovernor);
    HubEvmSpokeVoteDecoder evmDecoder = new HubEvmSpokeVoteDecoder(_core, address(this));
    _registerQueryType(address(evmDecoder), QueryResponse.QT_ETH_CALL_WITH_FINALITY);
  }
```

`HubVotePool::registerSpoke`
```solidity
  function registerSpoke(uint16 _targetChain, bytes32 _spokeVoteAddress) external {
    _checkOwner();
    _registerSpoke(_targetChain, _spokeVoteAddress);
  }
```

`HubPool::setGovernor`
```solidity
 function setGovernor(address _newGovernor) external {
    _checkOwner();
    hubGovernor = IGovernor(_newGovernor);
}
```

`HubProposalExtender::setVoteExtenderAdmin`
```solidity
 function setVoteExtenderAdmin(address _voteExtenderAdmin) external {
    _checkOwner();
    _setVoteExtenderAdmin(_voteExtenderAdmin); //@audit @info missing zero address check
  }
```

**Recommended Mitigation:** 해당되는 경우 제로 주소 확인을 추가하는 것을 고려하십시오.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.


### Lack of limits of vote weight window can lead to unfair voting power estimation

**Description:** `HubGovernor::setVoteWeightWindow`에는 투표 가중치 윈도우 기간에 대한 확인이 없습니다. 거버넌스 전용 함수이긴 하지만, 투표 가중치 기간을 매우 짧게 설정하거나 매우 길게 설정하면 불공정한 투표권 추정으로 이어질 수 있습니다.

```solidity
  function setVoteWeightWindow(uint48 _weightWindow) external {
    _checkGovernance();
    _setVoteWeightWindow(_weightWindow);
  }
```

```solidity
  function _setVoteWeightWindow(uint48 _windowLength) internal { //@audit no check on windowLength
    emit VoteWeightWindowUpdated(
      SafeCast.toUint48(voteWeightWindowLengths.upperLookup(SafeCast.toUint48(block.timestamp))), _windowLength
    );
    voteWeightWindowLengths.push(SafeCast.toUint96(block.timestamp), uint160(_windowLength));
  }
```
**Impact:** 매우 짧은 투표 가중치 기간은 사용자의 토큰 조작을 유도할 수 있으며, 매우 긴 투표 기간은 최근 보유자에게 불이익을 줄 것입니다.

**Recommended Mitigation:** 투표 가중치 기간을 거버넌스에 전적으로 맡기는 대신 합리적인 최소 및 최대 제한을 추가하는 것을 고려하십시오.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.


### `HubGovernor` lacks validation on `HubProposalExtender` ownership

**Description:** `HubGovernor` 컨트랙트는 생성자에서 `HubProposalExtender` 주소를 불변(immutable) 매개변수로 받습니다. 그러나 이 익스텐더가 `timelock` 컨트랙트에 의해 소유되었는지 확인하지 않습니다.

이러한 간과로 인해 거버넌스 시스템에 의해 제어되지 않는 HubProposalExtender를 사용하여 HubGovernor 인스턴스를 배포할 수 있으며, 잠재적으로 중요한 거버넌스 제어를 우회할 수 있습니다.

`DeployHubContractsBaseImpl.s.sol`의 배포 스크립트는 `timelock` 주소가 아닌 `config.voteExtenderAdmin` 주소를 소유자로 하여 익스텐더를 배포합니다.

`DeployHubContractsBaseImpl.s.sol`
```solidity
 function run()
    public
    returns (
      TimelockController,
      HubVotePool,
      HubGovernor,
      HubProposalMetadata,
      HubMessageDispatcher,
      HubProposalExtender
    )
  {
    DeploymentConfiguration memory config = _getDeploymentConfiguration();
    Vm.Wallet memory wallet = _deploymentWallet();
    vm.startBroadcast(wallet.privateKey);
    TimelockController timelock =
      new TimelockController(config.minDelay, new address[](0), new address[](0), wallet.addr);

    HubProposalExtender extender = new HubProposalExtender(
      config.voteExtenderAdmin, config.voteTimeExtension, config.voteExtenderAdmin, config.minimumExtensionTime
    );

   // other code
}
```

그러나 `HubProposalExtender.t.sol`에 작성된 모든 테스트는 `timelock` 주소를 컨트랙트 소유자로 사용합니다.

**Impact:** 연장 기간이나 투표 익스텐더 관리자와 같은 중요한 매개변수의 변경이 적절한 거버넌스 프로세스를 거치지 않고 이루어질 수 있습니다.

**Proof of Concept:** 다음을 `HubGovernor.t.sol`에 추가하십시오

```solidity
contract ExtenderOwnerNotTimelockTest is HubGovernorTest {
    address attacker = address(200);
    HubProposalExtender maliciousExtender;

    function testExtenderOwnerNotTimelock() public {
        // Attacker deploys their own HubProposalExtender
        vm.prank(attacker);
        maliciousExtender = new HubProposalExtender(
            attacker, // voteExtenderAdmin
            1 hours,  // extensionDuration
            attacker, // owner
            30 minutes // MINIMUM_EXTENSION_DURATION
        );

        // Deploy HubGovernor with the malicious extender
        HubGovernor.ConstructorParams memory params = HubGovernor.ConstructorParams({
            name: "Vulnerable Gov",
            token: token,
            timelock: timelock,
            initialVotingDelay: 1 days,
            initialVotingPeriod: 3 days,
            initialProposalThreshold: 500_000e18,
            initialQuorum: 100e18,
            hubVotePoolOwner: address(timelock),
            wormholeCore: address(wormhole),
            governorProposalExtender: address(maliciousExtender),
            initialVoteWeightWindow: VOTE_WEIGHT_WINDOW
        });

        HubGovernor vulnerableGovernor = new HubGovernor(params);

        // Verify that the deployment succeeded and the malicious extender is set
        assertEq(address(vulnerableGovernor.HUB_PROPOSAL_EXTENDER()), address(maliciousExtender));

        // Verify that the attacker owns the extender, not the timelock
        assertEq(Ownable(address(maliciousExtender)).owner(), attacker);
        assertTrue(Ownable(address(maliciousExtender)).owner() != address(timelock));

       // initialize
       maliciousExtender.initialize(payable(vulnerableGovernor));

        // Attacker can change extension duration without governance
        vm.prank(attacker);
        maliciousExtender.setExtensionDuration(2 days);

        // The change is reflected in the extender used by the governor
        assertEq(HubProposalExtender(address(vulnerableGovernor.HUB_PROPOSAL_EXTENDER())).extensionDuration(), 2 days);
    }
}
```

**Recommended Mitigation:** `HubGovernor` 생성자에 `HubProposalExtender`가 `timelock`에 의해 소유되었는지 확인하는 체크를 추가하는 것을 고려하십시오. 또는 배포 스크립트에서 `HubProposalExtender`의 소유권을 `timelock` 컨트랙트로 이전하는 것을 고려하십시오.

`HubGovernor.sol`
```solidity
constructor(ConstructorParams memory _params) {
    // ... existing constructor logic ...

    if (Ownable(_params.governorProposalExtender).owner() != address(_params.timelock)) {
        revert ProposalExtenderNotOwnedByTimelock();
    } //@audit add this validation

    HUB_PROPOSAL_EXTENDER = IVoteExtender(_params.governorProposalExtender);

    // ... rest of the constructor ...
}
```

**Wormhole Foundation**
[PR 179](https://github.com/wormhole-foundation/example-multigov/pull/179/commits/00f8a96a86c8f7009e9ac281ba62545869a30f40#diff-79d0f3e579465b402696db82e7e860247f827d7ebd5519584f50937cad11879f)에서 수정됨.

**Cyfrin:** 확인됨.


### Contract accounts and account abstraction wallets face limitations in Cross-Chain proposal creation and voting power utilization

**Description:** 멀티체인 거버넌스 시스템을 통해 사용자는 시스템에 연결된 모든 체인(허브 체인 포함)에 분산된 투표권을 집계하여 새로운 제안을 생성하기에 충분한 투표권을 모을 수 있습니다.

그러나 스포크 체인에 토큰을 보유한 컨트랙트 계정 및 AA 지갑은 허브 체인에서 제안을 생성하기 위해 투표권을 사용할 수 없습니다. 이러한 계정은 토큰을 보유한 모든 체인에서 동일한 주소를 제어하지 않을 수 있으며, 이는 토큰을 전송하거나 신뢰할 수 있는 EOA(Externally Operated Account)에 투표를 위임하지 않는 한 허브 체인에서 투표권을 사용할 수 없음을 의미합니다.

`HubEvmSpokeAggregateProposer::checkAndProposeIfEligible`은 호출자(msg.sender)가 queriedAccount와 동일한지 확인하고 그렇지 않으면 되돌립니다(revert).

```solidity
 function _checkProposalEligibility(bytes memory _queryResponseRaw, IWormhole.Signature[] memory _signatures)
    internal
    view
    returns (bool)
  {
       // code

        for (uint256 i = 0; i < _queryResponse.responses.length; i++) {
            // code...

           // Check that the address being queried is the caller
           if (_queriedAccount != msg.sender) revert InvalidCaller(msg.sender, _queriedAccount);

          // code...
       }

  }
```

이러한 계정은 투표 가중치가 고려되기 위해 제안 생성 날짜보다 훨씬 전에 투표를 위임해야 합니다 - 제안 생성 직전에 전송하면 투표권이 일시적으로 손실될 수 있습니다. 이는 투표권이 투표 가중치 윈도우 동안의 최소 토큰 보유량으로 계산되기 때문입니다.

제안이 생성되기 훨씬 전에 투표권을 위임해야 하는 이 제약은 관련 위험을 초래합니다. 즉, 위임된 컨트랙트의 투표권은 다른 관련 없는 제안에 투표하는 데 오용될 수 있습니다. 이는 분산 투표 시스템의 전반적인 신뢰 가정을 증가시킵니다.

**Impact:** 두 가지 위험이 있습니다:
1. 최근 위임된 투표권은 투표 윈도우 제약으로 인해 일시적으로 손실됩니다.
2. 제안 생성 훨씬 전에 위임된 투표권은 다른 활성 제안에 투표하는 데 오용될 수 있습니다.

**Recommended Mitigation:** 컨트랙트 계정이 동일한 주소를 요구하지 않고 체인 간 소유권을 증명할 수 있는 메커니즘을 구현하는 것을 고려하십시오.

또는 기존 설계에서 컨트랙트 계정 및 계정 추상화 지갑에 대한 위임의 제약 및 위험을 문서화하는 것을 고려하십시오. 이는 관련 당사자가 제안 생성 시 EOA에 투표권을 위임할지, 어떻게, 언제 위임할지 미리 계획하는 데 도움이 될 것입니다.

**Wormhole Foundation**
[PR 178](https://github.com/wormhole-foundation/example-multigov/pull/178/commits/914aec3776273a5dd17a036af16c1fb981cea26b)에서 수정됨.

**Cyfrin:** 확인됨.



### Cross-Chain votes can be invalidated in some scenarios due to Hub-Spoke timestamp mismatch

**Description:** `HubVotePool` 컨트랙트의 크로스체인 투표 메커니즘의 현재 구현은 스포크 체인에서 투표가 행사된 타임스탬프를 고려하지 않습니다. `HubVotePool::crossChainVote` 함수가 호출될 때, 스포크 체인에서 투표가 실제로 행사된 타임스탬프가 아니라 현재 타임스탬프를 사용하여 투표 유효성을 결정합니다.

이는 스포크 체인의 유효한 투표가 유효한 투표 기간 내에 행사되었더라도 투표 기간이 종료된 후 허브에 의해 처리되면 거부될 수 있는 상황으로 이어집니다.

이 문제는 `castVote`가 호출될 때 현재 블록 타임스탬프를 기반으로 제안의 상태를 확인하는 OpenZeppelin의 Governor 컨트랙트 사용에서 비롯됩니다. 크로스체인 컨텍스트에서 이는 스포크 체인에서의 투표 행사와 허브 체인에서의 투표 처리 사이의 지연 시간을 고려하지 않습니다.


```solidity
// In HubVotePool's crossChainVote function (simplified)
function crossChainVote(bytes memory _queryResponseRaw, IWormhole.Signature[] memory _signatures) external {
    // ... (parsing and verification)

    // This call uses the current timestamp, not the spoke chain's timestamp
    hubGovernor.castVoteWithReasonAndParams(
        _proposalId,
        UNUSED_SUPPORT_PARAM,
        "rolled-up vote from governance spoke token holders",
        _votes
    );
}

// In OpenZeppelin's Governor contract
function _castVote(
    uint256 proposalId,
    address account,
    uint8 support,
    string memory reason,
    bytes memory params
) internal virtual returns (uint256) {
    // This check uses the current timestamp
    _validateStateBitmap(proposalId, _encodeStateBitmap(ProposalState.Active));
    // ...
}
```

**Impact:** 스포크 체인의 유효한 투표가 투표 기간 후에 처리되면 폐기될 수 있습니다. 최종 투표 집계는 네트워크 전체에서 행사된 모든 유효한 투표를 정확하게 나타내지 못할 수 있습니다.

이 위험은 스포크 체인에만 존재한다는 점에 주목할 가치가 있습니다. 허브 체인의 유권자는 자신의 투표가 유효할 것이라는 보장과 함께 마지막 순간까지 투표할 수 있지만, 스포크 체인 유권자에게는 그러한 보장이 존재하지 않습니다. 또한 대부분의 L2에 대한 최종성 시간은 ~15분이며 따라서 이 엣지 케이스는 크랭크 터너(crank-turner)의 효율성에 관계없이 현재 설계에 존재할 것입니다.

또한 `HubProposalExtender`는 체인 중단 및 투표가 허브 체인으로 다시 전파될 수 없는 시나리오와 관련된 엣지 케이스를 설명하기 위해 투표 기간을 일회성으로 연장할 수 있는 기능이 있습니다. 이는 확실히 위험을 상당히 완화하지만 투표 손실 위험을 완전히 제거하지는 않습니다.

**Proof of Concept:** 다음을 `HubGovernor.t.sol`에 추가하십시오

```solidity
contract CountHubPoolVote is HubGovernorTest {
  function test_Revert_HubPoolValidVotesRejected() external {
    uint256 proposalId = 0;
    uint16 queryChainId = 2;
    address _spokeContract= address(999);

   // set current hub vote pool
   governor.exposed_setHubVotePool(address(hubVotePool));

   // and register spoke contract
    {
      vm.startPrank(address(timelock));
        hubVotePool.registerSpoke(queryChainId, addressToBytes32(_spokeContract));
      vm.stopPrank();
    }

    {
      (, delegates) = _setGovernorAndDelegates();
      (ProposalBuilder builder) = _createArbitraryProposal();

      vm.startPrank(delegates[0]);
      proposalId =
        governor.propose(builder.targets(), builder.values(), builder.calldatas(), "hi");
      vm.stopPrank();

      _jumpToActiveProposal(proposalId);
    }


    bytes memory ethCall = QueryTest.buildEthCallWithFinalityRequestBytes(
      bytes("0x1296c33"), // random blockId: a hash of the block number
      "finalized", // finality
      1, // numCallData
      QueryTest.buildEthCallDataBytes(
        _spokeContract, abi.encodeWithSignature("proposalVotes(uint256)", proposalId)
      )
    );

    console2.log("block number %i", block.number);
    console2.log("block timestamp %i", block.timestamp);
    console2.log("voting period %i", governor.votingPeriod());
    console2.log("proposal vote start %i", governor.proposalSnapshot(proposalId));

    bytes memory ethCallResp = QueryTest.buildEthCallWithFinalityResponseBytes(
      uint64(block.number), // block number
      blockhash(block.number), // block hash
      uint64(block.timestamp), // block time US //@note send the timestamp before vote start
      1, // numResults
      QueryTest.buildEthCallResultBytes(
        abi.encode(
          proposalId,
          SpokeCountingFractional.ProposalVote({
            againstVotes: uint128(200),
            forVotes: uint128(10000),
            abstainVotes: uint128(800)
          })
        )
      ) // results
    );


    bytes memory _queryRequestBytes = QueryTest.buildOffChainQueryRequestBytes(
      VERSION, // version
      0, // nonce
      1, // num per chain requests
      abi.encodePacked(
        QueryTest.buildPerChainRequestBytes(
          queryChainId, // chainId
          hubVotePool.QT_ETH_CALL_WITH_FINALITY(),
          ethCall
        )
      )
    );


  bytes memory _resp = QueryTest.buildQueryResponseBytes(
      VERSION, // version
      OFF_CHAIN_SENDER, // sender chain id
      OFF_CHAIN_SIGNATURE, // signature
      _queryRequestBytes, // query request
      1, // num per chain responses
      abi.encodePacked(
        QueryTest.buildPerChainResponseBytes(queryChainId , hubVotePool.QT_ETH_CALL_WITH_FINALITY(), ethCallResp)
      )
    );

    vm.warp(governor.proposalDeadline(proposalId) + 1); // move beyond proposal deadline - no longer active
    IWormhole.Signature[] memory signatures = _getSignatures(_resp);

    vm.expectRevert(abi.encodeWithSelector(IGovernor.GovernorUnexpectedProposalState.selector, proposalId, IGovernor.ProposalState.Defeated, bytes32(1 << uint8(IGovernor.ProposalState.Active))));
    hubVotePool.crossChainVote(_resp, signatures);
  }
}
```

**Recommended Mitigation:** Governor에서 `castVote`를 즉시 호출하는 대신 `HubVotePool` 컨트랙트에서 사용자 정의 투표 집계를 구현하는 것을 고려하십시오. 투표 기간이 종료된 후 `HubVotePool`의 별도 함수가 집계된 크로스체인 투표를 `Governor`에 제출할 수 있습니다. 투표 기간 종료 후의 이 투표는 투표 종료 후 특정 기간 내에 `HubVotePool`에 대해서만 허용될 수 있습니다. 보안 관점에서의 주요 아이디어는 스포크 체인에서 행사된 합법적인 투표가 허브 체인의 최종 집계에서 고려되지 않는 시나리오를 피하는 것입니다.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.



### Potential de-synchronization of spoke registries between `HubVotePool` and `HubEvmSpokeAggregateProposer`

**Description:** 시스템은 스포크 체인 주소에 대해 두 개의 별도 레지스트리를 유지 관리합니다: 하나는 `HubVotePool`에, 다른 하나는 `HubEvmSpokeAggregateProposer`에 있습니다.

이러한 레지스트리는 서로 다른 메커니즘을 사용하며 독립적으로 업데이트될 수 있습니다:

`HubVotePool`은 체크포인트 기반 시스템을 사용합니다:

```solidity
mapping(uint16 emitterChain => Checkpoints.Trace256 emitterAddress) internal emitterRegistry;
```
`HubEvmSpokeAggregateProposer`는 간단한 매핑을 사용합니다:

```solidity
mapping(uint16 wormholeChainId => address spokeVoteAggregator) public registeredSpokes;
```
업데이트 메커니즘도 다릅니다:

- `HubVotePool` 레지스트리는 소유자가 직접 업데이트할 수 있습니다.
- `HubEvmSpokeAggregateProposer` 레지스트리는 HubGovernor가 실행하는 거버넌스 제안을 통해 업데이트됩니다.

이 설계는 두 레지스트리가 동기화되지 않을 위험을 초래하여 잠재적으로 크로스체인 투표 시스템에서 일관성 없는 동작을 유발할 수 있습니다.

**Impact:** 동기화되지 않은 스포크 레지스트리는 한 가지 작업(예: 제안자를 위한 투표 집계)에는 투표를 유효한 것으로 간주하지만 다른 작업(예: 허브 풀 투표)에는 유효하지 않은 것으로 간주할 수 있습니다. 결과적으로 사용자는 HubPool에 대해 더 이상 유효하지 않은 투표를 사용하여 제안을 생성할 수 있습니다.

**Recommended Mitigation:** `HubVotePool` 및 `HubEvmSpokeAggregateProposer` 컨트랙트 모두에 대해 단일 진실 공급원(single source of truth)을 갖는 것을 고려하십시오. 일반적으로 체크포인트 기반 레지스트리는 과거 변경 사항을 추적하지 않는 `HubEvmSpokeAggregateProposer`에서 사용되는 간단한 매핑보다 더 강력합니다.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.


### Lack of minimum threshold for `maxQueryTimestampOffset` in `HubEvmSpokeAggregateProposer`

**Description:** `HubEvmSpokeAggregateProposer` 컨트랙트에서 `setMaxQueryTimestampOffset` 함수는 소유자가 최소 임계값을 강제하지 않고 `maxQueryTimestampOffset`에 대한 새 값을 설정할 수 있도록 허용합니다. 현재 구현은 새 값이 0이 아닌지만 확인합니다:

```solidity
function _setMaxQueryTimestampOffset(uint48 _newMaxQueryTimestampOffset) internal {
    if (_newMaxQueryTimestampOffset == 0) revert InvalidOffset();
    emit MaxQueryTimestampOffsetUpdated(maxQueryTimestampOffset, _newMaxQueryTimestampOffset);
    maxQueryTimestampOffset = _newMaxQueryTimestampOffset;
}
```

**Impact:** 하한선이 부족하면 오프셋이 비실용적으로 낮은 값으로 설정될 수 있으며, 이는 제안 생성을 위한 크로스체인 투표 집계를 방해할 수 있습니다.

**Recommended Mitigation:** `maxQueryTimestampOffset`에 대한 최소 임계값 구현을 고려하십시오.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.



### Inconsistent vote weight windows enable potential voting power manipulation

**Description:** 현재 구현은 허브 체인(`HubGovernor`를 통해)과 스포크 체인(SpokeVoteAggregator를 통해)에서 투표 가중치 윈도우를 독립적으로 설정할 수 있도록 허용합니다. 이는 동일한 거버넌스 제안에 대해 서로 다른 체인에서 투표권이 계산되는 방식에 불일치를 초래할 수 있습니다.

`HubGovernor.sol`에서는 거버넌스를 통해 설정됩니다:

```solidity
function setVoteWeightWindow(uint48 _weightWindow) external {
    _checkGovernance();
    _setVoteWeightWindow(_weightWindow);
}
```

`SpokeVoteAggregator.sol`에서는 컨트랙트 소유자에 의해 설정됩니다:
```solidity
function setVoteWeightWindow(uint48 _voteWeightWindow) public {
    _checkOwner();
    _setVoteWeightWindow(_voteWeightWindow);
}
```
이러한 설정이 체인 간에 동기화된 상태를 유지하도록 보장하는 메커니즘이 없습니다.

**Impact:** 투표 가중치 윈도우의 불일치는 투표권 차이로 이어질 수 있습니다. 투표권은 투표 가중치 윈도우 동안의 최소 토큰 보유량으로 결정되므로, 윈도우가 다르면 서로 다른 스포크 체인에서 불공정한 투표 대표성을 초래할 수 있습니다.

**Recommended Mitigation:** `SpokeAirlock` 컨트랙트를 `SpokeVoteAggregator`의 소유자로 만드는 것을 고려하십시오. 그러면 모든 주요 매개변수 변경은 Hub Governance 승인을 받아야 합니다. 일반적으로 각 스포크 체인에 대해 여러 관리자를 두는 것을 피하고 모든 권한을 Hub Governance에 위임하는 것을 고려하십시오.

 [README](https://github.com/wormhole-foundation/example-multigov/tree/7513aa59000819dd21266c2be16e18a0ae8a595f?tab=readme-ov-file#spokevoteaggregator)는 `SpokeAirlock`이 실제로 `SpokeVoteAggregator`의 소유자임을 시사하지만, [`DeploySpokeContractsBaseImpl.sol`](https://github.com/wormhole-foundation/example-multigov/blob/7513aa59000819dd21266c2be16e18a0ae8a595f/evm/script/DeploySpokeContractsBaseImpl.sol#L48) 배포 스크립트에는 이에 대한 구현이 누락되어 있습니다.

 **Wormhole Foundation**
[PR 184](https://github.com/wormhole-foundation/example-multigov/pull/184/commits/5adab8a44b41ceac3ead4abc7d88983800e0f2eb)에서 수정됨.

**Cyfrin:** 확인됨.



\clearpage
## Informational


### Missing event emissions when changing critical parameters

**Description:** 중요한 매개변수가 해당 이벤트 방출 없이 변경됩니다. 이는 전반적인 투명성 부족으로 이어집니다.

`HubVotePool::setGovernor`는 거버너가 변경될 때 이벤트를 방출하지 않습니다.

```solidity
  function setGovernor(address _newGovernor) external {
    _checkOwner();
    hubGovernor = IGovernor(_newGovernor);
    //@audit @info missing event check for this critical change
  }
```

`HubProposalExtender::extendProposal`은 제안이 연장될 때 이벤트를 방출하지 않습니다.

```solidity
function extendProposal(uint256 _proposalId) external {
    uint256 exists = governor.proposalSnapshot(_proposalId);
    if (msg.sender != voteExtenderAdmin) revert AddressCannotExtendProposal();
    if (exists == 0) revert ProposalDoesNotExist();
    if (extendedDeadlines[_proposalId] != 0) revert ProposalAlreadyExtended();

    IGovernor.ProposalState state = governor.state(_proposalId);
    if (state != IGovernor.ProposalState.Active && state != IGovernor.ProposalState.Pending) {
      revert ProposalCannotBeExtended();
    }

    extendedDeadlines[_proposalId] = uint48(governor.proposalDeadline(_proposalId)) + extensionDuration;
   //@audit @info missing event emission
  }
```

`HubGovernor::_setWhitelistedProposer`는 화이트리스트된 제안자가 업데이트될 때 이벤트를 방출하지 않습니다.

```solidity
  function _setWhitelistedProposer(address _proposer) internal {
    emit WhitelistedProposerUpdated(whitelistedProposer, _proposer);
    whitelistedProposer = _proposer; //@audit lack of zero addy check
  }
```
**Recommended Mitigation:** 중요한 컨트랙트 매개변수가 변경될 때 이벤트를 추가하는 것을 고려하십시오.

**Wormhole Foundation**
[PR 173](https://github.com/wormhole-foundation/example-multigov/pull/173/commits)에서 수정됨.

**Cyfrin:** 확인됨.


### Potential long-term failure in `HubGovernor::getVotes` due to uint32 Casting of Timestamps

**Description:** `GovernorMinimumWeightedVoteWindow::_getVotes` 함수에는 `_windowStart` 값을 uint32로 캐스팅하는 부분이 있습니다:

```solidity
uint256 _startPos = _upperLookupRecent(_account, uint32(_windowStart), _numCheckpoints);
```
이 캐스팅은 처리할 수 있는 최대 타임스탬프를 4,294,967,295(2106년)로 제한합니다. 이 날짜 이후에는 더 큰 타임스탬프를 uint32로 캐스팅하려고 하면 산술 오버플로 오류로 인해 트랜잭션이 되돌려질(revert) 것입니다.

**Impact:** 이 문제의 영향은 향후 80년 동안 나타나지 않을 것이므로 현재로서는 미미합니다. 이 기간이 지나면 컨트랙트는 투표 처리에 실패하여 효과적으로 핵심 기능이 중단될 것입니다.

**Recommended Mitigation:** 캐스팅 작업에서 uint32를 uint64로 대체하는 것을 고려하십시오.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.


### Proposal deadline extension is allowed even in `Pending` state

**Description:** `HubProposalExtender::extendProposal` 함수는 제안이 `Pending` 상태일 때도 제안 마감일 연장을 허용합니다. 이를 통해 `VoteExtenderAdmin`은 투표 기간이 시작되기도 전에 제안의 마감일을 연장할 수 있습니다.

그러나 `Pending` 상태 동안에는 스포크 체인이 투표를 집계하여 허브 풀로 보낼 수 없으므로, 웜홀 쿼리가 허브 체인에 메시지를 제출할 수 없는 시나리오에서 투표 집계에 더 많은 시간을 허용하려는 의도된 목적에 대해 확장이 효과적이지 않습니다.

```solidity
function extendProposal(uint256 _proposalId) external {
    // ... (other checks)
    IGovernor.ProposalState state = governor.state(_proposalId);
    if (state != IGovernor.ProposalState.Active && state != IGovernor.ProposalState.Pending) {
        revert ProposalCannotBeExtended();
    }
    // ... (extension logic)
}
```
**Recommended Mitigation:** `extendProposal` 함수를 수정하여 활성(Active) 제안에 대해서만 연장을 허용하는 것을 고려하십시오.

**Wormhole Foundation**
[PR 170](https://github.com/wormhole-foundation/example-multigov/pull/170/commits/48c4965edf8b8c125199fc399758929c88e64f48)에서 수정됨.

**Cyfrin:** 확인됨.



### Missing deployment script for `HubEvmSpokeAggregateProposer`

**Description:** `DEployHubContractsBaseImpls.s.sol`의 배포 스크립트에는 `HubEvmSpokeAggregateProposer` 배포 코드가 포함되어 있지 않습니다.


**Recommended Mitigation:** `maxQueryTimestampOffset` 매개변수에 대해 문서화된 초기값을 사용하여 `HubEvmSpokeAggregateProposer`를 배포하도록 스크립트를 업데이트하는 것을 고려하십시오.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.



### Missing implementation to cast fractional vote with signature

**Description:** SpokeVoteAggregator에는 부분 투표(fractional voting) 매개변수로 투표 메시지에 서명하는 함수가 없습니다. 사용자는 서명을 통해 명목 투표를 행사할 수 있지만, 서명을 통해 부분 투표를 행사할 수 있는 규정은 없습니다.

**Recommended Mitigation:** 사용자가 부분 투표 매개변수를 전달하여 투표 메시지에 서명할 수 있는 함수를 추가하는 것을 고려하십시오.

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.


### Inconsistent documentation related to the deployment of `SpokeMessageExecutor`

**Description:** 프로젝트 README에서 `SpokeMessageExecutor`의 거버넌스 업그레이드 경로는 다음과 같이 제공됩니다.

```text
`SpokeMessageExecutor`: Deploy a new contract and then update the hub dispatcher to call the new spoke message executor.
```

SpokeMessageExecutor는 업그레이드 가능하므로 새 컨트랙트를 배포할 필요가 없습니다. README에서 제안한 대로 수행하면 실제로 재생 공격 위험이 발생합니다.

README는 오래되었으며 현재 시스템 설계와 일치하지 않습니다.


**Recommended Mitigation:** `SpokeExecutor`가 업그레이드 가능한 컨트랙트인 현재 설계를 반영하도록 프로젝트 README를 업그레이드하는 것을 고려하십시오.

**Wormhole Foundation**
[PR 176](https://github.com/wormhole-foundation/example-multigov/pull/176/commits/41b4bdaa14e045e817f525b5d4d9b9c792e6bf6c)에서 수정됨.

**Cyfrin:** 확인됨.

\clearpage
## Gas Optimization


### Hash comparison can be optimized

**Description:** `HubEvmSpokeDecoder::decode`는 블록이 확정되었는지 확인하기 위해 keccak256 해시 비교를 수행합니다. 비교자 중 하나는 항상 상수입니다. 즉, keccak256(bytes("finalized"))입니다. 해시 생성은 비용이 많이 들며 이 값을 미리 계산하여 단순화할 수 있습니다.

```solidity
    if (keccak256(_ethCalls.requestFinality) != keccak256(bytes("finalized"))) {
      revert InvalidQueryBlock(_ethCalls.requestBlockId);
    }
```
이것은 가스 비용이 많이 들며 이 경우 해싱이 불필요합니다.

**Recommended Mitigation:** 매번 계산하는 대신 `keccak256(bytes("finalized"))`를 상수로 저장하는 것을 고려하십시오.

**Wormhole Foundation**
[PR 169](https://github.com/wormhole-foundation/example-multigov/pull/169/commits/1230774ca68795e7f5ed30c9036a95ebe33b6c2c)에서 수정됨.

**Cyfrin:** 확인됨.


### `_checkProposalEligibility` gas optimization

**Description:** `HubEvmSpokeAggregateProposer::_checkProposalEligibility`의 현재 구현은 제안 임계값이 충족되었는지 확인하기 전에 모든 스포크 체인 응답을 처리합니다. 또한 허브 투표 가중치는 스포크 체인의 모든 투표가 집계된 후 총 투표 가중치에 추가됩니다.

이 접근 방식은 허브 체인에서 제안자의 투표권만으로 또는 스포크 체인의 일부와 결합하여 제안 임계값을 충족하기에 충분한 경우 불필요한 가스 소비로 이어질 수 있습니다.

**Recommended Mitigation:** 허브 투표 가중치를 마지막에 추가하는 대신 먼저 추가하는 것을 고려하십시오. 허브 투표 가중치를 추가한 후 그리고 각 스포크 체인 응답을 처리한 후 제안 임계값을 확인하도록 함수를 수정하십시오. 이를 통해 임계값이 충족되면 조기 종료가 가능하여 불필요한 계산에 대한 가스를 잠재적으로 절약할 수 있습니다.

```solidity
function _checkProposalEligibility(bytes memory _queryResponseRaw, IWormhole.Signature[] memory _signatures)
    internal
    view
    returns (bool)
{
    // ... (initial setup)

    uint256 _hubVoteWeight = HUB_GOVERNOR.getVotes(msg.sender, _sharedQueryBlockTime / 1_000_000);
    _totalVoteWeight = _hubVoteWeight;

    uint256 _proposalThreshold = HUB_GOVERNOR.proposalThreshold();
    if (_totalVoteWeight >= _proposalThreshold) {
        return true;
    }

    for (uint256 i = 0; i < _queryResponse.responses.length; i++) {
        // ... (process each spoke chain response)
        uint256 _voteWeight = abi.decode(_ethCalls.result[0].result, (uint256));
        _totalVoteWeight += _voteWeight;

        if (_totalVoteWeight >= _proposalThreshold) {
            return true;
        }
    }

    return false;
}
```

**Wormhole Foundation**
인지됨.

**Cyfrin:** 인지됨.

\clearpage
