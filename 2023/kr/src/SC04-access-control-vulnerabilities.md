## 취약점: 부적절한 접근 제어

### 설명:
접근 제어 취약점은 승인되지 않은 사용자가 컨트랙트의 데이터나 기능에 접근하거나 수정할 수 있게 하는 보안 결함입니다. 이러한 취약점은 컨트랙트의 코드가 사용자 권한 수준에 따라 접근을 적절하게 제한하지 못할 때 발생합니다. 스마트 컨트랙트의 접근 제어는 토큰 발행, 제안 투표, 자금 인출, 컨트랙트 일시 중지 및 업그레이드, 소유권 변경과 같은 거버넌스 및 중요한 로직과 관련될 수 있습니다.

### 예시 (HospoWise 해킹):
```
function burn(address account, uint256 amount) public { // burn 함수에 적절한 접근 제어가 구현되지 않음
        _burn(account, amount);
    }
}
```
### 영향:
- 공격자가 컨트랙트 내의 중요한 기능과 데이터에 무단으로 접근하여 무결성과 보안을 손상시킬 수 있습니다.
- 취약점으로 인해 컨트랙트가 관리하는 자금이나 자산이 도난당할 수 있으며, 사용자와 이해관계자에게 심각한 재정적 피해를 초래할 수 있습니다.

### 해결 방안:
- 초기화 함수가 한 번만, 그리고 승인된 엔티티에 의해서만 호출될 수 있도록 보장하세요.
- 컨트랙트에서 Ownable 또는 RBAC(역할 기반 접근 제어)와 같은 확립된 접근 제어 패턴을 사용하여 권한을 관리하고 승인된 사용자만 특정 기능에 접근할 수 있도록 하세요. 이는 민감한 함수에 `onlyOwner` 또는 커스텀 역할과 같은 적절한 접근 제어 수정자(modifier)를 추가하여 수행할 수 있습니다.

### 부적절한 접근 제어 공격 피해 사례:
1. [HospoWise 해킹](https://etherscan.io/address/0x952aa09109e3ce1a66d41dc806d9024a91dd5684#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
2. [LAND NFT 해킹](https://bscscan.com/address/0x1a62fe088F46561bE92BB5F6e83266289b94C154#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)
