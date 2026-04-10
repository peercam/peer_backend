# Database Schema Overview

PostgreSQL database. All timestamps are `TIMESTAMP WITHOUT TIME ZONE` defaulting to `CURRENT_TIMESTAMP`. UUIDs are used as primary keys throughout.

---

## Entity-Relationship Summary

```
users ──┬── users_info          (1:1)
        ├── user_preferences    (1:1)
        ├── user_referral_info  (1:1)
        ├── wallett             (1:1)
        ├── dailyfree           (1:1)
        ├── access_tokens       (1:1)
        ├── refresh_tokens      (1:1)
        ├── invisibles          (1:1)
        ├── eligibility_token   (1:N)
        ├── follows             (N:M self-ref)
        ├── user_block_user     (N:M self-ref)
        ├── posts ──┬── post_info            (1:1)
        │           ├── posts_media          (1:N)
        │           ├── post_tags ── tags    (N:M)
        │           ├── comments ── comment_info (1:1)
        │           ├── user_post_likes      (N:M)
        │           ├── user_post_dislikes   (N:M)
        │           ├── user_post_views      (N:M)
        │           ├── user_post_saves      (N:M)
        │           ├── user_post_shares     (N:M)
        │           ├── user_post_reports    (N:M)
        │           ├── user_post_comments   (1:1 per comment)
        │           ├── advertisements       (1:N)
        │           │   ├── advertisements_log  (1:N)
        │           │   └── advertisements_info (1:1)
        │           └── gems                 (1:N)
        ├── chats ──┬── chatparticipants (N:M)
        │           ├── chatmessages     (1:N)
        │           ├── user_chat_status (N:M)
        │           └── newsfeed         (1:1)
        ├── wallet / logwins             (1:N)
        ├── transactions                 (1:N)
        ├── shop_orders                  (1:N)
        ├── user_reports                 (1:N)
        ├── logdata / logdaten           (1:N)
        └── token_holders / password_resets / password_reset_requests
```

---

## Tables

### Users & Identity

#### `users`
Core user account table.

| Column | Type | Notes |
|--------|------|-------|
| uid | UUID | **PK** |
| email | VARCHAR(249) | UNIQUE, NOT NULL |
| username | VARCHAR(33) | NOT NULL |
| password | VARCHAR(255) | NOT NULL (argon2id) |
| status | SMALLINT | Default 0 |
| verified | SMALLINT | Default 0 |
| slug | INTEGER | NOT NULL; composite unique with username |
| roles_mask | INTEGER | Default 0 (bitmask) |
| ip | INET | Registration IP |
| img | VARCHAR(100) | Profile image path |
| biography | TEXT | |
| visibility_status | VARCHAR(25) | Default `'normal'` (added by moderation migration) |
| updatedat | TIMESTAMP | |
| createdat | TIMESTAMP | |

**Indexes:** uid, email, (username, slug)

#### `users_info`
Extended user profile counters and metadata. 1:1 with `users`.

| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**, FK → users(uid) CASCADE |
| liquidity | NUMERIC(30,10) | Token balance, default 0 |
| amountposts | INTEGER | |
| amountfollower | INTEGER | |
| amountfollowed | INTEGER | |
| amountfriends | INTEGER | |
| amountblocked | INTEGER | |
| isprivate | SMALLINT | |
| invited | UUID | FK → users(uid) CASCADE (inviter) |
| phone | VARCHAR(21) | |
| pkey | VARCHAR(44) | |
| reports | INTEGER | Default 0 |
| totalreports | INTEGER | Default 0 |
| count_content_moderation_dismissed | INTEGER | Default 0 |
| updatedat | TIMESTAMP | |

#### `user_preferences`
Per-user settings.

| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**, FK → users(uid) CASCADE |
| content_filtering_severity_level | SMALLINT | NULL or 0–10 |
| onboardingsWereShown | JSONB | Default `'[]'` |
| updatedat | TIMESTAMP | |

#### `user_referral_info`
Referral link data.

| Column | Type | Notes |
|--------|------|-------|
| uid | UUID | **PK**, FK → users(uid) CASCADE |
| referral_link | VARCHAR(255) | NOT NULL |
| qr_code_url | VARCHAR(255) | |
| referral_uuid | UUID | UNIQUE, NOT NULL |
| createdat | TIMESTAMP | |

#### `invisibles`
Users in invisible mode.

| Column | Type | Notes |
|--------|------|-------|
| invis_id | UUID | **PK**, FK → users(uid) CASCADE |

---

### Authentication & Tokens

#### `access_tokens`
| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**, FK → users(uid) CASCADE |
| access_token | TEXT | |
| createdat | INTEGER | Unix epoch |
| expiresat | INTEGER | Unix epoch |

#### `refresh_tokens`
| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**, FK → users(uid) CASCADE |
| refresh_token | TEXT | |
| createdat | INTEGER | Unix epoch |
| expiresat | INTEGER | Unix epoch |

#### `token_holders`
Email verification tokens.

| Column | Type | Notes |
|--------|------|-------|
| token | VARCHAR(128) | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| attempt | SMALLINT | Default 0 |
| expiresat | INTEGER | ≥ 0 |
| collected | SMALLINT | Default 0 |
| updatedat | TIMESTAMP | |

#### `password_resets`
| Column | Type | Notes |
|--------|------|-------|
| token | VARCHAR(128) | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| attempt | SMALLINT | |
| expiresat | INTEGER | |
| collected | SMALLINT | |
| updatedat | TIMESTAMP | |

#### `password_reset_requests`
| Column | Type | Notes |
|--------|------|-------|
| user_id | UUID | FK → users(uid) CASCADE |
| token | VARCHAR(255) | |
| collected | BOOLEAN | Default false |
| attempt_count | INTEGER | Default 1 |
| expires_at | TIMESTAMP | |
| last_attempt | TIMESTAMP | |
| updatedat | TIMESTAMP | |

#### `eligibility_token`
| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | FK → users(uid) CASCADE |
| token | TEXT | |
| status | VARCHAR(33) | Default `'NO_FILE'` |
| expiresat | TIMESTAMP | |
| createdat | TIMESTAMP | |

---

### Social Graph

#### `follows`
| Column | Type | Notes |
|--------|------|-------|
| followerid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| followedid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| createdat | TIMESTAMP | |

#### `user_block_user`
| Column | Type | Notes |
|--------|------|-------|
| blockerid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| blockedid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| createdat | TIMESTAMP | |

---

### Posts & Content

#### `posts`
| Column | Type | Notes |
|--------|------|-------|
| postid | UUID | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| feedid | UUID | FK → newsfeed(feedid) CASCADE, nullable |
| contenttype | VARCHAR(13) | CHECK IN (image, text, video, audio, imagegallery, videogallery, audiogallery, secretgallery) |
| title | TEXT | |
| mediadescription | TEXT | |
| media | TEXT | JSON array of media objects |
| cover | TEXT | |
| options | TEXT | |
| status | SMALLINT | Default 0 |
| visibility_status | VARCHAR(25) | Default `'normal'` |
| createdat | TIMESTAMP | |

**Indexes:** userid, postid, feedid, createdat

#### `post_info`
Aggregate counters per post. 1:1 with `posts`.

| Column | Type | Notes |
|--------|------|-------|
| postid | UUID | **PK**, FK → posts(postid) CASCADE |
| userid | UUID | FK → users(uid) CASCADE |
| likes | INTEGER | |
| dislikes | INTEGER | |
| reports | INTEGER | |
| views | INTEGER | |
| saves | INTEGER | |
| shares | INTEGER | |
| comments | INTEGER | |
| totalreports | INTEGER | Default 0 |
| count_content_moderation_dismissed | INTEGER | Default 0 |
| createdat | TIMESTAMP | |

#### `posts_media`
Individual media items for gallery posts.

| Column | Type | Notes |
|--------|------|-------|
| postid | UUID | **PK**(composite), FK → posts(postid) CASCADE |
| contenttype | VARCHAR(13) | CHECK IN (image, text, video, audio, cover) |
| media | VARCHAR(500) | **PK**(composite) |
| options | VARCHAR(500) | |

#### `tags`
| Column | Type | Notes |
|--------|------|-------|
| tagid | BIGSERIAL | **PK** |
| name | VARCHAR(62) | |

#### `post_tags`
| Column | Type | Notes |
|--------|------|-------|
| postid | UUID | **PK**(composite), FK → posts(postid) CASCADE |
| tagid | INTEGER | **PK**(composite), FK → tags(tagid) CASCADE |
| createdat | TIMESTAMP | |

---

### Post Interactions (Junction Tables)

These tables share the same structure: `(userid UUID, postid UUID, collected SMALLINT, createdat TIMESTAMP)` with composite PK.

| Table | Purpose |
|-------|---------|
| `user_post_likes` | Users who liked a post |
| `user_post_dislikes` | Users who disliked a post |
| `user_post_views` | Users who viewed a post |
| `user_post_saves` | Users who saved a post |
| `user_post_shares` | Users who shared a post |
| `user_post_reports` | Users who reported a post |

#### `user_post_comments`
| Column | Type | Notes |
|--------|------|-------|
| commentid | UUID | **PK**, FK → comments(commentid) CASCADE |
| userid | UUID | FK → users(uid) CASCADE |
| postid | UUID | FK → posts(postid) CASCADE |
| collected | SMALLINT | |
| createdat | TIMESTAMP | |

---

### Comments

#### `comments`
| Column | Type | Notes |
|--------|------|-------|
| commentid | UUID | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| postid | UUID | FK → posts(postid) CASCADE |
| parentid | UUID | FK → comments(commentid) CASCADE (threaded replies) |
| content | TEXT | |
| status | SMALLINT | Default 10 |
| visibility_status | VARCHAR(25) | Default `'normal'` |
| createdat | TIMESTAMP | |

#### `comment_info`
| Column | Type | Notes |
|--------|------|-------|
| commentid | UUID | **PK**, FK → comments(commentid) CASCADE |
| userid | UUID | FK → users(uid) CASCADE |
| likes | INTEGER | |
| reports | INTEGER | |
| comments | INTEGER | |
| totalreports | INTEGER | Default 0 |
| count_content_moderation_dismissed | INTEGER | Default 0 |
| createdat | TIMESTAMP | |

#### `user_comment_likes`
| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| commentid | UUID | **PK**(composite), FK → comments(commentid) CASCADE |
| collected | SMALLINT | |
| createdat | TIMESTAMP | |

#### `user_comment_reports`
| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| commentid | UUID | **PK**(composite), FK → comments(commentid) CASCADE |
| collected | SMALLINT | |
| createdat | TIMESTAMP | |

---

### Chats & Messaging

#### `chats`
| Column | Type | Notes |
|--------|------|-------|
| chatid | UUID | **PK** |
| creatorid | UUID | FK → users(uid) CASCADE |
| name | TEXT | |
| image | VARCHAR(100) | |
| ispublic | SMALLINT | Default 1 |
| createdat | TIMESTAMP | |
| updatedat | TIMESTAMP | |

#### `newsfeed`
A newsfeed is a specialization of a chat.

| Column | Type | Notes |
|--------|------|-------|
| feedid | UUID | **PK**, FK → chats(chatid) CASCADE |
| creatorid | UUID | FK → users(uid) CASCADE |
| name | VARCHAR(50) | |
| image | VARCHAR(100) | |
| createdat | TIMESTAMP | |
| updatedat | TIMESTAMP | |

#### `chatmessages`
| Column | Type | Notes |
|--------|------|-------|
| messid | BIGSERIAL | **PK** |
| chatid | UUID | FK → chats(chatid) CASCADE |
| userid | UUID | FK → users(uid) CASCADE |
| content | TEXT | |
| createdat | TIMESTAMP | |

#### `chatparticipants`
| Column | Type | Notes |
|--------|------|-------|
| chatid | UUID | **PK**(composite), FK → chats(chatid) CASCADE |
| userid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| hasaccess | SMALLINT | Default 0 |
| createdat | TIMESTAMP | |

#### `user_chat_status`
Tracks last-seen message per user per chat.

| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**(composite), FK → users(uid) CASCADE |
| chatid | UUID | **PK**(composite), FK → chats(chatid) CASCADE |
| last_seen_message_id | INTEGER | FK → chatmessages(messid) |
| createdat | TIMESTAMP | |

---

### Economy & Wallet

#### `wallett`
User's aggregate token balance.

| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**, FK → users(uid) CASCADE |
| liquidity | NUMERIC(30,10) | Default 0 |
| liquiditq | NUMERIC(64) | Default 0 |
| updatedat | TIMESTAMP | |
| createdat | TIMESTAMP | |

#### `wallet`
Individual token ledger entries.

| Column | Type | Notes |
|--------|------|-------|
| token | VARCHAR(12) | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| postid | UUID | Nullable |
| fromid | UUID | Nullable |
| numbers | NUMERIC(30,10) | |
| numbersq | NUMERIC(64) | |
| whereby | INTEGER | |
| createdat | TIMESTAMP | |

#### `transactions`
Full transaction log.

| Column | Type | Notes |
|--------|------|-------|
| transactionid | UUID | **PK** |
| operationid | UUID | NOT NULL (renamed from transuniqueid) |
| transactiontype | VARCHAR(255) | |
| senderid | UUID | FK → users(uid) OR mint_account(accountid) (trigger-validated) |
| recipientid | UUID | FK → users(uid) |
| tokenamount | NUMERIC(30,10) | NOT NULL (no default) |
| transferaction | VARCHAR(255) | |
| message | TEXT | |
| transactioncategory | VARCHAR(255) | CHECK IN (P2P_TRANSFER, AD_PINNED, POST_CREATE, LIKE, DISLIKE, COMMENT, TOKEN_MINT, SHOP_PURCHASE, FEE, INVITER_FEE_EARN) |
| createdat | TIMESTAMP | |

**Constraints:** `chk_not_self_transfer` (senderid ≠ recipientid), `chk_positive_amount` (tokenamount > 0)

#### `gems`
Gem rewards earned from post interactions.

| Column | Type | Notes |
|--------|------|-------|
| gemid | UUID | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| postid | UUID | FK → posts(postid) CASCADE |
| fromid | UUID | Nullable |
| gems | NUMERIC(30,10) | |
| whereby | INTEGER | |
| collected | SMALLINT | |
| mintid | UUID | FK → mints(mintid), nullable |
| transactionid | UUID | FK → transactions(transactionid), nullable |
| createdat | TIMESTAMP | |

#### `logwins`
Token win log.

| Column | Type | Notes |
|--------|------|-------|
| token | UUID | **PK** |
| userid | UUID | |
| postid | UUID | Nullable |
| fromid | UUID | Nullable |
| gems | NUMERIC(30,10) | |
| numbers | NUMERIC(30,10) | |
| numbersq | NUMERIC(64) | |
| whereby | INTEGER | |
| migrated | INTEGER | Default 1 |
| createdat | TIMESTAMP | |

#### `mcap`
Market-cap snapshots.

| Column | Type | Notes |
|--------|------|-------|
| capid | SERIAL | **PK** |
| coverage | NUMERIC(30,10) | |
| tokenprice | NUMERIC(30,10) | |
| gemprice | NUMERIC(30,10) | |
| daygems | NUMERIC(30,10) | |
| daytokens | NUMERIC(30,10) | |
| totaltokens | NUMERIC(30,10) | |
| createdat | TIMESTAMP | |

#### `action_prices`
Global pricing configuration (single row).

| Column | Type | Notes |
|--------|------|-------|
| post_price | NUMERIC(10,4) | |
| like_price | NUMERIC(10,4) | |
| dislike_price | NUMERIC(10,4) | |
| comment_price | NUMERIC(10,4) | |
| currency | VARCHAR(10) | Default `'EUR'` |
| createdat | TIMESTAMP | |
| updatedat | TIMESTAMP | |

#### `dailyfree`
Daily free-action counters per user.

| Column | Type | Notes |
|--------|------|-------|
| userid | UUID | **PK**, FK → users(uid) CASCADE |
| liken | SMALLINT | |
| comments | SMALLINT | |
| posten | SMALLINT | |
| createdat | TIMESTAMP | |

---

### Mint

#### `mint_account`
Central mint treasury (single row).

| Column | Type | Notes |
|--------|------|-------|
| accountid | UUID | **PK** |
| initial_balance | NUMERIC(30,10) | CHECK ≥ 0 |
| current_balance | NUMERIC(30,10) | CHECK ≤ initial_balance |
| createdat | TIMESTAMP | |
| updatedat | TIMESTAMP | |

#### `mints`
Daily mint operations.

| Column | Type | Notes |
|--------|------|-------|
| mintid | UUID | **PK** |
| day | DATE | UNIQUE |
| gems_in_token_ratio | NUMERIC(30,10) | CHECK > 0 |
| createdat | TIMESTAMP | |

---

### Advertisements

#### `advertisements`
| Column | Type | Notes |
|--------|------|-------|
| advertisementid | UUID | **PK** |
| postid | UUID | FK → posts(postid) CASCADE |
| userid | UUID | FK → users(uid) CASCADE |
| status | VARCHAR(12) | CHECK IN (basic, pinned) |
| timestart | TIMESTAMP | |
| timeend | TIMESTAMP | CHECK ≥ timestart |
| createdat | TIMESTAMP | |

**Indexes:** (postid, status, timestart, timeend)

#### `advertisements_log`
Immutable audit log of advertisement purchases.

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | **PK** |
| advertisementid | UUID | FK → advertisements CASCADE |
| postid | UUID | FK → posts CASCADE |
| userid | UUID | FK → users CASCADE |
| status | VARCHAR(12) | |
| timestart | TIMESTAMP | |
| timeend | TIMESTAMP | |
| tokencost | NUMERIC(20,5) | CHECK ≥ 0 |
| eurocost | NUMERIC(20,5) | CHECK ≥ 0 |
| operationid | UUID | Nullable |
| createdat | TIMESTAMP | |

#### `advertisements_info`
Aggregate engagement counters per advertisement.

| Column | Type | Notes |
|--------|------|-------|
| advertisementid | UUID | **PK**, FK → advertisements CASCADE |
| postid | UUID | FK → posts CASCADE |
| userid | UUID | FK → users CASCADE |
| likes | INTEGER | |
| dislikes | INTEGER | |
| reports | INTEGER | |
| views | INTEGER | |
| saves | INTEGER | |
| shares | INTEGER | |
| comments | INTEGER | |
| totalreports | INTEGER | Default 0 |
| updatedat | TIMESTAMP | |
| createdat | TIMESTAMP | |

---

### Shop

#### `shop_orders`
| Column | Type | Notes |
|--------|------|-------|
| shoporderid | UUID | **PK** |
| userid | UUID | FK → users(uid) CASCADE |
| transactionid | UUID | FK → transactions(transactionid) CASCADE |
| shopitemid | VARCHAR(255) | |
| size | VARCHAR(255) | |
| name | VARCHAR(255) | |
| email | VARCHAR(100) | |
| addressline1 | VARCHAR(255) | |
| addressline2 | VARCHAR(255) | |
| city | VARCHAR(255) | |
| zipcode | VARCHAR(100) | |
| country | VARCHAR(100) | CHECK IN (GERMANY) |
| createdat | TIMESTAMP | |

---

### Reporting & Moderation

#### `user_reports`
| Column | Type | Notes |
|--------|------|-------|
| reportid | UUID | **PK** |
| reporter_userid | UUID | FK → users(uid) CASCADE |
| targetid | UUID | |
| targettype | VARCHAR(13) | CHECK IN (post, user, comment) |
| message | TEXT | |
| collected | SMALLINT | |
| hash_content_sha256 | CHAR(64) | Part of unique constraint |
| moderationticketid | UUID | FK → moderation_tickets(uid) SET NULL |
| moderationid | UUID | FK → moderations(uid) SET NULL |
| createdat | TIMESTAMP | |

**Constraints:** UNIQUE (reporter_userid, targetid, hash_content_sha256), CHECK reporter ≠ target

#### `moderation_tickets`
| Column | Type | Notes |
|--------|------|-------|
| uid | UUID | **PK** |
| status | VARCHAR(25) | Default `'waiting_for_review'` |
| reportscount | INTEGER | Default 0 |
| contenttype | VARCHAR(25) | Nullable |
| targetcontentid | UUID | |
| createdat | TIMESTAMP | |
| updatedat | TIMESTAMP | |

#### `moderations`
| Column | Type | Notes |
|--------|------|-------|
| uid | UUID | **PK** |
| moderationticketid | UUID | FK → moderation_tickets(uid) CASCADE |
| moderatorid | UUID | FK → users(uid) CASCADE |
| status | VARCHAR(25) | Default `'waiting_for_review'` |
| createdat | TIMESTAMP | |

---

### Logging

#### `logdata`
Compact action log.

| Column | Type | Notes |
|--------|------|-------|
| logid | BIGSERIAL | **PK** |
| userid | UUID | |
| ip | INET | |
| browser | VARCHAR(1000) | |
| action_type | VARCHAR(30) | |
| createdat | TIMESTAMP | |

#### `logdaten`
Detailed HTTP request log.

| Column | Type | Notes |
|--------|------|-------|
| logid | BIGSERIAL | **PK** |
| userid | UUID | |
| ip | INET | |
| browser | VARCHAR(1000) | |
| url | VARCHAR(255) | |
| http_method | VARCHAR(10) | |
| status_code | INTEGER | |
| response_time | DECIMAL(10,2) | Milliseconds |
| location | VARCHAR(255) | |
| action_type | VARCHAR(30) | |
| auth_status | VARCHAR(50) | |
| createdat | TIMESTAMP | |

---

### Contact & Rate Limiting

#### `contactus`
| Column | Type | Notes |
|--------|------|-------|
| msgid | BIGSERIAL | **PK** |
| email | VARCHAR(249) | |
| name | VARCHAR(33) | |
| message | TEXT | |
| ip | INET | |
| collected | SMALLINT | |
| createdat | TIMESTAMP | |

#### `contactus_rate_limit`
| Column | Type | Notes |
|--------|------|-------|
| ip | INET | **PK** |
| request_count | INTEGER | Default 0 |
| last_request | TIMESTAMP | |

---

### Versioning

#### `versions`
| Column | Type | Notes |
|--------|------|-------|
| versid | UUID | **PK** |
| version | NUMERIC(3,2) | |
| wikiLink | TEXT | |
| createdat | TIMESTAMP | |

---

## Migration History

Migrations are applied in order from `sql_files_for_import/`:

| File | Summary |
|------|---------|
| `001_structure.sql` | Base schema — all core tables |
| `20250716…_user_settings_content_view_preferences.sql` | Creates `user_preferences` table |
| `20250723…_transactions_table.sql` | Creates `transactions` table |
| `20250729…_eligibility_token_expires_table.sql` | Creates `eligibility_token` table |
| `20250804…_advertisements_table.sql` | Creates `advertisements`, `advertisements_log`, `advertisements_info` |
| `20250824…_posts_status.sql` | Changes posts.status default from 10 → 0 |
| `20250827…_tokenamount_change.sql` | Converts transactions.tokenamount to NUMERIC; renames transuniqueid → operationid |
| `20250903…_getactionprice_change.sql` | Adds `onboardingsWereShown` to user_preferences; updates action prices |
| `20250903…_logwins_migration.sql` | Adds `migrated` column to logwins |
| `20250908…_db_timezone.sql` | Database timezone configuration |
| `20251001…_transactions_constraints.sql` | Transaction constraint updates |
| `20251002…_moderation_table.sql` | Creates `moderation_tickets`, `moderations`; adds moderation columns |
| `20251006…_increase_browser_field_length.sql` | Widens browser field |
| `20251014…_fix_follow_counters_migration.sql` | Recalculates follow counters |
| `20251105…_add_operationid_to_ads_info_log.sql` | Adds operationid to ads info/log |
| `20251106…_remove_default_on_transactions_tokenamount.sql` | Removes default on tokenamount |
| `20251202…_recalculate_users_info_counters.sql` | Recalculates users_info counters |
| `20251215…_add_transaction_category_to_transactions.sql` | Adds `transactioncategory` CHECK constraint |
| `20251218…_remove_logdaten_payload.sql` | Removes logdaten payload column |
| `20260106…_convert_varchar_columns_to_text.sql` | Converts varchar columns to text |
| `20260112…_shop_order_table.sql` | Creates `shop_orders` table |
| `20260115…_mint.sql` | Creates `mint_account`, `mints`; adds mintid/transactionid to gems |
| `20260115…_transactions_senderid_check.sql` | Replaces senderid FK with trigger allowing mint_account references |
| `20260210…_inviter_tnx_category.sql` | Adds `INVITER_FEE_EARN` to transactioncategory CHECK |
