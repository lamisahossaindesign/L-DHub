# Firestore Security Specification

## 1. Data Invariants
- **Users**: Must have `name`, `email`, `role`, and `user_id`. `user_id` must follow `LDH-USER-[0-9]{4}`. `role` defaults to `user`.
- **Services**: Must have `title`, `icon`, `description`. `packages` must be a list of objects with `title` and `price`.
- **Orders**: Must link to a valid `user_id`. Status transition must be linear (pending -> active -> completed/cancelled).
- **Settings**: System-wide configuration, only editable by verified admins.
- **Immutability**: `createdAt` and `ownerId` (or `user_id` in profile) cannot be changed after creation.

## 2. The "Dirty Dozen" Payloads

1. **Identity Theft**: `update /users/victim { role: 'admin' }` (Denied: Only owner/admin write, role immutable for owner)
2. **Shadow Field injection**: `create /services/1 { title: 'Hack', shadow: 'evil' }` (Denied: Strict key validation)
3. **ID Poisoning**: `get /users/VERY_LONG_STRING_MORE_THAN_128_CHARS` (Denied: isValidId check)
4. **Denial of Wallet**: `create /users/1 { name: 'A' * 1000000 }` (Denied: Size limits)
5. **Privilege Escalation**: `update /users/me { role: 'admin' }` (Denied: Role immutable for non-admin)
6. **Relational Orphan**: `create /orders/1 { service_name: 'FakeService' }` (Denied: exists() check for service)
7. **Temporal Fraud**: `create /orders/1 { createdAt: 9999999999999 }` (Denied: server timestamp check)
8. **PII Leak**: `list /users` as non-admin (Denied: users collection list query enforcement)
9. **Status Shortcut**: `update /orders/1 { status: 'completed' }` from 'pending' (Denied: state transition logic)
10. **Ghost User ID**: `create /users/1 { user_id: 'BAD-FORMAT' }` (Denied: Regex check)
11. **Email Spoofing**: Admin actions with `email_verified: false` (Denied: email_verified check)
12. **Admin Lockdown Bypass**: Editing `settings` as a registered user (Denied: isAdmin check)

## 3. The Test Runner (Mock)
See `firestore.rules.test.ts` (This is high-level logic for the agent turn). 
