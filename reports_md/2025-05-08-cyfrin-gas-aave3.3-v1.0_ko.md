**수석 감사자**

[Dacian](https://x.com/DevDacian)

**보조 감사자**



---

# 결과 (Findings)
## 가스 최적화 (Gas Optimization)


### `GenericLogic::calculateUserAccountData`에서 `currentReserve.configuration` 캐싱

**설명:** `GenericLogic::calculateUserAccountData`는 상태를 변경하지 않는 `view` 함수이므로 `currentReserve.configuration`을 캐시하십시오.

**파급력:** `snapshots/Pool.Setters.json`:
```diff
-  "setUserEMode: enter eMode, 1 borrow, 1 supply": "140836",
-  "setUserEMode: leave eMode, 1 borrow, 1 supply": "112635",
+  "setUserEMode: enter eMode, 1 borrow, 1 supply": "140695",
+  "setUserEMode: leave eMode, 1 borrow, 1 supply": "112494",
```

`snapshots/Pool.Operations.json`:
```diff
-  "borrow: first borrow->borrowingEnabled": "256480",
-  "borrow: recurrent borrow": "249018",
+  "borrow: first borrow->borrowingEnabled": "256479",
+  "borrow: recurrent borrow": "248877",
   "flashLoan: flash loan for one asset": "197361",
-  "flashLoan: flash loan for one asset and borrow": "279057",
+  "flashLoan: flash loan for one asset and borrow": "279056",
   "flashLoan: flash loan for two assets": "325455",
-  "flashLoan: flash loan for two assets and borrow": "484439",
+  "flashLoan: flash loan for two assets and borrow": "484295",
   "flashLoanSimple: simple flash loan": "170603",
-  "liquidationCall: deficit on liquidated asset": "392365",
-  "liquidationCall: deficit on liquidated asset + other asset": "491921",
-  "liquidationCall: full liquidation": "392365",
-  "liquidationCall: full liquidation and receive ATokens": "368722",
-  "liquidationCall: partial liquidation": "383166",
-  "liquidationCall: partial liquidation and receive ATokens": "359520",
+  "liquidationCall: deficit on liquidated asset": "392223",
+  "liquidationCall: deficit on liquidated asset + other asset": "491638",
+  "liquidationCall: full liquidation": "392223",
+  "liquidationCall: full liquidation and receive ATokens": "368581",
+  "liquidationCall: partial liquidation": "383024",
+  "liquidationCall: partial liquidation and receive ATokens": "359378",
   "repay: full repay": "176521",
   "repay: full repay with ATokens": "173922",
   "repay: partial repay": "189949",
   "supply: first supply->collateralEnabled": "176366",
   "withdraw: full withdraw": "165226",
   "withdraw: partial withdraw": "181916",
-  "withdraw: partial withdraw with active borrows": "239471"
+  "withdraw: partial withdraw with active borrows": "239329"
```

**권장되는 완화 방법:** 커밋 [3cd6639](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/3cd663998a91906460b7e9175862ba3fe794efb1)를 참조하십시오.


### `LiquidationLogic::executeLiquidationCall`에서 `usersConfig[params.user]` 캐싱

**설명:** `LiquidationLogic::executeLiquidationCall`에서 `usersConfig[params.user]`를 캐시하고, 이 사본을 뷰(view) 함수인 `GenericLogic::calculateUserAccountData` 및 `ValidationLogic::validateLiquidationCall`에 안전하게 전달할 수 있습니다.

더 강력한 최적화는 청산 프로세스 전체에서 캐시된 사본을 사용하고 업데이트한 다음 마지막에 스토리지에 기록하는 것입니다. 이는 더 많은 변경이 필요하므로 G-06에서 별도로 구현되었습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "392223",
-  "liquidationCall: deficit on liquidated asset + other asset": "491638",
-  "liquidationCall: full liquidation": "392223",
-  "liquidationCall: full liquidation and receive ATokens": "368581",
-  "liquidationCall: partial liquidation": "383024",
-  "liquidationCall: partial liquidation and receive ATokens": "359378",
+  "liquidationCall: deficit on liquidated asset": "392182",
+  "liquidationCall: deficit on liquidated asset + other asset": "491597",
+  "liquidationCall: full liquidation": "392182",
+  "liquidationCall: full liquidation and receive ATokens": "368539",
+  "liquidationCall: partial liquidation": "382983",
+  "liquidationCall: partial liquidation and receive ATokens": "359337",
```

**권장되는 완화 방법:** 커밋 [4ce346c](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/4ce346c4ba64667d049f2344cf2df9115d104c62)를 참조하십시오.


### `LiquidationLogic::executeLiquidationCall`에서 `collateralReserve.configuration` 캐싱

**설명:** `LiquidationLogic::executeLiquidationCall`에서 `collateralReserve.configuration`을 안전하게 캐시하여 자식 함수에 전달하면 많은 동일한 스토리지 읽기를 절약할 수 있습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "392182",
-  "liquidationCall: deficit on liquidated asset + other asset": "491597",
-  "liquidationCall: full liquidation": "392182",
-  "liquidationCall: full liquidation and receive ATokens": "368539",
-  "liquidationCall: partial liquidation": "382983",
-  "liquidationCall: partial liquidation and receive ATokens": "359337",
+  "liquidationCall: deficit on liquidated asset": "391606",
+  "liquidationCall: deficit on liquidated asset + other asset": "491021",
+  "liquidationCall: full liquidation": "391606",
+  "liquidationCall: full liquidation and receive ATokens": "367841",
+  "liquidationCall: partial liquidation": "382408",
+  "liquidationCall: partial liquidation and receive ATokens": "358639",
```

**권장되는 완화 방법:** 커밋 [414dc2d](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/414dc2d6cb9314bcda79cd72425407be75be22a6)를 참조하십시오.


### `LiquidationLogic::executeLiquidationCall`에서 `collateralReserve.id` 캐싱

**설명:** `LiquidationLogic::executeLiquidationCall`에서 `collateralReserve.id`를 안전하게 캐시하고 사본을 자식 함수에 전달할 수 있습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "391606",
-  "liquidationCall: deficit on liquidated asset + other asset": "491021",
-  "liquidationCall: full liquidation": "391606",
-  "liquidationCall: full liquidation and receive ATokens": "367841",
-  "liquidationCall: partial liquidation": "382408",
-  "liquidationCall: partial liquidation and receive ATokens": "358639",
+  "liquidationCall: deficit on liquidated asset": "391560",
+  "liquidationCall: deficit on liquidated asset + other asset": "490975",
+  "liquidationCall: full liquidation": "391560",
+  "liquidationCall: full liquidation and receive ATokens": "367673",
+  "liquidationCall: partial liquidation": "382476",
+  "liquidationCall: partial liquidation and receive ATokens": "358585",
```

**권장되는 완화 방법:** 커밋 [a82d552](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/a82d552a5bb901f9f4b0c81421f36953df686978)를 참조하십시오.


### `LiquidationLogic::_liquidateATokens`에서 캐시된 `vars.collateralAToken` 사용

**설명:** `LiquidationLogic::_liquidateATokens`에서 캐시된 `vars.collateralAToken`을 사용하십시오. 스토리지에서 다시 읽을 필요가 없습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: full liquidation and receive ATokens": "367673",
+  "liquidationCall: full liquidation and receive ATokens": "367553",
   "liquidationCall: partial liquidation": "382476",
-  "liquidationCall: partial liquidation and receive ATokens": "358585",
+  "liquidationCall: partial liquidation and receive ATokens": "358465",
```

**권장되는 완화 방법:** 커밋 [f6af2e1](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/f6af2e13b8e3b9960abb63ebb4eaeb82e271b718)를 참조하십시오.


### `LiquidationLogic::executeLiquidationCall`에서 `usersConfig[params.user]`에 대해 스토리지를 한 번만 읽고 쓰기

**설명:** `usersConfig[params.user]`는 `LiquidationLogic::executeLiquidationCall`의 시작 부분에서 한 번 읽을 수 있으며, 캐시된 사본을 `memory`를 사용하여 전달하고 필요에 따라 수정한 다음, 함수의 마지막에 한 번 스토리지에 쓸 수 있습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "391560",
-  "liquidationCall: deficit on liquidated asset + other asset": "490975",
-  "liquidationCall: full liquidation": "391560",
-  "liquidationCall: full liquidation and receive ATokens": "367553",
-  "liquidationCall: partial liquidation": "382476",
-  "liquidationCall: partial liquidation and receive ATokens": "358465",
+  "liquidationCall: deficit on liquidated asset": "391305",
+  "liquidationCall: deficit on liquidated asset + other asset": "489972",
+  "liquidationCall: full liquidation": "391305",
+  "liquidationCall: full liquidation and receive ATokens": "367366",
+  "liquidationCall: partial liquidation": "382734",
+  "liquidationCall: partial liquidation and receive ATokens": "358795",
```

이 변경은 부분적으로 성능이 약간 저하되는 부분 청산(partial liquidations)을 제외한 모든 부분에 이점을 주는 것으로 보입니다.

**권장되는 완화 방법:** 커밋 [f419f3c](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/f419f3c638401cef897a265ff0407da762e84021)를 참조하십시오.


### `LiquidationLogic::_calculateAvailableCollateralToLiquidate`에서 명명된 반환 변수를 사용하여 메모리 사용량 및 가스 감소

**설명:** `LiquidationLogic::_calculateAvailableCollateralToLiquidate`에서 명명된 반환 변수(named return variables)를 사용하여 메모리 사용량과 가스를 줄이십시오.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "391305",
-  "liquidationCall: deficit on liquidated asset + other asset": "489972",
-  "liquidationCall: full liquidation": "391305",
-  "liquidationCall: full liquidation and receive ATokens": "367366",
-  "liquidationCall: partial liquidation": "382734",
-  "liquidationCall: partial liquidation and receive ATokens": "358795",
+  "liquidationCall: deficit on liquidated asset": "391070",
+  "liquidationCall: deficit on liquidated asset + other asset": "489736",
+  "liquidationCall: full liquidation": "391070",
+  "liquidationCall: full liquidation and receive ATokens": "367131",
+  "liquidationCall: partial liquidation": "382507",
+  "liquidationCall: partial liquidation and receive ATokens": "358569",
```

**권장되는 완화 방법:** 커밋 [af61e44](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/af61e44a43c488ba1a3a569482a7fed18b8518c9)를 참조하십시오.


### `AvailableCollateralToLiquidateLocalVars` 메모리 구조체를 제거하고 `LiquidationLogic::_calculateAvailableCollateralToLiquidate`에서 로컬 변수만 사용

**설명:** 인메모리(in-memory) "컨텍스트" 구조체를 사용하여 변수를 저장하는 것은 "stack too deep errors"를 피하기 위한 좋은 트릭이지만, 함수 내 로컬 변수를 사용하는 것보다 훨씬 더 많은 가스를 사용합니다.

인메모리 "컨텍스트" 구조체가 필요하지 않을 때는 사용하지 않는 것이 더 저렴합니다. 따라서 `AvailableCollateralToLiquidateLocalVars` 메모리 구조체를 제거하고 `LiquidationLogic::_calculateAvailableCollateralToLiquidate`에서 로컬 변수만 사용하십시오.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "391070",
-  "liquidationCall: deficit on liquidated asset + other asset": "489736",
-  "liquidationCall: full liquidation": "391070",
-  "liquidationCall: full liquidation and receive ATokens": "367131",
-  "liquidationCall: partial liquidation": "382507",
-  "liquidationCall: partial liquidation and receive ATokens": "358569",
+  "liquidationCall: deficit on liquidated asset": "390891",
+  "liquidationCall: deficit on liquidated asset + other asset": "489556",
+  "liquidationCall: full liquidation": "390891",
+  "liquidationCall: full liquidation and receive ATokens": "366954",
+  "liquidationCall: partial liquidation": "382335",
+  "liquidationCall: partial liquidation and receive ATokens": "358397",
```

**권장되는 완화 방법:** 커밋 [84d3925](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/84d392575dfd0da6224a43500315962430455af0)를 참조하십시오.


### `LiquidationCallLocalVars` 구조체의 3개 변수를 `LiquidationLogic::executeLiquidationCall`의 본문으로 이동

**설명:** G-07과 유사하게, "stack too deep" 오류를 유발하지 않으면서 인메모리 "컨텍스트" 구조체 `LiquidationCallLocalVars`에서 `LiquidationLogic::executeLiquidationCall`의 함수 본문으로 적어도 3개의 변수를 이동할 수 있습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "liquidationCall: deficit on liquidated asset": "390891",
-  "liquidationCall: deficit on liquidated asset + other asset": "489556",
-  "liquidationCall: full liquidation": "390891",
-  "liquidationCall: full liquidation and receive ATokens": "366954",
-  "liquidationCall: partial liquidation": "382335",
-  "liquidationCall: partial liquidation and receive ATokens": "358397",
+  "liquidationCall: deficit on liquidated asset": "390795",
+  "liquidationCall: deficit on liquidated asset + other asset": "489459",
+  "liquidationCall: full liquidation": "390795",
+  "liquidationCall: full liquidation and receive ATokens": "366840",
+  "liquidationCall: partial liquidation": "382220",
+  "liquidationCall: partial liquidation and receive ATokens": "358265",
```

**권장되는 완화 방법:** 커밋 [8bf12e2](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/8bf12e272001306d07ba2ebf07ba2d3668792784)를 참조하십시오.


### `GenericLogic::calculateUserAccountData`에서 명명된 반환 변수 사용

**설명:** G-7과 유사하게, `GenericLogic::calculateUserAccountData`에서 명명된 반환 변수를 사용하는 것이 더 저렴합니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "borrow: first borrow->borrowingEnabled": "256479",
-  "borrow: recurrent borrow": "248877",
+  "borrow: first borrow->borrowingEnabled": "256117",
+  "borrow: recurrent borrow": "248451",
   "flashLoan: flash loan for one asset": "197361",
-  "flashLoan: flash loan for one asset and borrow": "279056",
+  "flashLoan: flash loan for one asset and borrow": "278694",
   "flashLoan: flash loan for two assets": "325455",
-  "flashLoan: flash loan for two assets and borrow": "484295",
+  "flashLoan: flash loan for two assets and borrow": "483384",
   "flashLoanSimple: simple flash loan": "170603",
-  "liquidationCall: deficit on liquidated asset": "390795",
-  "liquidationCall: deficit on liquidated asset + other asset": "489459",
-  "liquidationCall: full liquidation": "390795",
-  "liquidationCall: full liquidation and receive ATokens": "366840",
-  "liquidationCall: partial liquidation": "382220",
-  "liquidationCall: partial liquidation and receive ATokens": "358265",
+  "liquidationCall: deficit on liquidated asset": "390368",
+  "liquidationCall: deficit on liquidated asset + other asset": "489010",
+  "liquidationCall: full liquidation": "390368",
+  "liquidationCall: full liquidation and receive ATokens": "366414",
+  "liquidationCall: partial liquidation": "381793",
+  "liquidationCall: partial liquidation and receive ATokens": "357839",
-  "withdraw: partial withdraw with active borrows": "239329"
+  "withdraw: partial withdraw with active borrows": "238904"
```

**권장되는 완화 방법:** 커밋 [f6f7cb6](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/f6f7cb6c2ff160d722ebc25a6f121a59528f26a3)를 참조하십시오.


### `GenericLogic::calculateUserAccountData`에서 사용되는 컨텍스트 구조체 `CalculateUserAccountDataVars`에서 5개의 변수 제거

**설명:** G-7 및 G-8과 유사하게, "stack too deep" 오류를 유발하지 않으면서 인메모리 구조체 `CalculateUserAccountDataVars`에서 이러한 변수를 제거하는 것이 더 저렴합니다.

**파급력:** `snapshots/Pool.Operations`:
```diff
-  "borrow: first borrow->borrowingEnabled": "256117",
-  "borrow: recurrent borrow": "248451",
+  "borrow: first borrow->borrowingEnabled": "255807",
+  "borrow: recurrent borrow": "248112",
   "flashLoan: flash loan for one asset": "197361",
-  "flashLoan: flash loan for one asset and borrow": "278694",
+  "flashLoan: flash loan for one asset and borrow": "278384",
   "flashLoan: flash loan for two assets": "325455",
-  "flashLoan: flash loan for two assets and borrow": "483384",
+  "flashLoan: flash loan for two assets and borrow": "482657",
   "flashLoanSimple: simple flash loan": "170603",
-  "liquidationCall: deficit on liquidated asset": "390368",
-  "liquidationCall: deficit on liquidated asset + other asset": "489010",
-  "liquidationCall: full liquidation": "390368",
-  "liquidationCall: full liquidation and receive ATokens": "366414",
-  "liquidationCall: partial liquidation": "381793",
-  "liquidationCall: partial liquidation and receive ATokens": "357839",
+  "liquidationCall: deficit on liquidated asset": "390029",
+  "liquidationCall: deficit on liquidated asset + other asset": "488641",
+  "liquidationCall: full liquidation": "390029",
+  "liquidationCall: full liquidation and receive ATokens": "366075",
+  "liquidationCall: partial liquidation": "381454",
+  "liquidationCall: partial liquidation and receive ATokens": "357501",
-  "withdraw: partial withdraw with active borrows": "238904"
+  "withdraw: partial withdraw with active borrows": "238566"
```

**권장되는 완화 방법:** 커밋 [01e1024](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/01e10245711584b0fffad75029a2ce1ea1201498), [48d773e](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/48d773e84d2f4a57422926b4d6246fb24f97a177)를 참조하십시오.


### `ReserveLogic::cache` 및 `cumulateToLiquidityIndex`에서 명명된 반환 사용

**설명:** `ReserveLogic::cache` 및 `cumulateToLiquidityIndex`에서 명명된 반환을 사용하면 여러 함수에 걸쳐 좋은 가스 절감 효과를 얻을 수 있습니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "borrow: first borrow->borrowingEnabled": "255775",
-  "borrow: recurrent borrow": "248098",
-  "flashLoan: flash loan for one asset": "197361",
-  "flashLoan: flash loan for one asset and borrow": "278352",
-  "flashLoan: flash loan for two assets": "325455",
-  "flashLoan: flash loan for two assets and borrow": "482562",
-  "flashLoanSimple: simple flash loan": "170603",
-  "liquidationCall: deficit on liquidated asset": "390014",
-  "liquidationCall: deficit on liquidated asset + other asset": "488644",
-  "liquidationCall: full liquidation": "390014",
-  "liquidationCall: full liquidation and receive ATokens": "366060",
-  "liquidationCall: partial liquidation": "381439",
-  "liquidationCall: partial liquidation and receive ATokens": "357486",
-  "repay: full repay": "176521",
-  "repay: full repay with ATokens": "173922",
-  "repay: partial repay": "189949",
-  "repay: partial repay with ATokens": "185129",
-  "supply: collateralDisabled": "146755",
-  "supply: collateralEnabled": "146755",
-  "supply: first supply->collateralEnabled": "176229",
-  "withdraw: full withdraw": "165226",
-  "withdraw: partial withdraw": "181916",
-  "withdraw: partial withdraw with active borrows": "238552"
+  "borrow: first borrow->borrowingEnabled": "255438",
+  "borrow: recurrent borrow": "247760",
+  "flashLoan: flash loan for one asset": "197044",
+  "flashLoan: flash loan for one asset and borrow": "278015",
+  "flashLoan: flash loan for two assets": "324816",
+  "flashLoan: flash loan for two assets and borrow": "481887",
+  "flashLoanSimple: simple flash loan": "170288",
+  "liquidationCall: deficit on liquidated asset": "389335",
+  "liquidationCall: deficit on liquidated asset + other asset": "487617",
+  "liquidationCall: full liquidation": "389335",
+  "liquidationCall: full liquidation and receive ATokens": "365723",
+  "liquidationCall: partial liquidation": "380761",
+  "liquidationCall: partial liquidation and receive ATokens": "357148",
+  "repay: full repay": "176189",
+  "repay: full repay with ATokens": "173590",
+  "repay: partial repay": "189617",
+  "repay: partial repay with ATokens": "184797",
+  "supply: collateralDisabled": "146423",
+  "supply: collateralEnabled": "146423",
+  "supply: first supply->collateralEnabled": "175897",
+  "withdraw: full withdraw": "164894",
+  "withdraw: partial withdraw": "181583",
+  "withdraw: partial withdraw with active borrows": "238216"
```

**권장되는 완화 방법:** 커밋 [f51ced5](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/f51ced5571da85b33ca3ff4a109f9dfa2e84e87a)를 참조하십시오.


### 로컬 변수를 제거하거나 `memory` 반환을 위해 명명된 반환 사용

**설명:** 일반적으로 로컬 변수를 제거할 수 있거나 반환 값이 `memory`일 때 명명된 반환을 사용하는 것이 더 효율적입니다.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "borrow: first borrow->borrowingEnabled": "255438",
-  "borrow: recurrent borrow": "247760",
+  "borrow: first borrow->borrowingEnabled": "255409",
+  "borrow: recurrent borrow": "247710",
   "flashLoan: flash loan for one asset": "197044",
-  "flashLoan: flash loan for one asset and borrow": "278015",
+  "flashLoan: flash loan for one asset and borrow": "277986",
   "flashLoan: flash loan for two assets": "324816",
-  "flashLoan: flash loan for two assets and borrow": "481887",
+  "flashLoan: flash loan for two assets and borrow": "481858",
-  "repay: full repay": "176189",
-  "repay: full repay with ATokens": "173590",
-  "repay: partial repay": "189617",
-  "repay: partial repay with ATokens": "184797",
+  "repay: full repay": "176156",
+  "repay: full repay with ATokens": "173565",
+  "repay: partial repay": "189587",
+  "repay: partial repay with ATokens": "184775",
   "supply: collateralDisabled": "146423",
   "supply: collateralEnabled": "146423",
-  "supply: first supply->collateralEnabled": "175897",
+  "supply: first supply->collateralEnabled": "175868",
```

`snapshots/RewardsController.json`:
```diff
-  "claimAllRewards: one reward type": "50167",
-  "claimAllRewardsToSelf: one reward type": "49963",
-  "claimRewards partial: one reward type": "48299",
-  "claimRewards: one reward type": "48037",
-  "configureAssets: one reward type": "264184",
+  "claimAllRewards: one reward type": "50131",
+  "claimAllRewardsToSelf: one reward type": "49927",
+  "claimRewards partial: one reward type": "48250",
+  "claimRewards: one reward type": "47988",
+  "configureAssets: one reward type": "264175",
```

`snapshots/StataTokenV2.json`:
```diff
-  "claimRewards": "359669",
+  "claimRewards": "359522",
```

참고: 이후 커밋에서 더 많은 이점이 있었습니다.

**권장되는 완화 방법:** 커밋 [ff2f190](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/ff2f190e4454175b6594c3c4fc6aebd35461013e), [460e574](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/460e5744635fe856320eebcfeeb8091211c59039), [47933e7](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/47933e7e74c906fe475cfe8eb9fc2537bca30922), [2ad50db](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/2ad50db0d160f2a45dca9dcbebb4b0931e356b71), [9e76a0b](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/9e76a0b71aa1f523d4afdcc6be03bf184b7a2a7a), [757b9a7](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/757b9a7a3825bd2bc596b11fd2bf51884f6ff5d9)를 참조하십시오.


### `SupplyLogic::executeWithdraw`에서 `userConfig` 스토리지를 한 번만 읽고 쓰기

**설명:** `SupplyLogic::executeWithdraw`에서 `userConfig` 스토리지를 한 번만 읽고 쓰십시오.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "supply: first supply->collateralEnabled": "175868",
-  "withdraw: full withdraw": "164894",
-  "withdraw: partial withdraw": "181583",
-  "withdraw: partial withdraw with active borrows": "238208"
+  "supply: first supply->collateralEnabled": "175872",
+  "withdraw: full withdraw": "164728",
+  "withdraw: partial withdraw": "181499",
+  "withdraw: partial withdraw with active borrows": "237972"
```

**권장되는 완화 방법:** 커밋 [ba371a8](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/ba371a8dc77295dfe27a3c4b6b62d63213ec4a62)를 참조하십시오.


### `SupplyLogic::executeFinalizeTransfer`에서 `usersConfig[params.from]` 캐싱

**설명:** `SupplyLogic::executeFinalizeTransfer`에서 `usersConfig[params.from]`을 캐싱하고 `if` 문 안에 `uint256 reserveId = reserve.id;`를 넣으면 여러 함수에 걸쳐 가스를 줄일 수 있습니다.

**파급력:** `snapshots/AToken.transfer.json`:
```diff
-  "full amount; sender: ->disableCollateral;": "103316",
-  "full amount; sender: ->disableCollateral; receiver: ->enableCollateral": "145062",
-  "full amount; sender: ->disableCollateral; receiver: dirty, ->enableCollateral": "132989",
+  "full amount; sender: ->disableCollateral;": "103282",
+  "full amount; sender: ->disableCollateral; receiver: ->enableCollateral": "145028",
+  "full amount; sender: ->disableCollateral; receiver: dirty, ->enableCollateral": "132955",
-  "partial amount; sender: collateralEnabled;": "103347",
-  "partial amount; sender: collateralEnabled; receiver: ->enableCollateral": "145093"
+  "partial amount; sender: collateralEnabled;": "103208",
+  "partial amount; sender: collateralEnabled; receiver: ->enableCollateral": "144954"
```

`snapshots/StataTokenV2.json`:
```diff
-  "depositATokens": "219313",
+  "depositATokens": "219279",
-  "redeemAToken": "152637"
+  "redeemAToken": "152498"
```

`snapshots/WrappedTokenGatewayV3.json`:
```diff
-  "withdrawETH": "258800"
+  "withdrawETH": "258766"
```

**권장되는 완화 방법:** 커밋 [aba92d2](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/aba92d2afb5c15275bd0b00cf844e26905020a5f)를 참조하십시오.


### `BorrowLogic::executeBorrow`에서 `userConfig` 캐싱

**설명:** `BorrowLogic::executeBorrow`에서 `userConfig`를 캐시하십시오.

**파급력:** `snapshots/Pool.Operations.json`:
```diff
-  "borrow: first borrow->borrowingEnabled": "255409",
-  "borrow: recurrent borrow": "247710",
+  "borrow: first borrow->borrowingEnabled": "255253",
+  "borrow: recurrent borrow": "247555",
   "flashLoan: flash loan for one asset": "197044",
-  "flashLoan: flash loan for one asset and borrow": "277986",
+  "flashLoan: flash loan for one asset and borrow": "277830",
   "flashLoan: flash loan for two assets": "324816",
-  "flashLoan: flash loan for two assets and borrow": "481858",
+  "flashLoan: flash loan for two assets and borrow": "481547",
```

**권장되는 완화 방법:** 커밋 [204a894](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/204a894f194eecdb2ea8d32edc996e52501a988e)를 참조하십시오.


### `RewardsDistributor`에서 `storage` 배열 길이를 캐시하고, 3번 이상 읽을 것으로 예상될 때 `memory`에 대해서도 캐시

**설명:** `storage`에 대한 배열 길이를 캐시하고, 3번 이상 읽을 것으로 예상되는 경우 `memory`에 대해서도 캐시하십시오.

**파급력:** `snapshots/RewardsController.json`:
```diff
-  "getUserAccruedRewards: one reward type": "2182"
+  "getUserAccruedRewards: one reward type": "2090"
```

**권장되는 완화 방법:** 커밋 [a3117ba](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/a3117ba9a5a3e1eb7134219e1797c49168436484)를 참조하십시오.


### `RewardsDistributor:::setDistributionEnd`, `setEmissionPerSecond`에서 이벤트 방출 매개변수 캐싱

**설명:** `RewardsDistributor:::setDistributionEnd`, `setEmissionPerSecond`에서 이벤트 방출 매개변수를 캐시하십시오.

**파급력:** `snapshots/RewardsController.json`:
```diff
-  "setDistributionEnd": "5972"
+  "setDistributionEnd": "5940"
-  "setEmissionPerSecond: one reward one emission": "11541"
+  "setEmissionPerSecond: one reward one emission": "11455"
```

**권장되는 완화 방법:** 커밋 [8f488dc](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/8f488dcc67a0a8469e1b0f99924d5653c03f92a9), [f516e0c](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/f516e0c65b0940c5d8cfc8a23171e1f0eae598ee), [c932b96](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/c932b96c7733c6f3fa5472c8895ebf39ae20df12), [38408b1](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/38408b112b3bb4c2cf4b8581b6e7e484965bb282)를 참조하십시오.


### `RewardsDistributor::_configureAssets`에서 동일한 구문으로 `availableRewardsCount`를 읽고 증가시키기

**설명:** `RewardsDistributor::_configureAssets`에서 동일한 구문으로 `availableRewardsCount`를 읽고 증가시키십시오.

**파급력:** `snapshots/RewardsController.json`:
```diff
-  "configureAssets: one reward type": "264175",
+  "configureAssets: one reward type": "263847",
```

**권장되는 완화 방법:** 커밋 [654ecad](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/654ecada3c3514f9c351258fa4a6c59ec2e93080)를 참조하십시오.


### `numAvailableRewards == 0`일 때 `RewardsDistributor::_updateData`에서 빠르게 종료

**설명:** 종료 전 불필요한 작업을 피하기 위해 `numAvailableRewards == 0`일 때 `RewardsDistributor::_updateData`에서 빠르게 종료하십시오.

이 최적화는 가장 일반적인 경우가 `numAvailableRewards = 0`인 모든 함수의 성능을 향상시키지만, `numAvailableRewards != 0`일 때 보상을 청구할 경우 성능이 더 나빠집니다.

따라서 가장 가능성이 높은 경우를 기준으로 고려해야 하는 트레이드오프입니다.

**파급력:** `snapshots/AToken.transfer.json`:
```diff
-  "full amount; receiver: ->enableCollateral": "144885",
-  "full amount; sender: ->disableCollateral;": "103282",
-  "full amount; sender: ->disableCollateral; receiver: ->enableCollateral": "145028",
-  "full amount; sender: ->disableCollateral; receiver: dirty, ->enableCollateral": "132955",
-  "full amount; sender: collateralDisabled": "103139",
-  "partial amount; sender: collateralDisabled;": "103139",
-  "partial amount; sender: collateralDisabled; receiver: ->enableCollateral": "144885",
-  "partial amount; sender: collateralEnabled;": "103208",
-  "partial amount; sender: collateralEnabled; receiver: ->enableCollateral": "144954"
+  "full amount; receiver: ->enableCollateral": "144757",
+  "full amount; sender: ->disableCollateral;": "103154",
+  "full amount; sender: ->disableCollateral; receiver: ->enableCollateral": "144900",
+  "full amount; sender: ->disableCollateral; receiver: dirty, ->enableCollateral": "132827",
+  "full amount; sender: collateralDisabled": "103011",
+  "partial amount; sender: collateralDisabled;": "103011",
+  "partial amount; sender: collateralDisabled; receiver: ->enableCollateral": "144757",
+  "partial amount; sender: collateralEnabled;": "103080",
+  "partial amount; sender: collateralEnabled; receiver: ->enableCollateral": "144826"
```

`snapshots/Pool.Operations.json`:
```diff
-  "borrow: first borrow->borrowingEnabled": "255253",
-  "borrow: recurrent borrow": "247555",
+  "borrow: first borrow->borrowingEnabled": "255189",
+  "borrow: recurrent borrow": "247491",

-  "flashLoan: flash loan for one asset and borrow": "277830",
+  "flashLoan: flash loan for one asset and borrow": "277766",

-  "flashLoan: flash loan for two assets and borrow": "481547",
+  "flashLoan: flash loan for two assets and borrow": "481419",

-  "liquidationCall: deficit on liquidated asset": "389335",
-  "liquidationCall: deficit on liquidated asset + other asset": "487617",
-  "liquidationCall: full liquidation": "389335",
-  "liquidationCall: full liquidation and receive ATokens": "365723",
-  "liquidationCall: partial liquidation": "380761",
-  "liquidationCall: partial liquidation and receive ATokens": "357148",
-  "repay: full repay": "176156",
-  "repay: full repay with ATokens": "173565",
-  "repay: partial repay": "189587",
-  "repay: partial repay with ATokens": "184775",
-  "supply: collateralDisabled": "146423",
-  "supply: collateralEnabled": "146423",
+  "liquidationCall: deficit on liquidated asset": "389079",
+  "liquidationCall: deficit on liquidated asset + other asset": "487297",
+  "liquidationCall: full liquidation": "389079",
+  "liquidationCall: full liquidation and receive ATokens": "365403",
+  "liquidationCall: partial liquidation": "380505",
+  "liquidationCall: partial liquidation and receive ATokens": "356828",
+  "repay: full repay": "176092",
+  "repay: full repay with ATokens": "173437",
+  "repay: partial repay": "189523",
+  "repay: partial repay with ATokens": "184647",
+  "supply: collateralDisabled": "146359",
+  "supply: collateralEnabled": "146359",
```

`snapshots/RewardsController.json`: (청구가 더 나빠짐)
```diff
-   "claimAllRewards: one reward type": "50131",
-   "claimAllRewardsToSelf: one reward type": "49927",
-   "claimRewards partial: one reward type": "48250",
-   "claimRewards: one reward type": "47988",
+  "claimAllRewards: one reward type": "50311",
+  "claimAllRewardsToSelf: one reward type": "50107",
+  "claimRewards partial: one reward type": "48430",
+  "claimRewards: one reward type": "48168"
```

`snapshots/StataTokenV2.json`: (일부는 더 나빠지고, 일부는 더 좋아짐)
```diff
-  "claimRewards": "359522",
-  "deposit": "280209",
-  "depositATokens": "219279",
-  "redeem": "205420",
-  "redeemAToken": "152498"
+  "claimRewards": "359882",
+  "deposit": "280145",
+  "depositATokens": "219151",
+  "redeem": "205356",
+  "redeemAToken": "152370"
```

`snapshots/WrappedTokenGatewayV3.json`:
```diff
-  "borrowETH": "249186",
-  "depositETH": "222292",
-  "repayETH": "192572",
-  "withdrawETH": "258766"
+  "borrowETH": "249122",
+  "depositETH": "222228",
+  "repayETH": "192508",
+  "withdrawETH": "258574"
```

**권장되는 완화 방법:** 커밋 [be7f13c](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/be7f13c5ddef7343f4a3911a6e573cfcd29ed271)를 참조하십시오.


### `Collector::createStream`에서 동일한 구문으로 `streamId`를 읽고 증가시키며, 명명된 반환 사용

**설명:** `Collector::createStream`에서 동일한 구문으로 `streamId`를 읽고 증가시키며, 명명된 반환을 사용하십시오.

**파급력:** `snapshots/Collector.json`:
```diff
-  "createStream": "211680",
+  "createStream": "211600",
```

**권장되는 완화 방법:** 커밋 [a008981](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/a008981433710d3603be8d44dacbb3d1d62e18b8)를 참조하십시오.


### `Collector::deltaOf`에서 `storage`의 전체 `Stream` 구조체를 `memory`로 복사하지 않음

**설명:** `Collector::deltaOf`에서 2개의 변수만 필요하므로 `storage`의 전체 `Stream` 구조체를 `memory`로 복사하지 마십시오.

**파급력:** `snapshots/Collector.json`:
```diff
-  "cancelStream: by funds admin": "18522",
-  "cancelStream: by recipient": "49489",
+  "cancelStream: by funds admin": "16710",
+  "cancelStream: by recipient": "47635",
-  "withdrawFromStream: final withdraw": "43594",
-  "withdrawFromStream: intermediate withdraw": "42252"
+  "withdrawFromStream: final withdraw": "42656",
+  "withdrawFromStream: intermediate withdraw": "41326"
```

**권장되는 완화 방법:** 커밋 [78c8150](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/78c81502825580181528787596cde4521c05194e)를 참조하십시오.


### `BalanceOfLocalVars` 구조체를 제거하고 `Collector::balanceOf`에서 로컬 변수 사용

**설명:** `BalanceOfLocalVars` 구조체를 제거하고 `Collector::balanceOf`에서 로컬 변수를 사용하십시오.

**파급력:** `snapshots/Collector.json`:
```diff
-  "cancelStream: by funds admin": "16710",
-  "cancelStream: by recipient": "47635",
+  "cancelStream: by funds admin": "16483",
+  "cancelStream: by recipient": "47407",
-  "withdrawFromStream: final withdraw": "42656",
-  "withdrawFromStream: intermediate withdraw": "41326"
+  "withdrawFromStream: final withdraw": "42560",
+  "withdrawFromStream: intermediate withdraw": "41230"
```

**권장되는 완화 방법:** 커밋 [cc31124](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/cc31124fd1f30081c053f9a1b1294afad2b9958f)를 참조하십시오.


### `CreateStreamLocalVars` 구조체를 제거하고 `Collector::createStream`에서 로컬 변수 사용

**설명:** `CreateStreamLocalVars` 구조체를 제거하고 `Collector::createStream`에서 로컬 변수를 사용하십시오.

**파급력:** `snapshots/Collector.json`:
```diff
-  "createStream": "211600",
+  "createStream": "211518",
```

**권장되는 완화 방법:** 커밋 [dc3da97](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/dc3da97e44f523045642364d301c08f578450361)를 참조하십시오.


### `Collector::withdrawFromStream` 및 `cancelStream`에서 `storage`의 전체 `Stream` 구조체를 `memory`로 복사하지 않고 `onlyAdminOrRecipient` 수정자를 내부 함수로 리팩터링

**설명:** `Collector::withdrawFromStream` 및 `cancelStream`에서 `storage`의 전체 `Stream` 구조체를 `memory`로 복사하지 말고 `onlyAdminOrRecipient` 수정자를 내부 함수로 리팩터링하십시오.

**파급력:** `snapshots/Collector.json`:
```diff
-  "cancelStream: by funds admin": "16483",
-  "cancelStream: by recipient": "47407",
+  "cancelStream: by funds admin": "15718",
+  "cancelStream: by recipient": "46456",
-  "withdrawFromStream: final withdraw": "42560",
-  "withdrawFromStream: intermediate withdraw": "41230"
+  "withdrawFromStream: final withdraw": "41449",
+  "withdrawFromStream: intermediate withdraw": "40224"
```

**권장되는 완화 방법:** 커밋 [73e6123](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/73e61234427c673fd1852f88862b3f154d2c74ce)를 참조하십시오.


### `PoolAddressesProviderRegistry::unregisterAddressesProvider`에서 `require` 확인 전에 `oldId`를 캐시하여 스토리지 읽기 1회 절약

**설명:** `PoolAddressesProviderRegistry::unregisterAddressesProvider`에서 `require` 확인 전에 `oldId`를 캐시하여 스토리지 읽기 1회를 절약하십시오:

**권장되는 완화 방법:**
```diff
  function unregisterAddressesProvider(address provider) external override onlyOwner {
-   require(_addressesProviderToId[provider] != 0, Errors.ADDRESSES_PROVIDER_NOT_REGISTERED);
    uint256 oldId = _addressesProviderToId[provider];
+   require(oldId != 0, Errors.ADDRESSES_PROVIDER_NOT_REGISTERED);
    _idToAddressesProvider[oldId] = address(0);
    _addressesProviderToId[provider] = 0;
```

커밋 [93935b8](https://github.com/devdacian/aave-v3-origin-liquidation-gas-fixes/commit/93935b820e52ea8ee8498eae928b2c2d9b695124)를 참조하십시오.

\clearpage
