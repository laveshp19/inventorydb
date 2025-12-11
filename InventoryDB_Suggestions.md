# Daily Diary & Chat — Improvement & Redesign Proposals

Focus: simplify schema, enforce integrity, optimize performance, and make it API-friendly and maintainable.

## Schema Corrections & Integrity
- Add proper PK on `MBL_ChatMaster.ChatMasterID`; keep `UserRelation` unique with a separate unique index.  
- Remove self-referencing FK on `MBL_ChatCounters.CounterMasterID`; add FK on `ChatMasterID` to `MBL_ChatMaster` with unique constraint (1 counter per chat).  
- Introduce FKs across diary tables: `DailyDiaryComments.DailyDiaryMasterID` → `DailyDiaryMaster`; `DailyDiaryAssociatedPeople.DailyDiaryMasterID` → `DailyDiaryMaster`; `DailyDiaryMessagesRead.ForeignTableID` → `DailyDiaryComments` and/or `DailyDiaryMaster` (via discriminator); `DDNotification.DDMasterID` → `DailyDiaryMaster`.  
- Standardize entity references via a consistent polymorphic pattern (e.g., `EntityType` + `EntityID` with check constraints) or explicit FKs where domain allows.

## Column & Naming Cleanup
- Rename `DDIsFor` → `AudienceType`; `RaisedByEntityType`/`OwnerEntityType` → `ActorType`; `RaisedByID`/`OwnerID` → `ActorID`.  
- In `DailyDiaryComments`, rename `ActionToStaffID` → `ActionTargetStaffID`; `NucleusComments` → `InternalNotes`.  
- In `MBL_ChatCounters`, rename `StaffUnread`/`ParentUnread` → `UnreadForStaff`/`UnreadForParent`; `TotalChatMessages` → `MessageCount`.  
- Normalize `IsThisRelatedToMasterRecord` → `IsMasterLevel`.  
- Ensure bit columns are NOT NULL with default 0; avoid nullable booleans (`StaffNotification`, `ParentNotification`, `IsBulk`, etc.).

## Normalization & Redesign
- **Diary**: Move common participant entity references to a junction table `DailyDiaryParticipant` with role enum (`Owner`, `RaisedBy`, `Associated`, `Recipient`). Replace multiple `StaffID`/`ParentsID`/`StudentID` columns with a unified `EntityType/EntityID` pair per role.  
- **Comments**: Same polymorphic author model; store `ContextStudentID` only if different from main diary context to reduce redundancy.  
- **Read tracking**: Replace `DailyDiaryMessagesRead` with `DailyDiaryReadReceipt` referencing a comment or master ID plus reader entity type/id; add unique constraint on (target, reader).  
- **Notifications**: Reference participants through normalized entity pattern; store template key + payload for localization instead of raw text only.  
- **Chat**: Make `MBL_ChatMaster` capture participants explicitly (`StaffID`, `ParentID`, optional `StudentID`) with FKs; `UserRelation` can be derived.  
- **Counters**: Enforce single row per chat; consider computed unread via `ChatMessage` table with `IsRead` rows instead of counters to avoid drift, or keep counters with triggers to maintain consistency.

## Indexing & Performance
- Add indexes:  
  - `DailyDiaryComments (DailyDiaryMasterID, CommentsDateAndTime)` for timeline queries.  
  - `DailyDiaryAssociatedPeople (DailyDiaryMasterID)` for joins in reports.  
  - `DailyDiaryMessagesRead (ForeignTableID, IsMasterLevel)` for read lookups.  
  - `MBL_ChatMessages (ChatMasterID, MsgID DESC)` to support recent-message paging.  
  - Unique index `MBL_ChatCounters (ChatMasterID)` to prevent duplicates.  
  - Covering indexes for `DDReports` subqueries or refactor to joins (see next section).  
- Remove redundant diagram extended properties to slim deploy scripts (optional).

## View & Query Refactors
- Refactor `DDReports` to replace correlated subqueries with explicit joins to `Staff`, `Student`, `Parents`, `CurrentGradeAndSection` and to pre-aggregated latest activity (CTE with MAX timestamp) for better performance.  
- Collapse `DailyDiaryMessagesReadPreConsolidation` + `DailyDiaryMessagesReadConsolidation` into a single view with distinct clause or use a materialized aggregation table maintained via triggers if performance-critical.  
- Remove duplicated message list procedures; create a single parameterized proc or view with role-based filtering handled at app layer.

## Stored Procedure Improvements
- Add server-side pagination with `OFFSET/FETCH` for chat message lists; return `TotalCount` or `HasMore` indicator.  
- Wrap message insert + counter update in a transaction; lock on `MBL_ChatCounters` row to avoid race conditions; use `MERGE` or `INSERT ... ON DUPLICATE` pattern to ensure single counter row.  
- Add validation (existence of `ChatMasterID`, participant authorization) inside procs.  
- Return standardized result shapes (DTO-aligned) rather than `SELECT *`.  
- Add error handling and set XACT_ABORT ON in write procs.

## API-Friendly Models & DTOs
- **Diary DTO**: `Diary { id, printableId, createdAt, audienceType, status, raisedBy {type,id,name}, owner {type,id,name}, contextStudentId, message, markdown, hindiMessage, bulkInfo {isBulk, bulkId, isFirst}, counts {comments, unread}, latestActivityAt }`.  
- **Comment DTO**: `Comment { id, diaryId, author {type,id,name}, text, hindiText, actionType, createdAt, isDeleted, contextStudentId, actionToStaffId }`.  
- **ReadReceipt DTO**: `ReadReceipt { targetType: 'master'|'comment', targetId, reader {type,id}, readAt }`.  
- **Chat DTO**: `Chat { id, staffId, parentId, studentId?, notifications {staff, parent}, lastMessage {text, mediaUrl, at, sender}, unread {staff, parent} }`.  
- **Message DTO**: `Message { id, chatId, sender, text, mediaUrl, sentAt }`.

## Workflow Enhancements
- Use triggers or application service to keep counters and read-receipts consistent (insert receipt when sender posts, update unread for counterpart).  
- Introduce soft-delete flags with `DeletedAt` + actor for auditing instead of simple bit.  
- Capture `UpdatedAt/UpdatedBy` on diary/comment/chat entities for traceability.  
- Localize messages using template IDs and parameters; avoid storing formatted text in multiple columns.

## Naming & Standards
- Adopt consistent casing (PascalCase for columns), singular table names, and standard `CreatedAt/CreatedBy/UpdatedAt/UpdatedBy`.  
- Use enumerations stored as smallint with lookup tables (`EntityType`, `ActionStatus`, `AudienceType`) with FK to enforce valid values.  
- Avoid `varchar(max)` when bounded sizes are known; use `nvarchar` for user-facing text for localization.

## Candidate New ER Outline
- Tables: `Diary`, `DiaryParticipant (DiaryID, Role, EntityType, EntityID)`, `DiaryComment`, `DiaryReadReceipt`, `DiaryNotification`, `Chat`, `ChatMessage`, `ChatCounter`, `ChatParticipant` (if future group chats).  
- Each table with FK integrity, audited timestamps, and role/type enums.  
- Indexes aligned to primary access patterns: recent activity feeds, unread calculations, participant lookups, and paginated message history.

## Migration Approach (high level)
- Add new keys/FKs and indexes side-by-side, backfill data, then swap procs/views to new structures.  
- Clean duplicated procedures after consolidation; add tests for unread logic and report outputs.  
- Provide compatibility views where needed during transition.

