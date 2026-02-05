## 취약점: 재진입 공격 (Reentrancy)

### 설명:
재진입 공격은 스마트 컨트랙트가 자신의 상태를 업데이트하기 전에 다른 컨트랙트에 외부 호출을 할 때 발생하는 취약점을 악용합니다. 이로 인해 악의적인 외부 컨트랙트가 원래 함수에 다시 진입하여 동일한 상태를 사용해 출금과 같은 특정 작업을 반복할 수 있습니다. 이러한 공격을 통해 공격자는 컨트랙트의 모든 자금을 탈취할 수 있습니다.

### 예시 (DAO 해킹):
```
function splitDAO(uint _proposalID, address _newCurator) noEther onlyTokenholders returns (bool _success) {
    ...

    uint fundsToBeMoved = (balances[msg.sender] * p.splitData[0].splitBalance) / p.splitData[0].totalSupply;
    // 잔액이 업데이트되지 않으므로 공격자는 이 수정자를 여러 번 통과할 수 있음
    if (p.splitData[0].newDAO.createTokenProxy.value(fundsToBeMoved)(msg.sender) == false) throw;

    ...

    // DAO 토큰 소각
    // 잔액이 업데이트되기 전에 자금이 전송됨

    Transfer(msg.sender, 0, balances[msg.sender]);
    withdrawRewardFor(msg.sender); // 친절하게 보상도 받아감
    // 자금 전송 후에야 비로소 잔액이 업데이트됨
    totalSupply -= balances[msg.sender];
    paidOut[msg.sender] = 0;
    return true;
}
```
### 영향:
- 가장 즉각적이고 심각한 결과는 자금 탈취입니다. 공격자는 취약점을 악용하여 권한 이상의 자금을 인출하며, 컨트랙트의 잔액을 완전히 고갈시킬 수 있습니다.
- 공격자가 승인되지 않은 함수 호출을 트리거할 수 있습니다. 이는 컨트랙트 또는 관련 시스템 내에서 의도하지 않은 동작이 실행될 수 있음을 의미합니다.

### 해결 방안:
- 외부 컨트랙트를 호출하기 전에 항상 모든 상태 변경이 먼저 이루어지도록 하세요. 즉, 외부 코드를 호출하기 전에 잔액을 업데이트하거나 내부 코드를 먼저 실행해야 합니다.
- Open Zeppelin의 Re-entrancy Guard와 같이 재진입을 방지하는 함수 수정자(modifier)를 사용하세요.

### 재진입 공격 피해 사례:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)
