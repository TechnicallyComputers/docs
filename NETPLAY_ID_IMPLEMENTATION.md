# Netplay ID Implementation

## Overview

This document describes the implementation of authentic netplay IDs in RomM, which allows users to have unique identifiers for netplay sessions that are separate from their login usernames.

## Background

Previously, RomM used user login usernames as identifiers in netplay sessions. This implementation introduces dedicated netplay IDs that:

- Are separate from login credentials
- Can be customized by users
- Support future federation capabilities
- Maintain full backwards compatibility

## Implementation Details

### Database Schema Changes

#### Migration: `0064_add_netplayid.py`
```sql
ALTER TABLE users ADD COLUMN netplayid VARCHAR(255) NULL UNIQUE;
CREATE INDEX ix_users_netplayid ON users(netplayid);
```

**Backwards Compatibility**: The column is nullable, so existing databases continue to work without modification.

#### Model Updates
```python
# In backend/models/user.py
netplayid: Mapped[str | None] = mapped_column(
    String(length=TEXT_FIELD_LENGTH), nullable=True, unique=True, index=True
)

# Kiosk user also updated
return cls(
    id=-1,
    username="kiosk",
    netplayid="kiosk",  # Added for consistency
    # ... other fields
)
```

### API Changes

#### User Profile Updates
- **Endpoint**: `PUT /api/users/{id}`
- **New Field**: `netplayid` in UserForm
- **Validation**:
  - 3-32 characters
  - Alphanumeric + underscore/dash only
  - Unique across all users
  - Optional (can be null/empty)

#### Database Handler
```python
# Added method in backend/handler/database/users_handler.py
@begin_session
def get_user_by_netplayid(
    self,
    netplayid: str,
    session: Session = None,
) -> User | None:
    query = select(User).filter(User.netplayid == netplayid)
    return session.scalar(query.limit(1))
```

### SFU Authentication Updates

#### Token Generation
```python
# In backend/endpoints/sfu.py mint_sfu_token()
sfu_identifier = user.netplayid or user.username  # Fallback for backwards compatibility

token_data = {
    "sub": sfu_identifier,  # Uses netplayid if available
    # ... other claims
}
```

#### Token Verification
```python
# SFU verify endpoint now returns netplay_username from database
user = db_user_handler.get_user_by_netplayid(sub)
if not user:
    user = db_user_handler.get_user_by_username(sub)  # Backwards compatibility

netplay_username = user.netplayid if user else None
return SFUVerifyResponse(sub=sub, netplay_username=netplay_username)
```

### Frontend Changes

#### User Profile Page
- **Location**: `frontend/src/views/Settings/UserProfile.vue`
- **Conditional Display**: Only shows when `EJS_NETPLAY_ENABLED = true`
- **Field Position**: Between password and email fields in Account Details section

#### Validation Rules
```typescript
// In frontend/src/stores/users.ts
const netplayIdLength = (v: string) =>
  (v.length >= 3 && v.length <= 32) || i18n.global.t("settings.netplay-id-length");

const netplayIdChars = (v: string) =>
  /^[a-zA-Z0-9_-]*$/.test(v) || i18n.global.t("settings.netplay-id-chars");

netplayIdRules: [
  (v: string) => !v || netplayIdLength(v), // Optional field
  (v: string) => !v || netplayIdChars(v),  // Only validate if not empty
]
```

#### TypeScript Types
Updated generated types in:
- `frontend/src/__generated__/models/UserSchema.ts`
- `frontend/src/__generated__/models/UserForm.ts`

## Backwards Compatibility

### Database Level
- ✅ Existing users have `NULL` netplayid (no migration data loss)
- ✅ Old RomM versions can read the database (unknown column is ignored)
- ✅ No breaking schema changes

### API Level
- ✅ Existing API calls work unchanged
- ✅ New `netplayid` field is optional in requests
- ✅ SFU tokens work with fallback logic

### Frontend Level
- ✅ Feature is hidden when `EJS_NETPLAY_ENABLED = false`
- ✅ Existing profile page functionality unchanged
- ✅ TypeScript types are backwards compatible

### SFU Level
- ✅ Authentication works with username fallback
- ✅ Existing tokens continue to function
- ✅ No changes required to SFU server logic

## Configuration

### Environment Variables
```bash
# Enable netplay ID feature in UI
EJS_NETPLAY_ENABLED=true
```

### Default Behavior
- **When disabled**: Netplay ID field is hidden, system uses usernames
- **When enabled**: Users can set custom netplay IDs, fallback to username

## Future Federation Support

This implementation is designed to support cross-instance netplay federation:

### Database Schema Ready
```sql
-- Can store federated IDs like "federated-romm.com:user123"
netplayid VARCHAR(255) UNIQUE
```

### Authentication Architecture
```python
# Future federated identifier format
federated_id = f"{issuer}:{user_id}"
# Examples:
# "romm:sfu:localuser" (local)
# "federated.com:sfu:remoteuser" (federated)
```

### SFU Server Extensions Needed
The SFU server can be extended to:
1. Accept multiple trusted issuers
2. Route federated users to appropriate instances
3. Handle cross-instance communication protocols

## Deployment Guide

### 1. Database Migration
```bash
# Run from backend directory
cd backend
alembic upgrade head
```

### 2. Environment Configuration
```bash
# Add to your RomM environment
EJS_NETPLAY_ENABLED=true
```

### 3. Frontend Rebuild
```bash
# Rebuild frontend to pick up type changes
npm run build
# or
npm run dev
```

### 4. Verification
1. Check user profile page shows "Netplay ID" field
2. Test setting and updating netplay IDs
3. Verify SFU authentication still works
4. Confirm backwards compatibility with existing users

## Security Considerations

### Input Validation
- Server-side validation prevents injection attacks
- Client-side validation provides immediate feedback
- Length and character restrictions prevent abuse

### Uniqueness Constraints
- Database-level UNIQUE constraint on netplayid
- Duplicate prevention at API level
- Case-sensitive uniqueness (follows SQL standard)

### Privacy
- Netplay IDs are public identifiers for netplay sessions
- Separate from private login credentials
- Users can change IDs (with appropriate validation)

## Testing Scenarios

### Backwards Compatibility
- ✅ Existing user without netplayid can authenticate
- ✅ Old SFU tokens continue working
- ✅ Database queries work with NULL values

### New Functionality
- ✅ User can set netplay ID via profile page
- ✅ Validation prevents invalid IDs
- ✅ Uniqueness prevents duplicate IDs
- ✅ SFU uses netplay ID for authentication

### Edge Cases
- ✅ Empty string clears netplay ID
- ✅ Username fallback when netplay ID not set
- ✅ Migration doesn't affect existing data

## Troubleshooting

### Common Issues

**Migration Fails**
```bash
# Check database permissions
# Ensure no duplicate netplayid values exist
# Verify alembic is properly configured
```

**Frontend Doesn't Show Field**
```bash
# Check EJS_NETPLAY_ENABLED=true
# Clear browser cache
# Rebuild frontend
```

**SFU Authentication Fails**
```bash
# Check token generation uses correct identifier
# Verify database has netplayid values
# Check SFU server logs for authentication errors
```

## Related Files

### Backend
- `backend/alembic/versions/0064_add_netplayid.py` - Database migration
- `backend/models/user.py` - User model updates
- `backend/endpoints/user.py` - API validation
- `backend/endpoints/sfu.py` - Token generation/verification
- `backend/handler/database/users_handler.py` - Database queries

### Frontend
- `frontend/src/views/Settings/UserProfile.vue` - Profile page UI
- `frontend/src/stores/users.ts` - Validation rules
- `frontend/src/__generated__/models/UserSchema.ts` - TypeScript types
- `frontend/src/__generated__/models/UserForm.ts` - Form types

### Configuration
- Environment variable: `EJS_NETPLAY_ENABLED`
- Conditional feature display based on netplay support

This implementation provides a solid foundation for user-controlled netplay identities while maintaining full backwards compatibility and preparing for future federation capabilities.