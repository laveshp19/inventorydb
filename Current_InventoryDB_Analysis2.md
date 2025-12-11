# Daily Diary & Chat Module - Comprehensive Database Analysis

**Document Version:** 1.0  
**Last Updated:** December 11, 2025  
**Scope:** Complete structural and functional analysis of Daily Diary and Chat related database objects

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Daily Diary Module](#daily-diary-module)
3. [Chat Module](#chat-module)
4. [Entity Relationships](#entity-relationships)
5. [Data Flow & Workflows](#data-flow--workflows)
6. [Anomalies & Issues](#anomalies--issues)

---

## Executive Summary

The Daily Diary and Chat modules represent two critical communication systems within the InventoryDB database:

- **Daily Diary Module**: A comprehensive system for logging, tracking, and managing daily communications/notes between staff, students, and parents with escalation capabilities
- **Chat Module**: A real-time messaging system enabling direct staff-to-parent and parent-to-staff conversations with unread message tracking

Both modules are heavily denormalized with significant functional overlap, redundant data structures, and suboptimal indexing strategies.

---

## Daily Diary Module

### Overview
The Daily Diary module manages detailed daily communications and updates related to students, with support for comments, escalations, bulk operations, and audit trails.

### Core Tables

#### 1. **DailyDiaryMaster** 
**Purpose:** Primary master table storing individual daily diary entries/communications

**Physical Characteristics:**
- Type: Table (with Identity Primary Key)
- Row Count: Variable (high volume - daily transactions)
- Primary Key: `DDMasterID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose | Notes |
|---|---|---|---|---|
| DDMasterID | DECIMAL (18) | NOT NULL | Unique identifier for each diary entry | Identity column, auto-increment |
| PrintableID | VARCHAR (30) | NULL | Human-readable identifier for reports | Format: Custom ID for display |
| DDDateAndTime | DATETIME | NOT NULL | Timestamp of entry creation | Critical filter for date-range queries |
| DDIsFor | SMALLINT | NOT NULL | Purpose/Category code (0=Staff, 1=Student, 2=Parent, 3=Other) | Enum-like field |
| RaisedByEntityType | SMALLINT | NOT NULL | Entity type of originator (0=Staff, 1=Student, 2=Parent) | Determines who created the entry |
| RaisedByID | DECIMAL (18) | NOT NULL | ID of person who created entry | FK reference to Staff/Student/Parent |
| ParentsID | DECIMAL (18) | NOT NULL | Associated parent ID | Direct reference (not conditional) |
| StudentsID | DECIMAL (18) | NOT NULL | Associated student ID | Direct reference (not conditional) |
| StaffID | INT | NOT NULL | Associated staff/teacher ID | Direct reference (not conditional) |
| OwnerEntityType | SMALLINT | NOT NULL | Entity type responsible for action (0=Staff, 1=Student, 2=Parent) | Determines ownership |
| OwnerID | DECIMAL (18) | NOT NULL | ID of responsible entity | FK reference |
| DetailedMessage | NVARCHAR (MAX) | NOT NULL | Primary content of the diary entry | Full message text |
| TicketID | DECIMAL (18) | NOT NULL | Linked support ticket ID | Default: 0 if no ticket |
| ActionStatus | TINYINT | NOT NULL | Status code (0=Open, 1=Closed, 2=Escalated, etc.) | Enum-like field |
| IsDeleted | BIT | NOT NULL | Soft delete flag | Logical deletion (0=Active, 1=Deleted) |
| ContextStudentID | DECIMAL (18) | NOT NULL | Student for whom entry is contextually relevant | Supports multi-child parents |
| MarkdownText | NVARCHAR (MAX) | NOT NULL | Structured markup version of message | For rich text rendering |
| HindiMessage | NVARCHAR (MAX) | NULL | Hindi language translation of message | Multi-language support |
| IsBulk | BIT | NULL | Flag indicating bulk operation | Marks entries created as group |
| BulkID | DECIMAL (18) | NULL | Reference to bulk operation group | Groups related entries |
| IsFirstMessageInBulk | BIT | NULL | Marks first message in bulk group | Workflow indicator |

**Constraints:**
- Primary Key (Clustered): `PK_DailyDiaryMaster` on `DDMasterID`
- No Foreign Keys defined (referenced via entity type + ID pattern)
- No Unique constraints
- No Check constraints

**Indexes (Non-Clustered):**
1. `IX_DailyDiaryMaster_RaisedByEnitytyType` → RaisedByEntityType ASC
2. `IX_DailyDiaryMaster_OwnerID` → OwnerID ASC
3. `IX_DailyDiaryMaster_RaisedByEntityID` → RaisedByID ASC
4. `IX_DailyDiaryMaster_StaffID` → StaffID ASC
5. `IX_DailyDiaryMaster_DDDateAndTime_And_ActionStatus` → DDDateAndTime ASC, ActionStatus ASC
6. `IX_DailyDiaryMaster_OwnerEnittyTYpe` → OwnerEntityType ASC (Note: Typo in name)
7. `IX_DailyDiaryMaster_StudentID` → StudentsID ASC
8. `IX_DailyDiaryMaster_ActionStatus` → ActionStatus ASC
9. `IX_DailyDiaryMaster_ContextStudentID` → ContextStudentID ASC
10. `IX_DailyDiaryMaster_ParentsID` → ParentsID ASC
11. `IX_DailyDiaryMaster_DDDateAndTime` → DDDateAndTime ASC

**Extended Properties:**
- `ContextStudentID`: "THIS IS USED FOR STORING STUDENT'S ID FOR WHOM THE DAILY DIARY MESSAGE IS CREATED. THIS IS NEEDED BECAUSE A PARENT CAN HAVE ONE OR MORE KIDS STUDYING IN THE SCHOOL."
- `TicketID`: "THIS WILL HOLD SOME VALUE WHEN THE A TICKET IS RAISED FROM THIS DAILY DIARY COMMUNICATION"

**Data Flow:**
- **Creation**: Initiated by staff, student, or parent; captures originator details
- **Update**: Status changes via soft-delete or action status updates
- **Consumption**: Queried via entity type + ID, date range filters, or status filters
- **Archival**: Soft-deleted records retained for audit trail

---

#### 2. **DailyDiaryComments**
**Purpose:** Thread comments/replies to diary entries, supporting nested conversations

**Physical Characteristics:**
- Type: Table
- Row Count: High volume (conversation dependent)
- Primary Key: `DailyDiaryCommentsID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose |
|---|---|---|---|
| DailyDiaryCommentsID | DECIMAL (18) | NOT NULL | Unique comment identifier |
| DailyDiaryMasterID | DECIMAL (18) | NOT NULL | FK to parent diary entry |
| StaffID | INT | NULL | Staff member adding comment |
| ParentsID | DECIMAL (18) | NULL | Parent adding comment |
| StudentID | DECIMAL (18) | NULL | Student adding comment |
| Comments | VARCHAR (MAX) | NULL | Comment text content |
| NucleusComments | VARCHAR (300) | NULL | Short summary/snippet for UI display |
| CommentsDateAndTime | DATETIME | NOT NULL | When comment was added |
| ActionType | NCHAR (10) | NOT NULL | Type of action (code-based) |
| IsDeleted | BIT | NOT NULL | Soft delete flag |
| ContextStudentID | DECIMAL (18) | NOT NULL | Student context (same multi-child purpose) |
| HindiMessage | NVARCHAR (MAX) | NULL | Hindi translation of comment |
| ActionToStaffID | DECIMAL (18) | NULL | Staff member targeted by action |

**Constraints:**
- Primary Key (Clustered): `PK_DailyDiaryComments` on `DailyDiaryCommentsID`
- No explicit Foreign Keys (DailyDiaryMasterID not FKed)
- No Unique constraints
- No Check constraints

**Indexes (Non-Clustered):**
1. `IX_DailyDiaryComments_StaffID` → StaffID ASC
2. `IX_DailyDiaryComments_ParentsID` → ParentsID ASC
3. `IX_DailyDiaryComments_StudentID` → StudentID ASC

**Data Flow:**
- **Creation**: Reply to master entry, captures commenter type + ID
- **Update**: Soft-delete only
- **Consumption**: Fetched with parent entry via DailyDiaryMasterID
- **Thread Reconstruction**: Requires JOIN with DailyDiaryMaster + sort by CommentsDateAndTime

---

#### 3. **DailyDiaryAssociatedPeople**
**Purpose:** Maps people (staff/parents/students) to diary entries with permission/action flags

**Physical Characteristics:**
- Type: Table
- Row Count: Variable (scales with people count per entry)
- Primary Key: `DailyDiaryAssociatedPeopleID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose |
|---|---|---|---|
| DailyDiaryAssociatedPeopleID | DECIMAL (18) | NOT NULL | Unique association record ID |
| DailyDiaryMasterID | DECIMAL (18) | NOT NULL | Reference to diary entry |
| StaffID | INT | NULL | Associated staff member ID |
| ParentsID | DECIMAL (18) | NULL | Associated parent ID |
| StudentID | DECIMAL (18) | NULL | Associated student ID |
| IsStarred | BIT | NOT NULL | User marked entry as important |
| CanEscalate | BIT | NOT NULL | User has escalation permission |
| CanForward | BIT | NOT NULL | User can forward/share entry |
| CanChangeActions | BIT | NOT NULL | User can modify actions/status |
| CanDelete | BIT | NOT NULL | User can delete entry |
| IsAdmin | BIT | NOT NULL | User has admin-level permissions |
| HasEscalated | BIT | NOT NULL | Entry has been escalated by this user |
| EscalatedByID | DECIMAL (18) | NULL | ID of user who escalated |
| RecordDateTime | DATE | NULL | Date record was created |

**Constraints:**
- Primary Key (Clustered): `PK_DailyDiaryAssociatedPeople` on `DailyDiaryAssociatedPeopleID`
- No Foreign Keys (DailyDiaryMasterID not explicitly FKed)

**Indexes (Non-Clustered):**
1. `IX_DailyDiaryAssociatedPeople_ParentsID` → ParentsID ASC
2. `IX_DailyDiaryAssociatedPeople_StudentID` → StudentID ASC
3. `IX_DailyDiaryAssociatedPeople_StaffID` → StaffID ASC

**Data Flow:**
- **Creation**: When entry created; associates all relevant parties
- **Update**: Permission flags updated based on user actions
- **Consumption**: Filter entries visible to specific user by querying this table

---

#### 4. **DailyDiaryMessagesRead**
**Purpose:** Tracks read/unread status of diary entries and comments per user

**Physical Characteristics:**
- Type: Audit/Status table
- Row Count: Very high (one per user per comment/message)
- Primary Key: `DailyDiaryMessagesReadID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose |
|---|---|---|---|
| DailyDiaryMessagesReadID | DECIMAL (18) | NOT NULL | Unique read record ID |
| ForeignTableID | DECIMAL (18) | NOT NULL | ID from DailyDiaryComments or DailyDiaryMaster |
| IsThisRelatedToMasterRecord | BIT | NOT NULL | 1=Master entry, 0=Comment entry |
| StaffID | INT | NULL | Staff member who read it |
| ParentsID | DECIMAL (18) | NULL | Parent who read it |
| StudentID | DECIMAL (18) | NULL | Student who read it |

**Constraints:**
- Primary Key (Clustered): `PK_DailyDiaryMessagesRead` on `DailyDiaryMessagesReadID`
- No Foreign Keys
- No Unique constraints

**Indexes (Non-Clustered):**
1. `IX_DailyDiaryMessagesRead_ParentsID` → ParentsID ASC
2. `IX_DailyDiaryMessagesRead_StudentID` → StudentID ASC
3. `IX_DailyDiaryMessagesRead_foreigntableid` → ForeignTableID ASC
4. `IX_DailyDiaryMessagesRead_STaffID` → StaffID ASC (Note: Typo in name)

**Data Flow:**
- **Creation**: When user opens/reads a message
- **Consumption**: To determine unread count for user dashboard
- **Reporting**: Message read statistics for compliance/audit

---

### Daily Diary Views

#### 1. **DailyDiaryConsolidatedComments**
```sql
Combines:
- DailyDiaryMaster (DetailedMessage, DDDateAndTime, DDMasterID)
- DailyDiaryComments (Comments, CommentsDateAndTime, DailyDiaryMasterID)

Orders: By DDDateAndTime DESC

Purpose: Single result set of all messages/comments for a diary thread
Usage: Timeline rendering, chronological display
```

#### 2. **DailyDiaryMessagesReadPreConsolidation**
```sql
Joins: DailyDiaryMessagesRead RIGHT OUTER JOIN DailyDiaryComments

Extracts:
- EntityType (derives from comment: 0=Staff, 1=Student, 2=Parent, 3=Unknown)
- EntityID (derives from comment source)
- MessagesReadEntityType (derives from reader: 0=Staff, 1=Student, 2=Parent)
- MessagesReadEntityID (derives from reader ID)
- DailyDiaryMasterID

Purpose: Pre-processing for read/unread consolidation
Complexity: Heavy CASE statements for entity type derivation
```

#### 3. **DailyDiaryMessagesReadConsolidation**
```sql
Groups: DailyDiaryMessagesReadPreConsolidation

Groups By:
- DailyDiaryMasterID
- EntityID
- EntityType
- MessagesReadEntityType
- MessagesReadEntityID

Purpose: Consolidated view of who read what by aggregation
Usage: Dashboard analytics, read-status summaries
```

#### 4. **DistinctPeopleInADailyDiaryWhoHasCommented**
```sql
Unions three queries:
1. Staff commenters (joins Staff table, uses StaffMasterID)
2. Parent commenters (joins Parents table, uses ParentsID)
3. Student commenters (joins Student table, uses StudentID)

Produces:
- DailyDiaryMasterID
- FirstName, LastName
- EntityType ('STAFF', 'PARENTS', 'STUDENT')
- EntityID (StaffMasterID/ParentsID/StudentID)
- EntityTypeNum (0=Staff, 2=Parent, 1=Student)

Purpose: List of unique people who commented on each entry
```

#### 5. **ParentsDDRecords**
```sql
Simple View:
SELECT DISTINCT ParentsID, DDMasterID
FROM DailyDiaryMaster 
LEFT JOIN DailyDiaryAssociatedPeople ON DDMasterID

Purpose: Map of which diary entries are associated with which parents
Usage: Quick parent-entry lookups
```

#### 6. **DDReports**
```sql
Complex reporting view combining:
- DailyDiaryMaster (all columns + derived fields)
- DailyDiaryAssociatedPeople LEFT JOIN

Adds Derived Fields:
- DefaultEntityType (from AssociatedPeople)
- DefaultEntityID (from AssociatedPeople)
- LastDateTime (from DailyDiaryConsolidatedComments)
- OwnerTitle (resolved via Student table lookup)
- RaisedByName (resolved via Staff/Parent/Student lookup)
- ImagePath (conditional formatting)

Order: By LastDateTime DESC, DDMasterID DESC

Purpose: Complete reporting dataset for dashboards/exports
Complexity: Extremely high (nested subqueries, multiple CASE statements)
Performance: Potential bottleneck due to complexity
```

#### 7. **DailyDiaryCommentsCountByAUser**
```sql
Groups: DistinctPeopleInADailyDiaryWhoHasCommented

Produces:
- DailyDiaryMasterID
- EntityType, EntityID
- FirstName, LastName
- EntityTypeNum
- cCount (correlated subquery COUNT of comments)

Purpose: Comment count statistics per person per entry
Usage: Contribution tracking, analytics
Performance: Correlated subquery (potential N+1 issue)
```

---

### Daily Diary Stored Procedures

**Note:** The provided SQL files show template placeholders but no actual implementations for common operations like INSERT, UPDATE, or complex queries. The two provided SPs are incomplete shells:

1. **mbl_spgChatParentUserListByParentIDAndChatMasterID** - Incomplete (Chat-related)
2. **mbl_spgChatStaffUserListByStaffMasterIDAndChatMasterID** - Incomplete (Chat-related)

**Likely Missing Procedures (Based on naming patterns and table structure):**
- `spg_DailyDiaryMasterByDateRange` (GET operations)
- `spg_DailyDiaryCommentsByMasterID` (GET operations)
- `spi_DailyDiaryMaster` (INSERT operations)
- `spd_DailyDiaryMaster` (DELETE operations)
- `spu_DailyDiaryMasterStatus` (UPDATE operations)
- `spg_DailyDiaryAssociatedPeopleByMasterID` (GET operations)

---

## Chat Module

### Overview
The Chat module enables real-time text-based communication between staff and parents with presence awareness and unread message tracking.

### Core Tables

#### 1. **MBL_ChatMaster**
**Purpose:** Defines a communication channel between two parties

**Physical Characteristics:**
- Type: Dimension table (relatively static)
- Row Count: Low volume (one per unique staff-parent relationship)
- Primary Key: `ChatMasterID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose | Notes |
|---|---|---|---|---|
| ChatMasterID | DECIMAL (18) | NOT NULL | Unique chat channel identifier | PK |
| UserRelation | NVARCHAR (100) | NOT NULL | Relationship descriptor | Format: "StaffID_ParentID" or similar |
| StaffNotification | BIT | NULL | Staff notified of new messages | 1=Yes, 0=No |
| ParentNotification | BIT | NULL | Parent notified of new messages | 1=Yes, 0=No |

**Issue Identified:**
- Primary Key on `UserRelation` (NVARCHAR 100) instead of `ChatMasterID`
- This enforces unique relationships but misaligns with typical ID-based design
- NVARCHAR PK is inefficient; should use surrogate key with UNIQUE constraint

**Constraints:**
- Primary Key (Clustered): `PK_MBL_ChatMaster` on `UserRelation` ❌ **WRONG FIELD**
- Should be: `PK_MBL_ChatMaster` on `ChatMasterID`

**Data Flow:**
- **Creation**: When staff initiates chat with parent (or vice versa)
- **Update**: Notification flags toggled by users
- **Consumption**: Queried to fetch chat details and notification preferences

---

#### 2. **MBL_ChatMessages**
**Purpose:** Individual messages within a chat thread

**Physical Characteristics:**
- Type: Fact table (high volume)
- Row Count: Very high (conversation-dependent)
- Primary Key: `MsgID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose |
|---|---|---|---|
| MsgID | DECIMAL (18) | NOT NULL | Unique message identifier |
| ChatMasterID | DECIMAL (18) | NULL | FK to chat channel |
| ChatText | NVARCHAR (MAX) | NULL | Message content |
| MediaPath | NVARCHAR (MAX) | NULL | Path to attached file/image |
| Sender | NVARCHAR (100) | NULL | Sender identifier (name or ID) |
| EntryTime | DATETIME | NULL | Timestamp of message |

**Issues Identified:**
1. `ChatMasterID` is nullable (should NOT NULL)
2. `Sender` is text-based (should be FK to Staff/Parent with type)
3. No sequence/order field (relies on EntryTime for ordering)
4. `ChatText` and `MediaPath` both NULL-able (message requires one)
5. No read/delivered status tracking
6. No soft-delete capability

**Constraints:**
- Primary Key (Clustered): `PK_MBL_ChatMessages` on `MsgID`
- No Foreign Keys (should FK to ChatMasterID)
- No Check constraint to enforce ChatText OR MediaPath populated

**Indexes:**
- Clustered index only (on MsgID)
- **MISSING**: Non-clustered index on (ChatMasterID, EntryTime DESC)

**Data Flow:**
- **Creation**: When user sends message
- **Consumption**: Ordered by EntryTime for thread display
- **Reporting**: Message volume/frequency analytics

---

#### 3. **MBL_ChatCounters**
**Purpose:** Tracks read/unread message counts and metadata per chat channel

**Physical Characteristics:**
- Type: Dimension/Status table
- Row Count: Low (one per ChatMaster + history)
- Primary Key: `CounterMasterID` (DECIMAL 18, IDENTITY 1,1)

**Column Definitions:**

| Column Name | Data Type | Nullable | Purpose |
|---|---|---|---|
| CounterMasterID | DECIMAL (18) | NOT NULL | Unique counter record ID |
| ChatMasterID | DECIMAL (18) | NULL | Reference to chat channel |
| StaffUnread | NUMERIC (18) | NULL | Count of unread messages for staff |
| ParentUnread | NUMERIC (18) | NULL | Count of unread messages for parent |
| TotalChatMessages | NUMERIC (18) | NULL | Total message count in channel |
| LastUpdateTime | DATETIME | NULL | When counters were last updated |

**Critical Issues:**
1. **Self-Referencing FK**: Constraints `FK_MBL_ChatCounters_MBL_ChatCounters` references its own table on CounterMasterID (nonsensical!)
2. `ChatMasterID` is nullable (should NOT NULL)
3. No denormalization benefit (should be computed via COUNT queries instead)
4. Multiple nullable numeric fields (poor design for counters)
5. Synchronization risk between stored counts and actual message counts

**Constraints:**
- Primary Key (Clustered): `PK_MBL_ChatCounters` on `CounterMasterID`
- Foreign Key (Broken): `FK_MBL_ChatCounters_MBL_ChatCounters` on self ❌ **INVALID**

**Data Flow:**
- **Creation**: When first message sent in a channel
- **Update**: After each new message (increment counters)
- **Reset**: When user opens chat (reset unread to 0)
- **Consumption**: Queried for notification badges/counters

---

### Chat Stored Procedures

#### 1. **mbl_spgChatMessageListByChatMasterIDForStaff**
```sql
Input Parameters:
  - @chatMasterID: DECIMAL - Chat channel ID
  - @takePage: INT - Message count to fetch (pagination)

Logic:
  SELECT TOP (@takePage) *
  FROM MBL_ChatMessages
  WHERE ChatMasterID = @chatMasterID
  ORDER BY MsgID DESC

Output:
  - All MBL_ChatMessages columns (paginated, most recent first)

Issues:
  - SELECT * (poor practice, returns all columns)
  - No EntryTime in ORDER BY (relies on MsgID sequence)
  - Assumes MsgID strictly increases with time (true but implicit)
  - No offset/skip for true pagination (only TOP)
```

#### 2. **mbl_spgChatMessageListByChatMasterIDForParent**
```sql
Identical to Staff version above
- Input: @chatMasterID, @takePage
- Logic: TOP @takePage ordered by MsgID DESC
```

#### 3. **mbl_spiChatMessageByChatMasterIDForStaff**
**Purpose:** Insert new message from staff and update unread counters

```sql
Input Parameters:
  - @ChatMasterID: DECIMAL
  - @ChatText: NVARCHAR(MAX)
  - @Sender: NVARCHAR(50)
  - @MediaPath: NVARCHAR(MAX)
  - @EntryTime: DATETIME

Logic Flow:
  1. INSERT into MBL_ChatMessages (all parameters)
  2. SELECT CounterMasterID, StaffUnread FROM MBL_ChatCounters WHERE ChatMasterID
  3. COUNT total messages in channel
  4. IF @counter = 0 (first message):
       INSERT new MBL_ChatCounters (StaffUnread=1, ParentUnread=0)
     ELSE:
       UPDATE MBL_ChatCounters (StaffUnread+=1)

Issues:
  - Race condition: Check then insert/update not atomic
  - Sender is NVARCHAR (should be ID with foreign key)
  - No transaction/rollback handling
  - Manual counter increment error-prone
  - Unused variable initialization (@counter=0, @staffUnread=0)
```

#### 4. **mbl_spiChatMessageByChatMasterIDForParent**
**Purpose:** Insert new message from parent and update unread counters

```sql
Identical to Staff version except:
  - Updates ParentUnread instead of StaffUnread
  - Sets StaffUnread=0 when first message
```

#### 5. **mbl_spgNewMessageCountForStaffByChatMasterID**
**Purpose:** Get unread count for staff in a chat channel

```sql
Input Parameters:
  - @ChatMasterID: DECIMAL

Logic:
  SELECT ParentUnread FROM MBL_ChatCounters WHERE ChatMasterID = @ChatMasterID

Output:
  - Single NUMERIC value (unread parent messages for staff)

Issue:
  - Returns ParentUnread for staff (counterintuitive naming)
  - Should return StaffUnread for staff's own unread count from parent
```

#### 6. **mbl_spgNewMessageCountForParentByChatMasterID**
**Purpose:** Get unread count for parent in a chat channel

```sql
Input Parameters:
  - @ChatMasterID: DECIMAL

Logic:
  SELECT StaffUnread FROM MBL_ChatCounters WHERE ChatMasterID = @ChatMasterID

Output:
  - Single NUMERIC value (unread staff messages for parent)

Issue:
  - Returns StaffUnread for parent (counterintuitive naming)
  - Opposite logic from staff version (confusing symmetry)
```

#### 7. **mbl_spgChatParentUserListByParentIDAndChatMasterID**
**Purpose:** Get chat info for parent + associated student details

```sql
Input Parameters:
  - @ParentID: DECIMAL
  - @ChatMasterID: DECIMAL

Logic:
  1. Declare variables for unread count, last message, timestamp, notifications
  2. Query MBL_ChatCounters for ParentUnread
  3. If NULL, set to 0
  4. Query MBL_ChatMaster for StaffNotification, ParentNotification
  5. Query MBL_ChatMessages (TOP 1 DESC) for last message, time
  6. If last message NULL, set to empty string
  7. JOIN Student to Parents to get GradeSectionMapping
  8. Construct full name: "FirstName LastName (Grade-Section)"
  9. Conditional image path (default avatar or custom)
  10. SELECT student details + computed chat info

Output Columns:
  - FirstName, LastName, Email, FullName, Sex, StudentID, ParentsID
  - ImagePath (computed)
  - ChatMasterID, UnReadMessageCount, lastMessage, lastMessageTime
  - StaffNotification, ParentNotification

Observations:
  - Variable overloading (declares then overwrites)
  - Hard-coded image URL prefix
  - Multi-row output but TOP 1 on queries (potential data loss)
  - Inefficient: Multiple separate queries instead of single JOIN
```

#### 8. **mbl_spgChatStaffUserListByStaffMasterIDAndChatMasterID**
**Purpose:** Get chat info for staff + associated staff details

```sql
Identical structure to Parent version except:
  - Uses @StaffMasterID parameter
  - Joins to WorkingStaffList instead of Student/Parents
  - Returns staff columns (WorkEmail, MiddleName)
  - Queries StaffUnread from MBL_ChatCounters
```

#### 9. **Alumni_SP_GetAllAlumniChatDetailsBySenderId**
**Purpose:** Alumni module chat history

```sql
Input Parameters:
  - @SenderId: INT
  - @UserType: INT (enum: staff, parent, student, alumni)

Logic:
  SELECT all columns FROM AlumniChatMessages
  WHERE SenderId = @SenderId AND UserType = @UserType

Output:
  - AlumniChatMessageID, AlumniBufferId, ChatText, SenderId, UserType
  - CreatedBy, CreatedOn, ModifiedBy, ModifiedOn
  - isSender = 1 (literal, hard-coded)

Issues:
  - Not MBL_Chat tables (separate Alumni chat system!)
  - Hard-coded isSender = 1 (doesn't check if actually sender)
  - Created/Modified audit columns (best practice)
  - Separate from parent-staff chat module (code duplication)
```

---

## Entity Relationships

### Daily Diary Entity Relationship Diagram

```
┌─────────────────────────┐
│   DailyDiaryMaster      │
├─────────────────────────┤
│ DDMasterID (PK)         │
│ DDDateAndTime           │
│ RaisedByEntityType(0,1,2)
│ RaisedByID              │ ──┐
│ ParentsID               │   │ (polymorphic FK)
│ StudentsID              │   │
│ StaffID                 │ ◄─┤
│ OwnerEntityType(0,1,2)  │   │
│ OwnerID                 │ ──┴──┐
│ DetailedMessage         │      │
│ TicketID                │      │
│ ActionStatus            │      │
│ ContextStudentID        │      │
│ MarkdownText            │      │
│ HindiMessage            │      │
│ IsBulk                  │      │
│ BulkID                  │      │
│ IsFirstMessageInBulk    │      │
└─────────────────────────┘      │
         │                       │
         ├──────────┐            │
         │          │            │
         │    ┌─────▼──────────────────────┐
         │    │ DailyDiaryComments         │
         │    ├────────────────────────────┤
         │    │ DailyDiaryCommentsID (PK)  │
         │    │ DailyDiaryMasterID (FK) ◄──┼─┐
         │    │ StaffID (nullable)         │ │ (polymorphic FK)
         │    │ ParentsID (nullable)       │ │
         │    │ StudentID (nullable)       │ │
         │    │ Comments                   │ │
         │    │ CommentsDateAndTime        │ │
         │    │ ActionType                 │ │
         │    │ ContextStudentID           │ │
         │    │ ActionToStaffID            │ │
         │    │ HindiMessage               │ │
         │    │ IsDeleted                  │ │
         │    └────────────────────────────┘ │
         │                                    │
         └────────────────┬───────────────────┘
                          │
         ┌────────────────▼───────────────────┐
         │ DailyDiaryAssociatedPeople         │
         ├────────────────────────────────────┤
         │ DailyDiaryAssociatedPeopleID (PK)  │
         │ DailyDiaryMasterID (FK)            │
         │ StaffID (nullable)                 │
         │ ParentsID (nullable)               │
         │ StudentID (nullable)               │
         │ IsStarred                          │
         │ CanEscalate, CanForward, ...       │
         │ EscalatedByID                      │
         │ RecordDateTime                     │
         └────────────────────────────────────┘

┌──────────────────────────────────────┐
│  DailyDiaryMessagesRead              │
├──────────────────────────────────────┤
│  DailyDiaryMessagesReadID (PK)       │
│  ForeignTableID (FK to Comments/Master)
│  IsThisRelatedToMasterRecord (flag)  │
│  StaffID (nullable)                  │
│  ParentsID (nullable)                │
│  StudentID (nullable)                │
└──────────────────────────────────────┘

KEY ISSUES:
❌ No explicit foreign keys (polymorphic relationship implemented via code)
❌ Entity type stored as enum codes (magic numbers: 0, 1, 2, 3)
❌ Multiple nullable entity ID columns (StaffID, ParentsID, StudentID)
❌ Referential integrity not enforced by database
```

### Chat Entity Relationship Diagram

```
┌──────────────────────────┐
│   MBL_ChatMaster         │
├──────────────────────────┤
│ ChatMasterID (PK)        │
│ UserRelation (UNIQUE!) ◄─┼─── ❌ PK IS ON WRONG COLUMN
│ StaffNotification        │
│ ParentNotification       │
└──────────────────────────┘
         │
         │ (1:M)
         │
    ┌────▼───────────────────────┐
    │   MBL_ChatMessages         │
    ├────────────────────────────┤
    │ MsgID (PK)                 │
    │ ChatMasterID (FK, nullable)◄── ❌ SHOULD NOT NULL
    │ ChatText (nullable)        │
    │ MediaPath (nullable)       │
    │ Sender (NVARCHAR)          │
    │ EntryTime (nullable)       │
    └────────────────────────────┘

┌────────────────────────────────┐
│   MBL_ChatCounters             │
├────────────────────────────────┤
│ CounterMasterID (PK)           │
│ ChatMasterID (FK, nullable)    │
│ StaffUnread (nullable)         │
│ ParentUnread (nullable)        │
│ TotalChatMessages (nullable)   │
│ LastUpdateTime (nullable)      │
│ FK_Self: BROKEN ❌             │
└────────────────────────────────┘

KEY ISSUES:
❌ ChatMaster PK on UserRelation (wrong field)
❌ ChatCounters self-referencing FK (invalid)
❌ ChatMessages.ChatMasterID nullable
❌ Sender is text (should be ID with type)
❌ No message read/delivery status
❌ No polymorphic FK pattern (just text sender)
❌ Separate AlumniChatMessages table (duplication)
```

---

## Data Flow & Workflows

### Daily Diary Workflow

**Scenario 1: Staff Creates an Entry**
```
1. Staff opens app, navigates to Daily Diary
2. Staff creates entry with:
   - Message: "Student was absent, need parent contact"
   - Associated student, parents
   - Relevance: Class, Conduct, Health, etc.
   
3. Backend INSERT into DailyDiaryMaster:
   - RaisedByEntityType = 0 (Staff)
   - RaisedByID = Staff.StaffMasterID
   - DDDateAndTime = NOW()
   - DetailedMessage = user input
   - ParentsID, StudentsID, StaffID populated
   - OwnerEntityType = 0, OwnerID = Staff.StaffMasterID
   - ActionStatus = 0 (Open)
   
4. INSERT into DailyDiaryAssociatedPeople for each recipient:
   - Parent as recipient: CanDelete=1, CanChangeActions=0
   - Staff as recipient: CanDelete=1, CanChangeActions=1
   - Student as recipient: CanDelete=0, CanChangeActions=0

5. Push notification sent to parents/relevant parties
6. Entry appears in parent/staff dashboards
```

**Scenario 2: Parent Adds Comment**
```
1. Parent opens Daily Diary entry on app
2. Parent reads message and comments
3. Backend INSERT into DailyDiaryComments:
   - DailyDiaryMasterID = entry ID
   - ParentsID = parent ID
   - Comments = parent's response
   - CommentsDateAndTime = NOW()
   - ActionType = "RESPONSE" or similar code
   - ContextStudentID = student ID
   
4. INSERT into DailyDiaryMessagesRead to track parent read status
5. Notification sent to staff who can see this comment
6. Comment appears in DailyDiaryConsolidatedComments view
```

**Scenario 3: Staff Escalates Entry to Counselor**
```
1. Staff views entry and clicks "Escalate"
2. Backend UPDATE DailyDiaryMaster:
   - ActionStatus = 2 (Escalated)
   
3. INSERT into DailyDiaryAssociatedPeople for Counselor:
   - StaffID = Counselor ID
   - CanEscalate = 1
   - CanChangeActions = 1
   
4. UPDATE DailyDiaryAssociatedPeople for original creator:
   - HasEscalated = 1
   - EscalatedByID = escalating staff ID
   
5. Entry moves to Counselor's queue
6. Email sent to counselor
```

### Chat Workflow

**Scenario 1: Staff Initiates Chat**
```
1. Staff opens parent communication screen
2. Selects parent to chat with
3. System checks if ChatMaster exists for Staff-Parent pair:
   - If not, INSERT into MBL_ChatMaster:
     * UserRelation = "Staff_123_Parent_456" (or similar)
     * StaffNotification = 1, ParentNotification = 1
     * ChatMasterID = auto-generated
   
4. Staff sends message via mbl_spiChatMessageByChatMasterIDForStaff:
   - INSERT into MBL_ChatMessages
   - UPDATE MBL_ChatCounters (ParentUnread += 1)
   
5. Parent receives push notification
6. Message appears in parent's chat list
```

**Scenario 2: Parent Opens Chat with Staff**
```
1. Parent opens app notification for new message
2. Parent opens chat thread with Staff
3. System queries mbl_spgChatMessageListByChatMasterIDForParent:
   - Fetches last 50 messages (TOP 50, ORDER BY MsgID DESC)
   
4. System queries mbl_spgChatParentUserListByParentIDAndChatMasterID:
   - Gets chat metadata (last message, unread count, notifications)
   
5. Parent reads all messages
6. System resets ParentUnread count to 0 in MBL_ChatCounters
   (implicit in UI, not in provided SP)
```

**Scenario 3: Two-Way Conversation**
```
Staff → Parent (via mbl_spiChatMessageByChatMasterIDForStaff)
  Message inserted, ParentUnread incremented

Parent → Staff (via mbl_spiChatMessageByChatMasterIDForParent)
  Message inserted, StaffUnread incremented

Each opens their respective chat list:
  - Calls mbl_spgChatMessageListByChatMasterIDForStaff/Parent
  - Fetches last N messages
  - Displays in chronological order

Unread counts managed by MBL_ChatCounters
```

---

## Anomalies & Issues

### Critical Issues

| Severity | Issue | Location | Impact | Recommendation |
|---|---|---|---|---|
| **CRITICAL** | Self-referencing FK `FK_MBL_ChatCounters_MBL_ChatCounters` | MBL_ChatCounters | DB integrity violated | Remove immediately; add proper FK to MBL_ChatMaster |
| **CRITICAL** | PK on wrong column (UserRelation) | MBL_ChatMaster | Cannot use ChatMasterID as PK; violates relational design | Swap PK to ChatMasterID, make UserRelation UNIQUE |
| **CRITICAL** | Polymorphic FK without enforcement | DailyDiaryMaster, Comments | Orphaned records possible; no referential integrity | Implement proper FKs or use subtype tables (Staff, Parent, Student) |
| **HIGH** | Missing explicit FKs | DailyDiaryComments (ForeignTableID unclear) | Data corruption risk | Add constraints: DailyDiaryMasterID → DailyDiaryMaster.DDMasterID |
| **HIGH** | Nullable ForeignKey | MBL_ChatMessages.ChatMasterID | Orphaned messages without channel | Make NOT NULL; add FK constraint |
| **HIGH** | Race condition in insert | mbl_spiChatMessageByChatMasterIDForStaff/Parent | Counter increment may fail or double-count under concurrency | Use transaction; use triggers; or replace with COUNT query |

### Major Issues

| Issue | Location | Impact | Recommendation |
|---|---|---|---|
| Entity type encoding (0,1,2,3) | DailyDiaryMaster & all related tables | Magic numbers; error-prone; implicit business logic | Create `EntityType` lookup table; use enums in code |
| Hard-coded URL prefix | mbl_spgChatStaffUserListByStaffMasterIDAndChatMasterID | Breaks on URL change; not maintainable | Move to config; use stored config table |
| SELECT * in queries | mbl_spgChatMessageListByChatMasterIDForStaff/Parent | Overwinding; unnecessary data transfer | Specify columns: MsgID, ChatText, MediaPath, Sender, EntryTime |
| Correlated subquery for counting | DailyDiaryCommentsCountByAUser | N+1 query pattern; poor performance at scale | Use JOIN + GROUP BY instead |
| Nullable counter columns | MBL_ChatCounters | Ambiguous NULL vs 0; NULL check required in code | Default to 0; use NOT NULL constraint |
| Separate AlumniChatMessages table | Alumni module | Code duplication; maintenance burden | Consolidate with MBL_Chat tables |
| Missing indexes | MBL_ChatMessages | Query performance on ChatMasterID, EntryTime | Add: (ChatMasterID, EntryTime DESC) |
| No transactions in SPs | All insert/update SPs | Partial failures; inconsistent state | Wrap in BEGIN TRANSACTION...COMMIT |
| Soft-delete without audit | DailyDiaryMaster, Comments | No tracking of who/when deleted | Add: DeletedBy, DeletedDate columns |
| Missing audit columns | MBL_ChatMessages | No tracking of modifications | Add: CreatedBy, CreatedDate, ModifiedBy, ModifiedDate |
| Redundant message counters | MBL_ChatCounters | Sync risk between counts and actual data | Remove denormalization; use COUNT() in queries or materialized view |

### Design Issues

| Issue | Impact | Root Cause | Solution |
|---|---|---|---|
| Incomplete SP templates | SPs incomplete/placeholder-only | Development artifact left in prod | Implement missing procedures |
| No DELETE procedures | Logical deletes only; no purging | Soft-delete design | Implement retention policy; add archive proc |
| Views with complex logic | Hard to maintain; poor performance | Mixing reporting with transactional data | Separate reporting schema; use materialized view |
| Inconsistent naming | OwnerEnittyTYpe typo; STaffID typo | Code review gaps | Audit and fix all object names |
| No partitioning strategy | Large tables slow queries | High transaction volume | Implement time-based partitioning on DDDateAndTime |
| Missing statistics | Query optimizer can't optimize | No maintenance script | Add: UPDATE STATISTICS job |

### Performance Issues

| Issue | Table | Query | Impact | Fix |
|---|---|---|---|---|
| Full table scan | DailyDiaryMaster | Range queries on DDDateAndTime | Slow dashboards | Index exists but may need tuning |
| Unordered by EntryTime | MBL_ChatMessages | Message thread queries | Relying on MsgID sequence | Add (ChatMasterID, EntryTime DESC) index |
| No partition pruning | DailyDiaryMaster | Historical queries | Full scan over years of data | Partition by year; implement archiving |
| Complex view overhead | DDReports | Reporting dashboards | Multiple subqueries, CASE statements | Pre-aggregate; use materialized view |
| N+1 correlated subqueries | DailyDiaryCommentsCountByAUser | Analytics | One query per row | Refactor to window function or JOIN |

---

## Summary of Findings

### Strengths
✅ Comprehensive feature set (comments, escalation, bulk, translations)  
✅ Read/unread tracking infrastructure  
✅ Permission/authorization framework (DailyDiaryAssociatedPeople)  
✅ Multi-language support (HindiMessage)  
✅ Audit trail via soft-deletes  
✅ Multiple reporting views  

### Weaknesses
❌ Critical FK and PK integrity issues  
❌ Polymorphic relationships without proper constraints  
❌ Performance issues (missing indexes, correlated subqueries)  
❌ Incomplete SP implementations  
❌ Denormalization without proper synchronization  
❌ Poor naming consistency  
❌ Lack of audit columns  
❌ No transaction handling  
❌ Separate alumni chat duplicates code  

### Immediate Actions Required
1. Fix self-referencing FK in MBL_ChatCounters
2. Correct PK in MBL_ChatMaster
3. Add missing Foreign Key constraints
4. Implement transaction handling in SPs
5. Add missing non-clustered indexes
6. Complete SP implementations

---

**Document prepared for:** Development Team, Database Architects, Product Owners  
**Next Steps:** See InventoryDB_Suggestions.md for modernization recommendations
