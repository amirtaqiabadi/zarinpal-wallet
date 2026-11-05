# Wallet schema

Postgres tables for the ZarinPal wallet. Names, columns, and status maps match the ledger this skill is based on. Prisma is the reference; emit the same columns if the host uses another migration tool.

Point every `User` relation at the host user model. Do not create a second users table. Add the back-relations on that model (names can match the host's style):

```prisma
model User {
  // existing fields stay
  teamMemberships TeamMember[]
  teamInvitesSent TeamInvite[]     @relation("TeamInviteInviter")
  membersInvited  TeamMember[]     @relation("TeamMemberInvitedBy")
  gatewayAttempts GatewayAttempt[] @relation("GatewayAttemptInitiator")
  transactions    Transaction[]    @relation("TransactionInitiator")
  refundRequests  RefundRequest[]
}
```

`accommodation_id` and `booking_id` are nullable and have no relation. Add a foreign key only when the host already has those tables.

`GatewayType` still lists `zibal` and `sep` so the enum matches. This skill writes only `zarinpal`. `TransactionType` keeps the full set so later ledger rows fit the same table. This skill inserts `gateway_credit` and `reverse` only.

Wallet display name is `team_members.display_name`, not a column on `teams`.

## Enums

```prisma
enum MemberRole {
  owner  @map("owner")
  admin  @map("admin")
  member @map("member")
  viewer @map("viewer")
}

enum TeamMemberStatus {
  pendingOwnerApproval @map("pending_owner_approval")
  active               @map("active")
  revoked              @map("revoked")
  left                 @map("left")
}

enum TransactionType {
  Deposit                @map("deposit")
  BonusDeposit           @map("bonus_deposit")
  GiftCardCredit         @map("gift_card_credit")
  GiftCredit             @map("gift_credit")
  DirectBankDeposit      @map("direct_bank_deposit")
  WalletTransferIn       @map("wallet_transfer_in")
  GatewayCredit          @map("gateway_credit")
  AnonymousGatewayCredit @map("anonymous_gateway_credit")
  ServiceCredit          @map("service_credit")
  RefundCancellation     @map("refund_cancellation")
  ServiceCancelCredit    @map("service_cancel_credit")
  Refund                 @map("refund")
  BonusExpiration        @map("bonus_expiration")
  ServiceDebit           @map("service_debit")
  AccommodationRenewal   @map("accommodation_renewal")
  UserVerification       @map("user_verification")
  Reverse                @map("reverse")
}

enum GatewayType {
  zarinpal @map("zarinpal")
  zibal    @map("zibal")
  sep      @map("sep")
}

enum GatewayAttemptStatus {
  pending         @map("10-pending")
  successful      @map("50-successful")
  reverse_pending @map("70-reverse_pending")
  failed          @map("80-failed")
  reversed        @map("90-reversed")
}

enum RefundStatus {
  pending    @map("10-pending")
  processing @map("30-processing")
  paid       @map("50-paid")
  rejected   @map("80-rejected")
  failed     @map("90-failed")
}
```

Raw SQL must compare the mapped strings (`10-pending`, `50-successful`, `70-reverse_pending`, `80-failed`, `90-reversed`). Prisma client code compares the enum names (`pending`, `successful`, and so on).

## Tables

```prisma
model Team {
  id        Int      @id @default(autoincrement())
  isActive  Boolean  @default(true) @map("is_active")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @default(now()) @updatedAt @map("updated_at")

  members         TeamMember[]
  invites         TeamInvite[]
  transactions    Transaction[]
  gatewayAttempts GatewayAttempt[]
  refundRequests  RefundRequest[]

  @@index([updatedAt])
  @@map("teams")
}

model TeamMember {
  id              Int              @id @default(autoincrement())
  teamId          Int              @map("team_id")
  userId          Int              @map("user_id")
  displayName     String?          @map("display_name")
  role            MemberRole       @default(member) @map("role")
  isActive        Boolean          @default(true) @map("is_active")
  status          TeamMemberStatus @default(active) @map("status")
  teamInviteId    Int?             @map("team_invite_id")
  invitedByUserId Int?             @map("invited_by_user_id")
  requestedAt     DateTime?        @map("requested_at")
  approvedAt      DateTime?        @map("approved_at")
  revokedAt       DateTime?        @map("revoked_at")
  leftAt          DateTime?        @map("left_at")
  createdAt       DateTime         @default(now()) @map("created_at")
  updatedAt       DateTime         @default(now()) @updatedAt @map("updated_at")

  team      Team        @relation(fields: [teamId], references: [id])
  user      User        @relation(fields: [userId], references: [id])
  teamInvite TeamInvite? @relation(fields: [teamInviteId], references: [id])
  invitedBy User?       @relation("TeamMemberInvitedBy", fields: [invitedByUserId], references: [id])

  @@unique([teamId, userId])
  @@index([teamId])
  @@index([userId])
  @@index([teamInviteId])
  @@index([status])
  @@map("team_members")
}

/// Structural compatibility. Do not implement invite redemption in this skill.
model TeamInvite {
  id            Int        @id @default(autoincrement())
  teamId        Int        @map("team_id")
  inviterUserId Int        @map("inviter_user_id")
  tokenHash     String     @unique @map("token_hash")
  role          MemberRole @default(viewer) @map("role")
  maxUses       Int        @default(10) @map("max_uses")
  expiresAt     DateTime   @map("expires_at")
  revokedAt     DateTime?  @map("revoked_at")
  createdAt     DateTime   @default(now()) @map("created_at")
  updatedAt     DateTime   @default(now()) @updatedAt @map("updated_at")

  team        Team         @relation(fields: [teamId], references: [id])
  inviter     User         @relation("TeamInviteInviter", fields: [inviterUserId], references: [id])
  memberships TeamMember[]

  @@index([teamId])
  @@index([inviterUserId])
  @@index([expiresAt])
  @@map("team_invites")
}

model GatewayAttempt {
  id                     Int                  @id @default(autoincrement())
  merchantCorrelationKey String               @unique @map("merchant_correlation_key") @db.VarChar(64)
  initiatorId            Int                  @map("initiator_id")
  teamId                 Int                  @map("team_id")
  accommodationId        Int?                 @map("accommodation_id")
  type                   GatewayType          @map("type")
  transactionType        String               @default("gateway_credit") @map("transaction_type") @db.VarChar(64)
  status                 GatewayAttemptStatus @default(pending) @map("status")
  callbackUrl            String?              @map("callback_url")
  sessionToken           String?              @map("session_token")
  referenceId            String?              @map("reference_id")
  trackingNumber         String?              @map("tracking_number")
  maskedPan              String?              @map("masked_pan")
  hashedPan              String?              @map("hashed_pan")
  salt                   String?              @map("salt")
  amount                 BigInt               @map("amount")
  fee                    BigInt               @map("fee")
  metadata               Json?                @map("metadata") @db.JsonB
  notes                  String?              @map("notes")
  createdAt              DateTime             @default(now()) @map("created_at")
  updatedAt              DateTime             @default(now()) @updatedAt @map("updated_at")

  initiator    User          @relation("GatewayAttemptInitiator", fields: [initiatorId], references: [id])
  team         Team          @relation(fields: [teamId], references: [id])
  transactions Transaction[]

  @@index([type])
  @@index([transactionType])
  @@index([status])
  @@index([referenceId])
  @@index([createdAt])
  @@index([updatedAt])
  @@index([initiatorId, teamId, status])
  @@index([accommodationId])
  @@map("gateway_attempts")
}

model Transaction {
  id                     Int             @id @default(autoincrement())
  initiatorId            Int             @map("initiator_id")
  teamId                 Int             @map("team_id")
  gatewayAttemptId       Int?            @map("gateway_attempt_id")
  accommodationId        Int?            @map("accommodation_id")
  bookingId              Int?            @map("booking_id")
  type                   TransactionType @map("type")
  amount                 BigInt          @map("amount")
  availableBalanceAfter  BigInt          @map("available_balance_after")
  restrictedBalanceAfter BigInt          @map("restricted_balance_after")
  metadata               Json?           @map("metadata") @db.JsonB
  createdAt              DateTime        @default(now()) @map("created_at")
  updatedAt              DateTime        @default(now()) @updatedAt @map("updated_at")

  initiator      User           @relation("TransactionInitiator", fields: [initiatorId], references: [id])
  team           Team           @relation(fields: [teamId], references: [id])
  gatewayAttempt GatewayAttempt? @relation(fields: [gatewayAttemptId], references: [id])
  refundRequest  RefundRequest?

  @@unique([type, gatewayAttemptId])
  @@index([initiatorId])
  @@index([teamId])
  @@index([gatewayAttemptId])
  @@index([accommodationId])
  @@index([bookingId])
  @@index([type])
  @@index([createdAt])
  @@index([updatedAt])
  @@index([teamId, createdAt(sort: Desc)])
  @@map("transactions")
}

model RefundRequest {
  id            Int          @id @default(autoincrement())
  initiatorId   Int          @map("initiator_id")
  teamId        Int          @map("team_id")
  transactionId Int          @unique @map("transaction_id")
  cardNumber    String       @map("card_number") @db.Char(16)
  amount        BigInt
  status        RefundStatus @default(pending)
  metadata      Json?        @map("metadata") @db.JsonB
  createdAt     DateTime     @default(now()) @map("created_at")
  updatedAt     DateTime     @updatedAt @map("updated_at")

  initiator   User        @relation(fields: [initiatorId], references: [id])
  team        Team        @relation(fields: [teamId], references: [id])
  transaction Transaction @relation(fields: [transactionId], references: [id])

  @@index([teamId])
  @@index([initiatorId])
  @@index([status])
  @@map("refund_requests")
}
```

## Column meaning for ZarinPal

| Column | Stores |
| --- | --- |
| `gateway_attempts.session_token` | `authority` from the payment request |
| `gateway_attempts.reference_id` | `ref_id` after a successful verify |
| `gateway_attempts.masked_pan` | masked `card_pan` from verify |
| `gateway_attempts.hashed_pan` | `card_hash` from verify |
| `gateway_attempts.fee` | fee from the gateway response, else `0` |
| `gateway_attempts.salt` | 32 random bytes, Base64 |
| `gateway_attempts.merchant_correlation_key` | 64-character Crockford Base32 from a CSPRNG |
| `transactions.available_balance_after` | available balance after this row |
| `transactions.restricted_balance_after` | restricted balance after this row |

Available balance is the latest `available_balance_after` for that `team_id`, or `0` when the team has no rows. Never keep a second mutable balance counter.

`refund_requests.card_number` is the 16-digit destination card for a manual refund request. Validate digits only. Do not log it. Mask it in list responses (first 6 and last 4). Gateway attempts never store a full PAN.

`team_invites.token_hash` stores a hash of the invite token. The skill does not implement invites.

## Personal wallet

When the caller omits `teamId` and has no active membership, create one `teams` row and one `team_members` row in the same transaction: `display_name` `Personal wallet`, `role` `owner`, `status` `active`, `is_active` true.

When the caller sends `teamId`, the initiator must be an active `owner` of that team. Otherwise respond `403` with `TEAM_FORBIDDEN`.
