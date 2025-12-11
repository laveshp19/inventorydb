# Daily Diary & Chat Database Analysis (Inventory DB)

Scope covers all Daily Diary and Chat related objects in `dbo` schema of `DatabaseProjectINVENTORY.MDF`.

## Tables — Daily Diary

### DailyDiaryMaster
- **Purpose**: Root record for a diary thread/ticket (who raised, owner, context student, message text, status).
- **Columns**:  
  - `DDMasterID` (PK, identity, decimal18, not null)  
  - `PrintableID` (varchar30, null)  
  - `DDDateAndTime` (datetime, not null)  
  - `DDIsFor` (smallint, not null) — audience/target type.  
  - `RaisedByEntityType` / `RaisedByID` (smallint, decimal18, not null) — initiator.  
  - `ParentsID` (decimal18, not null); `StudentsID` (decimal18, not null); `StaffID` (int, not null).  
  - `OwnerEntityType` / `OwnerID` (smallint, decimal18, not null) — ticket owner.  
  - `DetailedMessage` (nvarchar(max), not null); `MarkdownText` (nvarchar(max), not null); `HindiMessage` (nvarchar(max), null).  
  - `TicketID` (decimal18, not null); `ActionStatus` (tinyint, not null); `IsDeleted` (bit, not null).  
  - `ContextStudentID` (decimal18, not null) — disambiguates multi-child parents.  
  - Bulk flags: `IsBulk` (bit, null), `BulkID` (decimal18, null), `IsFirstMessageInBulk` (bit, null).  
- **Constraints**: PK on `DDMasterID`. No FKs defined in script.  
- **Indexes (NC)**: on `RaisedByEntityType`, `OwnerID`, `RaisedByID`, `StaffID`, `DDDateAndTime+ActionStatus`, `OwnerEntityType`, `StudentsID`, `ActionStatus`, `ContextStudentID`, `ParentsID`, `DDDateAndTime`.  
- **Nullability/Defaults**: explicit NOT NULL for core fields; no defaults.  
- **Relationships**: Parent to comments, associated people, notifications, messages read. Entity type/ID pairs reference Staff/Student/Parents tables (soft references).  
- **Data flow**: Inserted when a diary is created (likely via app layer). Updated for status, ownership, bulk flags. Consumed heavily by views (`DDReports`, `DailyDiaryConsolidatedComments`) and downstream UI reporting.

### DailyDiaryComments
- **Purpose**: Individual comments/replies within a diary thread.  
- **Columns**: `DailyDiaryCommentsID` (PK identity), `DailyDiaryMasterID` (decimal18, FK logical), `StaffID` (int, null), `ParentsID` (decimal18, null), `StudentID` (decimal18, null), `Comments` (varchar(max)), `NucleusComments` (varchar300), `CommentsDateAndTime` (datetime, not null), `ActionType` (nchar10, not null), `IsDeleted` (bit, not null), `ContextStudentID` (decimal18, not null), `HindiMessage` (nvarchar(max), null), `ActionToStaffID` (decimal18, null).  
- **Constraints**: PK on `DailyDiaryCommentsID`. No FKs in script.  
- **Indexes (NC)**: on `StaffID`, `ParentsID`, `StudentID`.  
- **Relationships**: Child of `DailyDiaryMaster`. Drives read-status views via `DailyDiaryMessagesRead*`.  
- **Data flow**: Inserts on new comments; read by reporting views; no stored procs shown for diary comment insert/update.

### DailyDiaryAssociatedPeople
- **Purpose**: Permissions/context participants per diary record.  
- **Columns**: Identity PK `DailyDiaryAssociatedPeopleID`; `DailyDiaryMasterID`; optional `StaffID`/`ParentsID`/`StudentID`; flags: `IsStarred`, `CanEscalate`, `CanForward`, `CanChangeActions`, `CanDelete`, `IsAdmin`, `HasEscalated`; `EscalatedByID` (decimal18, null); `RecordDateTime` (date, null).  
- **Constraints**: PK on identity. No FKs.  
- **Indexes (NC)**: on `ParentsID`, `StudentID`, `StaffID`.  
- **Relationships**: Links actors to a diary master. Used by `DDReports` for default entity type/ID and escalation state.  
- **Data flow**: Populated alongside diary creation or escalation workflows; read for UI permissions.

### DailyDiaryMessagesRead
- **Purpose**: Tracks per-entity read receipts for diary comments or master records.  
- **Columns**: Identity PK `DailyDiaryMessagesReadID`; `ForeignTableID` (decimal18) — points to diary master or comment; `IsThisRelatedToMasterRecord` (bit); `StaffID`/`ParentsID`/`StudentID` (nullable).  
- **Constraints**: PK only; no FKs.  
- **Indexes (NC)**: on `ParentsID`, `StudentID`, `ForeignTableID`, `StaffID`.  
- **Relationships**: Combined with comments in `DailyDiaryMessagesReadPreConsolidation` to derive per-entity read coverage.  
- **Data flow**: Inserts when a recipient reads; consumed by views; update/insert logic not present in scripts.

### DDNotification
- **Purpose**: Notification messages tied to diary masters.  
- **Columns**: Identity PK `DDNotificationID`; `ForEntityType`/`ForEntityID`; `FromEntityEmailID`; `FromEntityType`/`FromEntityID`; `DDMasterID`; `DDDetails`; `TextToDisplay`; `IsRead`; `IsForTeamLeader`; `UpdateDateTime`.  
- **Constraints**: PK only.  
- **Indexes (NC)**: `FromEntityID`, `DDMasterID`, `FromEntityType`, `ForEntityID`, `FromEntityEmailID`, `ForEntityType`.  
- **Relationships**: Soft links to diary master and actors.  
- **Data flow**: Likely written by app logic on events (new comment/status); supports notification delivery queries.

## Views — Daily Diary

### DistinctPeopleInADailyDiaryWhoHasCommented
- **Purpose**: Distinct commenter list per diary, with name and type classification.  
- **Logic**: UNION of staff, parents, student joins on `DailyDiaryComments`; filters out null foreign IDs; maps `EntityTypeNum` (staff 0, student 1, parent 2).  
- **Sources**: `DailyDiaryComments`, `Staff`, `Parents`, `Student`.  
- **Consumption**: Used by `DailyDiaryCommentsCountByAUser`.

### DailyDiaryCommentsCountByAUser
- **Purpose**: Per diary, per commenter entity-type comment counts.  
- **Logic**: Select from `DistinctPeopleInADailyDiaryWhoHasCommented` with correlated subquery counting comments per entity type classification.  
- **Outputs**: `DailyDiaryMasterID`, entity identity fields, `CountOfCommentsMade`.  
- **Consumption**: Reporting/UI counts.

### DailyDiaryConsolidatedComments
- **Purpose**: Union of diary master message and all comments for chronological display.  
- **Logic**: UNION of `DailyDiaryMaster` (DetailedMessage) and `DailyDiaryComments` (Comments) selecting `DDMasterID` & timestamp, ordered desc.  
- **Consumption**: Used by `DDReports` to fetch latest interaction timestamp.

### DailyDiaryMessagesReadPreConsolidation
- **Purpose**: Expand read receipts joined to comments for entity/entity-type comparison.  
- **Logic**: RIGHT JOIN `DailyDiaryMessagesRead` to `DailyDiaryComments`; derives `EntityType/EntityID` from comment author; `MessagesReadEntityType/ID` from read rows; outputs comment text. Filters non-null comments.  
- **Consumption**: Source for consolidation view.

### DailyDiaryMessagesReadConsolidation
- **Purpose**: Deduplicate read receipt tuples.  
- **Logic**: GROUP BY over `DailyDiaryMessagesReadPreConsolidation`.  
- **Consumption**: Downstream reporting/analytics on who read whose messages.

### ParentsDDRecords
- **Purpose**: Distinct diary records associated with parents (either ownership or associated people).  
- **Logic**: LEFT JOIN `DailyDiaryMaster` to `DailyDiaryAssociatedPeople`; outputs `ParentsID`, `DDMasterID`.  
- **Consumption**: Likely parent-facing list generation/authorization filter.

### DDReports
- **Purpose**: Comprehensive diary report view for UI (names, images, profile paths, titles, owner, raised by, last interaction).  
- **Logic**: Selects from `DailyDiaryMaster` left join `DailyDiaryAssociatedPeople`. Derives names, full names, images, profile paths, titles using correlated subqueries to `Staff`, `Student`, `Parents`, `CurrentGradeAndSection`. Computes `LastDateTime` via `DailyDiaryConsolidatedComments`. Emits default entity types/IDs from associated people and owner details. Orders by `LastDateTime` desc then `DDMasterID`.  
- **Consumption**: Main feed/reporting endpoint/UI cards.

## Tables — Chat

### MBL_ChatMaster
- **Purpose**: Defines a chat relationship (likely parent-staff pair).  
- **Columns**: `ChatMasterID` (identity, decimal18, not null), `UserRelation` (nvarchar100, not null), `StaffNotification` (bit), `ParentNotification` (bit).  
- **Constraints**: PK is **on `UserRelation`** (non-identity) — identity column not used as PK. No FK.  
- **Indexes**: none besides PK.  
- **Relationships**: Parent for chat messages & counters.  
- **Data flow**: Created per relation; notification flags mutated by app.

### MBL_ChatMessages
- **Purpose**: Stores chat messages per chat master.  
- **Columns**: `MsgID` (PK identity), `ChatMasterID` (decimal18), `ChatText` (nvarchar(max)), `MediaPath` (nvarchar(max)), `Sender` (nvarchar100), `EntryTime` (datetime).  
- **Constraints**: PK on `MsgID`. No FKs.  
- **Indexes**: none.  
- **Data flow**: Written by chat insert SPs; read by list SPs.

### MBL_ChatCounters
- **Purpose**: Tracks unread counts and totals per chat.  
- **Columns**: `CounterMasterID` (PK identity), `ChatMasterID`, `StaffUnread`, `ParentUnread`, `TotalChatMessages`, `LastUpdateTime`.  
- **Constraints**: PK plus **self-referencing FK on `CounterMasterID` to itself** (anomalous, probably unintended). No FK to `ChatMaster`.  
- **Indexes**: none except PK.  
- **Data flow**: Inserted/updated by chat insert SPs; read by count SPs and list SPs.

## Stored Procedures — Chat

### mbl_spgChatMessageListByChatMasterIDForParent / ForStaff
- **Purpose**: Fetch paginated latest chat messages for parent/staff client.  
- **Parameters**: `@chatMasterID` (decimal), `@takePage` (int).  
- **Logic**: `SELECT TOP (@takePage) * FROM MBL_ChatMessages WHERE ChatMasterID=@chatMasterID ORDER BY MsgID DESC`.  
- **Tables Read**: `MBL_ChatMessages`.  
- **Output**: Raw rows (all columns).  
- **Notes**: No pagination offset; descending ID; same logic for both procedures.

### mbl_spiChatMessageByChatMasterIDForParent / ForStaff
- **Purpose**: Insert a message and update unread counters for opposite party.  
- **Parameters**: `@ChatMasterID` (decimal), `@ChatText` (nvarchar(max)), `@Sender` (nvarchar50), `@MediaPath` (nvarchar(max)), `@EntryTime` (datetime).  
- **Logic**:  
  1. Insert into `MBL_ChatMessages`.  
  2. Fetch `CounterMasterID` and relevant unread (`ParentUnread` for parent sender? actually increments opposite side).  
  3. Compute `@totalMessage` by counting messages.  
  4. If no counter row (when `@counter=0`), insert new counter row with unread=1 for receiver side (parent or staff) and unread=0 for sender side.  
  5. Else increment receiver unread and update totals and `LastUpdateTime`.  
- **Tables**: Write `MBL_ChatMessages`, read/write `MBL_ChatCounters`.  
- **Business**: Maintains unread counts per role; assumes single counter row per chat.

### mbl_spgNewMessageCountForParentByChatMasterID / ForStaff
- **Purpose**: Return unread count for respective role.  
- **Parameters**: `@ChatMasterID` (decimal).  
- **Logic**: Simple select of `StaffUnread` or `ParentUnread` from `MBL_ChatCounters` by `ChatMasterID`.  
- **Tables**: Read `MBL_ChatCounters`.

### mbl_spgChatParentUserListByParentIDAndChatMasterID
- **Purpose**: Parent-side chat overview for a chat master (per child).  
- **Parameters**: `@ParentID` (decimal), `@ChatMasterID` (decimal).  
- **Logic**:  
  - Get `ParentUnread` from `MBL_ChatCounters`; default 0.  
  - Load notification flags from `MBL_ChatMaster`.  
  - Get last message text/time from `MBL_ChatMessages` (latest MsgID). Format time string.  
  - Select student info (name, email, full name incl grade/section, image path with default URLs) from `Student` joined `StudentCurrentGradeSectionDetails` filtered by parent ID.  
  - Emit counts, last message/time, notification flags.  
- **Tables**: `MBL_ChatCounters` (read), `MBL_ChatMaster` (read), `MBL_ChatMessages` (read), `Student`, `StudentCurrentGradeSectionDetails`.  
- **Notes**: Assumes single chat master; no ordering; date formatting done in DB.

### mbl_spgChatStaffUserListByStaffMasterIDAndChatMasterID
- **Purpose**: Staff-side chat overview.  
- **Parameters**: `@StaffMasterID` (decimal), `@ChatMasterID` (decimal).  
- **Logic**: Similar to parent version but uses `StaffUnread` and `WorkingStaffList` for identity and image defaults.  
- **Tables**: `MBL_ChatCounters`, `MBL_ChatMaster`, `MBL_ChatMessages`, `WorkingStaffList`.  
- **Notes**: Builds image URLs with defaults.

### Alumni_SP_GetAllAlumniChatDetailsBySenderId
- **Purpose**: Fetch alumni chat messages for a sender and user type.  
- **Parameters**: `@SenderId` (int), `@UserType` (int).  
- **Logic**: Simple select on `AlumniChatMessages` filtered by sender and user type; hardcodes `isSender=1`.  
- **Tables**: `AlumniChatMessages`.  
- **Notes**: Not directly tied to parent/staff chat but part of chat module.

## Derived Data Flows & Workflows
- **Diary creation**: `DailyDiaryMaster` insert (not shown) likely sets owner/raised-by/context; optional `DailyDiaryAssociatedPeople` entries for permissions; notifications via `DDNotification`.  
- **Diary interaction**: `DailyDiaryComments` insert; `DailyDiaryMessagesRead` rows track reads; views aggregate commenters, counts, consolidated timelines, read coverage; `DDReports` composes UI-ready record with names/images/profile links.  
- **Chat messaging**: `mbl_spiChatMessageByChatMasterIDFor*` insert messages and maintain unread counters in `MBL_ChatCounters`; list procedures fetch recent messages; counter procs expose unread counts; user list procs fetch participant info plus last message snapshot and notifications.  
- **Read tracking**: For diaries, view pipeline (`DailyDiaryMessagesRead*`) converts read rows + comments into deduped combinations per entity/reader.

## Anomalies / Inconsistencies
- Missing foreign keys across diary and chat tables (soft references everywhere).  
- `MBL_ChatMaster` PK on `UserRelation` while `ChatMasterID` is identity but not PK.  
- `MBL_ChatCounters` has self-referencing FK on `CounterMasterID` (likely erroneous) and lacks FK to `MBL_ChatMaster`.  
- Duplicate procedures for staff/parent message listing with identical logic.  
- Unread counter logic derives existence by `CounterMasterID` fetched via `ChatMasterID` and compares to 0; if no row, `@counter` remains 0 and insert occurs, but multiple rows could exist; no uniqueness constraint on `ChatMasterID`.  
- Extensive correlated subqueries in `DDReports` may cause performance issues; no indexes on foreign reference columns in diary tables for join targets.  
- No default constraints for flags; nullability used for booleans (e.g., notification bits) causing tri-state behaviors.  
- No pagination offset in message list procs; only TOP paging from latest without continuation token.


