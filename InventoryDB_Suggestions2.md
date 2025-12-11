# Database Modernization & Optimization Suggestions
## Daily Diary & Chat Modules - InventoryDB_Project

**Version:** 1.0  
**Date:** December 11, 2025  
**Scope:** Architectural improvements, schema normalization, API-ready design

---

## Table of Contents
1. [Executive Overview](#executive-overview)
2. [Schema Redesign](#schema-redesign)
3. [Data Normalization](#data-normalization)
4. [Index Strategy](#index-strategy)
5. [API-Ready DTOs](#api-ready-dtos)
6. [Stored Procedure Modernization](#stored-procedure-modernization)
7. [Performance Optimization](#performance-optimization)
8. [Implementation Roadmap](#implementation-roadmap)

---

## Executive Overview

### Goals
- **Simplify** schema by removing redundancy and enforcing referential integrity
- **Normalize** data structures to eliminate polymorphic FK workarounds
- **Optimize** queries for modern API patterns (pagination, filtering, sorting)
- **Secure** data with proper constraints and audit trails
- **Scale** to handle growth in users and messages
- **Maintain** backward compatibility during migration

### Key Principles
1. **Explicit Constraints** over implicit business logic
2. **Surrogate Keys** for all PK/FK relationships
3. **Audit Columns** for compliance and troubleshooting
4. **Entity Inheritance** (subtype tables) over polymorphic codes
5. **Composable Views** for different API use cases
6. **Materialized Reporting** for analytics
7. **Proper Indexing** for query performance

---

## Schema Redesign

### Current Problems
```
❌ Polymorphic FKs (RaisedByEntityType + RaisedByID)
❌ Multiple nullable entity columns (StaffID, ParentsID, StudentID)
❌ No referential integrity constraints
❌ Self-referencing FK broken in MBL_ChatCounters
❌ PK on wrong column in MBL_ChatMaster
❌ Sender as text instead of ID in MBL_ChatMessages
```

### Proposed New Schema

#### Phase 1: Create Entity Type Reference Tables

```sql
-- Create explicit entity type dimension
CREATE TABLE dbo.EntityType (
    EntityTypeID TINYINT PRIMARY KEY,
    EntityTypeName NVARCHAR(50) NOT NULL UNIQUE,
    Description NVARCHAR(255),
    IsActive BIT NOT NULL DEFAULT 1
);

GO

INSERT INTO dbo.EntityType (EntityTypeID, EntityTypeName, Description)
VALUES 
    (0, 'Staff', 'School staff member'),
    (1, 'Student', 'Student'),
    (2, 'Parent', 'Parent/Guardian'),
    (3, 'Other', 'Other entity type');

GO

-- Instead of polymorphic FK, create a Person/Actor table
-- This normalizes entity references
CREATE TABLE dbo.DailyDiaryActor (
    ActorID DECIMAL(18) IDENTITY(1,1) NOT NULL PRIMARY KEY,
    EntityTypeID TINYINT NOT NULL,
    EntityID DECIMAL(18) NOT NULL,
    ActorName NVARCHAR(150) NOT NULL,
    ActorEmail NVARCHAR(100) NULL,
    ActorImagePath NVARCHAR(500) NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_DailyDiaryActor_EntityType FOREIGN KEY (EntityTypeID) 
        REFERENCES dbo.EntityType(EntityTypeID),
    CONSTRAINT UC_DailyDiaryActor_Entity UNIQUE (EntityTypeID, EntityID)
);

GO

-- Index for quick lookups
CREATE NONCLUSTERED INDEX IX_DailyDiaryActor_Entity 
    ON dbo.DailyDiaryActor(EntityTypeID, EntityID);
```

#### Phase 2: Redesign DailyDiaryMaster

```sql
-- REDESIGNED DailyDiaryMaster (simplified, normalized)
CREATE TABLE dbo.DailyDiaryMaster (
    DDMasterID DECIMAL(18) IDENTITY(1,1) NOT NULL PRIMARY KEY CLUSTERED,
    
    -- Metadata
    DDDateAndTime DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ContextStudentID DECIMAL(18) NOT NULL,  -- Who the diary is about
    
    -- Ownership
    RaisedByActorID DECIMAL(18) NOT NULL,   -- Who created it
    OwnerActorID DECIMAL(18) NOT NULL,      -- Who owns it
    
    -- Content
    DetailedMessage NVARCHAR(MAX) NOT NULL,
    MarkdownText NVARCHAR(MAX) NULL,
    HindiMessage NVARCHAR(MAX) NULL,
    
    -- Status & Tracking
    ActionStatusID TINYINT NOT NULL DEFAULT 0,  -- FK to ActionStatus table
    TicketID DECIMAL(18) NULL,                  -- FK to Ticket system
    PrintableID VARCHAR(30) NULL,               -- Human-readable ID
    
    -- Bulk Operations
    IsBulk BIT NOT NULL DEFAULT 0,
    BulkID DECIMAL(18) NULL,
    IsFirstMessageInBulk BIT NULL,
    
    -- Audit
    IsDeleted BIT NOT NULL DEFAULT 0,
    DeletedDate DATETIME2 NULL,
    DeletedByActorID DECIMAL(18) NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CreatedByActorID DECIMAL(18) NOT NULL,
    ModifiedDate DATETIME2 NULL,
    ModifiedByActorID DECIMAL(18) NULL,
    
    -- Constraints
    CONSTRAINT FK_DDMaster_RaisedByActor FOREIGN KEY (RaisedByActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_DDMaster_OwnerActor FOREIGN KEY (OwnerActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_DDMaster_DeletedByActor FOREIGN KEY (DeletedByActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_DDMaster_CreatedByActor FOREIGN KEY (CreatedByActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_DDMaster_ModifiedByActor FOREIGN KEY (ModifiedByActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_DDMaster_ActionStatus FOREIGN KEY (ActionStatusID)
        REFERENCES dbo.ActionStatus(ActionStatusID),
    CONSTRAINT CK_DDMaster_NotDeleted CHECK (IsDeleted IN (0,1))
);

GO

-- Critical indexes for query performance
CREATE NONCLUSTERED INDEX IX_DDMaster_ContextStudentID 
    ON dbo.DailyDiaryMaster(ContextStudentID, DDDateAndTime DESC) 
    INCLUDE (ActionStatusID, IsDeleted);

CREATE NONCLUSTERED INDEX IX_DDMaster_RaisedByActorID 
    ON dbo.DailyDiaryMaster(RaisedByActorID, DDDateAndTime DESC);

CREATE NONCLUSTERED INDEX IX_DDMaster_OwnerActorID 
    ON dbo.DailyDiaryMaster(OwnerActorID, DDDateAndTime DESC);

CREATE NONCLUSTERED INDEX IX_DDMaster_ActionStatus 
    ON dbo.DailyDiaryMaster(ActionStatusID, DDDateAndTime DESC) 
    WHERE IsDeleted = 0;

CREATE NONCLUSTERED INDEX IX_DDMaster_DateRange 
    ON dbo.DailyDiaryMaster(DDDateAndTime DESC) 
    INCLUDE (ContextStudentID, ActionStatusID) 
    WHERE IsDeleted = 0;

GO

-- Action Status lookup table
CREATE TABLE dbo.ActionStatus (
    ActionStatusID TINYINT PRIMARY KEY,
    StatusName NVARCHAR(50) NOT NULL UNIQUE,
    Description NVARCHAR(255),
    DisplayColor NVARCHAR(10) NULL
);

GO

INSERT INTO dbo.ActionStatus VALUES
    (0, 'Open', 'Entry is open and unresolved', '#FF9800'),
    (1, 'InProgress', 'Entry being addressed', '#2196F3'),
    (2, 'Escalated', 'Entry escalated to higher authority', '#F44336'),
    (3, 'Resolved', 'Entry resolved', '#4CAF50'),
    (4, 'Closed', 'Entry closed', '#757575');
```

#### Phase 3: Redesign DailyDiaryComments

```sql
CREATE TABLE dbo.DailyDiaryComments (
    CommentID DECIMAL(18) IDENTITY(1,1) PRIMARY KEY CLUSTERED,
    DDMasterID DECIMAL(18) NOT NULL,
    CommentByActorID DECIMAL(18) NOT NULL,
    CommentText VARCHAR(MAX) NOT NULL,
    CommentSummary VARCHAR(300) NULL,  -- Short preview
    ContextStudentID DECIMAL(18) NOT NULL,
    ActionTypeID TINYINT NULL,
    ActionToActorID DECIMAL(18) NULL,  -- If assigning action to someone
    
    -- Audit
    CommentDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsDeleted BIT NOT NULL DEFAULT 0,
    DeletedDate DATETIME2 NULL,
    
    -- Multi-language
    HindiMessage NVARCHAR(MAX) NULL,
    
    -- Constraints
    CONSTRAINT FK_DDComment_Master FOREIGN KEY (DDMasterID)
        REFERENCES dbo.DailyDiaryMaster(DDMasterID),
    CONSTRAINT FK_DDComment_CommentByActor FOREIGN KEY (CommentByActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_DDComment_ActionToActor FOREIGN KEY (ActionToActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID)
);

GO

CREATE NONCLUSTERED INDEX IX_DDComment_Master 
    ON dbo.DailyDiaryComments(DDMasterID, CommentDate DESC) 
    WHERE IsDeleted = 0;

CREATE NONCLUSTERED INDEX IX_DDComment_Actor 
    ON dbo.DailyDiaryComments(CommentByActorID, CommentDate DESC) 
    WHERE IsDeleted = 0;
```

#### Phase 4: Redesign Permissions/Accessibility

```sql
-- Replace DailyDiaryAssociatedPeople with explicit permission table
CREATE TABLE dbo.DailyDiaryPermission (
    PermissionID DECIMAL(18) IDENTITY(1,1) PRIMARY KEY,
    DDMasterID DECIMAL(18) NOT NULL,
    ActorID DECIMAL(18) NOT NULL,
    
    -- Permissions (should be separate table for true normalization)
    CanView BIT NOT NULL DEFAULT 1,
    CanComment BIT NOT NULL DEFAULT 0,
    CanEscalate BIT NOT NULL DEFAULT 0,
    CanResolve BIT NOT NULL DEFAULT 0,
    CanDelete BIT NOT NULL DEFAULT 0,
    CanAssignActions BIT NOT NULL DEFAULT 0,
    
    -- Audit
    IsStarred BIT NOT NULL DEFAULT 0,
    PermissionGrantedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    PermissionRevokedDate DATETIME2 NULL,
    
    CONSTRAINT FK_DDPerm_Master FOREIGN KEY (DDMasterID)
        REFERENCES dbo.DailyDiaryMaster(DDMasterID),
    CONSTRAINT FK_DDPerm_Actor FOREIGN KEY (ActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT UC_DDPerm_Unique UNIQUE (DDMasterID, ActorID)
);

GO

CREATE NONCLUSTERED INDEX IX_DDPerm_Entry 
    ON dbo.DailyDiaryPermission(DDMasterID) 
    INCLUDE (ActorID, CanView, CanComment);

CREATE NONCLUSTERED INDEX IX_DDPerm_Actor 
    ON dbo.DailyDiaryPermission(ActorID) 
    INCLUDE (DDMasterID, CanView);
```

#### Phase 5: Redesign Chat System

```sql
-- Redesigned MBL_ChatMaster
CREATE TABLE dbo.ChatChannel (
    ChannelID DECIMAL(18) IDENTITY(1,1) PRIMARY KEY CLUSTERED,
    StaffActorID DECIMAL(18) NOT NULL,
    ParentActorID DECIMAL(18) NOT NULL,
    
    -- Metadata
    ChannelCreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastMessageDate DATETIME2 NULL,
    
    -- Settings (remove from counters table)
    StaffMutedUntil DATETIME2 NULL,
    ParentMutedUntil DATETIME2 NULL,
    StaffArchivedDate DATETIME2 NULL,
    ParentArchivedDate DATETIME2 NULL,
    
    -- Constraints
    CONSTRAINT FK_Chat_StaffActor FOREIGN KEY (StaffActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT FK_Chat_ParentActor FOREIGN KEY (ParentActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT UC_Chat_Unique UNIQUE (StaffActorID, ParentActorID),
    CONSTRAINT CK_Chat_DifferentActors CHECK (StaffActorID <> ParentActorID)
);

GO

-- Redesigned MBL_ChatMessages
CREATE TABLE dbo.ChatMessage (
    MessageID DECIMAL(18) IDENTITY(1,1) PRIMARY KEY CLUSTERED,
    ChannelID DECIMAL(18) NOT NULL,
    SenderActorID DECIMAL(18) NOT NULL,
    
    -- Content (enforce one or both exists)
    MessageText NVARCHAR(MAX) NULL,
    MediaPath NVARCHAR(500) NULL,
    
    -- Message Status
    MessageTypeID TINYINT DEFAULT 1,  -- 1=Text, 2=Image, 3=File, 4=Audio, 5=System
    IsRead BIT NOT NULL DEFAULT 0,
    DeliveredDate DATETIME2 NULL,
    ReadDate DATETIME2 NULL,
    
    -- Timestamps
    SentDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    EditedDate DATETIME2 NULL,
    
    -- Audit
    IsDeleted BIT NOT NULL DEFAULT 0,
    DeletedDate DATETIME2 NULL,
    
    -- Constraints
    CONSTRAINT FK_Msg_Channel FOREIGN KEY (ChannelID)
        REFERENCES dbo.ChatChannel(ChannelID),
    CONSTRAINT FK_Msg_Sender FOREIGN KEY (SenderActorID)
        REFERENCES dbo.DailyDiaryActor(ActorID),
    CONSTRAINT CK_Msg_Content CHECK (MessageText IS NOT NULL OR MediaPath IS NOT NULL)
);

GO

-- Critical indexes for chat performance
CREATE NONCLUSTERED INDEX IX_ChatMsg_Channel 
    ON dbo.ChatMessage(ChannelID, SentDate DESC) 
    WHERE IsDeleted = 0;

CREATE NONCLUSTERED INDEX IX_ChatMsg_Sender 
    ON dbo.ChatMessage(SenderActorID, SentDate DESC);

CREATE NONCLUSTERED INDEX IX_ChatMsg_Unread 
    ON dbo.ChatMessage(ChannelID, IsRead, SentDate DESC) 
    WHERE IsDeleted = 0;

GO

-- Message Type lookup
CREATE TABLE dbo.MessageType (
    MessageTypeID TINYINT PRIMARY KEY,
    TypeName NVARCHAR(50) NOT NULL
);

GO

INSERT INTO dbo.MessageType VALUES
    (1, 'Text'),
    (2, 'Image'),
    (3, 'File'),
    (4, 'Audio'),
    (5, 'SystemMessage');

GO

-- Remove MBL_ChatCounters entirely, use materialized view instead
CREATE MATERIALIZED VIEW dbo.vwChatChannelStats
WITH SCHEMABINDING
AS
SELECT 
    c.ChannelID,
    c.StaffActorID,
    c.ParentActorID,
    COALESCE(SUM(CASE WHEN m.IsRead = 0 AND m.SenderActorID = c.ParentActorID THEN 1 ELSE 0 END), 0) AS StaffUnreadCount,
    COALESCE(SUM(CASE WHEN m.IsRead = 0 AND m.SenderActorID = c.StaffActorID THEN 1 ELSE 0 END), 0) AS ParentUnreadCount,
    COALESCE(COUNT(m.MessageID), 0) AS TotalMessageCount,
    MAX(m.SentDate) AS LastMessageDate
FROM dbo.ChatChannel c
LEFT JOIN dbo.ChatMessage m ON c.ChannelID = m.ChannelID AND m.IsDeleted = 0
GROUP BY c.ChannelID, c.StaffActorID, c.ParentActorID;

GO

-- Indexes on materialized view
CREATE UNIQUE CLUSTERED INDEX PK_vwChatStats ON dbo.vwChatChannelStats(ChannelID);
CREATE NONCLUSTERED INDEX IX_vwChatStats_Staff ON dbo.vwChatChannelStats(StaffActorID);
CREATE NONCLUSTERED INDEX IX_vwChatStats_Parent ON dbo.vwChatChannelStats(ParentActorID);
```

---

## Data Normalization

### Issue #1: Polymorphic Relationships
**Current:** Entity type + ID columns  
**Problem:** No constraint enforcement; allows orphaned records  
**Solution:** Normalize to DailyDiaryActor junction table with entity type

**Migration:**
```sql
-- Insert all actors from existing tables
INSERT INTO dbo.DailyDiaryActor (EntityTypeID, EntityID, ActorName)
SELECT 0, StaffMasterID, FirstName + ' ' + LastName
FROM dbo.Staff
UNION ALL
SELECT 2, ParentsID, UserID
FROM dbo.Parents
UNION ALL
SELECT 1, StudentID, FirstName + ' ' + LastName
FROM dbo.Student;
```

### Issue #2: Multiple Nullable Entity Columns
**Current:**
```
| StaffID | ParentsID | StudentID |
|---------|-----------|-----------|
| 123     | NULL      | NULL      |
| NULL    | 456       | NULL      |
| NULL    | NULL      | 789       |
```

**Problem:** Wasted space; ambiguity; data quality issues  
**Solution:** Single ActorID column with entity type reference

**Migration:** Use DailyDiaryActor table created above

### Issue #3: Sender as Text
**Current:** Sender = "John Doe" (NVARCHAR)  
**Problem:** Duplicated data; difficult to update; no FK  
**Solution:** Sender as FK to DailyDiaryActor

---

## Index Strategy

### Current Problems
- Missing index on (ChatMasterID, EntryTime DESC)
- Single-column indexes inefficient
- No filtered indexes (exclude deleted/inactive records)
- No included columns for covering queries

### Recommended Index Plan

#### Daily Diary Indexes (Priority Order)

```sql
-- 1. CRITICAL: Date range queries (dashboards, filtering)
CREATE NONCLUSTERED INDEX IX_DDMaster_DateStudent ON dbo.DailyDiaryMaster
    (ContextStudentID, DDDateAndTime DESC)
    INCLUDE (ActionStatusID, IsDeleted)
    WHERE IsDeleted = 0;

-- 2. CRITICAL: Filtering by action status
CREATE NONCLUSTERED INDEX IX_DDMaster_StatusDate ON dbo.DailyDiaryMaster
    (ActionStatusID, DDDateAndTime DESC)
    WHERE IsDeleted = 0;

-- 3. HIGH: Comment threads (get all comments for entry)
CREATE NONCLUSTERED INDEX IX_DDComment_EntryDate ON dbo.DailyDiaryComments
    (DDMasterID, CommentDate DESC)
    INCLUDE (CommentByActorID, CommentText)
    WHERE IsDeleted = 0;

-- 4. HIGH: Permission checks (access control)
CREATE NONCLUSTERED INDEX IX_DDPerm_ActorEntry ON dbo.DailyDiaryPermission
    (ActorID, DDMasterID)
    INCLUDE (CanView, CanComment, CanDelete)
    WHERE PermissionRevokedDate IS NULL;

-- 5. MEDIUM: Search by actor (owner/creator)
CREATE NONCLUSTERED INDEX IX_DDMaster_CreatorDate ON dbo.DailyDiaryMaster
    (RaisedByActorID, DDDateAndTime DESC)
    INCLUDE (ActionStatusID);

-- 6. MEDIUM: Bulk operation tracking
CREATE NONCLUSTERED INDEX IX_DDMaster_Bulk ON dbo.DailyDiaryMaster
    (BulkID, IsFirstMessageInBulk)
    WHERE IsBulk = 1;
```

#### Chat Indexes (Priority Order)

```sql
-- 1. CRITICAL: Message retrieval for thread display
CREATE NONCLUSTERED INDEX IX_ChatMsg_ChannelDate ON dbo.ChatMessage
    (ChannelID, SentDate DESC)
    INCLUDE (MessageText, SenderActorID, IsRead)
    WHERE IsDeleted = 0;

-- 2. CRITICAL: Unread message count
CREATE NONCLUSTERED INDEX IX_ChatMsg_Unread ON dbo.ChatMessage
    (ChannelID, IsRead, SentDate DESC)
    WHERE IsDeleted = 0;

-- 3. HIGH: Channel lookup (unique staff-parent pair)
CREATE UNIQUE NONCLUSTERED INDEX UC_Chat_StaffParent ON dbo.ChatChannel
    (StaffActorID, ParentActorID);

-- 4. HIGH: User's active conversations
CREATE NONCLUSTERED INDEX IX_Chat_StaffChannels ON dbo.ChatChannel
    (StaffActorID, LastMessageDate DESC)
    WHERE StaffArchivedDate IS NULL;

CREATE NONCLUSTERED INDEX IX_Chat_ParentChannels ON dbo.ChatChannel
    (ParentActorID, LastMessageDate DESC)
    WHERE ParentArchivedDate IS NULL;

-- 5. MEDIUM: Sender activity tracking
CREATE NONCLUSTERED INDEX IX_ChatMsg_Sender ON dbo.ChatMessage
    (SenderActorID, SentDate DESC);
```

### Index Maintenance

```sql
-- Add to maintenance job (nightly)
ALTER INDEX ALL ON dbo.DailyDiaryMaster REBUILD;
ALTER INDEX ALL ON dbo.DailyDiaryComments REBUILD;
ALTER INDEX ALL ON dbo.ChatMessage REBUILD;

-- Update statistics
UPDATE STATISTICS dbo.DailyDiaryMaster;
UPDATE STATISTICS dbo.DailyDiaryComments;
UPDATE STATISTICS dbo.ChatMessage;

-- Refresh materialized view
ALTER MATERIALIZED VIEW dbo.vwChatChannelStats REBUILD;
```

---

## API-Ready DTOs

### Request/Response Models

#### Daily Diary API

**Create Diary Entry Request:**
```json
{
  "contextStudentId": 12345,
  "raisedByActorId": 67890,
  "ownerActorId": 67890,
  "detailedMessage": "Student was late to class",
  "markdownText": "**Student** was late to class",
  "actionStatusId": 0,
  "recipients": [
    {
      "actorId": 98765,
      "canComment": true,
      "canEscalate": true,
      "canDelete": false
    }
  ],
  "tags": ["attendance", "behavior"]
}
```

**Diary Entry Response:**
```json
{
  "ddMasterId": 11111,
  "ddDateAndTime": "2025-12-11T10:30:00Z",
  "contextStudentId": 12345,
  "contextStudentName": "John Doe",
  "raisedByActor": {
    "actorId": 67890,
    "entityType": "Staff",
    "name": "Ms. Smith",
    "email": "smith@school.org"
  },
  "ownerActor": { ... },
  "detailedMessage": "Student was late to class",
  "actionStatus": {
    "statusId": 0,
    "statusName": "Open",
    "displayColor": "#FF9800"
  },
  "comments": [
    {
      "commentId": 22222,
      "commentByActor": { ... },
      "commentText": "I spoke with the student",
      "commentDate": "2025-12-11T11:00:00Z",
      "canDelete": true,
      "canEdit": true
    }
  ],
  "permissions": {
    "canComment": true,
    "canEscalate": true,
    "canDelete": false,
    "canResolve": true
  },
  "isStarred": false,
  "printableId": "DD-2025-001234"
}
```

**Diary List Request (Filtering & Pagination):**
```json
{
  "pageNumber": 1,
  "pageSize": 20,
  "sortBy": "ddDateAndTime",
  "sortOrder": "descending",
  "filters": {
    "studentId": 12345,
    "actionStatusIds": [0, 2],
    "dateFrom": "2025-12-01T00:00:00Z",
    "dateTo": "2025-12-31T23:59:59Z",
    "searchText": "attendance",
    "myEntriesOnly": false
  }
}
```

**Diary List Response:**
```json
{
  "totalRecords": 45,
  "pageNumber": 1,
  "pageSize": 20,
  "totalPages": 3,
  "entries": [
    { ... diary entry ... },
    { ... diary entry ... }
  ]
}
```

#### Chat API

**Send Message Request:**
```json
{
  "channelId": 55555,
  "senderActorId": 67890,
  "messageText": "Hi, just checking on attendance",
  "messageType": "Text"
}
```

**Chat Thread Response:**
```json
{
  "channelId": 55555,
  "staffActor": { ... },
  "parentActor": { ... },
  "messages": [
    {
      "messageId": 33333,
      "senderActor": { ... },
      "messageText": "Hi, just checking on attendance",
      "sentDate": "2025-12-11T10:00:00Z",
      "isRead": true,
      "readDate": "2025-12-11T10:05:00Z",
      "canDelete": true,
      "canEdit": true
    },
    { ... }
  ],
  "unreadCount": 0,
  "lastMessageDate": "2025-12-11T10:00:00Z",
  "channelStats": {
    "staffUnreadCount": 0,
    "parentUnreadCount": 0,
    "totalMessageCount": 45
  }
}
```

**Chat Channel List Response:**
```json
{
  "channels": [
    {
      "channelId": 55555,
      "otherPartyActor": { ... },
      "lastMessage": "Hi, just checking on attendance",
      "lastMessageDate": "2025-12-11T10:00:00Z",
      "unreadCount": 2,
      "isMuted": false,
      "isArchived": false,
      "canUnarchive": true
    }
  ],
  "totalChannels": 23,
  "unreadChannelCount": 3
}
```

### Naming Standards

**Table Naming:**
- Master/Dimension: `EntityName` (e.g., `ChatChannel`, `ActionStatus`)
- Junction/Bridge: `Entity1Entity2` (e.g., `DailyDiaryPermission`)
- Audit/History: `EntityHistory` or `EntityAudit`
- View: `v[Purpose]` (e.g., `vChatChannelStats`)

**Column Naming:**
- PK: `TableNameID` or just `ID`
- FK: `ReferencedTableID`
- Flag: `Is[Description]` (e.g., `IsDeleted`)
- DateTime: `[Action]Date` (e.g., `CreatedDate`, `DeletedDate`)
- Name/Text: `[Entity]Name` or `[Entity]Text`

**Procedure Naming:**
- GET (SELECT): `spg_[Entity]By[Filter]` (e.g., `spg_DailyDiaryByStudent`)
- INSERT: `spi_[Entity]` (e.g., `spi_DailyDiaryMaster`)
- UPDATE: `spu_[Entity][Action]` (e.g., `spu_DailyDiaryStatus`)
- DELETE: `spd_[Entity]` (e.g., `spd_DailyDiaryComment`)

---

## Stored Procedure Modernization

### Current Problems
- Template placeholders, not real implementations
- No transaction handling
- Race conditions in counter updates
- Poor error handling
- No pagination support
- Hard-coded values (URLs, limits)

### Redesigned Procedures

#### 1. Create Diary Entry (Transaction-Safe)

```sql
CREATE PROCEDURE dbo.spi_DailyDiaryMaster
    @ContextStudentID DECIMAL(18),
    @RaisedByActorID DECIMAL(18),
    @OwnerActorID DECIMAL(18),
    @DetailedMessage NVARCHAR(MAX),
    @MarkdownText NVARCHAR(MAX) = NULL,
    @HindiMessage NVARCHAR(MAX) = NULL,
    @ActionStatusID TINYINT = 0,
    @TicketID DECIMAL(18) = NULL,
    @IsBulk BIT = 0,
    @BulkID DECIMAL(18) = NULL,
    @IsFirstMessageInBulk BIT = NULL,
    @NewDDMasterID DECIMAL(18) OUTPUT,
    @ErrorMessage NVARCHAR(255) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Validate inputs
        IF @ContextStudentID IS NULL OR @ContextStudentID <= 0
        BEGIN
            SET @ErrorMessage = 'Invalid ContextStudentID';
            ROLLBACK TRANSACTION;
            RETURN 1;
        END;
        
        IF NOT EXISTS(SELECT 1 FROM dbo.DailyDiaryActor WHERE ActorID = @RaisedByActorID)
        BEGIN
            SET @ErrorMessage = 'RaisedByActorID not found';
            ROLLBACK TRANSACTION;
            RETURN 1;
        END;
        
        -- Insert diary entry
        INSERT INTO dbo.DailyDiaryMaster (
            ContextStudentID,
            RaisedByActorID,
            OwnerActorID,
            DetailedMessage,
            MarkdownText,
            HindiMessage,
            ActionStatusID,
            TicketID,
            IsBulk,
            BulkID,
            IsFirstMessageInBulk,
            CreatedByActorID,
            DDDateAndTime
        )
        VALUES (
            @ContextStudentID,
            @RaisedByActorID,
            @OwnerActorID,
            @DetailedMessage,
            @MarkdownText,
            @HindiMessage,
            @ActionStatusID,
            @TicketID,
            @IsBulk,
            @BulkID,
            @IsFirstMessageInBulk,
            @RaisedByActorID,
            GETUTCDATE()
        );
        
        SET @NewDDMasterID = SCOPE_IDENTITY();
        
        -- Generate PrintableID
        UPDATE dbo.DailyDiaryMaster
        SET PrintableID = 'DD-' + YEAR(DDDateAndTime) + '-' + CAST(DDMasterID AS VARCHAR(10))
        WHERE DDMasterID = @NewDDMasterID;
        
        COMMIT TRANSACTION;
        SET @ErrorMessage = NULL;
        RETURN 0;
        
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        SET @ErrorMessage = ERROR_MESSAGE();
        RETURN 1;
    END CATCH;
END;

GO
```

#### 2. Get Diary Entries with Pagination

```sql
CREATE PROCEDURE dbo.spg_DailyDiaryByStudent
    @StudentID DECIMAL(18),
    @ActionStatusID TINYINT = NULL,
    @DateFrom DATETIME2 = NULL,
    @DateTo DATETIME2 = NULL,
    @PageNumber INT = 1,
    @PageSize INT = 20,
    @SortBy NVARCHAR(50) = 'DDDateAndTime',
    @SortOrder NVARCHAR(10) = 'DESC'
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Input validation
    IF @PageNumber < 1 SET @PageNumber = 1;
    IF @PageSize < 1 SET @PageSize = 20;
    IF @PageSize > 100 SET @PageSize = 100;  -- Max page size
    
    DECLARE @Offset INT = (@PageNumber - 1) * @PageSize;
    DECLARE @TotalRecords INT;
    
    -- Count total matching records
    SELECT @TotalRecords = COUNT(*)
    FROM dbo.DailyDiaryMaster
    WHERE ContextStudentID = @StudentID
        AND IsDeleted = 0
        AND (@ActionStatusID IS NULL OR ActionStatusID = @ActionStatusID)
        AND (@DateFrom IS NULL OR DDDateAndTime >= @DateFrom)
        AND (@DateTo IS NULL OR DDDateAndTime <= @DateTo);
    
    -- Return metadata
    SELECT @TotalRecords AS TotalRecords, @PageSize AS PageSize, @PageNumber AS PageNumber;
    
    -- Return paginated results
    SELECT 
        d.DDMasterID,
        d.DDDateAndTime,
        d.ContextStudentID,
        d.DetailedMessage,
        d.ActionStatusID,
        a.StatusName,
        d.RaisedByActorID,
        ra.ActorName AS RaisedByName,
        d.OwnerActorID,
        oa.ActorName AS OwnerName,
        d.PrintableID,
        d.IsBulk,
        d.BulkID,
        (SELECT COUNT(*) FROM dbo.DailyDiaryComments WHERE DDMasterID = d.DDMasterID AND IsDeleted = 0) AS CommentCount,
        (SELECT COUNT(*) FROM dbo.DailyDiaryPermission WHERE DDMasterID = d.DDMasterID) AS PermissionCount
    FROM dbo.DailyDiaryMaster d
    LEFT JOIN dbo.ActionStatus a ON d.ActionStatusID = a.ActionStatusID
    LEFT JOIN dbo.DailyDiaryActor ra ON d.RaisedByActorID = ra.ActorID
    LEFT JOIN dbo.DailyDiaryActor oa ON d.OwnerActorID = oa.ActorID
    WHERE d.ContextStudentID = @StudentID
        AND d.IsDeleted = 0
        AND (@ActionStatusID IS NULL OR d.ActionStatusID = @ActionStatusID)
        AND (@DateFrom IS NULL OR d.DDDateAndTime >= @DateFrom)
        AND (@DateTo IS NULL OR d.DDDateAndTime <= @DateTo)
    ORDER BY 
        CASE WHEN @SortOrder = 'ASC' AND @SortBy = 'DDDateAndTime' THEN d.DDDateAndTime END ASC,
        CASE WHEN @SortOrder = 'DESC' AND @SortBy = 'DDDateAndTime' THEN d.DDDateAndTime END DESC
    OFFSET @Offset ROWS
    FETCH NEXT @PageSize ROWS ONLY;
END;

GO
```

#### 3. Send Chat Message (Atomic)

```sql
CREATE PROCEDURE dbo.spi_ChatMessage
    @ChannelID DECIMAL(18),
    @SenderActorID DECIMAL(18),
    @MessageText NVARCHAR(MAX) = NULL,
    @MediaPath NVARCHAR(500) = NULL,
    @MessageTypeID TINYINT = 1,
    @NewMessageID DECIMAL(18) OUTPUT,
    @ErrorMessage NVARCHAR(255) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Validate
        IF @ChannelID <= 0 OR @SenderActorID <= 0
        BEGIN
            SET @ErrorMessage = 'Invalid ChannelID or SenderActorID';
            ROLLBACK TRANSACTION;
            RETURN 1;
        END;
        
        IF @MessageText IS NULL AND @MediaPath IS NULL
        BEGIN
            SET @ErrorMessage = 'Message text or media path required';
            ROLLBACK TRANSACTION;
            RETURN 1;
        END;
        
        -- Verify sender is part of channel
        DECLARE @ChannelExists INT;
        SELECT @ChannelExists = COUNT(*)
        FROM dbo.ChatChannel
        WHERE ChannelID = @ChannelID
            AND (StaffActorID = @SenderActorID OR ParentActorID = @SenderActorID);
        
        IF @ChannelExists = 0
        BEGIN
            SET @ErrorMessage = 'Sender not authorized for this channel';
            ROLLBACK TRANSACTION;
            RETURN 1;
        END;
        
        -- Insert message
        INSERT INTO dbo.ChatMessage (
            ChannelID,
            SenderActorID,
            MessageText,
            MediaPath,
            MessageTypeID,
            SentDate,
            IsRead
        )
        VALUES (
            @ChannelID,
            @SenderActorID,
            @MessageText,
            @MediaPath,
            @MessageTypeID,
            GETUTCDATE(),
            0
        );
        
        SET @NewMessageID = SCOPE_IDENTITY();
        
        -- Update channel last message date
        UPDATE dbo.ChatChannel
        SET LastMessageDate = GETUTCDATE()
        WHERE ChannelID = @ChannelID;
        
        -- Refresh materialized view (if needed for real-time stats)
        -- ALTER MATERIALIZED VIEW dbo.vwChatChannelStats REBUILD;
        
        COMMIT TRANSACTION;
        SET @ErrorMessage = NULL;
        RETURN 0;
        
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        SET @ErrorMessage = ERROR_MESSAGE();
        RETURN 1;
    END CATCH;
END;

GO
```

#### 4. Get Chat Messages (Paginated)

```sql
CREATE PROCEDURE dbo.spg_ChatMessagesByChannel
    @ChannelID DECIMAL(18),
    @PageNumber INT = 1,
    @PageSize INT = 50,
    @IncludeRead BIT = 1
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @PageNumber < 1 SET @PageNumber = 1;
    IF @PageSize < 1 SET @PageSize = 50;
    IF @PageSize > 500 SET @PageSize = 500;
    
    DECLARE @Offset INT = (@PageNumber - 1) * @PageSize;
    
    SELECT 
        m.MessageID,
        m.ChannelID,
        m.SenderActorID,
        sa.ActorName AS SenderName,
        sa.ActorImagePath AS SenderImagePath,
        m.MessageText,
        m.MediaPath,
        m.MessageTypeID,
        mt.TypeName,
        m.SentDate,
        m.EditedDate,
        m.IsRead,
        m.ReadDate,
        m.IsDeleted
    FROM dbo.ChatMessage m
    LEFT JOIN dbo.DailyDiaryActor sa ON m.SenderActorID = sa.ActorID
    LEFT JOIN dbo.MessageType mt ON m.MessageTypeID = mt.MessageTypeID
    WHERE m.ChannelID = @ChannelID
        AND m.IsDeleted = 0
        AND (@IncludeRead = 1 OR m.IsRead = 0)
    ORDER BY m.SentDate DESC
    OFFSET @Offset ROWS
    FETCH NEXT @PageSize ROWS ONLY;
    
    -- Also return channel metadata
    SELECT 
        c.ChannelID,
        c.StaffActorID,
        c.ParentActorID,
        cs.StaffUnreadCount,
        cs.ParentUnreadCount,
        cs.TotalMessageCount,
        cs.LastMessageDate
    FROM dbo.ChatChannel c
    LEFT JOIN dbo.vwChatChannelStats cs ON c.ChannelID = cs.ChannelID
    WHERE c.ChannelID = @ChannelID;
END;

GO
```

---

## Performance Optimization

### Query Optimization Strategies

#### 1. Materialized Views for Reporting

```sql
-- Pre-aggregate common dashboard metrics
CREATE MATERIALIZED VIEW dbo.vwDailyDiaryDashboard
WITH SCHEMABINDING
AS
SELECT 
    d.ContextStudentID,
    d.ActionStatusID,
    COUNT(*) AS EntryCount,
    COUNT(DISTINCT d.RaisedByActorID) AS UniqueRaisers,
    COUNT(DISTINCT c.CommentID) AS TotalComments,
    MAX(d.DDDateAndTime) AS LastEntryDate,
    MIN(d.DDDateAndTime) AS FirstEntryDate
FROM dbo.DailyDiaryMaster d
LEFT JOIN dbo.DailyDiaryComments c ON d.DDMasterID = c.DDMasterID AND c.IsDeleted = 0
WHERE d.IsDeleted = 0
GROUP BY d.ContextStudentID, d.ActionStatusID;

GO

CREATE UNIQUE CLUSTERED INDEX PK_vwDDDash 
    ON dbo.vwDailyDiaryDashboard(ContextStudentID, ActionStatusID);
```

#### 2. Computed Columns for Denormalized Data

```sql
-- Add computed comment count to DailyDiaryMaster (if needed frequently)
ALTER TABLE dbo.DailyDiaryMaster
ADD CommentCount AS (
    SELECT COUNT(*)
    FROM dbo.DailyDiaryComments
    WHERE DDMasterID = dbo.DailyDiaryMaster.DDMasterID AND IsDeleted = 0
);

-- Index for better performance
CREATE NONCLUSTERED INDEX IX_DDMaster_CommentCount
    ON dbo.DailyDiaryMaster(CommentCount DESC)
    INCLUDE (ContextStudentID, ActionStatusID);
```

#### 3. Query Hints for Common Patterns

```sql
-- Use table hints for specific scenarios
SELECT TOP 50 *
FROM dbo.ChatMessage WITH (NOLOCK)  -- For read-heavy scenarios
WHERE ChannelID = @ChannelID
ORDER BY SentDate DESC;

-- Use RECOMPILE for parameter sniffing issues
CREATE PROCEDURE dbo.spg_DailyDiaryByDateRange
    @DateFrom DATETIME2,
    @DateTo DATETIME2
WITH RECOMPILE
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT * FROM dbo.DailyDiaryMaster
    WHERE DDDateAndTime BETWEEN @DateFrom AND @DateTo
        AND IsDeleted = 0;
END;
```

#### 4. Partitioning Strategy

```sql
-- Partition DailyDiaryMaster by year for large tables
CREATE PARTITION FUNCTION pfDDMasterYear (DATETIME2)
AS RANGE LEFT FOR VALUES ('2024-01-01', '2025-01-01', '2026-01-01');

GO

CREATE PARTITION SCHEME psDDMasterYear
AS PARTITION pfDDMasterYear
TO ([PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY]);

GO

-- Rebuild table with partitioning (offline operation)
-- Recreate DailyDiaryMaster with ON psDDMasterYear
```

### Caching Strategy

```json
{
  "chatChannelStats": {
    "ttl": 300,
    "key": "chat:stats:{channelId}",
    "refreshOn": ["message_sent", "message_read"]
  },
  "diaryEntryPermissions": {
    "ttl": 600,
    "key": "diary:perms:{entryId}:{actorId}",
    "refreshOn": ["permissions_changed"]
  },
  "actorDetails": {
    "ttl": 3600,
    "key": "actor:{actorId}",
    "refreshOn": ["actor_updated"]
  },
  "actionStatusList": {
    "ttl": 86400,
    "key": "action:statuses",
    "refreshOn": ["startup"]
  }
}
```

---

## Implementation Roadmap

### Phase 1: Preparation (Week 1)
- [ ] Create backup of existing database
- [ ] Create new schema objects (EntityType, DailyDiaryActor, etc.)
- [ ] Validate data migration scripts
- [ ] Create rollback procedures
- [ ] Set up monitoring/alerting

### Phase 2: Data Migration (Week 2-3)
- [ ] Migrate entities to DailyDiaryActor table
- [ ] Backfill new tables from legacy data
- [ ] Validate data integrity
- [ ] Run parallel testing (old vs new queries)
- [ ] Performance comparison

### Phase 3: API Updates (Week 3-4)
- [ ] Update application APIs to use new stored procedures
- [ ] Implement new DTOs
- [ ] Add pagination/filtering support
- [ ] Update error handling
- [ ] Test end-to-end scenarios

### Phase 4: Cutover (Week 4)
- [ ] Schedule maintenance window
- [ ] Migrate remaining data
- [ ] Switch application to new schema
- [ ] Monitor for issues
- [ ] Have rollback ready

### Phase 5: Cleanup (Week 5)
- [ ] Drop old tables (after grace period)
- [ ] Rename new tables to original names
- [ ] Rebuild indexes
- [ ] Analyze performance
- [ ] Document changes

### Timeline Estimate
- **Total Duration:** 4-5 weeks
- **Downtime Required:** 2-4 hours (during cutover)
- **Testing Required:** Parallel run 1-2 weeks before cutover

---

## Success Criteria

✅ All foreign key constraints enforced  
✅ Zero orphaned records  
✅ Query performance improved 30%+  
✅ Pagination support in all GET APIs  
✅ Unread message counts accurate (via materialized view)  
✅ 99.9% data integrity validation  
✅ Zero critical bugs in 2-week stabilization period  
✅ API response times < 200ms (p95)  

---

## Recommended Reading

- SQL Server Best Practices: https://learn.microsoft.com/en-us/sql/
- Entity-Relationship Modeling: Check IDEF1X patterns
- API Design: JSON:API specification
- Database Partitioning: https://learn.microsoft.com/en-us/sql/relational-databases/partitions/

---

**Document prepared for:** Development Team, Data Architects  
**Recommendation:** Begin Phase 1 immediately; allocate dedicated resources
