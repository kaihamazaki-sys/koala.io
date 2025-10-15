# koala.io


# Developer Workflow Documentation

## System Overview
This platform connects Small & Medium Businesses (SMBs) with influencers for campaign collaborations. SMBs create campaigns, influencers apply, and payments flow through an escrow-like wallet system with Stripe integration.

---

## 1. Authentication & Role Assignment

**Flow:**
- Users sign up via email/password through Supabase Auth
- On signup, a trigger (`handle_new_user`) creates a profile in `users` table with default role "SMB"
- A separate `user_roles` table maintains role security (prevents privilege escalation)
- After login, users are redirected based on role:
  - SMB → `/smb-dashboard`
  - Influencer → `/influencer-dashboard`

**Key Components:**
- `src/pages/Auth.tsx` - Login/signup UI
- `src/hooks/useUserRole.ts` - Secure role fetching via `user_roles` table
- Supabase Auth with session persistence in localStorage

---

## 2. SMB Dashboard Overview

**Purpose:** Central hub for SMBs to manage campaigns, view applications, handle payments, and communicate.

**Core Sections:**
- **Campaign List:** Displays all campaigns created by SMB (draft, live, completed)
- **Wallet Balance:** Shows available funds and transaction history
- **Pending Applications:** List of influencers who applied to campaigns
- **Chat Access:** Direct messaging with approved influencers

**Data Sources:**
- `campaigns` table (filtered by `user_id`)
- `campaign_applicants` table (joined with campaigns)
- `smb_wallet_balances` table (balance_cents converted to dollars)
- `wallet_ledger` table (transaction history)
- `conversations` + `messages` tables (chat threads)

**Navigation:**
- Tab-based layout switching between Campaigns, Applications, Wallet, Chat
- Modal overlays for detailed views (campaign details, influencer profiles)

---

## 3. Creating a Campaign (Deal Creation Flow)

**Form Fields:**
- Campaign Name, Description, Hashtags
- Budget (total campaign budget)
- Influencer Budget (per-influencer payout)
- Max Influencers (optional capacity limit)
- Start Date, End Date
- Content Types, Guidelines, Niche
- Privacy (Public/Private - private requires invitations)

**Flow:**
1. User fills form in `/create-campaign`
2. Validation: Budget > 0, dates valid, required fields present
3. Insert into `campaigns` table with status = "Draft"
4. Redirect to `/campaigns` to see newly created campaign

**Key Files:**
- `src/pages/CreateCampaign.tsx` - Form UI and submission logic
- `campaigns` table - Stores all campaign data

---

## 4. Campaign Management

**Actions:**
- **View:** Display campaign details in modal (name, budget, dates, applicants)
- **Edit:** Navigate to `/edit-campaign/:id` to modify draft campaigns
- **Publish:** Change status from "Draft" → "Live" (requires sufficient wallet balance)
  - Edge function `rpc_campaign_publish` reserves budget from wallet
  - Deducts funds and creates ledger entry
- **Complete:** Automatically triggered when end_date passes (via `campaign-scheduler` edge function)
  - Refunds unused budget back to wallet

**Status Transitions:**
- Draft (initial) → Live (published) → Completed (ended/budget exhausted)

**Visibility:**
- SMBs see all their campaigns
- Influencers see only "Live" public campaigns or campaigns they were invited to

**Real-time:**
- Campaign status changes sync via `sync_campaign_status_from_campaigns` trigger
- Updates reflected immediately in `campaign_analytics` table

---

## 5. Influencer Applications Review

**Flow:**
1. SMB views "Applications" tab on dashboard
2. List shows all applicants from `campaign_applicants` table (status: pending, approved, rejected, invited)
3. Clicking applicant opens full profile modal (`InfluencerFullProfile.tsx`)
4. SMB can:
   - **Approve:** Sets status = "approved" → triggers match creation → auto-creates chat conversation
   - **Reject:** Sets status = "rejected"
   - **Invite:** Manually invite influencers to private campaigns (status = "invited")

**Key Tables:**
- `campaign_applicants` - Application records (user_id, campaign_id, status, message)
- `users` - Influencer profile data (followers, bio, portfolio)
- `influencer_stats_public` - Aggregate stats (views, engagement rate, campaigns completed)

**Profile Visibility:**
- RLS policies ensure SMBs can only view profiles of influencers who applied to their campaigns

---

## 6. Chat & Communication System

**Architecture:**
- **Conversations Table:** Stores conversation metadata (participants, last_message_at)
- **Conversation Participants:** Junction table linking users to conversations
- **Messages Table:** Individual message records

**Thread Creation:**
- Auto-created when SMB approves an application
- Function `get_or_create_dm(other_user_id)` handles duplicate prevention
- Links conversation to specific campaign_id (optional)

**Real-time Updates:**
- Supabase Realtime channels listen for new messages
- UI updates instantly when messages arrive
- Unread count tracked via `message_reads` table

**Key Files:**
- `src/pages/ChatBusiness.tsx` / `ChatCreator.tsx` - Chat UI components
- `conversations`, `messages`, `message_reads` tables

---

## 7. Wallet & Payments System

**Deposit Flow:**
1. SMB clicks "Add Funds" on wallet page
2. Frontend calls edge function `create-deposit-session`
3. Edge function creates Stripe Checkout Session
4. User completes payment on Stripe
5. Stripe webhook (`stripe-webhook` edge function) verifies payment
6. On success:
   - Inserts record in `wallet_ledger` (type: "deposit", status: "completed")
   - Updates `smb_wallet_balances.balance_cents`

**Balance Display:**
- Fetched from `smb_wallet_balances` table (stored as cents, displayed as dollars)
- Transaction history from `wallet_ledger` (deposits, campaign funds, refunds, payouts)

**Key Components:**
- `src/pages/PaymentsSMB.tsx` - Deposit UI
- Edge functions: `create-deposit-session`, `stripe-webhook`
- Tables: `smb_wallet_balances`, `wallet_ledger`

---

## 8. Payout Authorization Flow

**Process:**
1. Influencer uploads content to campaign (video URL to `campaign_analytics`)
2. SMB reviews deliverable on dashboard
3. SMB manually updates views/likes/engagement metrics
4. SMB verifies content meets requirements
5. SMB clicks "Approve Payout"
6. Edge function `request-withdrawal` (or similar):
   - Deducts from campaign's `remaining_budget`
   - Creates payout record in `wallet_ledger` for influencer
   - Initiates Stripe Connect transfer to influencer's connected account
7. Influencer receives funds in their bank account (via Stripe)

**Payout Calculation:**
- Based on `influencer_budget` field from campaign
- Can be adjusted based on performance metrics

**Key Tables:**
- `campaign_analytics` - Stores influencer submissions (video_url, views, likes, earnings)
- `wallet_ledger` - Records payout transactions
- `campaigns.remaining_budget` - Tracks available funds

---

## 9. Notifications & State Feedback

**Notification Types:**
- **Campaign Application:** SMB notified when influencer applies
- **Campaign Invitation:** Influencer notified when invited to private campaign
- **New Message:** User notified of unread messages (count tracked)
- **Payout Complete:** Influencer notified when payment transfers

**Implementation:**
- Database triggers create records in `notifications` table
- Examples: `notify_campaign_application`, `notify_new_message`, `notify_campaign_invitation`
- Frontend polls or uses realtime subscriptions to display notifications
- Toast notifications via Sonner library for success/error feedback

**UI Components:**
- `src/components/NotificationsDropdown.tsx` - Bell icon with unread count
- Toast notifications throughout app for immediate feedback

---

## 10. Admin Oversight (Optional)

**Current Status:** Limited admin functionality

**Potential Features:**
- View all users, campaigns, transactions
- Manual balance adjustments for testing
- Force campaign status changes
- View system-wide analytics
- Resolve disputes between SMBs and influencers

**Implementation Notes:**
- Would require "admin" role in `user_roles` table
- RLS policies would need admin bypass clauses
- Separate admin dashboard UI
- Audit logging for admin actions

**Current Workaround:**
- Direct database access via Supabase dashboard
- Edge functions with service_role key for manual operations

---

## Database Schema Summary

**Core Tables:**
- `users` - User profiles (SMB & Influencer data)
- `user_roles` - Role assignments (security-critical)
- `campaigns` - Campaign definitions
- `campaign_applicants` - Application records
- `campaign_analytics` - Influencer submissions & metrics
- `conversations` + `messages` - Chat system
- `smb_wallet_balances` - SMB fund balances
- `wallet_ledger` - All financial transactions
- `notifications` - User notifications

**Key Enums:**
- `app_role`: "Influencer" | "SMB"
- `campaign_status_enum`: "Draft" | "Live" | "Completed"
- `transaction_type_enum`: "deposit" | "campaign_fund" | "influencer_payout" | "campaign_refund"
- `transaction_status_enum`: "pending" | "completed" | "failed"

---

## Security Highlights

1. **Row-Level Security (RLS):** All tables have policies restricting data access
2. **Role Verification:** Uses `user_roles` table (not `users.role`) to prevent privilege escalation
3. **Security Definer Functions:** Role checks bypass RLS to prevent recursive issues
4. **Stripe Webhooks:** Signature verification ensures legitimate payments
5. **Service Role Edge Functions:** Critical operations (payouts, deposits) use elevated permissions

---

## Edge Functions Summary

- `create-deposit-session` - Creates Stripe Checkout for deposits
- `stripe-webhook` - Processes Stripe payment events
- `request-withdrawal` - Handles influencer payout requests
- `campaign-scheduler` - Auto-completes expired campaigns
- `rpc_campaign_publish` - Publishes campaign (reserves budget)
- `rpc_campaign_deduct` - Deducts from campaign budget for payouts
- `create-onboarding-link` - Stripe Connect onboarding for influencers
- `check-payout-status` - Verifies influencer payout setup

---

*This document will be expanded with detailed subsections for triggers, RLS policies, webhook flows, and edge function logic in subsequent updates.*
