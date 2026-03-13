**Lead Auditors**

[Giovanni Di Siena](https://x.com/giovannidisiena)

**Assisting Auditors**

[Blckhv](https://x.com/blckhv)

[Slavcheww](https://x.com/Slavcheww)


---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### 주어진 배치 작업 내에서 동일한 토큰 ID를 여러 번 소각하여 총 공급량 및 잔액 불변성을 깨뜨릴 수 있음

**설명:** `_burnBatch(uint256[])` 함수는 제공된 모든 `tokenIds`를 루프하고 관련 상태를 업데이트합니다. 그러나 소유권/승인 로직은 상태 업데이트가 발생하기 전에 `_burnBatch(address,uint256[],bool)`에서 모든 토큰에 대해 실행됩니다. 따라서 배열 내에 동일한 `tokenId`가 여러 번 지정되어 이미 소각된 경우 `DEAD_ADDRESS`가 이미 해당 `tokenId`의 소유자인 시나리오를 명시적으로 고려하지 못합니다.

`totalSupply()`에 확인되지 않은(unchecked) 블록이 있기 때문에 동일한 ID를 여러 개 사용하여 `burnBatch()`를 호출함으로써 `burnCounter`를 임의로 증가시킬 수 있으며, 이는 결국 언더플로우를 통해 총 공급량 불변성을 깨뜨릴 수 있습니다.

```solidity
function totalSupply() public view returns (uint256) {
    unchecked {
        return _mintCounter - _burnCounter;
    }
}
```

이는 소각이 현재 소유자 주소에서 `DEAD_ADDRESS`로의 전송을 수행하지만 실행 시작 시에만 소유권을 검증하기 때문에 발생합니다. 대신 이 로직은 이전에 소각된 토큰의 승인/소유권/잔액 상태가 업데이트된 후 배치의 각 ID에 대해 차례로 실행되어야 합니다. 첫 번째 루프 반복 후 소유권이 `DEAD_ADDRESS`로 업데이트되므로 후속 실행은 `DEAD_ADDRESS`에서 자신으로의 전송을 수행하며, 이는 다음 가정에도 불구하고 잔액을 감소시킬 때 언더플로우를 발생시킵니다.

```solidity
unchecked {
    // Cannot overflow, as that would require more tokens to be burned/transferred
    // out than the owner initially received through minting and transferring in.
    _balances[owner] -= 1;
}
```

**영향:** 동일한 토큰 ID를 여러 번 배치 소각함으로써 총 공급량 및 잔액 불변성이 깨집니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `VFTokenC.test.ts`에 추가해야 합니다.

```typescript
it("burning non-existent tokens", async function () {
  const { vfTokenC, owner } = await connection.networkHelpers.loadFixture(
    fixtureWithMintedTokensForBurning
  );

  await vfTokenC.connect(owner).toggleBurnActive();

  const balanceBefore = await vfTokenC.balanceOf(owner.address);
  const deadBefore = await vfTokenC.balanceOf(DEAD_ADDRESS);
  const totalSupplyBefore = await vfTokenC.totalSupply();

  await expect(vfTokenC.connect(owner).burnBatchAdmin([102, 102, 102, 102]))
    .to.emit(vfTokenC, "Transfer")
    .withArgs(owner.address, DEAD_ADDRESS, 102);

  const balanceAfter = await vfTokenC.balanceOf(owner.address);
  const deadAfter = await vfTokenC.balanceOf(DEAD_ADDRESS);
  const totalSupplyAfter = await vfTokenC.totalSupply();

  console.log("Balance before:", balanceBefore.toString());
  console.log("Balance after:", balanceAfter.toString());

  console.log("Dead before:", deadBefore.toString());
  console.log("Dead after:", deadAfter.toString());

  console.log("Total Supply before:", totalSupplyBefore.toString());
  console.log("Total Supply after:", totalSupplyAfter.toString());

  // Tokens should now be owned by DEAD_ADDRESS
  expect(await vfTokenC.ownerOf(102)).to.equal(DEAD_ADDRESS);

  // Owner balance should have decreased by one
  expect(balanceAfter).to.equal(balanceBefore - 1n);

  // Total supply should have only decreased by one (violated)
  expect(totalSupplyAfter).to.equal(totalSupplyBefore - 1n);

  // DEAD_ADDRESS balance should have increased by one (violated)
  expect(deadAfter).to.equal(deadBefore + 1n);
});
```

**권장 완화 조치:** 배열에서 중복 ID를 방지하거나 상태 업데이트가 발생하기 전에 배치 처리하는 대신 각 루프 반복 후에 각 ID에 대한 소유권을 차례로 검증하는 것을 고려하십시오.

**VeeFriends:** 커밋 [dc834fa](https://github.com/veefriends/smart-contracts-v2/commit/dc834fad574e9f7caf460ba5eae2983f8d0e4488)에서 수정됨.

**Cyfrin:** 확인함. 소유자가 이미 데드 어드레스(dead address)인 경우 실행이 이제 되돌려집니다(revert).


### 토큰이 `DEAD_ADDRESS`로 직접 소각 및/또는 전송될 때 일관성 없는 상태 업데이트

**설명:** 토큰을 배치 소각할 때 다음과 같은 상태 업데이트가 수행됩니다.

```solidity
    function _burnBatch(uint256[] calldata tokenIds) internal virtual {
        for (uint256 i; i < tokenIds.length; i++) {
            uint256 tokenId = tokenIds[i];
            address owner = ownerOf(tokenId);

            _beforeTokenTransfer(owner, DEAD_ADDRESS, tokenId, 1);

            // Clear approvals
@>          delete _tokenApprovals[tokenId];

            unchecked {
                // Cannot overflow, as that would require more tokens to be burned/transferred
                // out than the owner initially received through minting and transferring in.
@>              _balances[owner] -= 1;
            }

@>          _owners[tokenId] = DEAD_ADDRESS;

            emit Transfer(owner, DEAD_ADDRESS, tokenId);

            _afterTokenTransfer(owner, DEAD_ADDRESS, tokenId, 1);
        }

        unchecked {
@>          _burnCounter += tokenIds.length;
        }
    }
```

`_balances[owner]` 상태는 감소하는 반면, 소유권이 부여되었음에도 불구하고 `DEAD_ADDRESS`의 상태는 증가하지 않는다는 점에 유의하십시오.

이제 `DEAD_ADDRESS`로의 직접적인 토큰 전송을 고려해 보십시오. 이것이 바람직하지 않을 수 있지만, 이번에는 잔액이 업데이트되는 반면 `_burnCounter` 상태는 변경되지 않는 비대칭이 존재합니다.

```solidity
    function _transfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {
        if (to == address(0)) {
            revert ERC721VFTransferToTheZeroAddress();
        }

        if (ownerOf(tokenId) != from) {
            revert ERC721VFTransferFromIncorrectOwner(from, tokenId);
        }

        _beforeTokenTransfer(from, to, tokenId, 1);

        // Clear approvals from the previous owner
        delete _tokenApprovals[tokenId];

        unchecked {
            // `_balances[from]` cannot overflow for the same reason as described in `_burn`:
            // `from`'s balance is the number of token held, which is at least one before the current
            // transfer.
            // `_balances[to]` could overflow in the conditions described in `_mint`. That would require
            // all 2**256 token ids to be minted, which in practice is impossible.
            _balances[from] -= 1;
@>          _balances[to] += 1;
        }
@>      _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);

        _afterTokenTransfer(from, to, tokenId, 1);
    }
```

**영향:** 토큰을 `DEAD_ADDRESS`로 직접 소각 및/또는 전송할 때 상태가 부분적으로 손상됩니다.

**권장 완화 조치:** 토큰이 소각될 때 `_balances[DEAD_ADDRESS]`를 증가시키십시오. 또한 `DEAD_ADDRESS`로의 직접적인 토큰 전송을 제한하거나, 적어도 예상되는 소각 메커니즘과 일관성을 유지하기 위해 이 엣지 케이스가 처리되도록 하십시오.

**VeeFriends:** 커밋 [dc834fa](https://github.com/veefriends/smart-contracts-v2/commit/dc834fad574e9f7caf460ba5eae2983f8d0e4488) 및 [ed976c6](https://github.com/veefriends/smart-contracts-v2/commit/ed976c671033b145b9055337218196dfb2e642ae)에서 수정됨.

**Cyfrin:** 확인함. 데드 어드레스 잔액이 이제 소각 시 증가하며 직접 전송은 더 이상 허용되지 않습니다.

\clearpage
## 정보 (Informational)


### 불필요한 unchecked 블록 제거 가능

**설명:** 산술 연산이 수행되지 않는 unchecked 블록은 불필요하며 제거할 수 있습니다.

```solidity
function totalMinted() public view returns (uint256) {
    unchecked {
        return _mintCounter;
    }
}

function totalBurned() public view returns (uint256) {
    unchecked {
        return _burnCounter;
    }
}
```

**권장 완화 조치:**
```diff
function totalMinted() public view returns (uint256) {
-   unchecked {
        return _mintCounter;
-   }
}

function totalBurned() public view returns (uint256) {
-   unchecked {
        return _burnCounter;
-   }
}
```

**VeeFriends:** 커밋 [ed976c6](https://github.com/veefriends/smart-contracts-v2/commit/ed976c671033b145b9055337218196dfb2e642ae)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 바이트코드 크기를 줄이기 위해 사용되지 않는 내부 함수 제거

**설명:** 내부 함수 `_safeMint()`, `_safeMintBatch()`, `_airdrop()`, `_mintBatch()` 및 `_burn()`은 사용되지 않는 것으로 보이며 현재 `VFTokenC`에 추가 기능을 추가할 요구 사항이 없는 것으로 이해됩니다. 컨트랙트는 업그레이드할 수 없으므로 향후 업그레이드를 위해서는 별도의 배포가 필요합니다. 즉, 바이트코드 크기를 줄이기 위해 현재 릴리스에서 이러한 함수를 제외해야 할 가능성이 높습니다.

**VeeFriends:** 커밋 [dc834fa](https://github.com/veefriends/smart-contracts-v2/commit/dc834fad574e9f7caf460ba5eae2983f8d0e4488) 및 [ed976c6](https://github.com/veefriends/smart-contracts-v2/commit/ed976c671033b145b9055337218196dfb2e642ae)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

