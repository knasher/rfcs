# RFC: Private Content in ATProto

**RFC:** atproto-001  
**Author:** Dave Nash  
**Date:** July 19, 2025  
**Version:** 0.1  
**Status:** Draft

- RFC: Private Content in ATProto - [Codeberg](https://codeberg.org/davenash/rfcs/src/branch/main/atproto/001-private-content.md)
- RFC: Private Content in ATProto - [GitHub](https://github.com/knasher/rfcs/blob/main/atproto/001-private-content.md)

## Summary

This RFC proposes a mechanism for private content in ATProto by modifying the data flow between Personal Data Servers (PDSes) and AppViews. Instead of transmitting private content through the Firehose, PDSes would send metadata notifications for interested AppViews to receive, which can then fetch the actual content directly from the PDS using existing OAuth authentication.

## Motivation

ATProto currently lacks support for private content, which is a fundamental feature expected by users migrating from other social platforms. Users need the ability to:

- Post content visible only to approved followers
- Share content with specific mentioned users
- Control granular privacy settings for individual posts

The current architecture broadcasts all content through the Firehose to all AppViews, making privacy impossible without significant architectural changes. This limitation prevents ATProto from serving use cases where users want more control over their content visibility.

## Detailed Design

### High-Level Flow

1. **Content Creation**: User creates content marked as private on their PDS
2. **Metadata Broadcast**: PDS sends metadata notification through Firehose (not the actual content)
3. **Authorization Check**: AppView receives metadata and determines if it holds valid access tokens with relevant permissions for the content
4. **Direct Fetch**: Authorized AppView fetches actual content directly from the PDS using OAuth
5. **Content Handling**: AppView processes and displays content according to privacy settings

### Technical Implementation

#### 1. New Lexicon for Private Content Metadata

```json
{
  "lexicon": 1,
  "id": "com.atproto.repo.privateContentNotification",
  "defs": {
    "main": {
      "type": "object",
      "properties": {
        "repo": { "type": "string" },
        "collection": { "type": "string" },
        "rkey": { "type": "string" },
        "cid": { "type": "string" },
        "contentType": { "type": "string" },
        "visibility": {
          "type": "string",
          "knownValues": ["followers", "mentioned", "custom"]
        },
        "timestamp": { "type": "string", "format": "datetime" }
      }
    }
  }
}
```

#### 2. PDS Modifications

- **Content Storage**: PDSes store private content normally in their repositories
- **Firehose Filtering**: Before broadcasting to Firehose, check content for privacy markers
- **Metadata Generation**: Generate and send metadata notifications instead of full content
- **Direct Access Endpoint**: Provide authenticated endpoint for AppViews to fetch private content

#### 3. AppView Integration

- **Metadata Processing**: Listen for private content notifications on Firehose
- **Authorization Check**: Check if the AppView holds valid access tokens with appropriate permissions for the user/content
- **Content Fetching**: Use existing OAuth tokens to fetch private content from origin PDS
- **Caching Strategy**: Implement appropriate caching respecting privacy constraints

#### 4. OAuth Integration and Scope Extensions

The existing OAuth framework provides the foundation, but requires new scopes for explicit private content authorization:

**New OAuth Scopes:**
- `atproto:read:private` - General permission to access user's private content
- `atproto:read:private:followers` - Access to followers-only content
- `atproto:read:private:mentioned` - Access to content where user is mentioned
- `atproto:read:private:custom` - Access to custom privacy level content (may require additional parameters)
- `atproto:write:private` - Permission to create and modify private content on user's behalf

**Authorization Flow:**
1. AppViews request appropriate private content scopes (both read and write) during user authorization
2. Users explicitly consent to sharing private content with the AppView and allowing it to create private content
3. PDSes validate scope permissions before serving private content or accepting private content writes
4. Tokens can be revoked to immediately cut off private content access

**Scope Validation:**
- PDSes must validate that requesting AppViews hold appropriate scopes before serving private content
- Scopes should be granular enough to allow users to authorize different privacy levels separately
- Existing token refresh and revocation mechanisms work unchanged

### Privacy Levels

1. **Followers Only**: Content visible to approved followers
2. **Mentioned Users**: Content visible only to mentioned users
3. **Custom Lists**: Content visible to specific user-defined groups

## Drawbacks

- **Increased Complexity**: AppViews must implement additional fetching and authorization logic
- **Performance Impact**: Direct PDS requests may be slower than Firehose consumption
- **Trust Model**: AppViews must be trusted by users to handle private content appropriately and not redistribute it publicly
- **Caching Challenges**: Private content caching becomes more complex to maintain privacy

## Alternatives

### Alternative 1: Encrypted Firehose Content
Encrypt private content before broadcasting, with keys shared only to authorized AppViews.
- **Rejected because**: Key distribution is complex and doesn't scale well

### Alternative 2: Private Content Relays
Create separate relay networks for private content.
- **Rejected because**: Adds significant infrastructure complexity and cost

## Unresolved Questions

1. **Caching Strategy**: What caching policies should AppViews implement for private content?
2. **Content Administration**: How do users manage private content they've posted if their chosen AppView failed to retrieve it? Is this a new problem or does it already exist with public content?
3. **Privacy Level Storage**: Should privacy levels be specified in OAuth scopes, request parameters, or request body when creating content? Privacy level should be stored as PDS metadata rather than embedded in the content record itself.
4. **Trust Indicators**: Should there be mechanisms to help users evaluate AppView trustworthiness?
5. **Migration Path**: How do existing public posts transition if a user changes their privacy settings?

## Future Possibilities

- **Granular Permissions**: More sophisticated privacy controls (time-based, location-based, etc.)
- **Private Groups**: Support for private group conversations and content
- **End-to-End Encryption**: Layer E2E encryption on top of this privacy mechanism
- **Privacy Analytics**: Allow users to see who has accessed their private content

## Implementation Plan

1. **Phase 1**: Define lexicon for private content metadata and specify new OAuth scopes
2. **Phase 2**: Update OAuth implementation to support private content scopes
3. **Phase 3**: Implement PDS-side filtering and metadata broadcasting  
4. **Phase 4**: Create reference AppView implementation with private content support
5. **Phase 5**: Test with volunteer PDSes and AppViews
6. **Phase 6**: Broader rollout and ecosystem adoption

## Conclusion

This proposal provides a privacy mechanism for ATProto that leverages existing OAuth infrastructure while maintaining the protocol's decentralized architecture. By using metadata notifications and direct fetching, users gain content privacy without sacrificing the benefits of federation.

## Feedback and Discussion

This RFC is open for community review and feedback. Key areas where input would be particularly valuable:

- OAuth scope design and granularity
- Privacy level specification mechanism
- Implementation complexity for AppView developers
- Security and trust model considerations

Please provide feedback through [Codeberg issues/discussions/etc.].