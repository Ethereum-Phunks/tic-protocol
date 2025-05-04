![TIC Protocol Logo](assets/tic-icon.png)

# TIC (Transaction Inscribed Comments) Protocol Specification (alpha)


### Note:
TIC is currently in early alpha development. The specification and implementation are subject to change as we refine the protocol based on community feedback and testing.

## Overview
TIC is a child protocol built on top of [Ethscriptions](https://docs.ethscriptions.com/) that enables on-chain commenting capabilities through Ethereum transaction calldata. It provides a standardized way to associate comments with blockchain entities such as addresses, transaction hashes, or other on-chain identifiers. Comments can be nested and replied to by setting the topic to the hash of the parent comment.

## Protocol Rules

### 1. Data Structure
Comments must be formatted as JSON and encoded as a data URL with the following MIME type:
`data:message/vnd.tic+json;rule=esip6`

The `rule=esip6` parameter is mandatory and cannot be omitted. This ensures:
- Guaranteed message delivery for all comments
- Protection against front-running attacks
- Consistent behavior across all TIC implementations
- Reliable smart contract integration
- Future-proofing for protocol evolution

Applications must always include the `rule=esip6` parameter in their data URLs. Comments without this parameter are considered invalid according to the TIC protocol.

For more information about ESIP-6 and its benefits, see [ESIP-6: Opt-in Ethscription Non-uniqueness](https://docs.ethscriptions.com/esips/accepted-esips/esip-6-opt-in-ethscription-non-uniqueness).

### 2. Comment Object Schema
```typescript
interface TIC {
  // Required: Identifier that links the comment to a blockchain entity
  // Can be a single hex string or multiple hex strings separated by colons
  // Example: "0x1234" or "0xContractAddress:0xTokenId:0xExtraData"
  topic: `0x${string}` | `0x${string}:0x${string}` | `0x${string}:0x${string}:0x${string}`; // Can have unlimited parts (within calldata limits)
  
  // Required: The comment content
  content: string;
  
  // Required: Protocol version in hex format
  version: `0x${string}`;
  
  // Optional: Specifies the content encoding format (defaults to utf8)
  encoding?: EncodingType;

  // Optional: Specifies the comment type for application-specific features.
  type?: CommentType;
}

type EncodingType = 'utf8' | 'base64' | 'hex' | 'json' | 'markdown' | 'ascii';

type CommentType = 
  | 'comment'  // Basic comment
  | 'reaction' // Like/emoji reaction to content
```

### 3. Field Specifications

#### topic (Required)
- Must be a valid hexadecimal string with 0x prefix
- Can be a single hex string or multiple hex strings separated by colons
- All parts must be valid hex strings with 0x prefix
- Can represent:
  - Ethereum addresses (40 characters)
  - Transaction hashes (64 characters)
  - Comment hashes (for replies/nested comments)
  - Any combination of hex strings separated by colons
- No length restriction for standard topics, but must be a valid hex value
- When used for replies, the topic should be set to the hash of the parent comment
- Example: For NFT comments, the topic would be `0xContractAddress:0xTokenId` where:
  - ContractAddress is a valid Ethereum address
  - TokenId is converted to hex format

#### content (Required)
- The actual comment data
- Must be encoded according to the specified `encoding` field
- No length restriction beyond Ethereum calldata limits

#### version (Required)
- Must be a hexadecimal string
- Current protocol version: `0x0`
- Used for future protocol upgrades and compatibility

#### encoding (Optional)
Must be one of the following values:
- `utf8`: UTF-8 encoded text
- `base64`: Base64 encoded data
- `hex`: Hexadecimal encoded data
- `json`: JSON formatted data
- `markdown`: Markdown formatted text
- `ascii`: ASCII encoded text

#### type (Optional)
- Specifies the intended use of the comment
- Current supported types:
  - `comment`: Standard text comment (default if not specified)
  - `reaction`: Represents a reaction to content (like, emoji, etc.)
- Applications can use this field to implement specialized features
- Omitting this field defaults to standard comment behavior

### 4. Example Comment Inscription

```typescript
// Original comment
const comment = {
  topic: "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  content: "Great project!",
  version: "0x0",
  encoding: "utf8"
};

// Reaction to the above comment
const reaction = {
  topic: "0x123...", // Hash of the parent comment
  content: "👍",
  version: "0x0",
  encoding: "utf8",
  type: "reaction"
};

// Comment on a specific NFT
const nftComment = {
  topic: "0x742d35Cc6634C0532925a3b844Bc454e4438f44e:0x4d2", // Contract:TokenId (1234 in hex)
  content: "Love this NFT!",
  version: "0x0",
  encoding: "utf8"
};

// Data URL format
const dataUrl = `data:message/vnd.tic+json;rule=esip6,${JSON.stringify(comment)}`;
```

### 5. Inscription Process
1. Create a valid comment object following the schema
2. Convert the object to a JSON string
3. Create a data URL with the MIME type `message/vnd.tic+json;rule=esip6`
4. Submit the data URL as calldata in an Ethereum transaction following the Ethscriptions protocol

### 6. Comment Deletion
Comments can be marked as deleted by transferring the comment's Ethscription to the zero address:
- Zero address: `0x0000000000000000000000000000000000000000`

The transfer of a comment Ethscription to the zero address signals that the original author intends to delete their comment. While the comment data remains on-chain due to the immutable nature of blockchain data, indexers and applications should:
1. Mark these comments as deleted
2. Hide them from default views
3. Maintain the comment in the tree structure to preserve reply chains

Example deletion process:
```typescript
// Using the Ethscriptions protocol transfer functionality to mark comment as deleted
await transferEthscription({
  ethscriptionId: "0x123...", // The comment's ethscription ID
  to: "0x0000000000000000000000000000000000000000" // Zero address
});
```

### 7. Topic Format
Topics can be either a single hex string or multiple hex strings separated by colons:
```
0xHexString
```
or
```
0xHexString1:0xHexString2
```

Where:
- Each part must be a valid hex string with 0x prefix
- `:` is the delimiter between hex strings
- Example for NFTs: `0xContractAddress:0xTokenId` where TokenId is converted to hex

Example implementation for parsing topics:
```typescript
function parseTopic(topic: string) {
  if (topic.includes(':')) {
    const parts = topic.split(':');
    return {
      type: 'multi',
      parts: parts.map(part => part.toLowerCase())
    };
  }
  
  // Single hex string topic
  return {
    type: 'single',
    value: topic.toLowerCase()
  };
}
```

## Validation Rules
1. All required fields must be present
2. `version` must be a valid hex string
3. If `encoding` is present, it must be one of the specified EncodingType values
4. `topic` must be a valid hex string with 0x prefix
5. For multi-part topics: each part must be a valid hex string with 0x prefix
6. `content` must be properly encoded according to the specified encoding type (defaults to utf8)
7. If `type` is present, it must be one of the specified CommentType values
8. A comment should be considered deleted if its Ethscription has been transferred to the zero address

## Indexer Implementation Notes

### Topic Validation
Indexers should implement the following validation logic for topics:

```typescript
function validateTopic(topic: string): boolean {
  // Check if topic starts with 0x
  if (!topic.startsWith('0x')) return false;
  
  // Split by colon if multi-part
  const parts = topic.split(':');
  
  // Validate each part
  return parts.every(part => {
    // Check if part starts with 0x
    if (!part.startsWith('0x')) return false;
    
    // Check if remaining characters are valid hex
    const hexPart = part.slice(2);
    return /^[0-9a-fA-F]+$/.test(hexPart);
  });
}
```

### Topic Identification and Interpretation
The protocol treats all valid topics equally, regardless of their structure. Indexers should:

1. Validate that the topic follows the basic format (all parts are valid hex strings)
2. Store the topic as-is without making assumptions about its meaning
3. Allow applications to interpret the topic structure based on their needs

For example, these are all valid topics:
```
0x1234                                    // Single hex string
0xEOA:0xRandomHexString                  // EOA + random hex
0xContractAddress:0xTokenId              // NFT format
0xTransactionHash:0xEventIndex:0xData    // Transaction + event + data
0xAddress:0xTimestamp:0xExtraData        // Address + timestamp + data
```

Applications can implement their own interpretation logic:

```typescript
function interpretTopic(topic: string): TopicInterpretation {
  if (!validateTopic(topic)) return { type: 'invalid' };
  
  const parts = topic.split(':');
  
  // Single part topics
  if (parts.length === 1) {
    const value = parts[0];
    if (value.length === 42) { // 0x + 40 chars
      return { type: 'address', value };
    }
    if (value.length === 66) { // 0x + 64 chars
      return { type: 'transaction', value };
    }
    return { type: 'unknown', value };
  }
  
  // Multi-part topics
  if (parts.length === 2) {
    const [first, second] = parts;
    
    // NFT interpretation
    if (first.length === 42) { // 0x + 40 chars
      return { 
        type: 'nft',
        contractAddress: first,
        tokenId: second
      };
    }
    
    // Other interpretations can be added here
    return {
      type: 'multi',
      parts: [first, second]
    };
  }
  
  // More than 2 parts
  return {
    type: 'multi',
    parts
  };
}
```

### Additional Considerations
- The protocol itself makes no assumptions about the meaning of topics
- Applications are free to interpret topics based on their structure
- Indexers should store the raw topic and allow applications to implement their own interpretation logic
- Common interpretations (like NFTs) can be standardized by the community
- Token IDs should be converted to hex format before being used in topics
- Consider caching validation results for frequently used topics
- Implement proper error handling for malformed topics

## Notes
- TIC is in early alpha, and the specification may evolve as the protocol matures.
- Comments are immutable once inscribed
- Comments inherit all security and decentralization properties of the Ethscriptions protocol
- Comments can be nested by using the parent comment's hash as the topic
- Indexers should validate all comments against these protocol rules
- Future versions may introduce additional fields or functionality through version updates
- While comments can be marked as deleted, the underlying data remains on-chain due to the immutable nature of blockchain data
- Applications should respect deletion markers when displaying comments
- Deleted comments' replies should still be preserved and displayed

## References
- [Ethscriptions Protocol Specification](https://docs.ethscriptions.com/overview/protocol-specification)
