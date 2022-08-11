# Lazy Minting 설명

## 목적

블록체인 "메인넷"에서 NFT를 발행하려면 일반적으로 어느 정도의 비용이 듭니다. 
블록체인에 데이터를 기록하려면 계산 및 저장 비용을 지불하기 위해 수수료(_gas_)가 필요하기 때문입니다. 
이것은 아티스트 및 기타 NFT 제작자, 특히 자신의 작업이 판매될지 여부를 알기 전에 많은 돈을 미리 투자하고 싶지 않은 NFT를 처음 접하는 사람들에게는 장벽이 될 수 있습니다

몇 가지 고급 기술을 사용하면 첫 번째 구매자에게 판매되는 순간까지 NFT 발행 비용을 연기할 수 있습니다. 
민팅에 대한 가스 수수료는 NFT를 구매자에게 할당하는 것과 동일한 트랜잭션으로 통합되므로 NFT 작성자는 민팅시 비용을 지불할 필요가 없습니다. 
대신, 구매 가격의 일부는 초기 NFT 레코드를 생성하는 데 필요한 추가 가스를 충당하기 위해 사용됩니다.

구매 순간에 민팅이 이루어 지는것을 _lazy minting_ 이라고 부릅다.
[OpenSea와 같은 NFT마켓에서 채택](https://opensea.io/blog/announcements/introducing-the-collection-manager/)
이는 초기 비용 없이 NFT를 생성할 수 있도록 하여 NFT 제작자의 진입 장벽을 낮추기 위한 것입니다.

이 가이드에서는 다음의 일부 도우미 라이브러리를 사용하여 이더리움에서 게으른 발행이 작동하는 방식을 보여줍니다. [OpenZeppelin](https://openzeppelin.org).

## Overview

lazy minting 이 작동하려면 NFT 구매자가 호출할 수 있는 스마트 계약 기능이 필요합니다. 
이 기능은 원하는 NFT를 발행하고 이를 자신의 계정에 할당할 수 있습니다. 모두 한 번의 거래에서 이루어집니다

시작하기 전에 _non-lazy_minting (보통의 민팅) 기능을 살펴보겠습니다.  

```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract EagerNFT is ERC721URIStorage, AccessControl {
  using Counters for Counters.Counter;
  Counters.Counter _tokenIds;

  bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

  constructor(address minter)
    ERC721("EagerNFT", "EGR") {
      _setupRole(MINTER_ROLE, minter);
    }

  function mint(address buyer, string tokenURI) {
    require(hasRole(MINTER_ROLE, msg.sender), "Unauthorized");
    
    // minting logic
    _tokenIds.increment();
    uint256 tokenId = _tokenIds.current();
    _mint(buyer, tokenId);
    _setTokenURI(tokenId, tokenURI);
  }
}
```

이 계약은 [OpenZeppelin ERC721 기본 계약](https://docs.openzeppelin.com/contracts/4.x/erc721)을 기반으로 하며 
역할 기반 [접근 제어](https://docs.openzeppelin.com/)를 추가합니다. 

`mint` 함수에서 호출자에게 `MINTER_ROLE`이 있어야 승인된 발행인만 새 NFT를 생성할 수 있습니다.
lazy minting 패턴에서 우리는 약간의 변화가 필요합니다:

```solidity
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

contract LazyNFT {
  bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

  constructor(address minter)
    ERC721("LazyNFT", "LAZ") {
      _setupRole(MINTER_ROLE, minter);
    }

  struct NFTVoucher {
    uint256 tokenId;
    uint256 minPrice;
    string uri;
  }

  function redeem(address redeemer, NFTVoucher calldata voucher, bytes memory signature) public payable {
    address signer = _verify(voucher, signature);
    require(hasRole(MINTER_ROLE, signer), "Invalid signature - unknown signer");

    // minting logic...
  }

  function _verify(NFTVoucher voucher, bytes memory signature) private returns (address signer) {
    // verify signature against input and recover address, or revert transaction if signature is invalid
  }
}
```

잠시 서명 및 발행 논리에 대해 알아보겠습니다. 
먼저 `NFTVoucher`라는 새 `구조체`가 있습니다. 'NFTVoucher'는 아직 기록되지 않은 발행되지 않은 NFT를 나타냅니다. 
실제 NFT로 전환하기 위해 구매자는 '상환' 기능을 호출하고 바우처와 NFT 작성자가 준비한 바우처의 서명을 전달할 수 있습니다.

이 계약에는 여전히 역할 기반 액세스 제어가 있지만 약간 변경했습니다. 
`msg.sender`가 승인된 발행인인지 확인하는 대신 누구나 `redeem` 함수를 호출할 수 있습니다. 
액세스 제어는 NFT 발행 권한이 있는 사람이 서명을 생성했는지 확인하는 데 사용됩니다. 
이것은 Ethereum 서명을 확인하면 서명자의 주소를 반환하기 때문에 작동하므로 한 번의 작업으로 바우처를 검증하고 누가 생성했는지 알 수 있습니다.
