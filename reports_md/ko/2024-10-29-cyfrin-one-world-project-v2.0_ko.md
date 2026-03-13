**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[Gio](https://twitter.com/giovannidisiena)

**Assisting Auditors**



---

# Findings
## Critical Risk


### `MembershipERC1155` profit tokens can be drained due to missing `lastProfit` synchronization when minting and claiming profit

**Description:** DAO 멤버가 [`MembershipERC1155:claimProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L138-L147)을 호출하면 청구된 보상을 추적하기 위해 [`lastProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L184) 매핑이 업데이트됩니다. 그러나 멤버십 토큰을 민팅(minting)/소각(burning)하거나 멤버십 토큰을 새 계정으로 전송할 때 이 상태가 동기화되지 않습니다.

따라서 민팅하거나 전송할 때, 새로운 사용자는 DAO 멤버가 되기 전의 이전 수익에 대한 지분을 받을 자격이 있는 것으로 간주됩니다. 새로운 DAO 멤버가 다른 기존 멤버의 비용으로 수익을 청구하는 명백한 경우 외에도, 이는 `MembershipERC1155Contract`의 수익 토큰 잔고가 고갈될 때까지 동일한 멤버십 토큰을 새 계정 간에 재활용하여 수익을 청구함으로써 무기화될 수 있습니다.

**Impact:** DAO 멤버는 자격이 없는 수익을 청구할 수 있으며 악의적인 사용자는 `MembershipERC1155` 컨트랙트의 모든 수익 토큰(동일한 통화로 지불된 경우 멤버십 수수료 포함)을 고갈시킬 수 있습니다.

**Proof of Concept:** 다음 테스트를 `MembershipERC1155.test.ts`의 `describe("Profit Sharing")`에 추가할 수 있습니다:
```javascript
it("lets users steal steal account balance by transferring tokens and claiming profit", async function () {
    await membershipERC1155.connect(deployer).mint(user.address, 1, 100);
    await membershipERC1155.connect(deployer).mint(anotherUser.address, 1, 100);
    await testERC20.mint(nonAdmin.address, ethers.utils.parseEther("20"));
    await testERC20.connect(nonAdmin).approve(membershipERC1155.address, ethers.utils.parseEther("20"));
    await membershipERC1155.connect(nonAdmin).sendProfit(testERC20.address, ethers.utils.parseEther("2"));
    const userProfit = await membershipERC1155.profitOf(user.address, testERC20.address);
    expect(userProfit).to.be.equal(ethers.utils.parseEther("1"));

    const beforeBalance = await testERC20.balanceOf(user.address);
    const initialContractBalance = await testERC20.balanceOf(membershipERC1155.address);

    // user claims profit
    await membershipERC1155.connect(user).claimProfit(testERC20.address);

    const afterBalance = await testERC20.balanceOf(user.address);
    const contractBalance = await testERC20.balanceOf(membershipERC1155.address);

    // users balance increased
    expect(afterBalance.sub(beforeBalance)).to.equal(userProfit);
    expect(contractBalance).to.equal(initialContractBalance.sub(userProfit));

    // user creates a second account and transfers their tokens to it
    const userSecondAccount = (await ethers.getSigners())[4];
    await membershipERC1155.connect(user).safeTransferFrom(user.address, userSecondAccount.address, 1, 100, '0x');
    const newProfit = await membershipERC1155.profitOf(userSecondAccount.address, testERC20.address);
    expect(newProfit).to.be.equal(userProfit);

    // second account can claim profit
    const newBeforeBalance = await testERC20.balanceOf(userSecondAccount.address);
    await membershipERC1155.connect(userSecondAccount).claimProfit(testERC20.address);
    const newAfterBalance = await testERC20.balanceOf(userSecondAccount.address);
    expect(newAfterBalance.sub(newBeforeBalance)).to.equal(newProfit);

    // contract balance has decreased with twice the profit
    const contractBalanceAfter = await testERC20.balanceOf(membershipERC1155.address);
    expect(contractBalanceAfter).to.equal(initialContractBalance.sub(userProfit.mul(2)));
    expect(contractBalanceAfter).to.equal(0);

    // no profit left for other users
    const anotherUserProfit = await membershipERC1155.profitOf(anotherUser.address, testERC20.address);
    expect(anotherUserProfit).to.be.equal(ethers.utils.parseEther("1"));
    await expect(membershipERC1155.connect(anotherUser).claimProfit(testERC20.address)).to.be.revertedWith("ERC20: transfer amount exceeds balance");
});

it("lets users steal steal account balance by minting after profit is sent", async function () {
    await membershipERC1155.connect(deployer).mint(user.address, 1, 100);
    await membershipERC1155.connect(deployer).mint(anotherUser.address, 1, 100);
    await testERC20.mint(nonAdmin.address, ethers.utils.parseEther("20"));
    await testERC20.connect(nonAdmin).approve(membershipERC1155.address, ethers.utils.parseEther("20"));
    await membershipERC1155.connect(nonAdmin).sendProfit(testERC20.address, ethers.utils.parseEther("2"));
    const userProfit = await membershipERC1155.profitOf(user.address, testERC20.address);
    expect(userProfit).to.be.equal(ethers.utils.parseEther("1"));

    const beforeBalance = await testERC20.balanceOf(user.address);
    const initialContractBalance = await testERC20.balanceOf(membershipERC1155.address);

    // user claims profit
    await membershipERC1155.connect(user).claimProfit(testERC20.address);

    const afterBalance = await testERC20.balanceOf(user.address);
    const contractBalance = await testERC20.balanceOf(membershipERC1155.address);

    // users balance increased
    expect(afterBalance.sub(beforeBalance)).to.equal(userProfit);
    expect(contractBalance).to.equal(initialContractBalance.sub(userProfit));

    // new user mints a token after profit and can claim first users profit
    const newUser = (await ethers.getSigners())[4];
    await membershipERC1155.connect(deployer).mint(newUser.address, 1, 100);
    const newProfit = await membershipERC1155.profitOf(newUser.address, testERC20.address);
    expect(newProfit).to.be.equal(ethers.utils.parseEther("1"));

    // new user can claim profit
    const newBeforeBalance = await testERC20.balanceOf(newUser.address);
    await membershipERC1155.connect(newUser).claimProfit(testERC20.address);
    const newAfterBalance = await testERC20.balanceOf(newUser.address);
    expect(newAfterBalance.sub(newBeforeBalance)).to.equal(newProfit);

    // contract balance has decreased with twice the profit
    const contractBalanceAfter = await testERC20.balanceOf(membershipERC1155.address);
    expect(contractBalanceAfter).to.equal(initialContractBalance.sub(userProfit.mul(2)));
    expect(contractBalanceAfter).to.equal(0);

    // no profit left for first users
    const anotherUserProfit = await membershipERC1155.profitOf(anotherUser.address, testERC20.address);
    expect(anotherUserProfit).to.be.equal(ethers.utils.parseEther("1"));
    await expect(membershipERC1155.connect(anotherUser).claimProfit(testERC20.address)).to.be.revertedWith("ERC20: transfer amount exceeds balance");
});
```

**Recommended Mitigation:** `ERC1155::_beforeTokenTransfer`를 재정의하여 관련 작업이 수행될 때마다 수익 상태의 스냅샷을 찍는 것을 고려하십시오.

**One World Project:** 코드 구조 업데이트, 불필요한 코드 제거. [`a3980c1`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/a3980c17217a0b65ecbd28eb078d4d94b4bd5b80) 및 [`a836386`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/a836386bd48691078435d10df5671e3c25f23719)에서 토큰 전송 시 보상 업데이트됨.

**Cyfrin:** 확인됨. 보상은 이제 토큰의 모든 이동에 적용되는 `ERC1155Upgradeable::_update`에서 업데이트됩니다.


### DAO creator can inflate their privileges to mint/burn membership tokens, steal profits, and abuse approvals to `MembershipERC1155`

**Description:** [새로운 DAO 생성](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L66-L70) 중에 `MembershipFactory` 컨트랙트는 토큰을 [민팅](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L52-L59)/[소각](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L61-L67)하고 [`MembershipERC1155::callExternalContract`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L202-L210)를 통해 임의의 호출을 실행할 수 있는 특별한 권한이 있는 `OWP_FACTORY_ROLE`을 [부여](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L49)받습니다. 또한 호출하는 계정은 `DEFAULT_ADMIN_ROLE`을 [부여](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L48)받지만, [문서](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/access/AccessControl.sol#L40-L48)에 따르면 이는 다른 모든 역할을 관리할 수 있는 권한도 부여합니다.

이것은 주어진 DAO의 생성자가 `AccessControl::grantRole`을 호출하여 자신에게 `OWP_FACTORY_ROLE`을 부여할 수 있음을 의미하며 다음과 같은 여러 가지 함의가 있습니다:
- [`MembershipERC1155:sendProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L189-L200) 호출자로부터 선행 매매(front-running) 및/또는 매달린 승인(dangling approvals)을 악용하여 수익 토큰을 훔칠 수 있습니다.
- DAO 생성자는 DAO 및 멤버십 토큰에 대한 일방적인 통제권을 가지므로 모든 주소로/주소로부터 민팅/소각할 수 있습니다.
- `MembershipERC1155::sendProfit` 호출을 선행 매매하여 [`MembershipERC1155::burnBatchMultiple`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L85-L99) 호출로 멤버십 토큰의 총 공급량을 0으로 만들어 [이 조건부 블록](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L198-L200)이 실행되도록 함으로써 DAO로부터 수익을 훔칠 수 있습니다. 또는 호출이 실행될 때까지 기다렸다가 임의의 외부 호출을 사용하여 토큰을 직접 전송할 수 있습니다.

```solidity
if (_totalSupply > 0) {
    totalProfit[currency] += (amount * ACCURACY) / _totalSupply;
    IERC20(currency).safeTransferFrom(msg.sender, address(this), amount);
    emit Profit(amount);
} else {
    IERC20(currency).safeTransferFrom(msg.sender, creator, amount); // Redirect profit to creator if no supply
}
```

이 문제는 [`MembershipFactory::callExternalContract`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L155-L163)를 통해 `MembershipFactory` 컨트랙트와 모든 DAO를 제어하는 One World Project 소유자의 중앙 집중화 위험으로도 존재한다는 점에 유의해야 합니다. 이는 별도의 발견 사항으로 자세히 설명되어 있습니다.

**Impact:** DAO 생성자는 권한을 상승시켜 일방적인 통제권을 갖고 멤버로부터 수익을 훔칠 수 있으며, 컨트랙트에 대한 모든 수익 토큰 승인을 악용할 수 있습니다. 위의 모든 사항은 팩토리와 이에 의해 생성된 모든 DAO를 제어하는 One World Project 소유자에게도 가능합니다.

**Proof of Concept:** 다음 테스트를 `MembershipERC1155.test.ts`의 `describe("ERC1155 and AccessControl Interface Support")`에 추가할 수 있습니다:
```javascript
it("can give OWP_FACTORY_ROLE to an address and abuse priviliges", async function () {
    const [factory, creator, user] = await ethers.getSigners();
    const membership = await MembershipERC1155.connect(factory).deploy();
    await membership.deployed();
    await membership.initialize("TestToken", "TST", tokenURI, creator.address);

    await membership.connect(creator).grantRole(await membership.OWP_FACTORY_ROLE(), creator.address);
    expect(await membership.hasRole(await membership.OWP_FACTORY_ROLE(), creator.address)).to.be.true;

    // creator can mint and burn at will
    await membership.connect(creator).mint(user.address, 1, 100);
    await membership.connect(creator).burn(user.address, 1, 50);

    await testERC20.mint(user.address, ethers.utils.parseEther("1"));
    await testERC20.connect(user).approve(membership.address, ethers.utils.parseEther("1"));

    const creatorBalanceBefore = await testERC20.balanceOf(creator.address);

    // creator can abuse approvals
    const data = testERC20.interface.encodeFunctionData("transferFrom", [user.address, creator.address, ethers.utils.parseEther("1")]);
    await membership.connect(creator).callExternalContract(testERC20.address, data);

    const creatorBalanceAfter = await testERC20.balanceOf(creator.address);
    expect(creatorBalanceAfter.sub(creatorBalanceBefore)).to.equal(ethers.utils.parseEther("1"));
});
```

**Recommended Mitigation:** `DEFAULT_ADMIN_ROLE`을 부여하는 대신 DAO 생성자에 대해 더 세분화된 액세스 제어를 구현하십시오.

**One World Project:** [`a6b9d82`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/a6b9d82796c2d87a3924e8e80c3732474bf22506)에서 생성자에게 별도의 역할을 부여했습니다.

**Cyfrin:** 확인됨. `creator`는 이제 URI만 변경할 수 있는 별도의 역할 `DAO_CREATOR`를 갖습니다.

\clearpage
## High Risk


### `MembershipERC1155::sendProfit` can be front-run by calls to `MembershipFactory::joinDAO` to steal profit from existing DAO members

**Description:** [`MembershipERC1155::sendProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L189-L201) 호출 후 수익이 DAO 멤버에게 분배되며, 이는 [`totalProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L195)에서 추적되는 주당 수익을 증가시킵니다. DAO 가입 시 어떠한 수익 공유 지연도 없기 때문에, 충분한 재정적 동기가 있는 다른 사용자가 이 트랜잭션을 보고 실행되기 전에 DAO의 많은 지분을 매입할 수 있습니다. 이를 통해 기존 DAO 멤버의 비용으로 새로 추가된 수익에 대한 청구권을 갖게 됩니다.

**Impact:** `MembershipERC1155::sendProfit` 호출이 선행 매매(front-run)될 수 있어 기존 DAO 멤버에게 지급되는 수익이 불공정하게 감소합니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Join DAO")`에 추가할 수 있습니다:
```javascript
it("lets users front-run profit distribution", async function () {
  const tierIndex = 0;
  await testERC20.mint(addr1.address, ethers.utils.parseEther("1"));
  await testERC20.connect(addr1).approve(membershipFactory.address, TierConfig[tierIndex].price);
  await testERC20.mint(addr2.address, ethers.utils.parseEther("1"));
  await testERC20.connect(addr2).approve(membershipFactory.address, ethers.utils.parseEther("1"));
  await testERC20.mint(owner.address, ethers.utils.parseEther("1"));
  await testERC20.connect(owner).approve(membershipERC1155.address, ethers.utils.parseEther("1"));
  // user1 joins
  await membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, tierIndex);

  // time passes

  // user2 sees a pending sendProfit tx and front-runs it by buying a lot of membership tokens
  // this can be done with a deployed contract
  for(let i = 0; i < 9; i++) {
    await membershipFactory.connect(addr2).joinDAO(membershipERC1155.address, tierIndex);
  }

  // send profit tx is executed
  await membershipERC1155.sendProfit(testERC20.address, ethers.utils.parseEther("1"));

  const addr1Profit = await membershipERC1155.profitOf(addr1.address, testERC20.address);
  const addr2Profit = await membershipERC1155.profitOf(addr2.address, testERC20.address);

  // user2 has gotten 9x the profit of user1
  expect(addr1Profit).to.equal(ethers.utils.parseEther("0.1"));
  expect(addr2Profit).to.equal(ethers.utils.parseEther("0.9"));
});
```

**Recommended Mitigation:** 수익 공유가 활성화되기 전에 멤버십 지연을 구현하는 것을 고려하십시오.

**One World Project:** 멤버십은 구매해야 하며, 사용자가 sendProfit 함수를 선행 매매하기 위해 상당한 수의 주식을 획득하려는 경우, 얻을 수 있는 수익보다 훨씬 더 많은 금액을 지출해야 합니다.

**Cyfrin:** 인지됨. 그러나 One World Project는 수익 분배나 사용자 참여 시기를 통제하지 않으므로, 재정적 인센티브가 멤버십 가입 비용을 초과하는 시나리오를 방지하는 제한을 강제할 수 없습니다. 수익 분배가 상당한 경우, 의도하지 않았더라도 참가자에게 재정적으로 실행 가능한 상황이 될 수 있습니다. 프로토콜은 이러한 변수를 제어할 수 없으므로 DAO가 의도치 않게 이러한 시나리오를 만드는 것을 막을 수 없습니다. 따라서 새로운 DAO 온보딩 중 문서에서 이러한 잠재적 위험을 명확하게 전달할 것을 권장합니다.


### One World Project has unilateral control over all DAOs, allowing the owner to update tier configurations, mint/burn membership tokens, steal profits, and abuse token approvals to `MembershipFactory` and `MembershipERC1155` proxy contracts

**Description:** `MembershipFactory` 컨트랙트가 배포될 때 호출자에게 `EXTERNAL_CALLER` 역할이 부여됩니다. 이를 통해 One World Project는 [`MembershipFactory::updateDAOMembership`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L90-L117)를 통해 특정 DAO의 티어 구성을 업데이트하고 [`MembershipFactory::callExternalContract`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L155-L163)를 통해 임의의 호출을 실행할 수 있습니다. 또한 [새로운 DAO 생성](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L66-L70) 중에 `MembershipFactory` 컨트랙트는 토큰을 [민팅](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L52-L59)/[소각](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L61-L67)하고 [`MembershipERC1155::callExternalContract`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L202-L210)를 통해 임의의 호출을 실행할 수 있는 특별한 권한이 있는 `OWP_FACTORY_ROLE`을 [부여](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L49)받습니다.

DAO 티어 구성에 대한 일방적인 통제권만으로도 주의가 필요하지만, 대상 함수 선택자 및 호출할 컨트랙트에 대한 제한 없이 `MembershipFactory::callExternalContract` 및 `MembershipERC1155::callExternalContract` 호출을 연결하는 것은 매우 위험합니다. 결과적으로 다른 권한 상승 취약점과 유사하게, One World Project 소유자는 모든 DAO에 대해 멤버십 토큰을 임의로 민팅/소각하고, 수익을 훔치고, `MembershipERC1155` 프록시 컨트랙트에 대한 승인을 악용할 수 있는 능력을 갖게 됩니다. 또한 `MembershipFactory::callExternalContract`는 선행 매매 등을 통해 이 컨트랙트에 직접 부여된 승인을 악용하는 데 사용될 수 있습니다. 사용자가 DAO 가입 시 최대 `uint256` 허용량을 설정하면 One World Project 소유자는 주어진 통화에 대한 전체 토큰 잔고를 빼낼 수 있습니다.

**Impact:** One World Project 소유자는 `MembershipFactory` 컨트랙트 및 이에 의해 생성된 모든 DAO에 대한 일방적인 통제권을 가지므로 멤버로부터 수익을 훔칠 수 있고 프록시 컨트랙트에 대한 수익 토큰 승인을 악용할 수 있습니다. One World Project 소유자는 또한 `MembershipFactory` 컨트랙트에 대해 매달린 승인(dangling approvals)이 있는 모든 토큰의 잔고를 빼낼 수 있습니다. 이는 소유자 주소가 어떤 식으로든 손상되는 경우 특히 문제가 됩니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Call External Contract")`에 추가할 수 있습니다:
```javascript
it("allows admin to have unilateral power", async function() {
  await testERC20.mint(addr1.address, ethers.utils.parseEther("2"));
  await testERC20.connect(addr1).approve(membershipFactory.address, ethers.utils.parseEther("1"));

  await currencyManager.addCurrency(testERC20.address);  // Assume addCurrency function exists in CurrencyManager
  const tx = await membershipFactory.createNewDAOMembership(DAOConfig, TierConfig);
  const receipt = await tx.wait();
  const event = receipt.events.find((event:any) => event.event === "MembershipDAONFTCreated");
  const nftAddress = event.args[1];
  const membershipERC1155 = await MembershipERC1155.attach(nftAddress);

  let ownerBalanceBefore = await testERC20.balanceOf(owner.address);

  // admin can steal approvals made to factory
  const transferData = testERC20.interface.encodeFunctionData("transferFrom", [addr1.address, owner.address, ethers.utils.parseEther("1")]);
  await membershipFactory.callExternalContract(testERC20.address, transferData);

  let ownerBalanceAfter = await testERC20.balanceOf(owner.address);
  expect(ownerBalanceAfter.sub(ownerBalanceBefore)).to.equal(ethers.utils.parseEther("1"));

  // admin can mint/burn any DAO membership tokens
  const mintData = membershipERC1155.interface.encodeFunctionData("mint", [owner.address, 1, 100]);
  await membershipFactory.callExternalContract(nftAddress, mintData);

  let ownerBalanceERC1155 = await membershipERC1155.balanceOf(owner.address, 1);
  expect(ownerBalanceERC1155).to.equal(100);

  const burnData = membershipERC1155.interface.encodeFunctionData("burn", [owner.address, 1, 50]);
  await membershipFactory.callExternalContract(nftAddress, burnData);

  ownerBalanceERC1155 = await membershipERC1155.balanceOf(owner.address, 1);
  expect(ownerBalanceERC1155).to.equal(50);

  // admin can abuse approvals to any membership tokens as well
  await testERC20.connect(addr1).approve(membershipERC1155.address, ethers.utils.parseEther("1"));

  ownerBalanceBefore = await testERC20.balanceOf(owner.address);

  const data = membershipERC1155.interface.encodeFunctionData("callExternalContract", [testERC20.address, transferData]);
  await membershipFactory.callExternalContract(membershipERC1155.address, data);

  ownerBalanceAfter = await testERC20.balanceOf(owner.address);
  expect(ownerBalanceAfter.sub(ownerBalanceBefore)).to.equal(ethers.utils.parseEther("1"));
});
```

**Recommended Mitigation:** `MembershipFactory` 컨트랙트 소유권의 남용을 방지하기 위해 임의의 외부 호출에 의해 호출될 대상 컨트랙트 및 함수 선택자에 대한 제한을 구현하십시오.

**One World Project:** `EXTERNAL_CALLER` 지갑은 백엔드의 AWS Secrets Manager에 안전하게 저장되며 개인에게 액세스 권한이 부여되지 않습니다. 이 지갑은 오프체인 프로세스를 위한 온체인 트랜잭션을 실행하는 데 필요합니다. 또한 실행 가능한 함수는 특정 함수 서명으로 정의되지 않습니다. 향후 이 컨트랙트가 프로젝트에 자금을 분배하거나 오프체인 승인을 통해 실행함으로써 DAO를 통해 다른 작업을 수행하기 위해 컨트랙트와 상호 작용해야 할 수 있기 때문입니다.

**Cyfrin:** 인지됨. AWS Secrets Manager가 보안을 강화하지만 개인 키 또는 API 키 유출은 여전히 위험 요소로 남아 있습니다.

\clearpage
## Medium Risk


### DAO name can be stolen by front-running calls to `MembershipFactory::createNewDAOMembership`

**Description:** `MembershipFactory::createNewDAOMembership`이 호출되면 새로 생성된 `MembershipERC1155` 인스턴스는 이름 `ensname`과 [연결](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L61)됩니다:

```solidity
require(getENSAddress[daoConfig.ensname] == address(0), "DAO already exist.");
```

그러나 이 호출은 다른 생성자가 One World Project 멤버십 토큰을 설정하고 있음을 보고 그들보다 먼저 동일한 이름을 등록하여 이름을 "훔치는" 악의적인 사용자에 의해 선행 매매될 수 있습니다.

**Impact:** 누구나 DAO 멤버십 생성을 선행 매매할 수 있습니다. 이는 허니팟을 만들거나 단순히 DAO 생성자를 괴롭히는 데 사용될 수 있습니다.

**Recommended Mitigation:** DAO 생성자가 해당 ENS 이름과 연결되어 있는지 검증하는 것을 고려하십시오. 또는 이름을 임의의 문자열로 허용하고 생성자와 이름의 연결을 키로 사용하십시오.

**One World Project:** DAO 이름은 반드시 ENS 이름일 필요는 없으며 어떤 문자열이든 될 수 있습니다. 이름을 사용할 수 없는 경우 DAO 생성자는 프론트엔드 웹사이트에서 미리 인지하게 되며 다른 이름이나 해당 이름의 변형을 자유롭게 선택할 수 있습니다. 이름은 dao 생성자가 id를 기억할 필요 없이 dao를 쉽게 식별/기억할 수 있도록 문자열 형식으로 유지됩니다.

누군가 당신보다 먼저 그 이름으로 DAO를 생성할 수 있다면 허용되며, 사용자는 DAO에 대해 다른 이름이나 변형을 선택해야 합니다. DAO 이름을, 원하는 대로 결정하는 것은 전적으로 DAO 생성자에게 달려 있습니다.

**Cyfrin:** 인지됨.


### DAO membership fees cannot be retrieved by the creator

**Description:** [`MembershipFactory::joinDAO`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L120-L133)를 호출하는 사용자로부터 받는 DAO 멤버십 수수료는 One World Project와 DAO 생성자 간에 분할되어 각각 One World Project 지갑과 DAO `MembershipERC1155` 인스턴스로 전송됩니다:

```solidity
uint256 tierPrice = daos[daoMembershipAddress].tiers[tierIndex].price;
uint256 platformFees = (20 * tierPrice) / 100;
daos[daoMembershipAddress].tiers[tierIndex].minted += 1;
IERC20(daos[daoMembershipAddress].currency).transferFrom(msg.sender, owpWallet, platformFees);
IERC20(daos[daoMembershipAddress].currency).transferFrom(msg.sender, daoMembershipAddress, tierPrice - platformFees);
```

그러나 `daoMembershipAddress`로 [전송된](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L130) 수수료는 직접 회수할 수 있는 방법이 없기 때문에 DAO 생성자가 접근할 수 없습니다. 이 자금을 회수하여 생성자에게 전송할 수 있는 유일한 방법은 `MembershipFactory::EXTERNAL_CALLER` 역할이 [`MembershipFactory::callExternalContract`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L155-L163)를 통해 [`MembershipERC1155::callExternalContract`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L202-L210)를 호출하여 임의의 외부 호출을 실행하도록 허용하는 경우입니다.

**Impact:** DAO 생성자는 `EXTERNAL_CALLER` 역할에 의해 시작된 구조를 무시하고 `MembershipERC1155` 인스턴스에 지불된 멤버십 수수료를 회수할 직접적인 방법이 없습니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Create New DAO Membership")`에 추가할 수 있습니다:
```javascript
it("only allows owner to recover dao membership fees", async function () {
  await currencyManager.addCurrency(testERC20.address);
  const creator = addr1;

  await membershipFactory.connect(creator).createNewDAOMembership(DAOConfig, TierConfig);

  const ensAddress = await membershipFactory.getENSAddress("testdao.eth");
  const membershipERC1155 = await MembershipERC1155.attach(ensAddress);

  await testERC20.mint(addr2.address, ethers.utils.parseEther("20"));
  await testERC20.connect(addr2).approve(membershipFactory.address, ethers.utils.parseEther("20"));
  await expect(membershipFactory.connect(addr2).joinDAO(membershipERC1155.address, 1)).to.not.be.reverted;

  // fees are in the membership token but cannot be retrieved by the creator
  const daoMembershipBalance = await testERC20.balanceOf(membershipERC1155.address);
  expect(daoMembershipBalance).to.equal(160); // minus protocol fee

  const creatorBalanceBefore = await testERC20.balanceOf(creator.address);

  // only admin can recover them
  const transferData = testERC20.interface.encodeFunctionData("transfer", [creator.address, 160]);
  const data = membershipERC1155.interface.encodeFunctionData("callExternalContract", [testERC20.address, transferData]);
  await membershipFactory.callExternalContract(membershipERC1155.address, data);

  const creatorBalanceAfter = await testERC20.balanceOf(creator.address);
  expect(creatorBalanceAfter.sub(creatorBalanceBefore)).to.equal(160);
});
```

**Recommended Mitigation:** DAO 가입 시 사용자가 지불한 멤버십 수수료를 DAO 생성자가 회수할 수 있는 메서드를 추가하는 것을 고려하십시오.

**One World Project:** DAO 생성자는 의도적으로, 설계에 따라 DAO 자금에 접근할 수 없습니다. 자체 검증을 수행하는 `EXTERNAL_CONTRACT`에 의해서만 호출될 수 있는 `callExternalContract`를 통해서만 접근해야 합니다.

**Cyfrin:** 인지됨. 이 의존성은 추가적인 위험을 초래하므로 오프체인 서비스가 엄격한 보안 표준을 충족하는지 확인할 것을 권장합니다.


### Meta transactions do not work with most of the calls in `MembershipFactory`

**Description:** `MembershipFactory`는 중계자가 사용자를 대신하여 트랜잭션 수수료를 지불할 수 있도록 하는 `NativeMetaTransaction`을 상속하여 사용자 지정 메타 트랜잭션 구현을 사용합니다. 이는 ERC2771과 동일한 표준을 따르며, 사용자는 메타 트랜잭션에 서명하고 중계자가 이를 전달하며 사용자의 주소가 `msg.data`에 추가되어 실행됩니다.

따라서 `msg.sender`는 [`NativeMetaTransaction::executeMetaTransaction`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/NativeMetaTransaction.sol#L33)이 호출되는 경우 중계자가 되므로 실제 트랜잭션 전송자를 검색하는 데 사용할 수 없습니다. [여기](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L165-L185)에 이미 구현된 바와 같이, 해결책은 이러한 경우 `msg.data`의 마지막 20바이트에서 서명 사용자를 검색하는 `_msgSender()` 함수를 활용하는 것입니다.

이러한 이유로 `MembershipFactory`의 다음 함수들에 문제가 있습니다:
* `MembershipFactory::createNewDAOMembership` [[1](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L69), [2](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L84)].
* `MembershipFactory::joinDAO` [[1](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L129), [2](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L130), [3](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L131), [4](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L132)].
* `MembershipFactory::upgradeTier` [[1](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L141), [2](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L142), [3](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L143)].

**Impact:** 위의 호출 중 어느 것도 `NativeMetaTransaction::executeMetaTransaction`을 통해 시작될 때 올바르게 작동하지 않습니다. 가장 문제가 되는 것은 `MembershipFactory::createNewDAOMembership`인데, `creator`로 `MembershipFactory` 컨트랙트 주소를 사용하여 DAO 멤버십 토큰을 생성하기 때문입니다. `MembershipFactory::joinDAO` 및 `MembershipFactory::upgradeTier`는 `msg.sender`(`MembershipFactory`)가 `MembershipERC1155` 토큰이나 지불 `ERC20` 토큰을 보유해야 하므로(보유하지 않아야 함) 대부분 되돌아갈 것입니다.

**Proof of Concept:** `MembershipFactory.test.ts`에 추가할 수 있는 테스트:
```javascript
describe("Native meta transaction", function () {
  it("Meta transactions causes creation to use the wrong owner", async function () {
    await currencyManager.addCurrency(testERC20.address);

    const { chainId } = await ethers.provider.getNetwork();
    const salt = ethers.utils.hexZeroPad(ethers.utils.hexlify(chainId), 32)

    const domain = {
      name: 'OWP',
      version: '1',
      salt: salt,
      verifyingContract: membershipFactory.address,
    };
    const types = {
      MetaTransaction: [
        { name: 'nonce', type: 'uint256' },
        { name: 'from', type: 'address' },
        { name: 'functionSignature', type: 'bytes' },
      ],
    };
    const nonce = await membershipFactory.getNonce(addr1.address);
    const metaTransaction = {
      nonce,
      from: addr1.address,
      functionSignature: membershipFactory.interface.encodeFunctionData('createNewDAOMembership', [DAOConfig, TierConfig]),
    };
    const signature = await addr1._signTypedData(domain, types, metaTransaction);
    const {v,r,s} = ethers.utils.splitSignature(signature);

    const tx = await membershipFactory.executeMetaTransaction(metaTransaction.from, metaTransaction.functionSignature, r, s, v);
    const receipt = await tx.wait();
    const event = receipt.events.find((event:any) => event.event === "MembershipDAONFTCreated");
    const nftAddress = event.args[1];
    const creator = await MembershipERC1155.attach(nftAddress).creator();

    // creator becomes the membership factory not addr1
    expect(creator).to.equal(membershipFactory.address);
  });
});
```

**Recommended Mitigation:** 위에서 언급한 함수에서 `msg.sender` 대신 `_msgSender()`를 사용하는 것을 고려하십시오.

**One World Project:** MetaTransaction의 유일한 의도된 용도는 callExternalContract 함수를 호출하는 것입니다. 현재 구현은 `EXTERNAL_CALLER`가 백엔드에서 트랜잭션에 서명하고 서명된 객체를 사용자에게 보내면 사용자가 `executeMetaTransaction()` 함수를 통해 컨트랙트로 보내는 것입니다. 이렇게 하면 OWP 플랫폼은 관리자 트랜잭션에 대한 가스비를 지불할 필요가 없습니다. `_msgSender()`는 커밋 해시 [`83ba905`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/83ba905f581be57a56d521deff6d75e0837b2237)에서 추가되었습니다.

**Cyfrin:** 확인됨. `_msgSender()`는 이제 컨트랙트 전체에서 사용됩니다.


### Tier restrictions for `SPONSORED` DAOs can be bypassed by calling `MembershipFactory::upgradeTier`

**Description:** `MembershipFactory::upgradeTier` 호출의 `daoMembershipAddress` 매개변수에 지정된 DAO가 [`SPONSORED`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L139)로 등록된 경우, 멤버는 두 개의 하위 티어 토큰을 소각하여 하나의 상위 티어 토큰으로 티어를 업그레이드할 수 있습니다. 그러나 [`MembershipDAOStructs::DAOConfig`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L16)의 [`tiers.minted`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L35) 멤버는 구성된 [`tiers.amount`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L32)에 대해 업데이트되거나 검증되지 않습니다. 즉, DAO 멤버는 하위 티어 토큰을 민팅하고 업그레이드하여 의도한 것보다 더 많은 상위 티어 토큰을 민팅할 수 있습니다.

**Impact:** 주어진 티어에 대한 최대 멤버십 수는 하위 티어를 업그레이드함으로써 우회될 수 있습니다. 또한 `tiers.minted`가 원래 티어와 업그레이드된 티어에 대해 각각 감소/증가하지 않으므로 하위 티어에 대해 새로운 토큰을 민팅할 수 없습니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Upgrade Tier")`에 추가할 수 있습니다:

```javascript
it("can upgrade above max amount and minted not updated", async function () {
  const lowTier = 5;
  const highTier = 4;
  await testERC20.mint(addr1.address, ethers.utils.parseEther("1000000"));
  await testERC20.connect(addr1).approve(membershipFactory.address, ethers.utils.parseEther("1000000"));
  for(let i = 0; i < 40; i++) {
    await membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, highTier);
  }
  // cannot join anymore
  await expect(membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, highTier)).to.be.revertedWith("Tier full.");

  await membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, lowTier);
  await membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, lowTier);

  const tiersBefore = await membershipFactory.daoTiers(membershipERC1155.address);
  expect(tiersBefore[lowTier].minted).to.equal(2);
  expect(tiersBefore[highTier].minted).to.equal(40);

  // but can upgrade tier
  await membershipFactory.connect(addr1).upgradeTier(membershipERC1155.address, lowTier);

  // a total of 41 tokens for tier 4, max amount is 40
  const numberOfTokens = await membershipERC1155.balanceOf(addr1.address, highTier);
  expect(numberOfTokens).to.equal(41);

  // and minted hasn't changed
  const tiersAfter = await membershipFactory.daoTiers(membershipERC1155.address);
  expect(tiersAfter[lowTier].minted).to.equal(tiersBefore[lowTier].minted);
  expect(tiersAfter[highTier].minted).to.equal(tiersBefore[highTier].minted);
});
```

**Recommended Mitigation:** `tiers.minted` 멤버는 원래 티어에 대해 감소하고 업그레이드된 티어에 대해 증가해야 하며, `tier.amount`가 초과되지 않았는지 검증해야 합니다.

**One World Project:** 이것은 비즈니스 로직 요구 사항입니다. 티어가 꽉 찬 후에도 업그레이드를 허용해야 합니다. 따라서 총 민팅된 수는 민팅된 수를 유지하지만, 업그레이드된 멤버는 그 이상이 될 것입니다.

**Cyfrin:** 인지됨.


### No membership restrictions placed on `PRIVATE` DAOs allows anyone to join

**Description:** [`MembershipDAOStructs::DAOType`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L6-L10)은 DAO가 가질 수 있는 다양한 유형, 즉 `PRIVATE`, `SPONSORED` 및 제한이 없는 기본 `PUBLIC`을 노출합니다. `SPONSORED` 유형의 DAO는 개방되어 있지만 모든 티어를 사용해야 하며, `PRIVATE`는 멤버십에 추가 제한을 부과할 것으로 예상될 수 있지만, 이 경우는 처리되지 않아 누구나 이러한 DAO에 가입할 수 있습니다.

**Impact:** DAO 생성자가 `DAOType.PRIVATE`를 지정하더라도 가입이 허용된 계정에 제한을 둘 수 있는 가능성은 없습니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Create New DAO Membership")`에 추가할 수 있습니다:
```javascript
it("lets anyone join PRIVATE DAOs", async function () {
  await currencyManager.addCurrency(testERC20.address);

  // DAO membership is private
  DAOConfig.daoType = DAOType.PRIVATE;
  await membershipFactory.createNewDAOMembership(DAOConfig, TierConfig);

  const ensAddress = await membershipFactory.getENSAddress("testdao.eth");
  const membershipERC1155 = await MembershipERC1155.attach(ensAddress);

  await testERC20.mint(addr1.address, ethers.utils.parseEther("20"));
  await testERC20.connect(addr1).approve(membershipFactory.address, ethers.utils.parseEther("20"));

  // but anyone can join
  await expect(membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, 1)).to.not.be.reverted;
});
```

**Recommended Mitigation:** `PRIVATE` DAO의 생성자가 멤버십 제한을 적용하는 데 사용할 수 있는 허용 목록(allowlist) 옵션 또는 이와 유사한 기능을 구현하는 것을 고려하십시오.

**One World Project:** 스마트 컨트랙트에서 `PRIVATE` DAO에 가입하는 것을 막을 의도는 없습니다. 이는 단지 웹사이트에서 대중에게 보이지 않게 하기 위한 것입니다.

**Cyfrin:** 인지됨.


### DAO membership can exceed `MembershipDAOStructs::DAOConfig.maxMembers`

**Description:** [`MembershipDAOStructs::DAOConfig.maxMembers`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L18) 필드는 DAO 멤버십의 한도(cap)로 의도되었으며 초과해서는 안 되지만, 현재 사용되지 않고 있으며 각 해당 티어에 대한 한도 외에는 DAO에 가입할 수 있는 멤버 수에 제한이 없습니다.

**Impact:** 각 티어의 최대 수량에 의해서만 제한되며, 원하는 수의 멤버가 DAO에 가입할 수 있습니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Create New DAO Membership")`에 추가할 수 있습니다:
```javascript
it("can exceed maxMembers", async function () {
  // max members is 1
  DAOConfig.maxMembers = 1;
  await currencyManager.addCurrency(testERC20.address);
  await membershipFactory.createNewDAOMembership(DAOConfig, TierConfig);

  const ensAddress = await membershipFactory.getENSAddress("testdao.eth");
  const membershipERC1155 = await MembershipERC1155.attach(ensAddress);

  await testERC20.mint(addr1.address, ethers.utils.parseEther("20"));
  await testERC20.connect(addr1).approve(membershipFactory.address, ethers.utils.parseEther("20"));
  await testERC20.mint(addr2.address, ethers.utils.parseEther("20"));
  await testERC20.connect(addr2).approve(membershipFactory.address, ethers.utils.parseEther("20"));

  // two members can join
  await expect(membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, 1)).to.not.be.reverted;
  await expect(membershipFactory.connect(addr2).joinDAO(membershipERC1155.address, 1)).to.not.be.reverted;
});
```

**Recommended Mitigation:** DAO에 가입한 멤버 수를 검증하고 `maxMembers`를 초과하지 않도록 강제하는 것을 고려하십시오.

**One World Project:** maxMembers는 백엔드에서의 데이터 검증만을 위한 것입니다. 새로운 데이터에 따라 값을 업데이트했습니다. [`e60b078`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/e60b078f09d4ed0f1e509f36a2a6d42293815737) 및 [`510f305`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/510f305e24a89e0815934ab257a413b9e835607f)에서 수정됨.

**Cyfrin:** 확인됨. `tier.amount`의 합계는 `maxMembers`를 초과할 수 없으며 `tier.amount`는 가입 시 검증됩니다.


### Lowest tier (highest index) membership cannot be upgraded

**Description:** `SPONSORED` DAO의 경우, 멤버는 [`MembershipFactory::upgradeTier`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L135-L144) 호출 내에서 두 개의 토큰을 소각하여 하위 티어 멤버십에서 상위 티어 멤버십으로 업그레이드할 수 있습니다. 이 로직은 현재 티어가 업그레이드 가능한지 [검증](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L140)하려고 시도합니다:

```solidity
require(daos[daoMembershipAddress].noOfTiers > fromTierIndex + 1, "No higher tier available.");
```

그러나 여기서 주의해야 할 중요한 세부 사항은 `MembershipFactory::upgradeTier` 내에서 [참조](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L140-L142)될 때 가장 높은 티어 멤버십이 가장 낮은 티어 인덱스를 갖는다는 것입니다. 따라서 가장 높은 티어는 `0`으로 표시되고 가장 높은 인덱스를 가진 가장 낮은 티어는 `6`으로 표시됩니다. 즉, 위의 검증은 1만큼 차이가 납니다(off-by-one). `7 > 6 + 1`은 `false`이므로 가장 낮은 티어(가장 높은 인덱스) 멤버십에서 업그레이드할 수 없습니다. 또한 가장 높은 티어(가장 낮은 인덱스)에서의 업그레이드 시도는 민팅 시도 시 [언더플로로 인한 되돌리기(revert)](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L142)로 인해 실패한다는 점에 유의하십시오.

**Impact:** DAO 멤버는 가장 낮은 티어 멤버십을 더 높은 티어로 업그레이드할 수 없습니다.

**Proof of Concept:** 다음 테스트를 `MembershipFactory.test.ts`의 `describe("Upgrade Tier")`에 추가할 수 있습니다:
```javascript
it("cannot upgrade from lowest tier, highest index", async function () {
  const fromTierIndex = 6;
  await testERC20.mint(addr1.address, ethers.utils.parseEther("1000000"));
  await testERC20.connect(addr1).approve(membershipFactory.address, ethers.utils.parseEther("1000000"));

  await membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, fromTierIndex);
  await membershipFactory.connect(addr1).joinDAO(membershipERC1155.address, fromTierIndex);

  // cannot upgrade from highest index, lowest tier, because of off-by-one
  await expect(membershipFactory.connect(addr1).upgradeTier(membershipERC1155.address, fromTierIndex)).to.be.revertedWith("No higher tier available.");
});
```

**Recommended Mitigation:** `+ 1`을 제거하십시오:

```diff
-    require(daos[daoMembershipAddress].noOfTiers > fromTierIndex + 1, "No higher tier available.");
+    require(daos[daoMembershipAddress].noOfTiers > fromTierIndex, "No higher tier available.");
```

**One World Project:** [`0a94d44`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/0a94d44bd51b69bbaa2a624f545bdebff0785535)에서 수정됨.

**Cyfrin:** 확인됨. 비교는 이제 `>=`입니다.


### DAO members have no option to leave

**Description:** `MembershipFactory`는 DAO에 가입하고 `SPONSORED` 유형 DAO 내에서 티어를 업그레이드하는 메서드를 노출합니다. 그러나 DAO 멤버가 DAO를 떠나기로 결정한 경우 멤버십 토큰을 소각할 수 있는 로직은 DAO 멤버에게 직접 노출되지 않습니다. 이를 실행할 권한이 있는 유일한 역할은 사용자의 요청에 따라 사용자를 대신하여 이를 수행할 수 있는 `EXTERNAL_CALLER`입니다.

**Impact:** DAO 멤버는 `EXTERNAL_CALLER`의 협조 없이는 떠날 수 없습니다.

**Recommended Mitigation:** DAO 멤버가 떠날 수 있는 옵션을 갖도록 소각 로직을 직접 노출하는 것을 고려하십시오.

**One World Project:** 비즈니스 로직에 따라 멤버가 DAO를 탈퇴할 수 있는 절차는 의도적으로 마련되어 있지 않습니다. 이들은 `EXTERNAL_CALLER`에 의한 오프체인 프로세스를 통해 멤버십 NFT를 소각함으로써 제거될 수 있습니다.

**Cyfrin:** 인지됨. 이 의존성은 추가적인 위험을 초래하므로 오프체인 서비스가 엄격한 보안 표준을 충족하는지 확인할 것을 권장합니다.

\clearpage
## Low Risk


### `MembershipERC1155` should use OpenZeppelin upgradeable base contracts

**Description:** `MembershipERC1155`는 `ProxyAdmin` 인스턴스를 통해 제어되는 `TransparentUpgradeableProxy`와 함께 사용하기 위한 구현 컨트랙트입니다. 그러나 업그레이드 간의 스토리지 충돌을 피하기 위해 설계된 OpenZeppelin 업그레이드 가능 컨트랙트를 활용하지 않습니다.

**Impact:** 새로운 OpenZeppelin 라이브러리로 컨트랙트를 업그레이드하면 스토리지 충돌이 발생할 수 있습니다.

**Recommended Mitigation:** `ERC1155`, `AccessControl` 및 `Initializable`의 업그레이드 가능 버전을 사용하는 것을 고려하십시오.

**One World Project:** openzeppelin 버전 및 solidity 버전을 업데이트했습니다. openzeppelin 컨트랙트의 변경으로 인해 일부 함수를 변경해야 했습니다([`1c3e820`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/1c3e820adc53d977cd2337af1c2d524fc1ac2782)).

**Cyfrin:** 확인됨. `MembershipERC1155`는 이제 OpenZeppelin 컨트랙트의 업그레이드 가능 버전을 사용합니다. OpenZeppelin 라이브러리 버전도 업그레이드되었습니다.


### State update performed after external call in `MembershipERC1155::mint`

**Description:** `MembershipFactory::joinDAO` 호출 중 `MembershipERC1155::mint`가 호출될 때, `totalSupply` 증가는 `ERC1155::_mint` 호출 이후에 수행됩니다:

```solidity
function mint(address to, uint256 tokenId, uint256 amount) external override onlyRole(OWP_FACTORY_ROLE) {
    _mint(to, tokenId, amount, "");
    totalSupply += amount * 2 ** (6 - tokenId); // Update total supply with weight
}
```

즉각적인 영향은 없어 보이지만, 이는 Checks-Effects-Interactions (CEI) 패턴을 위반하므로 [`ERC1155::_doSafeTransferAcceptanceCheck`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/token/ERC1155/ERC1155.sol#L467-L486) [호출](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/token/ERC1155/ERC1155.sol#L285)로 인해 잠재적으로 안전하지 않습니다:

```solidity
if (to.isContract()) {
    try IERC1155Receiver(to).onERC1155Received(operator, from, id, amount, data) returns (bytes4 response) {
        if (response != IERC1155Receiver.onERC1155Received.selector) {
            revert("ERC1155: ERC1155Receiver rejected tokens");
        }
```

**Impact:** 즉각적인 영향은 없어 보이지만, 수신자(receiver) 스마트 컨트랙트 내에서 실행되는 모든 코드는 잘못된 `totalSupply` 상태로 작동합니다.

**Recommended Mitigation:** `_mint()`를 호출하기 전에 `totalSupply`를 증가시키는 것을 고려하십시오.

**One World Project:** [`30465a3`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/30465a3197adea883413298a9ac17fe8a1f0289e)에서 패턴 업데이트됨.

**Cyfrin:** 확인됨. 상태 변경은 이제 외부 호출이 이루어지기 전에 수행됩니다.


### `TierConfig::price` is not validated to follow `TierConfig::power` which itself is not used or validated

**Description:** 새로운 DAO 멤버십을 생성할 때 생성자는 [`TierConfig::power`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L34)를 지정할 수 있습니다. 그러나 이 값은 사용되거나 검증되지 않으며 코드베이스 전체에서 `2`로 가정됩니다. 예를 들어 `MembershipFactory::upgradeTier`에서는 두 개의 하위 티어 토큰이 하나의 상위 티어 토큰을 위해 소각될 수 있다고 가정합니다:

```solidity
IMembershipERC1155(daoMembershipAddress).burn(msg.sender, fromTierIndex, 2);
IMembershipERC1155(daoMembershipAddress).mint(msg.sender, fromTierIndex - 1, 1);
```

그리고 승수가 하드코딩된 [`MembershipERC1155::shareOf`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L165-L176)에서도 마찬가지입니다:

```solidity
function shareOf(address account) public view returns (uint256) {
    return (balanceOf(account, 0) * 64) +
           (balanceOf(account, 1) * 32) +
           (balanceOf(account, 2) * 16) +
           (balanceOf(account, 3) * 8) +
           (balanceOf(account, 4) * 4) +
           (balanceOf(account, 5) * 2) +
           balanceOf(account, 6);
}
```

이 외에도 `TierConfig::price`는 [`MembershipFactory::createNewDAOMembership`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L56) 또는 [`MembershipFactory::updateDAOMembership`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L94)에서 `TierConfig::power`와 함께 실제로 증가하는지 검증되지 않습니다:

```solidity
for (uint256 i = 0; i < tierConfigs.length; i++) {
    dao.tiers.push(tierConfigs[i]);
}
```

따라서 `power` 사양을 준수하지 않는 가격으로 DAO를 생성할 수 있습니다. `MembershipFactory::upgradeTier`에서 `power`가 `2`로 가정되기 때문에, 이로 인해 의도한 것보다 더 저렴하게 업그레이드될 수 있습니다.

**Impact:** DAO 생성자가 보낸 `power` 구성은 사용되지 않으며 전체적으로 `2`로 가정됩니다. `TierConfig::price` 또한 제공된 `power`를 실제로 따르는지 검증되지 않습니다.

**Recommended Mitigation:** 위에서 언급한 곳에서 `TierConfig::power`를 사용하고 검증하는 것을 고려하십시오.

**One World Project:** 이것은 비즈니스 로직에 따른 것입니다. 업그레이드는 항상 하위 티어에서 두 개의 NFT를 가져와 하나의 상위 티어 NFT를 민팅합니다. 다른 값들 중에서도 power는 dao 생성자가 사용자 정의할 수 있지만, 오프체인 검증을 위해서만 컨트랙트에 유지되며 컨트랙트에서 직접 사용되지는 않습니다.

**Cyfrin:** 인지됨.


### DAOs of all types can be updated with a lower number of tiers and are not validated to be above zero

**Description:** `MembershipFactory::createNewDAOMembership`에서 새로운 DAO 멤버십을 생성할 때, 병렬 데이터 구조가 동일한지 [검증](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L59)된 후 티어는 0이 아니고 최댓값을 초과하지 않는지 [검증](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L60)됩니다:

```solidity
require(daoConfig.noOfTiers == tierConfigs.length, "Invalid tier input.");
require(daoConfig.noOfTiers > 0 && daoConfig.noOfTiers <= 7, "Invalid tier count.");
```

`SPONSORED` DAO의 경우 티어 수는 최댓값과 동일한지 [검증](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L62-L64)됩니다:

```solidity
if (daoConfig.daoType == DAOType.SPONSORED) {
    require(daoConfig.noOfTiers == 7, "Invalid tier count for sponsored.");
}
```

그러나 `MembershipFactory::updateDAOMembership`이 호출될 때는 티어 수에 대한 [한도(cap)](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L97) 외에는 이러한 검증이 없습니다.

**Impact:** 모든 유형의 DAO는 티어 수를 0으로 업데이트하여 효과적으로 폐쇄될 수 있습니다.

**Recommended Mitigation:** 이 동작이 의도되지 않은 경우 원래 검증을 유지하여 모든 DAO의 티어 수가 0보다 크고 `SPONSORED` DAO가 최대 티어 수를 가져야 함을 보장하는 것을 고려하십시오.

**One World Project:** [`1b05816`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/1b05816da53ecefa02483141eeef689b331b328d)에서 확인 추가됨.

**Cyfrin:** 확인됨. `tiers`는 이제 `> 0`인지 확인하고 DAO가 `SPONSORED`인 경우 `7`과 같은지 확인합니다.


### `NativeMetaTransaction::executeMetaTransaction` is unnecessarily `payable`

**Description:** `NativeMetaTransaction::executeMetaTransaction`은 [`payable`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/NativeMetaTransaction.sol#L33-L39)로 표시되어 있지만, [OpenZeppelin 구현](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/metatx/MinimalForwarder.sol#L55)과 달리 함수 본문의 [`저수준 호출(low-level call)`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/NativeMetaTransaction.sol#L62-L64)은 네이티브 토큰을 전달하지 않습니다. 따라서 트랜잭션의 일부로 전송된 네이티브 토큰 잔고는 구현 컨트랙트에 묶이게 됩니다.

**Impact:** `MembershipFactory`의 경우 네이티브 토큰 잔고는 `EXTERNAL_CALLER` 역할에 의해 구출될 수 있지만, `OWPIdentity`의 경우 네이티브 토큰은 영원히 묶이게 됩니다.

**Recommended Mitigation:** 네이티브 토큰은 컨트랙트에서 사용되지 않아 불필요하므로 `NativeMetaTransaction::executeMetaTransaction`에서 `payable`을 제거하는 것을 고려하십시오.

또한 `MetaTransactionStruct`에 대한 [주석](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/NativeMetaTransaction.sol#L22-L23)을 _"value isn't included because it is not used in the implementing contracts"_라고 다시 작성할 수 있습니다.

**One World Project:** [`e60b078`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/e60b078f09d4ed0f1e509f36a2a6d42293815737)에서 업데이트됨.

**Cyfrin:** 확인됨. `msg.value`는 이제 전달됩니다.

\clearpage
## Informational


### `MembershipERC1155` implementation contract can be initialized

**Description:** `MembershipERC1155`는 Transparent upgradeable proxy 패턴과 함께 사용하기 위한 구현 컨트랙트입니다. 그러나 `initialize()` 함수는 누구나 호출할 수 있으므로 초기화될 수 있습니다.

**Impact:** 이는 구현 컨트랙트를 초기화하는 것 외에는 어떤 방식으로도 악용될 수 없으며, 이는 프록시에 영향을 미치지 않지만 소비자에게 혼란을 줄 수 있습니다.

**Recommended Mitigation:** 생성자 본문 내에서 [`Initializable::_disableInitializers`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/72c152dc1c41f23d7c504e175f5b417fccc89426/contracts/proxy/utils/Initializable.sol#L184-L203)를 호출하는 것을 고려하십시오.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 추가됨.

**Cyfrin:** 확인됨. `_disabledInitializers()`는 이제 생성자에서 호출됩니다.


### Consider making `MembershipERC1155::totalSupply` `public`

**Description:** `MembershipERC1155` 컨트랙트의 [`totalSupply`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L23) 변수는 현재 `private`으로 표시되어 있습니다:

```solidity
uint256 private totalSupply;
```

이 상태 변수는 오프체인 계산에 유용할 수 있으므로 더 쉽게 접근할 수 있도록 `public`으로 만드는 것을 고려하는 것이 좋습니다.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 업데이트됨.

**Cyfrin:** 확인됨. `totalSupply`는 이제 public입니다.


### Mixed use of `uint` and `uint256` in `MembershipERC1155`

**Description:** `MembershipERC1155`의 상태 선언은 `uint`와 `uint256`을 모두 사용합니다:

```solidity
mapping(address => uint256) public totalProfit;
mapping(address => mapping(address => uint)) internal lastProfit;
mapping(address => mapping(address => uint)) internal savedProfit;

uint256 internal constant ACCURACY = 1e30;

event Claim(address indexed account, uint amount);
event Profit(uint amount);
```

이는 일관성이 없고 혼란스럽습니다. 더 명시적인 `uint256`을 모든 곳에서 사용하는 것을 고려하십시오.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 업데이트됨.

**Cyfrin:** 확인됨. `uint256`이 이제 사용됩니다.


### Unnecessary storage gap in `MembershipERC1155` can be removed

**Description:** `MembershipERC1155`는 컨트랙트의 맨 끝에 [스토리지 갭(storage gap)](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L212-L213)을 선언합니다:

```solidity
uint256[50] private __gap;
```

이러한 갭은 추상 기본 컨트랙트에서 사용하기 위한 것으로, 상태 변수를 컨트랙트 스토리지 레이아웃에 추가할 때 사용된 총 스토리지 슬롯 수를 "아래로 이동"시키지 않도록 하여 상속하는 컨트랙트에서 잠재적인 스토리지 충돌을 일으키지 않도록 합니다.

`MembershipERC1155`는 다른 컨트랙트가 상속하는 기본 컨트랙트로 사용할 의도가 없으므로 스토리지 갭이 필요하지 않으며, 존재하는 갭은 불필요하므로 제거할 수 있습니다.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 제거됨.

**Cyfrin:** 확인됨. `__gap`이 제거되었습니다.


### `MembershipFactory::owpWallet` lacks explicitly declared visibility

**Description:** [`MembershipFactory::owpWallet`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L20)에는 선언된 가시성(visibility)이 없습니다:

```solidity
address owpWallet;
```

이는 기본 `internal` 가시성을 부여하지만, 컨트랙트의 상태 변수에 대해 가시성을 명시적으로 지정하는 것이 모범 사례입니다.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 public으로 변경됨.

**Cyfrin:** 확인됨. `owpWallet`은 public입니다.



### Unnecessarily complex `ProxyAdmin` ownership setup

**Description:** `ProxyAdmin` 컨트랙트는 `MembershipFactory` 생성자에서 생성됩니다:

```solidity
constructor(address _currencyManager, address _owpWallet, string memory _baseURI, address _membershipImplementation) {
    // ...
    proxyAdmin = new ProxyAdmin();
```

`ProxyAdmin`은 `Ownable`을 상속하고 컨트랙트 소유자를 `msg.sender`로 설정합니다. 즉, 소유자는 `MembershipFactory` 컨트랙트가 됩니다.

이 소유권 구조는 `EXTERNAL_CALLER` 역할이 프록시 업그레이드를 관리할 때 `MembershipFactory::callExternalContract`를 호출해야 한다는 요구 사항으로 인해 더욱 복잡해집니다. 더 간단한 해결책은 `ProxyAdmin`을 독립적으로 배포하고 그 주소를 `MembershipFactory` 생성자에 전달하는 것입니다.

**Recommended Mitigation:** `ProxyAdmin`의 별도 인스턴스를 배포하고 그 주소를 생성자 매개변수로 전달하여 소유권 구조를 덜 복잡하고 관리하기 쉽게 만드는 것을 고려하십시오.

**One World Project:** 인지됨. 의도적임. 그대로 유지함.

**Cyfrin:** 인지됨.


### Upgrading DAO tier emits same event as minting the same tier

**Description:** DAO가 SPONSORED로 등록된 경우, 멤버는 `MembershipFactory::upgradeTier` 호출에서 두 개의 하위 티어 토큰을 하나의 상위 티어 토큰으로 소각하여 멤버십 티어를 업그레이드할 수 있습니다.

이것은 처음으로 DAO에 가입할 때 방출되는 것과 동일한 `UserJoinedDAO` 이벤트를 [방출](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L143)하므로, 이 두 가지 작업을 구별하는 것이 불가능합니다.

**Recommended Mitigation:** DAO 멤버가 티어를 업그레이드할 때 별도의 이벤트를 방출하는 것을 고려하십시오.

**One World Project:** 업그레이드는 새 티어에서 새 토큰을 민팅하므로 백엔드에서 이벤트를 효율적으로 추적하기 위해 동일한 이벤트가 유지됩니다.

**Cyfrin:** 인지됨.


### Inconsistent indentation formatting in `CurrencyManager`

**Description:** `CurrencyManager`의 들여쓰기는 2칸인 반면 나머지 코드베이스의 들여쓰기는 4칸입니다.

**Recommended Mitigation:** `CurrencyManager`를 4칸 들여쓰기 규칙과 일치하도록 포맷하는 것을 고려하십시오.

**One World Project:** 인지됨.

**Cyfrin:** 인지됨.


### Unused variables should be used or removed

**Description:** 코드베이스 전체에서 다음 변수들이 선언되었지만 사용되지 않습니다:

* [`MembershipDAOStructs::UINT64_MAX`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L4)
* [`MembershipERC1155::deployer`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L21)
* [`CurrencyManager::admin`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/CurrencyManager.sol#L21)

이 변수들을 사용하거나 제거하는 것을 고려하십시오.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 제거됨.

**Cyfrin:** 확인됨. 위의 변수들은 모두 제거되었습니다.


### Incorrect `EIP712Base` constructor documentation

**Description:** `EIP712` 생성자를 문서화하는 [이 주석](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/EIP712Base.sol#L21-L29)은 부정확합니다:

```solidity
// supposed to be called once while initializing.
// one of the contractsa that inherits this contract follows proxy pattern
// so it is not possible to do this in a constructor
```

프로젝트에서 프록시 패턴을 사용하는 유일한 컨트랙트는 `MembershipERC1155`인데, 이는 `EIP712Base`를 직접적으로든 아니든 상속하지 않습니다. 따라서 주석은 필요하지 않습니다.

오타도 있습니다:

```diff
-  // one of the contractsa that inherits this contract follows proxy pattern
+  // one of the contracts that inherits this contract follows proxy pattern
```

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 제거됨.

**Cyfrin:** 확인됨. 문서가 이제 제거되었습니다.


### `chainId` is used as the `EIP712Base::EIP712Domain.salt` in `DOMAIN_TYPEHASH`

**Description:** `EIP712Base`는 EIP-712를 구현합니다. 그러나 `chainId`가 `salt` 매개변수로 사용되는 [`DOMAIN_TYPEHASH`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/EIP712Base.sol#L38) 정의에 실수가 있습니다.

[EIP-712 사양](https://eips.ethereum.org/EIPS/eip-712#definition-of-domainseparator)에 따르면, 솔트(salt)는 최후의 수단으로만 `DOMAN_TYPEHASH`에서 사용되어야 합니다.

`chainId` 매개변수를 사용해야 하지만, OpenZeppelin [EIP-712](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/EIP712.sol#L37-L39) 구현에서 수행된 것처럼 원시 체인 식별자로 사용해야 합니다:

```solidity
bytes32 private constant TYPE_HASH =
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
```

`DOMAIN_TYPEHASH`를 `salt` 대신 `chainId`를 사용하도록 변경하거나 OpenZeppelin 라이브러리를 직접 사용하는 것을 고려하십시오.

**One World Project:** 의도적임. 그대로 유지함.

**Cyfrin:** 인지됨.


### Tier indexing is confusing

**Description:** `MembershipERC1155` 전체에서 [가장 높은 티어](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L169-L175) 멤버십은 가장 낮은 티어 인덱스로 참조됩니다. 예를 들어, `6`개의 티어가 있는 DAO의 경우 가장 높은 티어 멤버십을 민팅하려면 가장 낮은 인덱스 `0`을 전달해야 하고, 가장 낮은 티어 멤버십을 민팅하려면 가장 높은 인덱스 `6`을 전달해야 합니다.

이는 매우 혼란스러우며 사용자나 타사 통합에 문제를 일으킬 수 있습니다. 가장 높은 티어 인덱스가 가장 높은 티어 멤버십에 해당하고 그 반대의 경우도 마찬가지가 되도록 이 규칙을 뒤집는 것을 고려하십시오.

**One World Project:** 의도적임. 티어 0 (웹사이트의 티어 1)이 가장 높은 레벨입니다. 티어 6 (웹사이트의 티어 7)은 가장 낮습니다.

**Cyfrin:** 인지됨.


### `MembershipFactory::tiers` will almost always return incorrect state

**Description:** [`MembershipFactory::tiers`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L45-L50)는 외부 소비를 위해 [`_tiers`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L23) 매핑을 노출하며, 여기에는 특정 티어에 대해 얼마나 많은 멤버십 토큰이 민팅되었는지를 나타내는 중요한 [`minted`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/libraries/MembershipDAOStructs.sol#L35) 상태 멤버가 포함됩니다. 그러나 [병렬 데이터 구조](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L128)와 달리 `MembershipFactory::joinDAO`나 두 상태 업데이트가 모두 누락된 `MembershipFactory::upgradeTier`에서는 업데이트되지 않습니다. 즉, 주어진 DAO에 대한 매핑이 [동기화](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L114)되는 `MembershipFactory::updateDAOMembership`에 대한 호출이 이루어지지 않는 한 [초기 구성 상태](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L85)만 반환됩니다. 이 또한 다른 멤버십이 민팅될 때까지만 정확하며, 그 후에는 특정 티어에 대해 실제로 민팅된 토큰 수가 매핑에 저장된 수를 초과하게 됩니다.

**Recommended Mitigation:** 두 병렬 데이터 구조를 모두 적절하게 업데이트하는 것을 고려하십시오. 다른 상태 업데이트 문제가 해결되었다고 가정하면, [`daos`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L22) 매핑을 사용하여 올바른 상태를 반환할 수 있습니다. 그러나 이를 위해서는 `MembershipFactory::tiers`를 수정하여 `daos.tiers` 배열을 반환하거나 특정 배열을 쿼리하는 별도의 호출을 구현해야 합니다. 공용 매핑은 `daos()`를 단순히 쿼리할 때 기본적으로 반환하지 않기 때문입니다. 이 경우 `_tiers` 매핑은 중복되므로 완전히 제거할 수 있습니다.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 제거됨.

**Cyfrin:** 확인됨. `_tiers`가 제거되었으며 `MembershipFactory::tiers`는 이제 `dao.tiers` 배열을 반환합니다.


### The Beacon proxy pattern is better suited to upgrading multiple instances of `MembershipERC1155`

**Description:** 현재 새로운 멤버십 DAO는 권한이 있는 `EXTERNAL_CALLER` 역할에 [노출](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L155-L163)된 `ProxyAdmin`의 [단일 인스턴스](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L40)에 의해 관리되는 [Transparent upgradeable proxies](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L66-L70)로 배포됩니다. `MembershipERC1155` 구현 업데이트가 필요한 경우 모든 DAO 프록시를 업그레이드하려는 의도라고 가정할 때, 업그레이드를 수행하기 위해 각 컨트랙트를 반복하는 것은 번거로울 것입니다. [Beacon proxy pattern](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/beacon/BeaconProxy.sol)은 관리되는 모든 프록시에 대해 이러한 유형의 전역 구현 업그레이드를 수행하는 데 더 적합하므로 기존 설계보다 권장됩니다.

**One World Project:** 업그레이드는 각 DAO별로 선택 사항이 될 것입니다. 따라서 그대로 유지합니다.

**Cyfrin:** 인지됨.


### DAO creators cannot freely update membership configuration

**Description:** `MembershipFactory::updateDAOMembership`은 특정 DAO에 대한 티어 구성을 업데이트하기 위한 것이지만, 이 함수는 [허가된](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L95) `EXTERNAL_CALLER` 역할에 의해서만 호출될 수 있습니다. 따라서 DAO 생성자는 `EXTERNAL_CALLER` 역할의 조정 없이는 멤버십 구성을 자유롭게 업데이트할 수 없습니다.

**Recommended Mitigation:** DAO 생성자가 자신의 DAO에 대한 멤버십 구성을 자유롭게 업데이트할 수 있도록 허용하십시오.

**One World Project:** DAO 생성자는 해당 액세스 권한을 직접 가질 수 없습니다.

**Cyfrin:** 인지됨.


### EIP-712 name and project symbol are misaligned

**Description:** `MembershipERC1155` 토큰에 사용된 [심볼](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L69)은 `1WP`입니다. 그러나 EIP-712 [이름 선언](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/meta-transaction/NativeMetaTransaction.sol#L31)에는 `OWP`가 사용됩니다.

이러한 불일치는 `1WP`를 이름으로 사용할 것이라고 생각하고 메시지에 서명하는 사용자에게 혼동을 줄 수 있습니다. 심볼과 동일한 이름을 사용하거나 그 반대로 사용하는 것을 고려하십시오.

**One World Project:** [`ba3603a`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/ba3603ad8c4d976bdf1fa76ea3fb91e8ea1d4462)에서 OWP로 변경했습니다.

**Cyfrin:** 확인됨. 토큰 심볼은 이제 `OWP`입니다.


### `OWPIdentity` token lacks a `name` and `symbol`

**Description:** `name`과 `symbol`은 ERC-1155 사양에서 필수 사항은 아니지만, 토큰을 식별하는 데 자주 사용됩니다. 그러나 [`OWPIdentity`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/OWPIdentity.sol) 컨트랙트는 이를 선언하지 않습니다.

**Recommended Mitigation:** 더 쉬운 식별을 위해 `name`과 `symbol`을 추가하는 것을 고려하십시오.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 추가됨.

**Cyfrin:** 확인됨. `name`과 `symbol`이 `public`으로 추가되었습니다.


### DAOs can be created with non-zero `TierConfig::minted`

**Description:** 새로운 DAO 멤버십을 생성할 때 병렬 `TierConfig` 구조체의 `minted` 멤버에 대한 검증이 없습니다. 즉, 특정 티어 인덱스에 대한 공급량이 실제로는 0이라도 민팅된 토큰이 0이 아닌 상태로 DAO가 생성될 수 있습니다.

**Recommended Mitigation:** 티어 구성의 민팅된 상태가 비어 있는 상태로 시작하도록 강제하는 것을 고려하십시오.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 추가됨.

**Cyfrin:** 확인됨. 티어를 `dao.tiers`에 푸시할 때 검사가 추가되었습니다.


### `MembershipFactory::joinDAO` will not function correctly with fee-on-transfer tokens

**Description:** 프로토콜이 전송 수수료(fee-on-transfer) 토큰을 지원할 의도가 없다는 것은 이해하지만, `MembershipFactory::joinDAO`는 이러한 유형의 토큰이 `CurrencyManager`에 추가되면 [올바르게 작동하지 않을 것](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L129-L130)이라는 점에 유의해야 합니다:

```solidity
IERC20(daos[daoMembershipAddress].currency).transferFrom(msg.sender, owpWallet, platformFees);
IERC20(daos[daoMembershipAddress].currency).transferFrom(msg.sender, daoMembershipAddress, tierPrice - platformFees);
```

여기서 `owpWallet`과 `daoMembershipAddress`가 받는 실제 토큰 수는 예상보다 적을 것입니다.

**One World Project:** 인지됨. 전송 수수료 토큰은 지원되지 않습니다.

**Cyfrin:** 인지됨.


### Constants should be used in place of magic numbers

**Description:** `MembershipFactory` [[1](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L60), [2](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L63
), [3](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/MembershipFactory.sol#L97)] 및 `MembershipERC1155` [[1](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L58), [2](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L71), [3](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L77), [4](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L92), [5](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L168-L175)]에는 함수 내에서 인라인으로 사용되는 매직 넘버가 다수 있습니다. 가독성을 높이고 반복을 피하며 오류 가능성을 줄이기 위해 이를 상수 변수로 대체해야 합니다.

**One World Project:** [`09b6f0f`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/09b6f0f978d2a8d2952a6938bf5756bec8a0170d)에서 반복적으로 사용되는 일부 위치에 상수를 추가했습니다.

**Cyfrin:** 확인됨. 그러나 `MembershipFactory`의 제안된 변경 사항만 구현되었으며 `MembershipERC1155`에는 구현되지 않았습니다.

\clearpage
## Gas Optimization


### The `savedProfit` mapping will always return zero

**Description:** DAO 멤버가 [`MembershipERC1155::claimProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L138-L147)을 호출하면 현재 수익이 []([`MembershipERC1155::saveProfit`](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L178-187):)에서 계산됩니다.

```solidity
function saveProfit(address account, address currency) internal returns (uint profit) {
    uint unsaved = getUnsaved(account, currency);
    lastProfit[account][currency] = totalProfit[currency];
    profit = savedProfit[account][currency] + unsaved;
    savedProfit[account][currency] = profit;
}
```

여기서 `savedProfit`은 계산된 저장되지 않은 수익만큼 증가합니다. 그런 다음 수익은 `savedProfit`을 0으로 재설정한 후 `MembershipERC1155::claimProfit`에서 지급됩니다:

```solidity
function claimProfit(address currency) external returns (uint profit) {
    profit = saveProfit(msg.sender, currency);
    require(profit > 0, "No profit available");
    savedProfit[msg.sender][currency] = 0;
    IERC20(currency).safeTransfer(msg.sender, profit);
    emit Claim(msg.sender, profit);
}
```

`savedProfit`은 초기화되는 동일한 호출의 수명 내에서 0으로 재설정되므로 매핑은 항상 주어진 통화/멤버 쌍에 대해 `0`을 반환합니다. 따라서 `savedProfit[account][currency] + unsaved` [표현식](https://github.com/OneWpOrg/audit-2024-10-oneworld/blob/416630e46ea6f0e9bd9bdd0aea6a48119d0b515a/contracts/dao/tokens/MembershipERC1155.sol#L185)에서의 사용은 중복이며, `savedProfit`에 저장된 값은 사용되지 않으므로 안전하게 제거할 수 있습니다.

**Recommended Mitigation:** `savedProfit` 제거를 고려하십시오.

**One World Project:** [`a3980c1`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/a3980c17217a0b65ecbd28eb078d4d94b4bd5b80)에서 savedProfit 매핑 사용 업데이트됨

**Cyfrin:** 닫힘. `savedProfit`은 이제 사용됩니다.

\clearpage
