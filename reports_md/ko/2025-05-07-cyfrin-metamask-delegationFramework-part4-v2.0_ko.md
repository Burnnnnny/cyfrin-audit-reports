**수석 감사자**

[0kage](https://twitter.com/0kage_eth)

**보조 감사자**



---

# 결과 (Findings)
## 정보성 (Informational)


### `LogicalOrWrapperEnforcer`의 실행 모드 제한 제거 가능

**설명:** `LogicalOrWrapperEnforcer` 컨트랙트는 현재 모든 후크(hook) 함수에 `onlyDefaultExecutionMode` 수정자를 포함하고 있습니다. 래핑된 개별 enforcer들이 이미 자체적인 실행 모드 제한을 적용하고 있기 때문에 이는 불필요한 제한을 생성합니다. 이러한 설계 결정은 래퍼의 유연성을 제한하고 기본이 아닌 실행 모드를 지원할 수 있는 caveat와 함께 작동하는 것을 방지할 수 있습니다.


**권장되는 완화 방법:** `LogicalOrWrapperEnforcer`의 모든 후크 함수에서 `onlyDefaultExecutionMode` 수정자를 제거하고 래핑된 개별 caveat enforcer가 자체 실행 모드 제한을 처리하도록 하는 것을 고려하십시오.

이는 가스 효율적일 뿐만 아니라, `LogicalOrWrapperEnforcer`가 미래의 caveat에 대해 더 유연하고 호환성을 갖도록 허용하는 동시에, 래핑된 enforcer 컨트랙트 자체를 통해 적절한 실행 모드 제한을 유지할 수 있게 합니다.

**Metamask:** 커밋 [d38d53d](https://github.com/MetaMask/delegation-framework/commit/d38d53dc467cc3b4faa7047cfca1844ea9cbc3be)에서 수정되었습니다.

**Cyfrin:** 해결되었습니다.


### `LogicalOrWrapperEnforcer`에서 위임자(Delegate)가 제어하는 권한 상승 위험

**설명:** `LogicalOrWrapperEnforcer` enforcer를 사용할 때, 위임자(delegator)는 모든 그룹이 적절한 보안 경계를 제공할 것이라고 기대하며 서로 다른 보안 속성을 가진 여러 caveat 그룹을 정의할 수 있습니다.

그러나 실행 중에 어떤 그룹이 평가될지 대리인(delegate)이 제어하므로, 대리인은 다른 그룹에 정의된 더 엄격한 보안 요구 사항을 우회하기 위해 가장 덜 제한적인 그룹을 선택할 수 있습니다.

예를 들어, 위임자가 두 그룹을 정의하는 경우:

- 그룹 0: 최소 100 토큰의 잔액 변경 필요
- 그룹 1: 최소 50 토큰의 잔액 변경 필요

대리인은 실행 시 그룹 1을 선택하여, 위임자가 100 토큰 최소값이 적용될 것으로 예상했을 때 50 토큰만 전송할 수 있습니다.

이러한 동작은 의도된 것이지만, 대리인이 가장 덜 제한적인 caveat 그룹을 선택하여 의도된 보안 제한을 우회할 수 있는 시나리오를 만듭니다. 이 enforcer의 동작이 대리인에게 그러한 통제권을 부여하기 때문에, 이는 효과적으로 정의된 모든 그룹에서 가장 덜 제한적인 옵션으로 권한을 상승시킬 수 있게 합니다.

**권장되는 완화 방법:** 모든 잔액 변경 enforcer에 추가된 것과 유사한 보안 공지를 추가하는 것을 고려하십시오.

```solidity
/**
 * @dev Security Notice: This enforcer allows delegates to choose which caveat group to use at execution time
 * via the groupIndex parameter. If multiple caveat groups are defined with varying levels of restrictions,
 * delegates can select the least restrictive group, bypassing stricter requirements in other groups.
 *
 * To maintain proper security:
 * 1. Ensure each caveat group represents a complete and equally secure permission set
 * 2. Never assume delegates will select the most restrictive group
 * 3. Design caveat groups with the understanding that delegates will choose the path of least resistance
 *
 * Use this enforcer at your own risk and ensure it aligns with your intended security model.
 */
```


**Metamask:** 커밋 [d38d53d](https://github.com/MetaMask/delegation-framework/commit/d38d53dc467cc3b4faa7047cfca1844ea9cbc3be)에서 수정되었습니다.

**Cyfrin:** 해결되었습니다.

\clearpage
