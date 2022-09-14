---
eip: 3525
title: Semi-Fungible Token
description: Defines a specification where EIP-721 compatible tokens with the same SLOT and different IDs are fungible.
author: Will Wang (@will-edge), Mike Meng <myan@solv.finance>, Ethan Y. Tsai (@YeeTsai), Ryan Chow <ryanchow@solv.finance>, Zhongxin Wu (@Nerverwind), AlvisDu (@AlvisDu)
discussions-to: https://ethereum-magicians.org/t/eip-3525-the-semi-fungible-token
status: Final
type: Standards Track
category: ERC
created: 2020-12-01
requires: 20, 165, 721
---

## Abstract

这是半可替代代币的标准。本文档中描述的智能合约接口集定义了一个 [EIP-721](./eip-721.md) 兼容的代币标准。 该标准引入了一个`<ID, SLOT, VALUE>` 三重标量模型，表示代币的半可替代结构。它还引入了新的转移模型以及反映代币半可替代性质的批准模型。

Token 包含一个 EIP-721 等效的 ID 属性，以将自己标识为一个普遍唯一的实体，以便令牌可以在地址之间转移并被批准以 EIP-721 兼容的方式操作。

代币还包含一个`value`属性，代表代币的数量性质。`value` 属性的含义很像 [EIP-20](./eip-20.md) 令牌的 `balance` 属性。 每个令牌都有一个'插槽'属性，确保具有相同插槽的两个令牌的值被视为可替代，从而为令牌的 value 属性增加了可替代性。

该 EIP 引入了新的代币转移模型以实现半可替代性，包括同一插槽的两个代币之间的价值转移以及从一个代币到一个地址的价值转移。

## Motivation

代币化是在加密货币中使用和控制数字资产的最重要趋势之一。 传统上，有两种方法可以做到这一点：可替代代币和不可替代代币。 可替代代币通常使用 EIP-20 标准，其中资产的每个单位都是相同的。 EIP-20 是一种灵活有效的操作可替代代币的方法。 不可替代的代币主要是 EIP-721 代币，这是一种能够根据身份区分数字资产的标准。

然而，两者都有明显的缺点。 例如，EIP-20 要求用户为每个单独的数据结构或可定制属性的组合创建单独的 EIP-20 合约。 在实践中，这导致需要创建大量的 EIP-20 合约。 另一方面，EIP-721 代币没有提供量化特征，大大削弱了它们的可计算性、流动性和可管理性。 例如，如果要使用 EIP-721 创建债券、保险单或归属计划等金融工具，则没有标准接口可供我们控制其中的价值，例如，无法转移一部分 代币所代表的合约中的权益。

解决问题的更直观和直接的方法是创建一个具有 EIP-20 的定量特征和 EIP-721 的定性属性的半同质代币。 这种半可替代代币与 EIP-721 的向后兼容性将有助于利用已经在使用的现有基础设施并导致更快的采用。

## Specification

本文档中的关键字"必须"、"不得"、"要求"、"应"、"不得"、"应该"、"不应"、"推荐"、"可以"和"可选"是为了 按照 RFC 2119 中的说明进行解释。

**Every [EIP-3525](./eip-3525.md) compliant contract must implement the EIP-3525, EIP-721 and [EIP-165](./eip-165.md) interfaces**

```solidity
pragma solidity ^0.8.0;

/**
 * @title EIP-3525 Semi-Fungible Token Standard
 * Note: the EIP-165 identifier for this interface is 0xd5358140.
 */
interface IERC3525 /* is IERC165, IERC721 */ {
    /**
     * @dev MUST emit when value of a token is transferred to another token with the same slot,
     *  including zero value transfers (_value == 0) as well as transfers when tokens are created
     *  (`_fromTokenId` == 0) or destroyed (`_toTokenId` == 0).
     * @param _fromTokenId The token id to transfer value from
     * @param _toTokenId The token id to transfer value to
     * @param _value The transferred value
     */
    event TransferValue(uint256 indexed _fromTokenId, uint256 indexed _toTokenId, uint256 _value);

    /**
     * @dev MUST emit when the approval value of a token is set or changed.
     * @param _tokenId The token to approve
     * @param _operator The operator to approve for
     * @param _value The maximum value that `_operator` is allowed to manage
     */
    event ApprovalValue(uint256 indexed _tokenId, address indexed _operator, uint256 _value);
    
    /**
     * @dev MUST emit when the slot of a token is set or changed.
     * @param _tokenId The token of which slot is set or changed
     * @param _oldSlot The previous slot of the token
     * @param _newSlot The updated slot of the token
     */ 
    event SlotChanged(uint256 indexed _tokenId, uint256 indexed _oldSlot, uint256 indexed _newSlot);

    /**
     * @notice Get the number of decimals the token uses for value - e.g. 6, means the user
     *  representation of the value of a token can be calculated by dividing it by 1,000,000.
     *  Considering the compatibility with third-party wallets, this function is defined as
     *  `valueDecimals()` instead of `decimals()` to avoid conflict with EIP-20 tokens.
     * @return The number of decimals for value
     */
    function valueDecimals() external view returns (uint8);

    /**
     * @notice Get the value of a token.
     * @param _tokenId The token for which to query the balance
     * @return The value of `_tokenId`
     */
    function balanceOf(uint256 _tokenId) external view returns (uint256);

    /**
     * @notice Get the slot of a token.
     * @param _tokenId The identifier for a token
     * @return The slot of the token
     */
    function slotOf(uint256 _tokenId) external view returns (uint256);

    /**
     * @notice Allow an operator to manage the value of a token, up to the `_value`.
     * @dev MUST revert unless caller is the current owner, an authorized operator, or the approved
     *  address for `_tokenId`.
     *  MUST emit the ApprovalValue event.
     * @param _tokenId The token to approve
     * @param _operator The operator to be approved
     * @param _value The maximum value of `_toTokenId` that `_operator` is allowed to manage
     */
    function approve(
        uint256 _tokenId,
        address _operator,
        uint256 _value
    ) external payable;

    /**
     * @notice Get the maximum value of a token that an operator is allowed to manage.
     * @param _tokenId The token for which to query the allowance
     * @param _operator The address of an operator
     * @return The current approval value of `_tokenId` that `_operator` is allowed to manage
     */
    function allowance(uint256 _tokenId, address _operator) external view returns (uint256);

    /**
     * @notice Transfer value from a specified token to another specified token with the same slot.
     * @dev Caller MUST be the current owner, an authorized operator or an operator who has been
     *  approved the whole `_fromTokenId` or part of it.
     *  MUST revert if `_fromTokenId` or `_toTokenId` is zero token id or does not exist.
     *  MUST revert if slots of `_fromTokenId` and `_toTokenId` do not match.
     *  MUST revert if `_value` exceeds the balance of `_fromTokenId` or its allowance to the
     *  operator.
     *  MUST emit `TransferValue` event.
     * @param _fromTokenId The token to transfer value from
     * @param _toTokenId The token to transfer value to
     * @param _value The transferred value
     */
    function transferFrom(
        uint256 _fromTokenId,
        uint256 _toTokenId,
        uint256 _value
    ) external payable;


    /**
     * @notice Transfer value from a specified token to an address. The caller should confirm that
     *  `_to` is capable of receiving EIP-3525 tokens.
     * @dev This function MUST create a new EIP-3525 token with the same slot for `_to`, 
     *  or find an existing token with the same slot owned by `_to`, to receive the transferred value.
     *  MUST revert if `_fromTokenId` is zero token id or does not exist.
     *  MUST revert if `_to` is zero address.
     *  MUST revert if `_value` exceeds the balance of `_fromTokenId` or its allowance to the
     *  operator.
     *  MUST emit `Transfer` and `TransferValue` events.
     * @param _fromTokenId The token to transfer value from
     * @param _to The address to transfer value to
     * @param _value The transferred value
     * @return ID of the token which receives the transferred value
     */
    function transferFrom(
        uint256 _fromTokenId,
        address _to,
        uint256 _value
    ) external payable returns (uint256);
}
```

插槽的枚举扩展是可选的。 这允许您的合约发布其完整的`SLOT`列表并使其可被发现。

```solidity
pragma solidity ^0.8.0;

/**
 * @title EIP-3525 Semi-Fungible Token Standard, optional extension for slot enumeration
 * @dev Interfaces for any contract that wants to support enumeration of slots as well as tokens 
 *  with the same slot.
 * Note: the EIP-165 identifier for this interface is 0x3b741b9e.
 */
interface IERC3525SlotEnumerable is IERC3525 /* , IERC721Enumerable */ {

    /**
     * @notice Get the total amount of slots stored by the contract.
     * @return The total amount of slots
     */
    function slotCount() external view returns (uint256);

    /**
     * @notice Get the slot at the specified index of all slots stored by the contract.
     * @param _index The index in the slot list
     * @return The slot at `index` of all slots.
     */
    function slotByIndex(uint256 _index) external view returns (uint256);

    /**
     * @notice Get the total amount of tokens with the same slot.
     * @param _slot The slot to query token supply for
     * @return The total amount of tokens with the specified `_slot`
     */
    function tokenSupplyInSlot(uint256 _slot) external view returns (uint256);

    /**
     * @notice Get the token at the specified index of all tokens with the same slot.
     * @param _slot The slot to query tokens with
     * @param _index The index in the token list of the slot
     * @return The token ID at `_index` of all tokens with `_slot`
     */
    function tokenInSlotByIndex(uint256 _slot, uint256 _index) external view returns (uint256);
}
```

插槽级别批准是可选的。 这允许任何想要支持插槽批准的合约，这允许操作员使用相同的插槽管理自己的代币。

```solidity
pragma solidity ^0.8.0;

/**
 * @title EIP-3525 Semi-Fungible Token Standard, optional extension for approval of slot level
 * @dev Interfaces for any contract that wants to support approval of slot level, which allows an
 *  operator to manage one's tokens with the same slot.
 *  See https://eips.ethereum.org/EIPS/eip-3525
 * Note: the EIP-165 identifier for this interface is 0xb688be58.
 */
interface IERC3525SlotApprovable is IERC3525 {
    /**
     * @dev MUST emit when an operator is approved or disapproved to manage all of `_owner`'s
     *  tokens with the same slot.
     * @param _owner The address whose tokens are approved
     * @param _slot The slot to approve, all of `_owner`'s tokens with this slot are approved
     * @param _operator The operator being approved or disapproved
     * @param _approved Identify if `_operator` is approved or disapproved
     */
    event ApprovalForSlot(address indexed _owner, uint256 indexed _slot, address indexed _operator, bool _approved);

    /**
     * @notice Approve or disapprove an operator to manage all of `_owner`'s tokens with the
     *  specified slot.
     * @dev Caller SHOULD be `_owner` or an operator who has been authorized through
     *  `setApprovalForAll`.
     *  MUST emit ApprovalSlot event.
     * @param _owner The address that owns the EIP-3525 tokens
     * @param _slot The slot of tokens being queried approval of
     * @param _operator The address for whom to query approval
     * @param _approved Identify if `_operator` would be approved or disapproved
     */
    function setApprovalForSlot(
        address _owner,
        uint256 _slot,
        address _operator,
        bool _approved
    ) external payable;

    /**
     * @notice Query if `_operator` is authorized to manage all of `_owner`'s tokens with the
     *  specified slot.
     * @param _owner The address that owns the EIP-3525 tokens
     * @param _slot The slot of tokens being queried approval of
     * @param _operator The address for whom to query approval
     * @return True if `_operator` is authorized to manage all of `_owner`'s tokens with `_slot`,
     *  false otherwise.
     */
    function isApprovedForSlot(
        address _owner,
        uint256 _slot,
        address _operator
    ) external view returns (bool);
}
```


### EIP-3525 Token Receiver

如果智能合约想要在收到来自其他地址的值时得到通知，它应该实现 `IERC3525Receiver` 接口中的所有功能，在实现中它可以决定是接受还是拒绝转账。 有关详细信息，请参阅"Transfer Rules"。

```solidity
 pragma solidity ^0.8.0;

/**
 * @title EIP-3525 token receiver interface
 * @dev Interface for a smart contract that wants to be informed by EIP-3525 contracts when receiving values from ANY addresses or EIP-3525 tokens.
 * Note: the EIP-165 identifier for this interface is 0x009ce20b.
 */
interface IERC3525Receiver {
    /**
     * @notice Handle the receipt of an EIP-3525 token value.
     * @dev An EIP-3525 smart contract MUST check whether this function is implemented by the recipient contract, if the
     *  recipient contract implements this function, the EIP-3525 contract MUST call this function after a 
     *  value transfer (i.e. `transferFrom(uint256,uint256,uint256,bytes)`).
     *  MUST return 0x009ce20b (i.e. `bytes4(keccak256('onERC3525Received(address,uint256,uint256,
     *  uint256,bytes)'))`) if the transfer is accepted.
     *  MUST revert or return any value other than 0x009ce20b if the transfer is rejected.
     * @param _operator The address which triggered the transfer
     * @param _fromTokenId The token id to transfer value from
     * @param _toTokenId The token id to transfer value to
     * @param _value The transferred value
     * @param _data Additional data with no specified format
     * @return `bytes4(keccak256('onERC3525Received(address,uint256,uint256,uint256,bytes)'))` 
     *  unless the transfer is rejected.
     */
    function onERC3525Received(address _operator, uint256 _fromTokenId, uint256 _toTokenId, uint256 _value, bytes calldata _data) external returns (bytes4);

}
```

### Token Manipulation

#### Scenarios

**_Transfer:_**

除了与 EIP-721 兼容的令牌传输方法外，该 EIP 还引入了两种新的传输模型：从 ID 到 ID 的值传输，以及从 ID 到地址的值传输。

```solidity
function transferFrom(uint256 _fromTokenId, uint256 _toTokenId, uint256 _value) external payable;
	
function transferFrom(uint256 _fromTokenId, address _to, uint256 _value) external payable returns (uint256 toTokenId_);
```

第一个允许在同一个槽内从一个令牌（由`_fromTokenId`指定）到另一个令牌（由`_toTokenId`指定）的值传输，导致从源令牌的值中减去`_value`并添加到 目标令牌的价值；

第二种允许从一个token（由`_fromTokenId`指定）到一个地址（由`_to`指定）的值转移，该值实际上转移到该地址拥有的token，并且应该返回目标token的id . 可以在此方法的'design decision'部分中找到进一步的解释。


#### Rules

**_approving rules:_**

本 EIP 提供四种审批功能，表示不同的审批级别，可描述为全级审批、槽级审批、代币 ID 级审批以及价值级审批。

- `setApprovalForAll`, 与 EIP-721 兼容，应表明完全批准级别，这意味着授权运营商能够管理所有者拥有的所有代币，包括其价值。
- `setApprovalForSlot` (optional) 应该表明批准的槽位级别，这意味着授权的操作员能够管理具有指定槽位的所有代币，包括它们的值，由所有者拥有。
- token ID 等级的 `approve` 函数, 与 EIP-721 兼容，应指示授权操作员只能管理所有者拥有的指定令牌 ID，包括其值。
- value 等级的 `approve` 函数, 应该表明授权运营商能够管理所有者拥有的指定令牌的指定最大值。
- 对于任何批准功能，调用者必须是所有者或已获得更高级别的授权。

**_transferFrom rules:_**

- `transferFrom(uint256 _fromTokenId, uint256 _toTokenId, uint256 _value)` 函数, 应根据以下规则指示从一个代币到另一个代币的`value`转移：

  - 必须还原除非 `msg.sender` 是 `_fromTokenId` 的所有者、授权操作员或已批准整个代币或至少 `_value` 的操作员。
  - 如果 `_fromTokenId` 或 `_toTokenId` 是零令牌 ID 或不存在，则必须还原。
  - 如果 `_fromTokenId` 和 `_toTokenId` 的插槽不匹配，则必须还原。
  - 如果 `_value` 超过 `_fromTokenId` 的值或其对运营商的允许，则必须还原。
  - 如果 _toTokenId 的所有者是智能合约，则必须检查 `onERC3525Received` 函数，如果该函数存在，则必须在值转移后调用此函数，如果结果不等于 0x009ce20b，则必须还原；
  - 必须发出`TransferValue`事件。

- The `transferFrom(uint256 _fromTokenId, address _to, uint256 _value)` function, 将`value`从一个`TokenId`转移到一个地址，应该遵循以下规则：

  - 必须要么找到地址 `_to` 拥有的 EIP-3525 令牌，要么创建一个新的 EIP-3525 令牌，具有相同的 `_fromTokenId` 槽，以接收传输的值。
  - 必须还原除非 `msg.sender` 是 `_fromTokenId` 的所有者、授权操作员或已批准整个代币或至少 `_value` 的操作员。
  - 如果 `_fromTokenId` 是0 或不存在，则必须还原。
  - 如果 `_to` 是零地址，则必须还原。
  - 如果 `_value` 超过 `_fromTokenId` 的值或其对运营商的允许，则必须还原。
  - 如果 _to 地址是智能合约，则必须检查 `onERC3525Received` 函数，如果该函数存在，则必须在值传输后调用此函数，如果结果不等于 0x009ce20b，则必须还原；
  - 必须发出 `Transfer` 和 `TransferValue` 事件。


### Metadata

#### Metadata Extensions

EIP-3525 元数据扩展是兼容的 EIP-721 元数据扩展。

这个可选接口可以通过 EIP-165 标准接口检测来识别。

```solidity
pragma solidity ^0.8.0;

/**
 * @title EIP-3525 Semi-Fungible Token Standard, optional extension for metadata
 * @dev Interfaces for any contract that wants to support query of the Uniform Resource Identifier
 *  (URI) for the EIP-3525 contract as well as a specified slot. 
 *  Because of the higher reliability of data stored in smart contracts compared to data stored in 
 *  centralized systems, it is recommended that metadata, including `contractURI`, `slotURI` and 
 *  `tokenURI`, be directly returned in JSON format, instead of being returned with a url pointing 
 *  to any resource stored in a centralized system. 
 *  See https://eips.ethereum.org/EIPS/eip-3525
 * Note: the EIP-165 identifier for this interface is 0xe1600902.
 */
interface IERC3525Metadata is
    IERC3525 /* , IERC721Metadata */
{
    /**
     * @notice Returns the Uniform Resource Identifier (URI) for the current EIP-3525 contract.
     * @dev This function SHOULD return the URI for this contract in JSON format, starting with
     *  header `data:application/json;`.
     *  See https://eips.ethereum.org/EIPS/eip-3525 for the JSON schema for contract URI.
     * @return The JSON formatted URI of the current EIP-3525 contract
     */
    function contractURI() external view returns (string memory);

    /**
     * @notice Returns the Uniform Resource Identifier (URI) for the specified slot.
     * @dev This function SHOULD return the URI for `_slot` in JSON format, starting with header
     *  `data:application/json;`.
     *  See https://eips.ethereum.org/EIPS/eip-3525 for the JSON schema for slot URI.
     * @return The JSON formatted URI of `_slot`
     */
    function slotURI(uint256 _slot) external view returns (string memory);
}
```

#### EIP-3525 Metadata URI JSON Schema

This is the "EIP-3525 Metadata JSON Schema for `contractURI()`" referenced above.

```json
{
  "title": "Contract Metadata",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Contract Name"
    },
    "description": {
      "type": "string",
      "description": "Describes the contract"
    },
    "image": {
      "type": "string",
      "description": "Optional. Either a base64 encoded imgae data or a URI pointing to a resource with mime type image/* representing what this contract represents."
    },
    "external_link": {
      "type": "string",
      "description": "Optional. A URI pointing to an external resource."
    },
    "valueDecimals": {
      "type": "integer",
      "description": "The number of decimal places that the balance should display - e.g. 18, means to divide the token value by 1000000000000000000 to get its user representation."
    }
  }
}
```

This is the "EIP-3525 Metadata JSON Schema for `slotURI(uint)`" referenced above.

```json
{
  "title": "Slot Metadata",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Identifies the asset category to which this slot represents"
    },
    "description": {
      "type": "string",
      "description": "Describes the asset category to which this slot represents"
    },
    "image": {
      "type": "string",
      "description": "Optional. Either a base64 encoded imgae data or a URI pointing to a resource with mime type image/* representing the asset category to which this slot represents."
    },
    "properties": {
      "type": "array",
      "description": "Each item of `properties` SHOULD be organized in object format, including name, description, value, order (optional), display_type (optional), etc."
      "items": {
        "type": "object",
        "properties": {
          "name": {
            "type": "string",
            "description": "The name of this property."
          },
          "description": {
            "type": "string",
            "description": "Describes this property."
          }
          "value": {
            "description": "The value of this property, which may be a string or a number."
          },
          "is_intrinsic": {
            "type": "boolean",
            "description": "According to the definition of `slot`, one of the best practice to generate the value of a slot is utilizing the `keccak256` algorithm to calculate the hash value of multi properties. In this scenario, the `properties` field should contain all the properties that are used to calculate the value of `slot`, and if a property is used in the calculation, is_intrinsic must be TRUE."
          },
          "order": {
            "type": "integer",
            "description": "Optional, related to the value of is_intrinsic. If is_intrinsic is TRUE, it must be the order of this property appeared in the calculation method of the slot."
          },
          "display_type": {
            "type": "string",
            "description": "Optional. Specifies in what form this property should be displayed."
          }
        }
      }
    }
  }
}
```


This is the "EIP-3525 Metadata JSON Schema for `tokenURI(uint)`" referenced above.

```json
{
  "title": "Token Metadata",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Identifies the asset to which this token represents"
    },
    "description": {
      "type": "string",
      "description": "Describes the asset to which this token represents"
    },
    "image": {
      "type": "string",
      "description": "Either a base64 encoded imgae data or a URI pointing to a resource with mime type image/* representing the asset to which this token represents."
    },
    "balance": {
      "type": "integer",
      "description": "THe value held by this token."
    },
    "slot": {
      "type": "integer",
      "description": "The id of the slot that this token belongs to."
    },
    "properties": {
      "type": "object",
      "description": "Arbitrary properties. Values may be strings, numbers, objects or arrays. Optional, you can use the same schema as the properties section of EIP-3525 Metadata JSON Schema for slotURI(uint) if you need a better description attribute."
    }
  }
}
```


## Rationale

### Metadata generation

该代币标准旨在代表半可替代资产，这些资产最适合金融工具而不是收藏品或游戏内物品。 为了最大限度地提高数字资产的透明度和安全性，我们强烈建议所有实现都应直接从合约代码生成元数据，而不是提供链下服务器 URL。

### Design decision: Value transfer from token to address

代币的`value`是代币的属性，与地址无关，因此将`value`转移到地址实际上是将其转移到该地址拥有的代币，而不是地址本身。

从实现的角度来看，将`value`从代币转移到地址的过程可以如下完成：（1）为接收者的地址创建一个新代币，（2）将`value`从`source token`转移到新代币。所以这个方法并不完全独立于 ID 到 ID 的传输方法，可以看作是包装了上述过程的语法糖。

在特殊情况下，如果目标地址拥有一个或多个与源令牌具有相同槽值的令牌，则该方法将具有如下替代实现： (1) 找到该地址拥有的一个具有相同槽值的令牌源令牌，（2）将值转移到找到的令牌。

上述两种实现都应被视为符合本标准。

维护 id-to-address 传输功能的目的是最大限度地与大多数钱包应用程序兼容，因为对于大多数令牌标准，令牌传输的目的地是地址。这种句法包装将帮助钱包应用轻松实现从代币到任何地址的价值转移功能。

### Design decision: Notification/acceptance mechanism instead of 'Safe Transfer'

EIP-721 和后来的一些代币标准引入了`safeTransferFrom`模型，为了更好地控制转移代币时的'safety' ，这种机制将不同的转移模式（安全/不安全）的选择留给发送者，并可能导致一些潜在的 问题：

1. 在大多数情况下，发送方不知道如何在两种传输方式（安全/不安全）之间进行选择；
2. 如果发送方调用 `safeTransferFrom` 方法，当接收方合约没有实现回调函数时，传输可能会失败，即使该合约可以毫无问题地接收和操作代币。

此 EIP 定义了一个简单的'Check, Notify and Response'模型，以实现更好的灵活性和简单性：

1.不需要额外的`safeTransferFrom`方法，所有调用者只需要调用一种传输；
2. 所有 EIP-3525 合约都需要检查接收者合约上是否存在 `onERC3525Received` 并在存在时调用该函数；
3. 任何智能合约都可以实现`onERC3525Received`函数，用于在收到值后得到通知，在该函数中可以返回某个预定义的值来接受转账，或者任何其他值来拒绝转账。

这种通知/接受机制有一个特殊情况，因为 EIP-3525 允许从一个地址向其自身进行价值转移，因此当实现 `onERC3525Received` 的智能合约向自身转移价值时，也会调用此函数。 这样合约就有责任在自我价值转移和从其他地址接收价值之间选择不同的接受规则。

### Design decision: Relationship between different approval models

为了与 EIP-721 的语义兼容性以及代币值操作的灵活性，我们决定定义一些批准级别之间的关系，如下所示：

1. 一个 id 的批准将导致被批准的操作员从这个 id 部分转移值的能力，这将简化一个 id 的值批准。 但是，通证中总价值的批准不应导致被批准的运营商转移通证实体的能力。
2. `setApprovalForAll` 将导致能够从任何代币部分转移价值，以及批准从任何代币到第三方的部分价值转移的能力，这将简化一个地址拥有的所有代币的价值转移和批准 .

## Backwards Compatibility

如开头所述，此 EIP 向后兼容 EIP-721。

## Reference Implementation

- [EIP-3525 implementation](../assets/eip-3525/contracts/ERC3525.sol)

## Security Considerations

The value level approval and slot level approval(optional) is isolated from EIP-721 approval models, so that approving value should not affect EIP-721 level approvals, implementations of this EIP must obey this principle.

Since this EIP is EIP-721 compatible, any wallets and smart contracts that can hold and manipulate standard EIP-721 tokens will have no risks of asset loss for EIP-3525 tokens.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
