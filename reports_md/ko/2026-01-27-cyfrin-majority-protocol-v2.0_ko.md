**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Jorge](https://x.com/TamayoNft)

**Assisting Auditors**




---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### `DepositManager::_refundEntryFee`가 추천인 보상을 차감하지 않아, 사용자가 게임에 참여했다가 나가면서 받을 자격이 없는 추천인 보상을 부풀려 토큰을 유출할 수 있음

**설명:** `DepositManager::_payEntryFee`는 사용자의 추천인에게 추천 보상을 적립하지만, `DepositManager::_refundEntryFee`는 사용자가 게임을 떠나 참가비를 환불받을 때 그만큼을 다시 차감하지 않습니다.

**영향:** 악의적 사용자는 리스케줄된 게임에 참여했다가 나가는 과정을 반복해 컨트랙트에서 토큰을 빼낼 수 있습니다. 악의적 의도가 없더라도 참가와 이탈이 반복되면 추천인 보상이 과다 배정되어, 최종적으로 우승자 보상이나 creator/protocol fee를 지급할 토큰이 부족해질 수 있습니다.

**권장 완화 조치:** 환불 시 `DepositManager::_payEntryFee`의 반대 방향으로 추천인 보상을 차감하십시오.

**Majority Games:** 커밋 [50a1e6b](https://github.com/Engage-Protocol/engage-protocol/commit/50a1e6bb3a48a6056cbf0678030be0e9424ba052)에서 수정됨.

**Cyfrin:** 확인함.


### `SessionManager::refundCancelledGame`가 호출자가 실제 참가자인지 검증하지 않아, 공격자가 취소된 게임의 토큰을 모두 빼갈 수 있음

**설명:** 취소된 게임 환불 함수가 호출자가 해당 게임에 실제로 참가했는지 확인하지 않습니다.

**영향:** permissionless 공격자는 취소된 게임의 자금을 전부 환불받아 고갈시킬 수 있습니다.

**권장 완화 조치:** 해당 게임에 실제로 참가한 사용자만 환불을 청구할 수 있도록 제한하십시오.

**Majority Games:** 커밋 [7692203](https://github.com/Engage-Protocol/engage-protocol/commit/7692203e579204d829bbb558716a5c8637ac2ef5)에서 수정됨.

**Cyfrin:** 확인함.


### 사용자는 참가하지 않은 무한한 수의 게임에 참여해 모든 참가비 요구사항을 우회하면서도 우승 및 보상 청구가 가능함

**설명:** `SessionManager::commitReaction`과 `revealReaction`은 사용자가 `_gameId`에 참가했는지는 보지만, 입력된 `_questionId`가 실제로 그 `_gameId`에 속하는지 검증하지 않습니다. 전략 컨트랙트들도 반응을 `questionId` 기준으로만 저장해 게임과의 연계를 강제하지 않습니다.

**영향:** 공격자는 한 번만 입장료를 내거나, 심지어 입장료가 0인 게임에만 참가한 뒤 다른 게임들에는 무료로 참여할 수 있습니다. 그럼에도 우승자가 되어 상금을 가져갈 수 있으므로 정상 사용자 대비 매우 큰 이점을 얻습니다.

**권장 완화 조치:** 모든 게임 관련 동작에서 사용자가 실제로 해당 게임에 참가했는지, 그리고 입력된 `questionId`가 입력된 `gameId`에 속하는지 함께 검증하십시오.

**Majority Games:** 커밋 [62cafca](https://github.com/Engage-Protocol/engage-protocol/commit/62cafcafd1bfbdb73809d0cd01c746e3586183b9), [01d5cc2](https://github.com/Engage-Protocol/engage-protocol/commit/01d5cc27f9ce72f0666083a2e37959c61ba58649), [f0e77f9](https://github.com/Engage-Protocol/engage-protocol/commit/f0e77f97da55310d8cb8c0a04096f5d3ef480b55)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 높은 위험 (High Risk)


### 리스케줄된 게임에 재참여한 뒤 다시 취소되면 사용자가 환불을 받을 수 없음

**설명:** 사용자가 리스케줄된 게임에서 한 번 환불받은 후 다시 참가하면, 이후 게임이 취소되었을 때 환불 상태가 적절히 초기화되지 않아 재환불이 불가능해집니다.

**영향:** 사용자가 다시 낸 참가비가 불변 `SessionManager` 컨트랙트 안에 영구적으로 잠깁니다.

**권장 완화 조치:** 사용자가 재참가할 때 환불 상태를 초기화하거나, 이미 환불받은 사용자의 재참가 자체를 허용하지 마십시오.

**Majority Games:** 동일 게임에서 환불받은 사용자의 재참가를 막는 방식으로 커밋 [3ac5654](https://github.com/Engage-Protocol/engage-protocol/commit/3ac565495df69ba8936be9d3d91a77eeb639b366)에서 수정됨.

**Cyfrin:** 확인함.


### `DepositManager::getRewards`가 항상 `REFERRER_FEE`를 포함해 계산하므로, 추천인이 없을 때도 각 게임 보상의 2%가 우승자에게 분배되지 않음

**설명:** 게임에 실제 추천인이 없더라도 `getRewards`가 항상 `REFERRER_FEE`를 차감한 뒤 우승자 보상 풀을 계산합니다.

**영향:** 추천인이 없는 게임에서는 우승자에게 돌아가야 할 보상이 실제보다 작아지며, 그 2%는 배분되지 않은 채 남게 됩니다.

**권장 완화 조치:** 고정 `REFERRER_FEE` 대신 실제로 누적된 추천인 보상 총액만 차감하도록 회계를 바꾸십시오.

**Majority Games:** `address(0)`에 귀속된 추천 수수료를 `CLAIMER_ROLE`이 회수할 수 있도록 하는 방식으로 커밋 [e090f2e](https://github.com/Engage-Protocol/engage-protocol/commit/e090f2e1b5f42eb212fdbda7be94ccf295281075)에서 수정됨.

**Cyfrin:** 확인함. 다만 `AccessControl::grantRole`이 override되지 않아, referrer에게 `CLAIMER_ROLE`이 부여되면 해당 주소에 귀속된 referral fee를 청구하지 못하게 될 수 있다는 점은 남아 있습니다.


### ranked rewards나 우승자 수가 설정되지 않으면 게임 종료 후 보상을 청구할 수 없어 토큰이 영구 잠김

**설명:** `FixedRanksReward::setRankedRewards`와 우승자 수 설정은 게임이 `Created` 상태일 때만 가능하지만, 설정이 빠진 채 게임이 끝날 수 있습니다.

**영향:** 게임 종료 후 우승자들이 보상을 청구하면 `RankedRewardsNotSet` 또는 `NumberOfWinnersMismatch`로 revert하고, 이미 `Concluded` 상태이므로 취소도 불가능해 자금이 잠깁니다.

**권장 완화 조치:** ranked rewards와 우승자 수가 모두 설정되기 전에는 게임 시작을 허용하지 마십시오.

**Majority Games:** 커밋 [a2e353e](https://github.com/Engage-Protocol/engage-protocol/commit/a2e353e664f7707d49a3ca9ca2bea792d731711c), [96d5fbe](https://github.com/Engage-Protocol/engage-protocol/commit/96d5fbe3132bbbecb509c8ca90cc785587da5e61)에서 수정됨.

**Cyfrin:** 확인함.


### `XPTiers`가 설정되지 않으면 게임 종료 후 보상을 청구할 수 없어 토큰이 잠김

**설명:** `DefaultSession::setXPTiers`도 `Created` 상태에서만 호출할 수 있는데, 이 설정이 빠진 상태로 게임이 종료될 수 있습니다.

**영향:** 게임이 종료된 뒤 우승자들은 사실상 0에 해당하는 보상을 받거나 보상 분배가 깨져 자금이 잠길 수 있습니다.

**권장 완화 조치:** `XPTiers`가 설정되기 전에는 게임 시작을 허용하지 마십시오.

**Majority Games:** 커밋 [65727de](https://github.com/Engage-Protocol/engage-protocol/commit/65727de8b15027a8ac8b61b7203be692e60f34cb)에서 수정됨.

**Cyfrin:** 확인함.


### 여러 사용자가 `DefaultSession::assertResults`를 호출하면 첫 번째 호출자를 제외한 나머지는 bond를 잃음

**설명:** `assertResults`는 permissionless 함수인데, 동일 `sessionId`에 대해 중복 assertion이 진행될 때 후속 호출자의 bond 회수 경로가 안전하게 처리되지 않습니다.

**영향:** 첫 번째 호출자 외의 사용자는 돈을 잃을 수 있습니다.

**권장 완화 조치:** 같은 `sessionId`에 대해 진행 중인 assertion을 하나만 허용하거나, 이미 처리된 assertion에 대한 callback에서는 revert하지 않고 조용히 종료하십시오.

**Majority Games:** 이미 처리된 assertion이면 callback에서 revert하지 않고 return하는 방식으로 커밋 [4c5483f](https://github.com/Engage-Protocol/engage-protocol/commit/4c5483fd6f39b49f2fcd93055151244b4b6cd262)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 중간 위험 (Medium Risk)


### 첫 번째 문제가 공개된 뒤에도 사용자가 게임에 참가해 다른 사용자보다 유리한 위치를 점할 수 있음

**설명:** `joinGame`이 게임이 `Ongoing` 상태일 때도 참가를 허용합니다.

**영향:** 사용자는 첫 번째 문제가 공개된 뒤 답을 알고 있을 때만 참가하는 식으로 부당한 이점을 얻을 수 있습니다.

**권장 완화 조치:** 게임이 이미 진행 중이면 참가를 허용하지 마십시오.

**Majority Games:** 커밋 [6ec205f](https://github.com/Engage-Protocol/engage-protocol/commit/6ec205f68d5f0d2bcf25035d5da09fe859f065b7)에서 수정됨.

**Cyfrin:** 확인함.


### `MajorityChoicePrompt`, `SPBinaryPrompt`, `TriviaChoicePrompt`는 서로 다른 `SessionManager` 인스턴스와 함께 사용할 때 올바르게 동작하지 않음

**설명:** 각 `SessionManager` 인스턴스가 같은 범위의 `questionId`를 재사용할 수 있는데, 프롬프트 전략은 이를 인스턴스별로 분리해 저장하지 않습니다.

**영향:** 서로 다른 `SessionManager` 인스턴스의 상태가 충돌해 결과 계산이나 반응 저장이 깨질 수 있습니다.

**권장 완화 조치:** 각 `SessionManager`마다 별도의 프롬프트 인스턴스를 사용하거나, 상태 저장 key에 `SessionManager` 식별자를 포함하십시오.

**Majority Games:** 커밋 [4b151db](https://github.com/Engage-Protocol/engage-protocol/commit/4b151db34ae5e0adb59076e472f292cfbeb9f571), [46d00d3](https://github.com/Engage-Protocol/engage-protocol/commit/46d00d3096a86694ccfa2b6ddec2ba90265e6ba2), [35c63e7](https://github.com/Engage-Protocol/engage-protocol/commit/35c63e77e829f48d6bb6b74bf6f3f5446399ec9b)에서 수정됨.

**Cyfrin:** 확인함.


### `recordResults`에서 각 문제에 잘못된 `recordResult`가 기록됨

**설명:** assertion이 truthful로 판정된 뒤 `recordResults`가 각 문제 결과를 기록할 때 잘못된 값을 넘깁니다.

**영향:** 전략 전반에서 `getResult`가 틀린 값을 반환하게 되어 결과 조회와 점수 계산이 왜곡됩니다.

**권장 완화 조치:** 각 문제에 대해 올바른 결과 값을 전달하도록 `recordResults` 호출 로직을 수정하십시오.

**Majority Games:** 커밋 [a3bcfb6](https://github.com/Engage-Protocol/engage-protocol/commit/a3bcfb6518f0eb33a6a37089e9e0a2c14ea7b210)에서 수정됨.

**Cyfrin:** 확인함.


### 모든 사용자의 XP가 0이면 게임 종료 후 `SessionManager::claimRewards`가 0으로 나누기 때문에 panic revert하며, 게임도 취소할 수 없어 토큰이 잠김

**설명:** `ProportionalToXPReward::getReward`는 총 XP로 나누는데, 모든 사용자가 XP를 얻지 못하면 `totalXP == 0`이 됩니다.

**영향:** 보상 청구가 division by zero로 revert하고, 이미 게임은 `Concluded` 상태라 취소도 못 해 자금이 잠깁니다.

**권장 완화 조치:** `DefaultSession::setXPTiers`에서 XP tier를 모두 0이 아닌 값으로 강제하십시오.

**Majority Games:** XP tier를 0이 아닌 값으로 강제하고 상한도 도입하는 방식으로 커밋 [951a454](https://github.com/Engage-Protocol/engage-protocol/commit/951a45490d2867f80dbf56bb4ce915c44a9a1281)에서 수정됨.

**Cyfrin:** 확인함.


### `reactionDeadline` 검증이 없어 여러 griefing 시나리오가 가능함

**설명:** 게임 생성 시 입력되는 `reactionDeadline` 범위가 검증되지 않습니다.

**영향:** creator가 `reactionDeadline`을 0 또는 `type(uint256).max` 같은 값으로 설정하면 답변 제출이 사실상 불가능해질 수 있고, 게임 종료 시점과도 어긋나면서 사용자 경험과 공정성이 깨집니다.

**권장 완화 조치:** `reactionDeadline`에 관리자 제어 최소/최대 범위를 두고, 게임 `endTime`을 넘지 못하게 하며, 관련 입력 배열 길이도 함께 검증하십시오.

**Majority Games:** `reactionDeadline` 제한을 추가하는 방식으로 커밋 [4cb2e42](https://github.com/Engage-Protocol/engage-protocol/commit/4cb2e42fa67194bea32775cba86c15045a2f56ba)에서 수정됨.

**Cyfrin:** 확인함. 다만 `media`와 `choices`는 반드시 1:1 대응일 필요는 없으므로, 의미 있는 조합을 만드는 책임은 creator와 프론트엔드에 있습니다.


### 게임 creator가 `TriviaChoicePrompt::revealSolutions`를 `reactionDeadline` 이전이나 게임 종료 전에 호출해, 참가비는 유지한 채 사용자의 답변 제출을 방해할 수 있음

**설명:** `revealSolutions`가 너무 이른 시점에 호출되어도 답 제출 경로가 이를 막지 못합니다.

**영향:** 악의적 creator는 해답을 즉시 공개해 사용자가 XP를 쌓지 못하게 만들면서도 참가비는 유지할 수 있습니다.

**권장 완화 조치:** `reactionDeadline`이 지났거나 게임이 종료된 뒤에만 해답 공개를 허용하십시오.

**Majority Games:** 커밋 [4d3f8b5](https://github.com/Engage-Protocol/engage-protocol/commit/4d3f8b5be490bdba368d3d5e961ba3e678dcad9e)에서 수정됨.

**Cyfrin:** 확인함. 선택된 수정은 creator가 해답을 너무 일찍 공개하면 스스로도 게임 종료, 보상 분배, 수수료 청구를 못 하게 만들어 grief 유인을 제거하는 더 깔끔한 방식입니다.


### `SPBinaryPrompt::getScore`와 `getResult`가 불참자의 점수 처리에서 충돌하며, `getScore`는 오답 사용자에게도 보상을 줌

**설명:** `getScore`는 불참자에게 0을 반환하지만 `getResult`는 이를 다시 XP tier에 매핑해 0보다 큰 점수를 줄 수 있습니다. 또한 정답 여부를 확인하지 않은 채 확률 예측만으로 점수를 줄 수 있습니다.

**영향:** 불참자나 오답자가 점수를 얻는 비정상 결과가 발생합니다.

**권장 완화 조치:** `getScore`와 `getResult`의 의미를 일치시키고, 오답 사용자에게는 보상을 주지 마십시오.

**Majority Games:** 커밋 [50657e9](https://github.com/Engage-Protocol/engage-protocol/commit/50657e94bb54245a456520c41982b882d2d08433), [a55eb19](https://github.com/Engage-Protocol/engage-protocol/commit/a55eb1998d08ebfff17e668887986878409cd8d5)에서 수정됨.

**Cyfrin:** 확인함.


### `SessionManager::rescheduleGame`이 시작 시간만 늦추고 종료 시간은 갱신하지 않아, creator가 참가비를 거두면서도 실제 참여를 방해하는 griefing 공격이 가능함

**설명:** 게임을 리스케줄할 때 새 시작 시간만 바꾸고 기존 종료 시간은 그대로 둡니다.

**영향:** 공식 종료 시간이 이미 지나버린 상태에서 게임이 시작될 수 있으며, creator는 수수료를 받으면서도 정상적인 진행을 막을 수 있습니다.

**권장 완화 조치:** 새 시작 시간에 맞춰 원래 게임 길이만큼 종료 시간도 함께 이동시키십시오.

**Majority Games:** 커밋 [ddb690f](https://github.com/Engage-Protocol/engage-protocol/commit/ddb690f09fc49e0a5d9191f1c8c9ce74c434aa58)에서 수정됨.

**Cyfrin:** 확인함.


### 사용자가 답변의 확률 값을 `uint16.max`로 설정해 `result.probabilityAverage`를 자신에게 유리하게 조작할 수 있음

**설명:** `SPBinaryPrompt::revealReaction`은 사용자가 공개하는 확률 값이 10,000을 넘는지 검증하지 않습니다.

**영향:** 공격자는 `probabilityAverage`를 왜곡해 더 높은 순위를 차지할 수 있습니다.

**권장 완화 조치:** 확률 값 상한을 10,000으로 제한하십시오.

**Majority Games:** 커밋 [2eaae4d](https://github.com/Engage-Protocol/engage-protocol/commit/2eaae4d5a6213f9728d04c96b346c28b3a618c3c)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `DefaultSession::assertionResolvedCallback`에서 이전 truthful assertion 뒤에 negative assertion이 오는 경우를 막아야 함

**설명:** 같은 `assertionId`에 대해 먼저 truthful 처리 후 나중에 false 처리되면, 이미 기록된 결과와 상충하는 삭제 로직이 실행될 수 있습니다.

**영향:** 이미 반영된 결과가 뒤늦게 지워질 수 있습니다.

**권장 완화 조치:** `assertions[assertionId].resolved`가 이미 `true`면 revert하십시오.

**Majority Games:** 커밋 [99ec735](https://github.com/Engage-Protocol/engage-protocol/commit/99ec735d6b0a42c22fd0af6ae6ec8c91ef2e922d)에서 수정됨.

**Cyfrin:** 확인함.


### `maximumContestants`가 지나치게 크면 `DefaultSession::recordResults`가 out-of-gas로 revert할 수 있음

**설명:** 기본 `maximumContestants`가 100만으로 설정되어 있어 결과 기록 시 과도한 반복이 일어날 수 있습니다.

**영향:** 게임이 정상적으로 종료되지 못할 수 있습니다.

**권장 완화 조치:** 보다 현실적인 참가자 상한을 두거나, 결과 기록을 청크 단위로 나누십시오.

**Majority Games:** 커밋 [a2e353e](https://github.com/Engage-Protocol/engage-protocol/commit/a2e353e664f7707d49a3ca9ca2bea792d731711c)에서 수정됨.

**Cyfrin:** 확인함.


### 추천받지 않은 플레이어의 보상이 `address(0)`에 누적됨

**설명:** `Registry(registry).referrers(player)`가 추천인이 없는 경우 `address(0)`을 반환하는데, 현재 로직은 이 경우에도 추천 보상을 할당합니다.

**영향:** 청구할 수 없는 추천 보상이 `address(0)`에 쌓입니다.

**권장 완화 조치:** 실제 추천인이 존재할 때만 추천 보상을 할당하십시오.

**Majority Games:** `address(0)`에 할당된 추천 수수료를 별도 회수할 수 있게 하는 방식으로 커밋 [e090f2e](https://github.com/Engage-Protocol/engage-protocol/commit/e090f2e1b5f42eb212fdbda7be94ccf295281075)에서 수정됨.

**Cyfrin:** 확인함. 다만 referrer에게 `CLAIMER_ROLE`이 나중에 부여될 수 있는 경로는 여전히 주의가 필요합니다.


### 게임에 질문이 너무 많으면 `SessionManager::cancelGameIfCreatorMissing`와 `endGame`이 out-of-gas로 revert할 수 있음

**설명:** 질문 수에 상한이 없어, 질문이 과도하게 많은 게임은 종료 또는 creator 부재 취소 처리 중 전 질문을 순회하면서 OOG가 날 수 있습니다.

**영향:** `endGame`이 실패하면 사용자와 creator의 자금이 묶일 수 있고, `cancelGameIfCreatorMissing`이 실패하면 사용자의 환불도 막힐 수 있습니다.

**권장 완화 조치:** 게임당 질문 수에 상한을 두십시오.

**Majority Games:** 커밋 [cb88233](https://github.com/Engage-Protocol/engage-protocol/commit/cb8823378ef74d688ff15eefb7b6ac0d2b0e5bc2)에서 수정됨.

**Cyfrin:** 확인함.


### 같은 사용자가 같은 게임에 여러 번 참가해 다른 플레이어의 자리를 차지하며 당첨 확률을 높일 수 있음

**설명:** `SessionManager::joinGame`은 이미 참가한 사용자인지 확인하지 않습니다.

**영향:** 동일 사용자가 대부분의 참가 슬롯을 차지해 다른 사용자의 참여를 막고, 자신의 승률을 크게 높일 수 있습니다.

**권장 완화 조치:** `contestants[_gameId][msg.sender] == true`이면 revert하도록 하십시오.

**Majority Games:** 커밋 [2bba52d](https://github.com/Engage-Protocol/engage-protocol/commit/2bba52d8a8dfecf45566b2d0b1790161102becd2)에서 수정됨.

**Cyfrin:** 확인함.


### `DepositManager::sponsorGame`는 게임이 `Cancelled` 또는 `Concluded` 상태일 때 revert해야 함

**설명:** `sponsorGame`은 sponsorship을 받을 때 게임 상태를 확인하지 않아, 취소되었거나 종료된 게임, 심지어 존재하지 않는 게임에도 자금이 들어갈 수 있습니다.

**영향:** 후원자는 회수할 수 없는 게임에 자금을 넣을 수 있으며, 특히 종료된 게임에 보낸 토큰은 사실상 영구 손실됩니다.

**권장 완화 조치:** 존재하는 게임 중 `Created` 또는 `Ongoing` 상태에서만 sponsorship을 허용하십시오.

**Majority Games:** 기존 게임이면서 `Created` 또는 `Ongoing` 상태일 때만 sponsorship을 허용하는 방식으로 커밋 [e01a1df](https://github.com/Engage-Protocol/engage-protocol/commit/e01a1df84bf8dc1cfda40ef9a52ac7bcc5e6fc75)에서 수정됨.

**Cyfrin:** 확인함.


### `SessionManager::revealGameQuestion`가 입력 `_questionId`가 입력 `_gameId`에 속하는지 검증하지 않음

**설명:** 질문 공개 경로 어디에서도 `_questionId`와 `_gameId`의 실제 연계를 강제하지 않습니다.

**영향:** creator는 다른 진행 중 게임의 `_gameId`를 끼워 넣어, 자기 게임이 아직 `Ongoing`이 아니더라도 질문을 공개하는 식으로 상태 제약을 우회할 수 있습니다.

**권장 완화 조치:** `revealGameQuestion`뿐 아니라 `startAndRevealGameQuestion`에도 동일하게 `questionId`가 `gameId`에 속하는지 검증하십시오.

**Majority Games:** 커밋 [15a2459](https://github.com/Engage-Protocol/engage-protocol/commit/15a24591dd9e1987e0f5383cc2d7de28e3072c77)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보성 (Informational)


### key와 value의 목적이 드러나도록 named mapping을 사용해야 함

**설명:** 일부 매핑은 named mapping을 쓰지만, 일부는 그렇지 않아 코드 읽기가 어렵습니다.

**Majority Games:** 커밋 [130e0a3](https://github.com/Engage-Protocol/engage-protocol/commit/130e0a33b69cc7381a0eea12719f14156bcc3446)에서 반영됨.

**Cyfrin:** 확인함.


### 외부 호출 전에 storage를 먼저 갱신해야 함

**설명:** 몇몇 함수는 외부 호출 후 storage를 갱신하고 있어, 순서를 바꾸는 편이 더 안전합니다.

**Majority Games:** 커밋 [6525ee1](https://github.com/Engage-Protocol/engage-protocol/commit/6525ee1547e0b7834cb99a786773bed1861369c7)에서 수정됨.

**Cyfrin:** 확인함.


### `revealSolution`의 주석을 수정해야 함

**설명:** 해당 주석은 여전히 session manager만 호출할 수 있는 것처럼 설명하지만, 현재는 그렇지 않습니다.

**Majority Games:** 커밋 [acb42cb](https://github.com/Engage-Protocol/engage-protocol/commit/acb42cbc4422d6ade640864b4d74dd3beffbcebc)에서 수정됨.

**Cyfrin:** 확인함.


### 악성 사용자가 `revealSolutions`를 프런트런해 정답을 맞춘 커밋을 넣을 수 있음

**설명:** 사용자는 `revealSolutions` 트랜잭션이 포함되기 전에 정답을 보고 맞는 커밋을 넣을 수 있습니다.

**영향:** 정답 제출 시점 공정성이 훼손될 수 있습니다.

**Majority Games:** 인지됨.


### named return value를 사용할 때 불필요한 `return` 문을 제거해야 함

**설명:** 일부 함수는 named return value를 사용하면서도 구식 `return` 문을 남겨두고 있습니다.

**Majority Games:** 커밋 [cc1c9d1](https://github.com/Engage-Protocol/engage-protocol/commit/cc1c9d17c69b817de2ff03e2b64ce5519a14df15)에서 수정됨.

**Cyfrin:** 확인함.


### 일관성을 위해 모든 `sessionId`를 `gameId`로 바꾸거나 그 반대로 통일해야 함

**설명:** 같은 의미를 가리키는 값이 `sessionId`와 `gameId`로 섞여 사용됩니다.

**Majority Games:** 모든 명칭을 `sessionId`로 통일하는 방식으로 커밋 [75663d1](https://github.com/Engage-Protocol/engage-protocol/commit/75663d17c2a514eac4ccc7c01bb2d780ed344ab3)에서 수정됨.

**Cyfrin:** 확인함.


### 게임 creator가 게임 종료 후 취소를 호출해 우승자의 보상 수령을 방해할 수 있음

**설명:** `SessionManager::cancelGame`은 게임이 `Ended` 상태일 때도 creator가 취소할 수 있게 합니다.

**영향:** creator가 우승자가 마음에 들지 않으면 게임을 취소해 우승자 보상을 막을 수 있습니다.

**Majority Games:** 오라클이 계산 규칙에 따라 정산하지 못하는 상황을 고려해 인지됨.


### `FixedRanksReward::getRewards`, `getReward`의 배열 길이 검사 comparator가 잘못됨

**설명:** 배열 길이 비교가 `>` 기준으로 되어 있어 필요한 길이 검증이 올바르게 이뤄지지 않습니다.

**Majority Games:** 커밋 [6717163](https://github.com/Engage-Protocol/engage-protocol/commit/6717163d9d0fbe98a8c2af006b00da9edc20796f)에서 수정됨.

**Cyfrin:** 확인함.


### `DefaultSession::assertResults`는 `proposedWinners`, `totalXPs`, `totalTimes` 배열 길이가 다르면 revert해야 함

**설명:** 세 입력 배열 길이가 다를 때도 검증이 부족합니다.

**영향:** asserter가 실수로 길이가 다른 배열을 넣으면 bond를 잃을 수 있습니다.

**권장 완화 조치:** 세 배열 길이가 모두 같은지 검사하십시오.

**Majority Games:** 커밋 [aafd672](https://github.com/Engage-Protocol/engage-protocol/commit/aafd672a20ba8771c36a860fd8b9b59ab966a594)에서 수정됨.

**Cyfrin:** 확인함.


### 우승자가 결정된 뒤에는 누구나 게임을 conclude할 수 있어야 함

**설명:** 현재는 오직 creator만 `concludeGame`을 호출할 수 있습니다.

**영향:** creator가 결과가 마음에 들지 않으면 게임 conclude를 거부해 보상 지급을 지연시킬 수 있습니다.

**권장 완화 조치:** 적절한 타임아웃 뒤에는 누구나 `concludeGame`을 호출할 수 있게 하십시오.

**Majority Games:** 커밋 [dca8622](https://github.com/Engage-Protocol/engage-protocol/commit/dca86228c93ad73486766a8d06f0e63eb292ee26)에서 수정됨.

**Cyfrin:** 확인함.


### `Prompt::finalizedAnswer`가 한 번도 설정되지 않음

**설명:** `Prompt` 구조체에 `finalizedAnswer`가 있지만, 해답 공개 이후에도 이 값이 설정되지 않습니다.

**영향:** 사실상 사용되지 않는 필드가 남아 있어 의도를 흐립니다.

**권장 완화 조치:** 실제로 값을 설정하거나, 필요 없다면 제거하십시오.

**Majority Games:** 커밋 [581a98d](https://github.com/Engage-Protocol/engage-protocol/commit/581a98d91b0246443f5c51bde665ae3641441fd3)에서 수정됨.

**Cyfrin:** 확인함.


### `Prompt::gameId`는 `questionId`와의 연계가 검증되지 않고 실제로도 사용되지 않아 제거 후보임

**설명:** `Prompt::gameId`는 본래 `questionId`와의 관계를 표현하지만, 실제로 검증에 쓰이지 않습니다.

**Majority Games:** 처음에는 커밋 [3644561](https://github.com/Engage-Protocol/engage-protocol/commit/36445618404482096f9170f605e88a7e7039a1bd)에서 제거했지만, 이후 `finalizedAnswer` 문제를 해결하기 위해 커밋 [581a98d](https://github.com/Engage-Protocol/engage-protocol/commit/581a98d91b0246443f5c51bde665ae3641441fd3)에서 다시 추가됨.

**Cyfrin:** 확인함.


### `DefaultSession::assertResults`는 입력 `sessionId`가 자기 인스턴스와 연계된 게임인지 검증해야 함

**설명:** 서로 다른 게임이 서로 다른 `DefaultSession` 인스턴스를 쓸 수 있는데, 현재는 입력 `sessionId`가 현재 인스턴스용 게임인지 확인하지 않습니다.

**영향:** 잘못된 세션 인스턴스에서 assertion을 처리할 수 있습니다.

**권장 완화 조치:** 입력 `sessionId`가 현재 `DefaultSession` 인스턴스에 속하는 게임인지 검증하십시오.

**Majority Games:** 커밋 [462c01a](https://github.com/Engage-Protocol/engage-protocol/commit/462c01a157f287014e14585bbb4008379a3126c2)에서 수정됨.

**Cyfrin:** 확인함.


### `getReactionTime`은 사용자가 게임에 참여하지 않았더라도 `reactionDeadline`을 반환함

**설명:** 현재 구현은 미참여자에게도 0이 아니라 `reactionDeadline`을 반환합니다.

**영향:** 반응 시간을 조회하는 뷰 함수가 직관적이지 않은 값을 줄 수 있습니다.

**권장 완화 조치:** 미참여자에게는 0을 반환하거나 revert하는 방식을 고려하십시오.

**Majority Games:** 의도된 동작으로 판단되어, 커밋 [9cacc0c](https://github.com/Engage-Protocol/engage-protocol/commit/9cacc0c94b6d9e343177d820accf6ceee5d387e8)에서 NatSpec만 명확히 갱신됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `DepositManager::protocolFee`, `maxCreatorFee`를 `uint128`로 줄여 같은 storage slot에 packing할 수 있음

**설명:** 두 값 모두 `BASIS_POINTS` 범위를 넘지 않으므로 더 작은 타입으로 선언해 같은 슬롯에 넣을 수 있습니다.

**Majority Games:** 커밋 [adedfc2](https://github.com/Engage-Protocol/engage-protocol/commit/adedfc2224c118fd2ac88eeec826bd4be48d8b6e)에서 수정됨.

**Cyfrin:** 확인함.


### 동일한 storage read를 캐시하고, write는 한 번만 수행해야 함

**설명:** 반복되는 storage 접근을 캐시하면 gas를 줄일 수 있습니다.

**Majority Games:** 커밋 [4e56c11](https://github.com/Engage-Protocol/engage-protocol/commit/4e56c1123865865224b24583a6abadb4348fcc69)에서 수정됨.

**Cyfrin:** 확인함.


### 외부의 읽기 전용 함수 입력은 `memory`보다 `calldata`를 선호해야 함

**설명:** 읽기 전용 external 입력은 `calldata`가 더 저렴합니다.

**Majority Games:** 커밋 [be290a6](https://github.com/Engage-Protocol/engage-protocol/commit/be290a6eae3b11b32c40699af6b7d072bbcf85d3)에서 수정됨.

**Cyfrin:** 확인함.


### Solidity에서는 기본값으로 초기화하지 않는 편이 좋음

**설명:** `0`, `false` 같은 기본값 초기화는 불필요한 gas를 늘립니다.

**Majority Games:** 커밋 [6686df5](https://github.com/Engage-Protocol/engage-protocol/commit/6686df583945b33ab7ab2ad64e432c0395fabeb4)에서 수정됨.

**Cyfrin:** 확인함.


### storage를 읽기 전에 입력 관련 검사를 먼저 수행해야 함

**설명:** 실패 가능성이 있는 입력 검사를 앞에 두면 불필요한 storage read를 줄일 수 있습니다.

**Majority Games:** 커밋 [02b8fd8](https://github.com/Engage-Protocol/engage-protocol/commit/02b8fd81a5098d581332aecc00d28437e2dc631b)에서 수정됨.

**Cyfrin:** 확인함.


### 더 나은 storage packing으로 `SessionManager::joinGame`을 더 효율적으로 구현할 수 있음

**설명:** `joinGame`은 같은 필드를 여러 번 읽고 있어 packing과 캐싱 여지가 있습니다.

**Majority Games:** 커밋 [c7eafa2](https://github.com/Engage-Protocol/engage-protocol/commit/c7eafa2037270b0358e37c6f547950a37df01fe6)에서 수정됨.

**Cyfrin:** 확인함.


### 더 나은 storage packing을 위해 timestamp는 `uint32`를 사용할 수 있음

**설명:** 이 프로토콜 수명 범위에서는 `uint32` 타임스탬프로 충분합니다.

**Majority Games:** 커밋 [5902894](https://github.com/Engage-Protocol/engage-protocol/commit/5902894a3c21e684298b639307ec950bc34be74b)에서 수정됨.

**Cyfrin:** 확인함.


### `DefaultSession::assertionResolvedCallback`에서 전체 `Assertion` 구조체를 `storage`에서 `memory`로 복사하지 말아야 함

**설명:** 큰 구조체 전체 복사는 비효율적이므로 storage reference를 쓰는 편이 낫습니다.

**Majority Games:** 커밋 [fc5e0fa](https://github.com/Engage-Protocol/engage-protocol/commit/fc5e0faa81eae1edbef05d47c2e7652a236a7895)에서 수정됨.

**Cyfrin:** 확인함.
