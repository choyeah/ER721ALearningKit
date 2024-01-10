# ERC721A 학습

721A는 Azuki에서 개발자들이 오픈제플린 IERC721을 수정하여 가스비를 크게 개선한 NFT 규약
https://www.erc721a.org/

여러 유명 프로젝트들에서 사용 됨

- @moonbirds
- @doodles (dooplicator)
- @rtfkt (forged token)
- @goblintown
- @adidas (airdrop)

## 주요 컨셉

다중 민팅을 위해 중복되는 코드를 개선하여 배치 민팅을 ERC721에서 지원하는 것이 포인트이다.
가스비를 줄이기 위해 여러 방안을 사용하였으며 사용자 입장에서는 ERC271과 동일하게 느낄 수 있다.

## 최적화

1. 오픈제플린의 ERC721 Enumerable의 중복된 스토리지 제거
2. 각각의 NFT를 발행할 때마다가 아니라 배치 발행 요청당 한 번씩 소유자의 잔액과 소유자의 정보를 업데이트
   > 한명의 유저가 민팅을 여러번 요청할 경우 요청한 모든 nft들의 오너를 다 세팅하는 대신 첫번째 민팅한 nft 오너만 세팅해주고, 조회할때는 오너가 없는 경우는 뛰어 넘는 식으로 조회 함수를 개선하여 가스비를 최적화 하였다.

## packed 방식을 사용한 가스비 개선

ERC721A에서는 어셈블리어와 바이트코드 등을 이용하여서 추가적으로 가스비를 개선하였습니다.
기존의 자료형과는 다르게 mapping의 value값에 uint256 형태의 결과 값을 매핑 시키고 바이트 코드 형태로 다양한 데이터를 저장합니다. 예를들어 ownership에 대한 정보 관리의 경우 다음과 같이 256 비트를 통해 여러 정보가 들어갈 위치를 미리 지정해 두어 저장 공간을 절약하였습니다. 저장공간의 절약은 곧 가스비 절감으로 이어집니다.

```
    // Bits Layout:
    // - [0..159]   `addr`
    // - [160..223] `startTimestamp`
    // - [224]      `burned`
    // - [225]      `nextInitialized`
    // - [232..255] `extraData`
    mapping(uint256 => uint256) private _packedOwnerships;
```

- 특정 주소에 대한 정보를 uint256안에 모두 넣고 활용

```
    // Mapping owner address to address data.
    //
    // Bits Layout:
    // - [0..63]    `balance`
    // - [64..127]  `numberMinted`
    // - [128..191] `numberBurned`
    // - [192..255] `aux`
    mapping(address => uint256) private _packedAddressData;
```

이러한 자료형들을 packed라고 정의 한 후에
unpacked라는 인터널 함수들을 통하여서 필요한 위치에 있는 정보만 뽑아서 가져다 사용합니다.
필요한 위치에 있는 정보만 뽑기 위하여 constants에서 미리 위치를 정의해두거나 마스크를 하기 위한 수치를 정해 둡니다.

\_packedAddressData 에서
BITMASK의 경우 자신이 원하는 위치에만 1을 넣은 자료형을 만들고 and 연산을 하여 나머지 위치의 값은 모두 0으로 만들어 버립니다.

BITPOS라고 되어 있는 변수의 경우 특정 위치를 의미하여 해당 위치에서 데이터를 가져올 때 사용합니다.

## 코드

https://github.com/chiru-labs/ERC721A

## 설치

```
npm install --save-dev erc721a
```

## 사용법

```
pragma solidity ^0.8.4;

import "erc721a/contracts/ERC721A.sol";

contract MyERC721A is ERC721A {
    constructor() ERC721A("Myerc721A", "721A") {}

    function mint(uint256 quantity) external payable {
        // `_mint`'s second argument now takes in a `quantity`, not a `tokenId`.
        _mint(msg.sender, quantity);
    }
}
```

> erc721과 동일하게 작동함
