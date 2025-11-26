ALTER PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250),
	@P__CanBeUsed BIT = 0  -- NEW PARAMETER with default value of 0
AS 
BEGIN 
    SET NOCOUNT ON
	SELECT 1;  
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @SystemUser NVARCHAR(250) = 'AlternaSystem'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],[Code],0,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Company] comp 
            WHERE comp.[Code] = input.[Code]
        );

        -- Temp tables to hold inserted IDs and link them back to input data 
        DECLARE @InsertedContainers TABLE( 
            RowId INT, 
            ContainerId INT 
        ); 

        DECLARE @InsertedFiles TABLE( 
            RowId INT, 
            FileId INT 
        ); 

        -- Insert Containers with unique Code + CompanyCode combination
        WITH UniqueContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate,
                   ROW_NUMBER() OVER (PARTITION BY BoxRef, Code ORDER BY RowId) as rn
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_Container] cont
                WHERE cont.[Code] = input.[BoxRef]
                AND cont.[CompanyCode] = input.[Code]
            )
        ),
        ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM UniqueContainerSource
            WHERE rn = 1
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @SystemUser, @Now, @SystemUser, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Capture existing container IDs for ALL rows (including duplicates within input)
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef]
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- For input rows with duplicate BoxRef + CompanyCode, map them to the inserted container
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, ic.ContainerId
        FROM @P__Old_Boxes input
        INNER JOIN @InsertedContainers ic ON EXISTS (
            SELECT 1 FROM @P__Old_Boxes input2 
            WHERE input2.RowId = ic.RowId 
            AND input2.BoxRef = input.BoxRef
            AND input2.Code = input.Code
        )
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- Insert new File Type with auto-incrementing FileTypeCode
        DECLARE @NewFileTypes TABLE (
            Description NVARCHAR(250),
            Entity NVARCHAR(50),
            ArchivingPeriod INT,
            NextCode INT
        );
        
        DECLARE @MaxFileTypeCode INT;
        SELECT @MaxFileTypeCode = ISNULL(MAX(CAST(Code AS INT)), 0) 
        FROM [dbo].[lkp_FileType] 
        WHERE ISNUMERIC(Code) = 1;
        
        -- Get distinct Entity+Description combinations, taking the first ArchivingPeriod encountered
        WITH UniqueNewFileTypes AS (
            SELECT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   MIN(input.[ArchivingPeriod]) as ArchivingPeriod
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
            GROUP BY input.[FileName], input.[Code]
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               @P__CanBeUsed,  -- MODIFIED: Use parameter value
               @SystemUser,
               @Now,
               @SystemUser,
               @Now
        FROM @NewFileTypes nft;

        -- Get the FileTypeCode for each file
        DECLARE @FileTypeCodes TABLE (
            RowId INT,
            FileTypeCode NVARCHAR(50)
        );
        
        INSERT INTO @FileTypeCodes (RowId, FileTypeCode)
        SELECT input.RowId, ft.Code
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.[Entity] = input.[Code] 
            AND ft.[Description] = input.[FileName];

        -- Insert Files
        INSERT INTO [dbo].[t_File] 
        ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
        OUTPUT inserted.Id
        INTO @InsertedFiles(FileId)
        SELECT null, 
               input.FileName, 
               ftc.FileTypeCode, 
               'FINAL', 
               input.Code, 
               null, 
               null, 
               input.AdditionalInfo, 
               0, 
               @SystemUser, 
               @Now, 
               @SystemUser, 
               @Now
        FROM @P__Old_Boxes input
        INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
        ORDER BY input.RowId;

        -- Map the inserted FileIds back to RowIds
        DECLARE @RowIdMapping TABLE (
            RowId INT,
            FileId INT,
            RowNum INT
        );

        INSERT INTO @RowIdMapping (RowId, RowNum)
        SELECT RowId, ROW_NUMBER() OVER (ORDER BY RowId) as RowNum
        FROM @P__Old_Boxes;

        WITH NumberedInsertedFiles AS (
            SELECT FileId, ROW_NUMBER() OVER (ORDER BY FileId) as RowNum
            FROM @InsertedFiles
        )
        UPDATE @InsertedFiles
        SET RowId = rm.RowId
        FROM @InsertedFiles if_target
        INNER JOIN NumberedInsertedFiles nif ON if_target.FileId = nif.FileId
        INNER JOIN @RowIdMapping rm ON nif.RowNum = rm.RowNum;

        -- Insert Container File Relationship
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @SystemUser, @Now, @SystemUser, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_CurrentContainerFileRelationship] rel
            WHERE rel.[FileId] = f.FileId AND rel.[ContainerId] = c.ContainerId
        );

        -- Insert new Sequence only if Owner doesn't already exist
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Sequence] seq
            WHERE seq.[Owner] = input.[Code]
        )
        AND input.[Code] NOT IN (
            SELECT i2.[Code] 
            FROM @P__Old_Boxes i2 
            WHERE i2.RowId < input.RowId
        );

        -- ========================================
        -- Insert Container Status History
        -- ========================================
        
        -- 1. Insert SENT status for all containers
        -- HoldingEntityCode = Code from input (the branch/entity code)
        -- User = BoxSentBy if not empty, otherwise AlternaSystem
        -- Always inactive (isCurrentStatus = 0)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'SENT',
               i.[Code],
               0,
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,
               i.[BoxSentDate],
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,
               i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'SENT'
        );

        -- 2. Insert RECEIVED status for containers with StatusCode RECEIVED, DESTROYED, or NOTFOUND
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Active only if final status is RECEIVED
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'RECEIVED',
               'WH',
               CASE WHEN i.[StatusCode] = 'RECEIVED' THEN 1 ELSE 0 END,
               @SystemUser,
               DATEADD(MINUTE, 1, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 1, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] IN ('RECEIVED', 'DESTROYED', 'NOTFOUND')
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'RECEIVED'
        );

        -- 3. Insert DESTROYED status for containers with StatusCode DESTROYED only
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Always active (final status)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'DESTROYED',
               'WH',
               1,
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'DESTROYED'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'DESTROYED'
        );

        -- 4. Insert NOTFOUND status for containers with StatusCode NOTFOUND
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Always active (final status)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'NOTFOUND',
               'WH',
               1,
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'NOTFOUND'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'NOTFOUND'
        );

        COMMIT TRANSACTION; 
    END TRY 
    BEGIN CATCH 
        ROLLBACK TRANSACTION; 
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(); 
        DECLARE @ErrSeverity INT = ERROR_SEVERITY(); 
        RAISERROR (@ErrMsg, @ErrSeverity, 1); 
    END CATCH 
END;




Answer on  3rd request:
----------------------
-- ========================================
-- MODIFIED: Insert new File Type with CanBeUsed from Excel or Parameter
-- ========================================
DECLARE @NewFileTypes TABLE (
    Description NVARCHAR(250),
    Entity NVARCHAR(50),
    ArchivingPeriod INT,
    CanBeUsed BIT,
    NextCode INT
);

DECLARE @MaxFileTypeCode INT;
SELECT @MaxFileTypeCode = ISNULL(MAX(CAST(Code AS INT)), 0) 
FROM [dbo].[lkp_FileType] 
WHERE ISNUMERIC(Code) = 1;

-- Get distinct Entity+Description combinations
-- Use CanBeUsed from Excel if any row has it set to 1, otherwise use parameter default
WITH UniqueNewFileTypes AS (
    SELECT 
           input.[FileName] as Description,
           input.[Code] as Entity,
           MIN(input.[ArchivingPeriod]) as ArchivingPeriod,
           MAX(CAST(input.[CanBeUsed] AS INT)) as CanBeUsedInt  -- Cast BIT to INT for MAX
    FROM @P__Old_Boxes input
    WHERE NOT EXISTS (
        SELECT 1 FROM [dbo].[lkp_FileType] ft
        WHERE ft.[Description] = input.[FileName]
        AND ft.[Entity] = input.[Code]
    )
    GROUP BY input.[FileName], input.[Code]
)
INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, CanBeUsed, NextCode)
SELECT Description, 
       Entity, 
       ArchivingPeriod,
       CASE WHEN CanBeUsedInt = 1 THEN 1 ELSE @P__CanBeUsed END as CanBeUsed,  -- Convert back to BIT
       @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
FROM UniqueNewFileTypes;

INSERT INTO [dbo].[lkp_FileType] 
([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
       nft.Entity,
       'Not Branch' as Category,
       nft.Description,
       0 as HasDate,
       0 as IsCustomer,
       nft.ArchivingPeriod,
       nft.CanBeUsed,
       @SystemUser,
       @Now,
       @SystemUser,
       @Now
FROM @NewFileTypes nft;
-------

3rd request
---------------
in usp_Insert_Into_All_Tables, i will be adding a new param that is CanBeUsed, set as 0 default vakue. this value can be changed in the FileTypeManagement option in Configuration Menu. I need the modified code to  integrate the CanBeUsed new field in the ExcelFile that is uploaded.I'll help you integrate the CanBeUsed field into the Excel upload process for usp_Insert_Into_All_Tables. Here are the modifications needed:
1. Table-Valued Parameter (TVP) Definition
First, check if your TVP TVP_Old_Boxes includes a CanBeUsed column. If not, you need to alter it:
sql-- Check current definition first
sp_help 'TVP_Old_Boxes'

-- If CanBeUsed doesn't exist, you need to DROP and RECREATE the type
-- (You can't ALTER a table type, you must recreate it)

-- Drop the type (make sure no procedures are using it first, or drop them temporarily)
DROP TYPE [dbo].[TVP_Old_Boxes]
GO

-- Recreate with CanBeUsed field
CREATE TYPE [dbo].[TVP_Old_Boxes] AS TABLE
(
    RowId INT,
    BoxRef NVARCHAR(50),
    Code NVARCHAR(50),
    CompanyName NVARCHAR(250),
    Mnemonic NVARCHAR(50),
    StatusCode NVARCHAR(50),
    BoxSentDate DATETIME,
    BoxSentBy NVARCHAR(250),
    FileName NVARCHAR(250),
    ArchivingPeriod INT,
    AdditionalInfo NVARCHAR(MAX),
    IsActive BIT,
    LastIndex INT,
    CanBeUsed BIT  -- ADD THIS FIELD
)
GO
2. Modified Stored Procedure - usp_Insert_Into_All_Tables
sqlALTER PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250),
	@P__CanBeUsed BIT = 0  -- Default value is 0
AS 
BEGIN 
    SET NOCOUNT ON
	SELECT 1;  
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @SystemUser NVARCHAR(250) = 'AlternaSystem'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],[Code],0,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Company] comp 
            WHERE comp.[Code] = input.[Code]
        );

        -- Temp tables to hold inserted IDs and link them back to input data 
        DECLARE @InsertedContainers TABLE( 
            RowId INT, 
            ContainerId INT 
        ); 

        DECLARE @InsertedFiles TABLE( 
            RowId INT, 
            FileId INT 
        ); 

        -- Insert Containers with unique Code + CompanyCode combination
        WITH UniqueContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate,
                   ROW_NUMBER() OVER (PARTITION BY BoxRef, Code ORDER BY RowId) as rn
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_Container] cont
                WHERE cont.[Code] = input.[BoxRef]
                AND cont.[CompanyCode] = input.[Code]
            )
        ),
        ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM UniqueContainerSource
            WHERE rn = 1
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @SystemUser, @Now, @SystemUser, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Capture existing container IDs for ALL rows (including duplicates within input)
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef]
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- For input rows with duplicate BoxRef + CompanyCode, map them to the inserted container
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, ic.ContainerId
        FROM @P__Old_Boxes input
        INNER JOIN @InsertedContainers ic ON EXISTS (
            SELECT 1 FROM @P__Old_Boxes input2 
            WHERE input2.RowId = ic.RowId 
            AND input2.BoxRef = input.BoxRef
            AND input2.Code = input.Code
        )
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- ========================================
        -- MODIFIED: Insert new File Type with CanBeUsed from Excel or Parameter
        -- ========================================
        DECLARE @NewFileTypes TABLE (
            Description NVARCHAR(250),
            Entity NVARCHAR(50),
            ArchivingPeriod INT,
            CanBeUsed BIT,  -- ADD THIS FIELD
            NextCode INT
        );
        
        DECLARE @MaxFileTypeCode INT;
        SELECT @MaxFileTypeCode = ISNULL(MAX(CAST(Code AS INT)), 0) 
        FROM [dbo].[lkp_FileType] 
        WHERE ISNUMERIC(Code) = 1;
        
        -- Get distinct Entity+Description combinations
        -- Use CanBeUsed from Excel if provided, otherwise use parameter default
        WITH UniqueNewFileTypes AS (
            SELECT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   MIN(input.[ArchivingPeriod]) as ArchivingPeriod,
                   COALESCE(MAX(input.[CanBeUsed]), @P__CanBeUsed) as CanBeUsed  -- MODIFIED: Use Excel value or parameter
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
            GROUP BY input.[FileName], input.[Code]
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, CanBeUsed, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               CanBeUsed,  -- ADD THIS LINE
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               nft.CanBeUsed,  -- MODIFIED: Use value from temp table
               @SystemUser,
               @Now,
               @SystemUser,
               @Now
        FROM @NewFileTypes nft;

        -- Get the FileTypeCode for each file
        DECLARE @FileTypeCodes TABLE (
            RowId INT,
            FileTypeCode NVARCHAR(50)
        );
        
        INSERT INTO @FileTypeCodes (RowId, FileTypeCode)
        SELECT input.RowId, ft.Code
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.[Entity] = input.[Code] 
            AND ft.[Description] = input.[FileName];

        -- Insert Files
        INSERT INTO [dbo].[t_File] 
        ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
        OUTPUT inserted.Id
        INTO @InsertedFiles(FileId)
        SELECT null, 
               input.FileName, 
               ftc.FileTypeCode, 
               'FINAL', 
               input.Code, 
               null, 
               null, 
               input.AdditionalInfo, 
               0, 
               @SystemUser, 
               @Now, 
               @SystemUser, 
               @Now
        FROM @P__Old_Boxes input
        INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
        ORDER BY input.RowId;

        -- Map the inserted FileIds back to RowIds
        DECLARE @RowIdMapping TABLE (
            RowId INT,
            FileId INT,
            RowNum INT
        );

        INSERT INTO @RowIdMapping (RowId, RowNum)
        SELECT RowId, ROW_NUMBER() OVER (ORDER BY RowId) as RowNum
        FROM @P__Old_Boxes;

        WITH NumberedInsertedFiles AS (
            SELECT FileId, ROW_NUMBER() OVER (ORDER BY FileId) as RowNum
            FROM @InsertedFiles
        )
        UPDATE @InsertedFiles
        SET RowId = rm.RowId
        FROM @InsertedFiles if_target
        INNER JOIN NumberedInsertedFiles nif ON if_target.FileId = nif.FileId
        INNER JOIN @RowIdMapping rm ON nif.RowNum = rm.RowNum;

        -- Insert Container File Relationship
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @SystemUser, @Now, @SystemUser, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_CurrentContainerFileRelationship] rel
            WHERE rel.[FileId] = f.FileId AND rel.[ContainerId] = c.ContainerId
        );

        -- Insert new Sequence only if Owner doesn't already exist
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Sequence] seq
            WHERE seq.[Owner] = input.[Code]
        )
        AND input.[Code] NOT IN (
            SELECT i2.[Code] 
            FROM @P__Old_Boxes i2 
            WHERE i2.RowId < input.RowId
        );

        -- ========================================
        -- Insert Container Status History
        -- ========================================
        
        -- 1. Insert SENT status for all containers
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'SENT',
               i.[Code],
               0,
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,
               i.[BoxSentDate],
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,
               i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'SENT'
        );

        -- 2. Insert RECEIVED status for containers with StatusCode RECEIVED, DESTROYED, or NOTFOUND
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'RECEIVED',
               'WH',
               CASE WHEN i.[StatusCode] = 'RECEIVED' THEN 1 ELSE 0 END,
               @SystemUser,
               DATEADD(MINUTE, 1, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 1, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] IN ('RECEIVED', 'DESTROYED', 'NOTFOUND')
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'RECEIVED'
        );

        -- 3. Insert DESTROYED status for containers with StatusCode DESTROYED only
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'DESTROYED',
               'WH',
               1,
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'DESTROYED'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'DESTROYED'
        );

        -- 4. Insert NOTFOUND status for containers with StatusCode NOTFOUND
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'NOTFOUND',
               'WH',
               1,
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'NOTFOUND'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'NOTFOUND'
        );

        COMMIT TRANSACTION; 
    END TRY 
    BEGIN CATCH 
        ROLLBACK TRANSACTION; 
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(); 
        DECLARE @ErrSeverity INT = ERROR_SEVERITY(); 
        RAISERROR (@ErrMsg, @ErrSeverity, 1); 
    END CATCH 
END;
3. C# Model for Excel Import (Update your model)
csharppublic class OldBoxModel
{
    public int RowId { get; set; }
    public string BoxRef { get; set; }
    public string Code { get; set; }
    public string CompanyName { get; set; }
    public string Mnemonic { get; set; }
    public string StatusCode { get; set; }
    public DateTime BoxSentDate { get; set; }
    public string BoxSentBy { get; set; }
    public string FileName { get; set; }
    public int ArchivingPeriod { get; set; }
    public string AdditionalInfo { get; set; }
    public bool IsActive { get; set; }
    public int LastIndex { get; set; }
    public bool CanBeUsed { get; set; }  // ADD THIS PROPERTY
}
4. Excel Reading Logic (Example with EPPlus or similar)
csharp// In your Excel import method, add column for CanBeUsed
public List<OldBoxModel> ReadExcelFile(string filePath)
{
    List<OldBoxModel> boxes = new List<OldBoxModel>();
    
    using (var package = new ExcelPackage(new FileInfo(filePath)))
    {
        var worksheet = package.Workbook.Worksheets[0];
        int rowCount = worksheet.Dimension.Rows;
        
        for (int row = 2; row <= rowCount; row++) // Start from row 2 (skip header)
        {
            var box = new OldBoxModel
            {
                RowId = row - 1,
                BoxRef = worksheet.Cells[row, 1].Value?.ToString(),
                Code = worksheet.Cells[row, 2].Value?.ToString(),
                CompanyName = worksheet.Cells[row, 3].Value?.ToString(),
                Mnemonic = worksheet.Cells[row, 4].Value?.ToString(),
                StatusCode = worksheet.Cells[row, 5].Value?.ToString(),
                BoxSentDate = Convert.ToDateTime(worksheet.Cells[row, 6].Value),
                BoxSentBy = worksheet.Cells[row, 7].Value?.ToString(),
                FileName = worksheet.Cells[row, 8].Value?.ToString(),
                ArchivingPeriod = Convert.ToInt32(worksheet.Cells[row, 9].Value),
                AdditionalInfo = worksheet.Cells[row, 10].Value?.ToString(),
                IsActive = Convert.ToBoolean(worksheet.Cells[row, 11].Value),
                LastIndex = Convert.ToInt32(worksheet.Cells[row, 12].Value),
                CanBeUsed = worksheet.Cells[row, 13].Value != null ? Convert.ToBoolean(worksheet.Cells[row, 13].Value) : false  // ADD THIS - Column 13 for CanBeUsed, default to false if empty
            };
            
            boxes.Add(box);
        }
    }
    
    return boxes;
}
5. BLL Method to Call Stored Procedure
csharppublic void InsertIntoAllTables(List<OldBoxModel> oldBoxes, string user, bool canBeUsedDefault = false)
{
    DAL.DAL iDAL = new();
    
    // Create DataTable from list
    DataTable dt = new DataTable();
    dt.Columns.Add("RowId", typeof(int));
    dt.Columns.Add("BoxRef", typeof(string));
    dt.Columns.Add("Code", typeof(string));
    dt.Columns.Add("CompanyName", typeof(string));
    dt.Columns.Add("Mnemonic", typeof(string));
    dt.Columns.Add("StatusCode", typeof(string));
    dt.Columns.Add("BoxSentDate", typeof(DateTime));
    dt.Columns.Add("BoxSentBy", typeof(string));
    dt.Columns.Add("FileName", typeof(string));
    dt.Columns.Add("ArchivingPeriod", typeof(int));
    dt.Columns.Add("AdditionalInfo", typeof(string));
    dt.Columns.Add("IsActive", typeof(bool));
    dt.Columns.Add("LastIndex", typeof(int));
    dt.Columns.Add("CanBeUsed", typeof(bool));  // ADD THIS COLUMN
    
    foreach (var box in oldBoxes)
    {
        dt.Rows.Add(
            box.RowId,
            box.BoxRef,
            box.Code,
            box.CompanyName,
            box.Mnemonic,
            box.StatusCode,
            box.BoxSentDate,
            box.BoxSentBy,
            box.FileName,
            box.ArchivingPeriod,
            box.AdditionalInfo,
            box.IsActive,
            box.LastIndex,
            box.CanBeUsed  // ADD THIS VALUE
        );
    }
    
    DynamicParameters parameters = new DynamicParameters();
    parameters.Add("@P__Old_Boxes", dt.AsTableValuedParameter("TVP_Old_Boxes"));
    parameters.Add("@P__User", user);
    parameters.Add("@P__CanBeUsed", canBeUsedDefault);  // ADD THIS PARAMETER
    
    iDAL.ExecuteQuery<object>("usp_Insert_Into_All_Tables", parameters, CommandType.StoredProcedure, CommandDirection.Select);
}
Summary
The logic now works as follows:

Excel has CanBeUsed column: The value from Excel will be used for that specific FileType
Excel doesn't have CanBeUsed column (or it's empty): The @P__CanBeUsed parameter value (default 0) will be used
Priority: Excel value > Parameter value > Default (0)

The COALESCE(MAX(input.[CanBeUsed]), @P__CanBeUsed) in the SQL ensures that if Excel has a value, it's used; otherwise, the parameter default is used.
Excel Template should have this column order:

Column 13: CanBeUsed (optional - can be 0/1 or TRUE/FALSE or blank)

If the column is not in Excel, you can handle it in your C# code by always setting CanBeUsed = false when reading.
-----------------

2nd request on 26-11-2025
==========================
My Existing Code as:
BLL.cs
--------
        public FileType UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
        {
            DAL.DAL iDAL = new();

            FileType Ret = new();

            OnPreEventUpdateFileType?.Invoke(ref updateFileTypeReq);

            DynamicParameters param = new();

            param.Add("Id", updateFileTypeReq.Id);
            param.Add("Code", updateFileTypeReq.Code);
            param.Add("Entity", updateFileTypeReq.Entity);
            param.Add("Description", updateFileTypeReq.Description);
            param.Add("Category", updateFileTypeReq.Category);
            param.Add("HasDate", updateFileTypeReq.HasDate);
            param.Add("IsCustomer", updateFileTypeReq.IsCustomer);
            param.Add("ArchivingPeriod", updateFileTypeReq.ArchivingPeriod);
            param.Add("CanBeUsed", updateFileTypeReq.CanBeUsed);
            param.Add("User", updateFileTypeReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<FileType>("usp_UpdateFileType", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateFileType?.Invoke(ref Ret, ref updateFileTypeReq);

            return Ret;
        }

Configuration.cs (As front end Controller)
-----------------
        public Boolean UpdateFileType(Int32 ModelID, String code, String Entity, String Description, Boolean IsBranch, Boolean IsCustomer, String ArchivingPeriod)
        {
            if (!Int32.TryParse(ArchivingPeriod, out Int32 AP))
            {
                AP = -2;
            }

            if (Entity is null)
            {
                Entity = String.Empty;
            }

            UpdateFileTypeReq updateFileTypeReq = new()
            {
                BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                Id = ModelID,
                Code = code,
                Entity = Entity,
                Description = Description,
                IsCustomer = IsCustomer,
                HasDate = !IsCustomer,
                Category = (IsBranch == true) ? "Branch" : "Not Branch",
                ArchivingPeriod = AP
            };
            UpdateFileTypeRes resp = new();
            resp = Common.ApiCall<UpdateFileTypeRes>(updateFileTypeReq, "UpdateFileType");
            return true;
        }
BLL.cs
--------
        public FileType UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
        {
            DAL.DAL iDAL = new();

            FileType Ret = new();

            OnPreEventUpdateFileType?.Invoke(ref updateFileTypeReq);

            DynamicParameters param = new();

            param.Add("Id", updateFileTypeReq.Id);
            param.Add("Code", updateFileTypeReq.Code);
            param.Add("Entity", updateFileTypeReq.Entity);
            param.Add("Description", updateFileTypeReq.Description);
            param.Add("Category", updateFileTypeReq.Category);
            param.Add("HasDate", updateFileTypeReq.HasDate);
            param.Add("IsCustomer", updateFileTypeReq.IsCustomer);
            param.Add("ArchivingPeriod", updateFileTypeReq.ArchivingPeriod);
            param.Add("CanBeUsed", updateFileTypeReq.CanBeUsed);
            param.Add("User", updateFileTypeReq.BaseReq.CurrentUser);

            Ret = iDAL.ExecuteQuery<FileType>("usp_UpdateFileType", param, CommandType.StoredProcedure,
                CommandDirection.Update).FirstOrDefault()!;

            OnPostEventUpdateFileType?.Invoke(ref Ret, ref updateFileTypeReq);

            return Ret;
        }

_FileTypeManagementTable.cshtml (New code)
@using Alterna.Archive.Core.Models.TableModel
@using Alterna.Archive.Core.Models
@model FileTypeModel

<table id="example" class="table table-striped table-bordered" style="width:100%">
    <thead>
        <tr>
            <th>Action</th>
            <th>Code Name</th>
            <th>Entity</th>
            <th>Description</th>
            <th>IsBranch</th>
            <th>IsCustomer</th>
            <th>ArchivingPeriod</th>
            <th>CanBeUsed</th>
        </tr>
    </thead>
    <tbody>
        @if (Model.FileTypeList.Count > 0)
        {
            StaticFileTypeModel.FileTypeList = Model.FileTypeList;
            for (int i = 0; i < Model.FileTypeList.Count; i++)
            {
                String entityTableTdId = "EntityTableTd" + Model.FileTypeList[i].Id;
                String hiddenClass = !Model.FileTypeList[i].IsBranch && !Model.FileTypeList[i].IsCustomer ? "" : "hidden";

                <tr>
                    <td>
                        <div style="text-align: center; cursor:pointer" onclick="editRow(this,@Model.FileTypeList[i].Id,@i)">
                            <span class="fa-regular fa-pen-to-square" title="Edit Details"></span>
                        </div>
                    </td>
                    @{
                        String CodeName = $"{Model.FileTypeList[i].Code} - {Model.FileTypeList[i].Description}";
                        String tdId = $"Code-{Model.FileTypeList[i].Id.ToString()}";
                    }

                    <td id="@tdId">@CodeName</td>

                    <td>
                        @Html.DropDownListFor(model => model.FileTypeList[i].Entity, @Model.EntityList, @Model.FileTypeList[i].Entity, new { @id = "Entity" + @Model.FileTypeList[i].Id, @disabled = "disabled", @class = hiddenClass })
                    </td>

                    <td>@Html.EditorFor(model => model.FileTypeList[i].Description, new { htmlAttributes = new { @id = "Description" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>

                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsBranch, new { @id = "IsBranch" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsCustomer, new { @id = "IsCustomer" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                    <td>@Html.EditorFor(model => model.FileTypeList[i].ArchivingPeriod, new { htmlAttributes = new { @type = "number", @min = "0", @step = "1", @id = "ArchivingPeriod" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>
                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].CanBeUsed, new { @id = "CanBeUsed" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                </tr>
            }
        }
    </tbody>
</table>

<script>
    $(document).ready(() => {
        $("#example").DataTable({
            pagingType: 'full_numbers'
        });
    });

    function editRow(element, id, index) {
        let row = $(element).closest('tr');
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        var prevData = {
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked  // NEW
        };
        sessionStorage.setItem('Ftt' + id, JSON.stringify(prevData))
        row.find('td:first-child').html('<div style="text-align: center;"><span style="cursor:pointer; color:green" class="fa-regular fa-floppy-disk" onclick="saveRow(this,' + id + ',' + index + ')" title="Save Details"></span>&nbsp;&nbsp;<span style="cursor:pointer; color:red" class="fa-solid fa-xmark" onclick="stopEdit(this,' + id + ')" title="Cancel"></span></div>');
        Fields_Switch(id, false)
    }

    function stopEdit(element, id) {
        var data = sessionStorage.getItem('Ftt' + id);
        var prevData = JSON.parse(data);
        let row = $(element).closest('tr');
        row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
        Fields_Switch(id, true)
        if (!prevData.IsCustomer && !prevData.IsBranch) {
            AddEntitySelectElementToDataTable(id);
        } else {
            RemoveEntitySelectElementToDataTable(id);
        }
        $('#Entity' + id)[0].value = prevData.Entity;
        $('#Description' + id)[0].value = prevData.Description;
        $('#IsBranch' + id)[0].checked = prevData.IsBranch,
            $('#IsCustomer' + id)[0].checked = prevData.IsCustomer;
        $('#ArchivingPeriod' + id)[0].value = prevData.ArchivingPeriod;
        $('#CanBeUsed' + id)[0].checked = prevData.CanBeUsed;  // NEW
    }

    function saveRow(element, id, Aindex) {
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        let code = $("#Code-" + id).html();
        let match = code.match(/\d+/);
        if (match) {
            code = match[0];
        }
        var dat = {
            ModelId: id,
            code: code,
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked  // NEW
        };
        $.ajax({
            type: 'POST',
            url: '/Configuration/UpdateFileType/',
            data: dat,
            dataType: 'html',
            success: function (data) {
                let row = $(element).closest('tr');
                row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
                Fields_Switch(id, true)
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
    }

    function Fields_Switch(id, x) {
        $('#Description' + id).prop('disabled', x);
        $('#IsCustomer' + id).prop('disabled', x);
        $('#ArchivingPeriod' + id).prop('disabled', x);
        $('#IsBranch' + id).prop('disabled', x);
        $('#Entity' + id).prop('disabled', x);
        $('#CanBeUsed' + id).prop('disabled', x);  // NEW
    }

    $('*[id*=IsBranch]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if (!$("#IsBranch" + index)[0].checked) {
            $('#IsCustomer' + index).prop('checked', false).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    $('*[id*=IsCustomer]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if ($("#IsCustomer" + index)[0].checked) {
            $('#IsBranch' + index).prop('checked', true).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    function AddEntitySelectElementToDataTable(index) {
        $("#Entity" + index).removeClass("hidden");
    }

    function RemoveEntitySelectElementToDataTable(index) {
        $("#Entity" + index).addClass("hidden");
    }
</script>
----------------------------------------------------------------------------

===========================
Answer on 26-11-25
===========================================================
in _FileTypeManagement.cshtml when i edit the row, and change the checkbox value of CanBeUsed, the save button does not save the value in the database, on save it looks like it has change, but when refreshing the page, the old value come back,.The issue is likely in your JavaScript - the CanBeUsed checkbox value might not be captured correctly in the saveRow function. Let me provide you with a corrected version of the JavaScript section:
Updated JavaScript in _FileTypeManagementTable.cshtml
Replace the entire <script> section with this corrected version:
cshtml<script>
    $(document).ready(() => {
        $("#example").DataTable({
            pagingType: 'full_numbers'
        });
    });

    function editRow(element, id, index) {
        let row = $(element).closest('tr');
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        var prevData = {
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked
        };
        sessionStorage.setItem('Ftt' + id, JSON.stringify(prevData))
        row.find('td:first-child').html('<div style="text-align: center;"><span style="cursor:pointer; color:green" class="fa-regular fa-floppy-disk" onclick="saveRow(this,' + id + ',' + index + ')" title="Save Details"></span>&nbsp;&nbsp;<span style="cursor:pointer; color:red" class="fa-solid fa-xmark" onclick="stopEdit(this,' + id + ')" title="Cancel"></span></div>');
        Fields_Switch(id, false)
    }

    function stopEdit(element, id) {
        var data = sessionStorage.getItem('Ftt' + id);
        var prevData = JSON.parse(data);
        let row = $(element).closest('tr');
        row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
        Fields_Switch(id, true)
        if (!prevData.IsCustomer && !prevData.IsBranch) {
            AddEntitySelectElementToDataTable(id);
        } else {
            RemoveEntitySelectElementToDataTable(id);
        }
        $('#Entity' + id)[0].value = prevData.Entity;
        $('#Description' + id)[0].value = prevData.Description;
        $('#IsBranch' + id)[0].checked = prevData.IsBranch;
        $('#IsCustomer' + id)[0].checked = prevData.IsCustomer;
        $('#ArchivingPeriod' + id)[0].value = prevData.ArchivingPeriod;
        $('#CanBeUsed' + id)[0].checked = prevData.CanBeUsed;
    }

    function saveRow(element, id, Aindex) {
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        let code = $("#Code-" + id).html();
        let match = code.match(/\d+/);
        if (match) {
            code = match[0];
        }
        
        // IMPORTANT: Get the current checkbox state
        let canBeUsed = $('#CanBeUsed' + id).is(':checked');
        
        var dat = {
            ModelId: id,
            code: code,
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: canBeUsed  // FIXED: Use the variable instead of accessing the DOM again
        };
        
        console.log('Saving data:', dat); // Debug: Check what's being sent
        
        $.ajax({
            type: 'POST',
            url: '/Configuration/UpdateFileType/',
            data: dat,
            dataType: 'json',  // CHANGED: Expect JSON response
            success: function (data) {
                console.log('Server response:', data); // Debug: Check server response
                let row = $(element).closest('tr');
                row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
                Fields_Switch(id, true);
                
                // Update session storage with new values
                var newData = {
                    Entity: entity,
                    Description: $('#Description' + id)[0].value,
                    IsBranch: $('#IsBranch' + id)[0].checked,
                    IsCustomer: $('#IsCustomer' + id)[0].checked,
                    ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
                    CanBeUsed: canBeUsed
                };
                sessionStorage.setItem('Ftt' + id, JSON.stringify(newData));
            },
            error: function (xhr) {
                console.log('Error:', xhr.responseText); // Debug: Check error
                alert('Error updating file type: ' + xhr.responseText);
            }
        });
    }

    function Fields_Switch(id, x) {
        $('#Description' + id).prop('disabled', x);
        $('#IsCustomer' + id).prop('disabled', x);
        $('#ArchivingPeriod' + id).prop('disabled', x);
        $('#IsBranch' + id).prop('disabled', x);
        $('#Entity' + id).prop('disabled', x);
        $('#CanBeUsed' + id).prop('disabled', x);
    }

    $('*[id*=IsBranch]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if (!$("#IsBranch" + index)[0].checked) {
            $('#IsCustomer' + index).prop('checked', false).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    $('*[id*=IsCustomer]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if ($("#IsCustomer" + index)[0].checked) {
            $('#IsBranch' + index).prop('checked', true).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    function AddEntitySelectElementToDataTable(index) {
        $("#Entity" + index).removeClass("hidden");
    }

    function RemoveEntitySelectElementToDataTable(index) {
        $("#Entity" + index).addClass("hidden");
    }
</script>
Also Update Your Controller Action
Make sure your UpdateFileType action in ConfigurationController.cs returns proper JSON:
csharp[HttpPost]
public ActionResult UpdateFileType(int ModelId, string code, string Entity, string Description, bool IsBranch, bool IsCustomer, int ArchivingPeriod, bool CanBeUsed)
{
    try
    {
        String session = GetSession("ArchiveData");
        
        // Log the incoming values for debugging
        System.Diagnostics.Debug.WriteLine($"UpdateFileType called - ModelId: {ModelId}, CanBeUsed: {CanBeUsed}");
        
        UpdateFileTypeReq request = new UpdateFileTypeReq()
        {
            BaseReq = new BaseRequest(HttpContext, session, false),
            FileTypeId = ModelId,
            Code = code,
            Entity = Entity,
            Description = Description,
            IsBranch = IsBranch,
            IsCustomer = IsCustomer,
            ArchivingPeriod = ArchivingPeriod,
            CanBeUsed = CanBeUsed
        };

        UpdateFileTypeRes response = Common.ApiCall<UpdateFileTypeRes>(request, "UpdateFileType");

        if (response.WebResp.HttpResponseCode == HttpStatusCode.OK)
        {
            return Json(new { success = true, message = "File Type updated successfully" }, JsonRequestBehavior.AllowGet);
        }
        else
        {
            return Json(new { success = false, message = response.WebResp.ResponseMessage }, JsonRequestBehavior.AllowGet);
        }
    }
    catch (Exception ex)
    {
        System.Diagnostics.Debug.WriteLine($"Error in UpdateFileType: {ex.Message}");
        return Json(new { success = false, message = ex.Message }, JsonRequestBehavior.AllowGet);
    }
}
Verify Your Stored Procedure
Make sure the stored procedure is correctly updating the CanBeUsed field:
sqlALTER PROCEDURE [dbo].[usp_UpdateFileType]
    @P__Id INT,
    @P__Code NVARCHAR(50),
    @P__Entity NVARCHAR(50),
    @P__Description NVARCHAR(250),
    @P__Category NVARCHAR(50),
    @P__IsCustomer BIT,
    @P__ArchivingPeriod INT,
    @P__CanBeUsed BIT,
    @P__User NVARCHAR(250)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Now DATETIME = GETDATE();
    DECLARE @RowsAffected INT = 0;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Debug: Log the values
        PRINT 'Updating FileType Id: ' + CAST(@P__Id AS NVARCHAR(10)) + ', CanBeUsed: ' + CAST(@P__CanBeUsed AS NVARCHAR(1));
        
        UPDATE [dbo].[lkp_FileType]
        SET 
            [Code] = @P__Code,
            [Entity] = @P__Entity,
            [Description] = @P__Description,
            [Category] = @P__Category,
            [IsCustomer] = @P__IsCustomer,
            [ArchivingPeriod] = @P__ArchivingPeriod,
            [CanBeUsed] = @P__CanBeUsed,
            [LastModifiedBy] = @P__User,
            [LastModifiedDate] = @Now
        WHERE [Id] = @P__Id;
        
        SET @RowsAffected = @@ROWCOUNT;
        
        COMMIT TRANSACTION;
        
        SELECT @RowsAffected as RowsAffected;
        
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrSeverity INT = ERROR_SEVERITY();
        RAISERROR (@ErrMsg, @ErrSeverity, 1);
    END CATCH
END;
GO
Testing Steps:

Open your browser's Developer Console (F12)
Edit a row and change the CanBeUsed checkbox
Click Save
Check the Console for the two debug messages:

Saving data: - Should show CanBeUsed: true or CanBeUsed: false
Server response: - Should show the server's response


Check your server logs for the debug output

The key changes:

Used .is(':checked') instead of [0].checked for more reliability
Changed AJAX dataType to 'json' to properly handle JSON responses
Added debug console.log statements to track the data flow
Updated session storage after successful save
Added JsonRequestBehavior.AllowGet to the controller

This should fix the issue with the CanBeUsed checkbox value not being saved properly.

============================================================
Coming Back to our previous chats, and returning back to our project Archiving . Same logic. We newly  included a new logic. Some file type that will be added to lkp_FileType table,  with a new flag (new field added to the table), canbeused=1 for new filetype , =0 to oldone.
I will provide with a code to a view in the Configuration Menu, where we can edit Files, we have to add a new column to the datatable. CanBeUsed, disabled at first, then will enabled when user click on edit button. I want to add at the end of the table. 

Here is the code complete, view, partial view, controller and action, front end. back end controller with  BLL and the Stored procedures. make what is needed.

I'll help you add the CanBeUsed column to the FileType management interface. Here are all the necessary modifications:

1. Database Changes
Add Column to Table (if not already added)
sql
-- Add CanBeUsed column to lkp_FileType table if it doesn't exist
IF NOT EXISTS (SELECT * FROM sys.columns 
               WHERE object_id = OBJECT_ID(N'[dbo].[lkp_FileType]') 
               AND name = 'CanBeUsed')
BEGIN
    ALTER TABLE [dbo].[lkp_FileType]
    ADD [CanBeUsed] BIT NOT NULL DEFAULT 0
END
GO
Update Stored Procedure - usp_GetAllFileType
sql
ALTER PROCEDURE [dbo].[usp_GetAllFileType] 
AS 
BEGIN 
    SELECT 
        Id, 
        Code, 
        Entity, 
        Description, 
        Category, 
        HasDate, 
        IsCustomer, 
        ArchivingPeriod, 
        CanBeUsed 
    FROM lkp_FileType 
END
Update Stored Procedure - usp_Insert_Into_All_Tables
Modify the INSERT INTO lkp_FileType section to set CanBeUsed = 1 for new entries:

sql
INSERT INTO [dbo].[lkp_FileType] 
([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
       nft.Entity,
       'Not Branch' as Category,
       nft.Description,
       0 as HasDate,
       0 as IsCustomer,
       nft.ArchivingPeriod,
       1 as CanBeUsed,  -- NEW: Set to 1 for newly imported file types
       @SystemUser,
       @Now,
       @SystemUser,
       @Now
FROM @NewFileTypes nft;
2. Model Changes
FileType.cs (Model)
csharp
public class FileType
{
    public int Id { get; set; }
    public string Code { get; set; }
    public string Entity { get; set; }
    public string Description { get; set; }
    public string Category { get; set; }
    public bool HasDate { get; set; }
    public bool IsCustomer { get; set; }
    public int ArchivingPeriod { get; set; }
    public bool CanBeUsed { get; set; }  // NEW PROPERTY
    public bool IsBranch { get; set; }
}
3. View Changes
_FileTypeManagementTable.cshtml (Partial View)
cshtml
@using Alterna.Archive.Core.Models.TableModel
@using Alterna.Archive.Core.Models
@model FileTypeModel

<table id="example" class="table table-striped table-bordered" style="width:100%">
    <thead>
        <tr>
            <th>Action</th>
            <th>Code Name</th>
            <th>Entity</th>
            <th>Description</th>
            <th>IsBranch</th>
            <th>IsCustomer</th>
            <th>ArchivingPeriod</th>
            <th>CanBeUsed</th>
        </tr>
    </thead>
    <tbody>
        
@if
 
(
Model
.
FileTypeList
.
Count 
>
 
0
)

        
{

            StaticFileTypeModel
.
FileTypeList 
=
 Model
.
FileTypeList
;

            
for
 
(
int
 i 
=
 
0
;
 i 
<
 Model
.
FileTypeList
.
Count
;
 i
++
)

            
{

                
String
 entityTableTdId 
=
 
"EntityTableTd"
 
+
 Model
.
FileTypeList
[
i
]
.
Id
;

                
String
 hiddenClass 
=
 
!
Model
.
FileTypeList
[
i
]
.
IsBranch 
&&
 
!
Model
.
FileTypeList
[
i
]
.
IsCustomer 
?
 
""
 
:
 
"hidden"
;


                
<
tr
>

                    
<
td
>

                        
<
div
 
style
=
"
text-align
:
 center
;
 
cursor
:
pointer
"
 
onclick
=
"
editRow
(
this
,
@Model
.
FileTypeList
[
i
]
.
Id
,
@i
)
"
>

                            
<
span
 
class
=
"
fa-regular fa-pen-to-square
"
 
title
=
"
Edit Details
"
>
</
span
>

                        
</
div
>

                    
</
td
>

                    
@
{

                        
String
 CodeName 
=
 
$"
{
Model
.
FileTypeList
[
i
]
.
Code
}
 - 
{
Model
.
FileTypeList
[
i
]
.
Description
}
"
;

                        
String
 tdId 
=
 
$"Code-
{
Model
.
FileTypeList
[
i
]
.
Id
.
ToString
(
)
}
"
;

                    
}


                    
<
td
 
id
=
"
@tdId
"
>
@
CodeName
</
td
>


                    
<
td
>

                        
@
Html
.
DropDownListFor
(
model 
=>
 model
.
FileTypeList
[
i
]
.
Entity
,
 @Model
.
EntityList
,
 @Model
.
FileTypeList
[
i
]
.
Entity
,
 
new
 
{
 @id 
=
 
"Entity"
 
+
 @Model
.
FileTypeList
[
i
]
.
Id
,
 @disabled 
=
 
"disabled"
,
 @
class
 
=
 hiddenClass 
}
)

                    
</
td
>


                    
<
td
>
@
Html
.
EditorFor
(
model 
=>
 model
.
FileTypeList
[
i
]
.
Description
,
 
new
 
{
 htmlAttributes 
=
 
new
 
{
 @id 
=
 
"Description"
 
+
 @Model
.
FileTypeList
[
i
]
.
Id
,
 @disabled 
=
 
"disabled"
 
}
 
}
)
</
td
>


                    
<
td
>
@
Html
.
CheckBoxFor
(
model 
=>
 model
.
FileTypeList
[
i
]
.
IsBranch
,
 
new
 
{
 @id 
=
 
"IsBranch"
 
+
 @Model
.
FileTypeList
[
i
]
.
Id
,
 @disabled 
=
 
"disabled"
 
}
)
</
td
>

                    
<
td
>
@
Html
.
CheckBoxFor
(
model 
=>
 model
.
FileTypeList
[
i
]
.
IsCustomer
,
 
new
 
{
 @id 
=
 
"IsCustomer"
 
+
 @Model
.
FileTypeList
[
i
]
.
Id
,
 @disabled 
=
 
"disabled"
 
}
)
</
td
>

                    
<
td
>
@
Html
.
EditorFor
(
model 
=>
 model
.
FileTypeList
[
i
]
.
ArchivingPeriod
,
 
new
 
{
 htmlAttributes 
=
 
new
 
{
 @type 
=
 
"number"
,
 @min 
=
 
"0"
,
 @step 
=
 
"1"
,
 @id 
=
 
"ArchivingPeriod"
 
+
 @Model
.
FileTypeList
[
i
]
.
Id
,
 @disabled 
=
 
"disabled"
 
}
 
}
)
</
td
>

                    
<
td
>
@
Html
.
CheckBoxFor
(
model 
=>
 model
.
FileTypeList
[
i
]
.
CanBeUsed
,
 
new
 
{
 @id 
=
 
"CanBeUsed"
 
+
 @Model
.
FileTypeList
[
i
]
.
Id
,
 @disabled 
=
 
"disabled"
 
}
)
</
td
>

                
</
tr
>

            
}

        
}

    </tbody>
</table>

<script>
    $(document).ready(() => {
        $("#example").DataTable({
            pagingType: 'full_numbers'
        });
    });

    function editRow(element, id, index) {
        let row = $(element).closest('tr');
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        var prevData = {
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked  // NEW
        };
        sessionStorage.setItem('Ftt' + id, JSON.stringify(prevData))
        row.find('td:first-child').html('<div style="text-align: center;"><span style="cursor:pointer; color:green" class="fa-regular fa-floppy-disk" onclick="saveRow(this,' + id + ',' + index + ')" title="Save Details"></span>&nbsp;&nbsp;<span style="cursor:pointer; color:red" class="fa-solid fa-xmark" onclick="stopEdit(this,' + id + ')" title="Cancel"></span></div>');
        Fields_Switch(id, false)
    }

    function stopEdit(element, id) {
        var data = sessionStorage.getItem('Ftt' + id);
        var prevData = JSON.parse(data);
        let row = $(element).closest('tr');
        row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
        Fields_Switch(id, true)
        if (!prevData.IsCustomer && !prevData.IsBranch) {
            AddEntitySelectElementToDataTable(id);
        } else {
            RemoveEntitySelectElementToDataTable(id);
        }
        $('#Entity' + id)[0].value = prevData.Entity;
        $('#Description' + id)[0].value = prevData.Description;
        $('#IsBranch' + id)[0].checked = prevData.IsBranch,
        $('#IsCustomer' + id)[0].checked = prevData.IsCustomer;
        $('#ArchivingPeriod' + id)[0].value = prevData.ArchivingPeriod;
        $('#CanBeUsed' + id)[0].checked = prevData.CanBeUsed;  // NEW
    }

    function saveRow(element, id, Aindex) {
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        let code = $("#Code-" + id).html();
        let match = code.match(/\d+/);
        if (match) {
            code = match[0];
        }
        var dat = {
            ModelId: id,
            code: code,
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked  // NEW
        };
        $.ajax({
            type: 'POST',
            url: '/Configuration/UpdateFileType/',
            data: dat,
            dataType: 'html',
            success: function (data) {
                let row = $(element).closest('tr');
                row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
                Fields_Switch(id, true)
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
    }

    function Fields_Switch(id, x) {
        $('#Description' + id).prop('disabled', x);
        $('#IsCustomer' + id).prop('disabled', x);
        $('#ArchivingPeriod' + id).prop('disabled', x);
        $('#IsBranch' + id).prop('disabled', x);
        $('#Entity' + id).prop('disabled', x);
        $('#CanBeUsed' + id).prop('disabled', x);  // NEW
    }

    $('*[id*=IsBranch]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if (!$("#IsBranch" + index)[0].checked) {
            $('#IsCustomer' + index).prop('checked', false).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    $('*[id*=IsCustomer]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if ($("#IsCustomer" + index)[0].checked) {
            $('#IsBranch' + index).prop('checked', true).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    function AddEntitySelectElementToDataTable(index) {
        $("#Entity" + index).removeClass("hidden");
    }

    function RemoveEntitySelectElementToDataTable(index) {
        $("#Entity" + index).addClass("hidden");
    }
</script>
4. Controller Changes
ConfigurationController.cs - Add UpdateFileType Action
csharp
[HttpPost]
public ActionResult UpdateFileType(int ModelId, string code, string Entity, string Description, bool IsBranch, bool IsCustomer, int ArchivingPeriod, bool CanBeUsed)
{
    try
    {
        String session = GetSession("ArchiveData");
        
        UpdateFileTypeReq request = new UpdateFileTypeReq()
        {
            BaseReq = new BaseRequest(HttpContext, session, false),
            FileTypeId = ModelId,
            Code = code,
            Entity = Entity,
            Description = Description,
            IsBranch = IsBranch,
            IsCustomer = IsCustomer,
            ArchivingPeriod = ArchivingPeriod,
            CanBeUsed = CanBeUsed  // NEW
        };

        UpdateFileTypeRes response = Common.ApiCall<UpdateFileTypeRes>(request, "UpdateFileType");

        if (response.WebResp.HttpResponseCode == HttpStatusCode.OK)
        {
            return Json(new { success = true, message = "File Type updated successfully" });
        }
        else
        {
            return Json(new { success = false, message = response.WebResp.ResponseMessage });
        }
    }
    catch (Exception ex)
    {
        return Json(new { success = false, message = ex.Message });
    }
}
5. Request/Response Models
UpdateFileTypeReq.cs
csharp
public class UpdateFileTypeReq
{
    public BaseRequest BaseReq { get; set; }
    public int FileTypeId { get; set; }
    public string Code { get; set; }
    public string Entity { get; set; }
    public string Description { get; set; }
    public bool IsBranch { get; set; }
    public bool IsCustomer { get; set; }
    public int ArchivingPeriod { get; set; }
    public bool CanBeUsed { get; set; }  // NEW
}
UpdateFileTypeRes.cs
csharp
public class UpdateFileTypeRes
{
    public UpdateFileTypeReq Req { get; set; }
    public bool Success { get; set; }
    public WebResponse WebResp { get; set; } = new WebResponse();
}
6. Backend Controller
ArchivingController.cs - Add UpdateFileType Endpoint
csharp
#region UpdateFileType Controller
[HttpPost]
[Route("UpdateFileType")]
public UpdateFileTypeRes UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
{
    UpdateFileTypeRes response = new()
    {
        Req = updateFileTypeReq
    };

    CorrelationInfo correlationInfo = new()
    {
        CorrelationId = updateFileTypeReq.BaseReq.CorrelationId,
        RDirection = RequestDirection.Request,
        RequestURL = "UpdateFileType",
        UserName = updateFileTypeReq.BaseReq.CurrentUser
    };

    try
    {
        String CorrelationId = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateFileTypeReq.BaseReq.CorrelationId;
        String CurrentUser = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateFileTypeReq.BaseReq.CurrentUser;
        String CurrentEntity = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : updateFileTypeReq.BaseReq.CurrentEntity;
        String CurrentBranch = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : updateFileTypeReq.BaseReq.CurrentBranch;

        LogInfo("UpdateFileType Has been called with the following Request", correlationInfo);
        LogInfoJson(updateFileTypeReq, correlationInfo);

        correlationInfo.RDirection = RequestDirection.Processing;

        #region Data Guard Check
        using (BLL.BLL oBLL = new(CurrentUser))
        {
            LogInfo("Data guard checks have started", correlationInfo);

            Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
            {
                { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateFileTypeReq) },
            };

            oBLL.DataIntegrityCheck(DataGuardDictionnary);

            LogInfo("Data guard check successful", correlationInfo);

            LogInfo("Start of UpdateFileType call", correlationInfo);

            response.Success = oBLL.UpdateFileType(updateFileTypeReq);

            if (!response.Success)
            {
                throw new SGBLInternalServerException($"Failed to update File Type");
            }

            response.WebResp.CorrelationId = CorrelationId;
            response.WebResp.User = CurrentUser;
            response.WebResp.Entity = CurrentEntity;
            response.WebResp.Branch = CurrentBranch;
            response.WebResp.HttpResponseCode = HttpStatusCode.OK;

            correlationInfo.RDirection = RequestDirection.Response;

            LogInfo("UpdateFileType Has Replied with the Following response", correlationInfo);
            LogInfoJson(response, correlationInfo);
            LogInfo("Calling the UpdateFileType is completed", correlationInfo);
        }

        return response;
        #endregion
    }
    catch (SGBLBadRequestException ex)
    {
        response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateFileTypeReq.BaseReq.CorrelationId!;
        response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateFileTypeReq.BaseReq.CurrentUser!;
        response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateFileTypeReq.BaseReq.CurrentEntity!;
        response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateFileTypeReq.BaseReq.CurrentBranch!;
        response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
        response.WebResp.ResponseMessage = ex.StackTrace;
        response.Success = false;

        correlationInfo.CorrelationId = response.WebResp.CorrelationId;
        correlationInfo.UserName = response.WebResp.User;
        correlationInfo.StatusCode = HttpStatusCode.BadRequest;
        correlationInfo.RDirection = RequestDirection.Response;

        LogError(ex.Message, correlationInfo, ex);
        LogErrorJson(response, correlationInfo, ex);

        return response;
    }
    catch (Exception ex)
    {
        response.WebResp.CorrelationId = updateFileTypeReq.BaseReq.CorrelationId!;
        response.WebResp.User = updateFileTypeReq.BaseReq.CurrentUser!;
        response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
        response.WebResp.ResponseMessage = ex.StackTrace;
        response.Success = false;

        correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
        correlationInfo.RDirection = RequestDirection.Response;

        LogError(ex.StackTrace, correlationInfo);
        LogErrorJson(response, correlationInfo, ex);

        return response;
    }
}
#endregion
7. BLL Changes
BLL.cs - Add UpdateFileType Method
csharp
#region UpdateFileType

public bool UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
{
    DAL.DAL iDAL = new();
    bool result = false;
    
    OnPreEventUpdateFileType?.Invoke(ref updateFileTypeReq);

    Dictionary<string, object> parameters = new Dictionary<string, object>
    {
        { "@P__Id", updateFileTypeReq.FileTypeId },
        { "@P__Code", updateFileTypeReq.Code },
        { "@P__Entity", updateFileTypeReq.Entity },
        { "@P__Description", updateFileTypeReq.Description },
        { "@P__Category", updateFileTypeReq.IsBranch ? "Branch" : "Not Branch" },
        { "@P__IsCustomer", updateFileTypeReq.IsCustomer },
        { "@P__ArchivingPeriod", updateFileTypeReq.ArchivingPeriod },
        { "@P__CanBeUsed", updateFileTypeReq.CanBeUsed },
        { "@P__User", this.CurrentUser }
    };

    int rowsAffected = iDAL.ExecuteNonQuery("usp_UpdateFileType", parameters, CommandType.StoredProcedure);
    result = rowsAffected > 0;
    
    OnPostEventUpdateFileType?.Invoke(ref result, ref updateFileTypeReq);
    
    return result;
}

#endregion
8. Stored Procedure - Create usp_UpdateFileType
sql
CREATE PROCEDURE [dbo].[usp_UpdateFileType]
    @P__Id INT,
    @P__Code NVARCHAR(50),
    @P__Entity NVARCHAR(50),
    @P__Description NVARCHAR(250),
    @P__Category NVARCHAR(50),
    @P__IsCustomer BIT,
    @P__ArchivingPeriod INT,
    @P__CanBeUsed BIT,
    @P__User NVARCHAR(250)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Now DATETIME = GETDATE();
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        UPDATE [dbo].[lkp_FileType]
        SET 
            [Code] = @P__Code,
            [Entity] = @P__Entity,
            [Description] = @P__Description,
            [Category] = @P__Category,
            [IsCustomer] = @P__IsCustomer,
            [ArchivingPeriod] = @P__ArchivingPeriod,
            [CanBeUsed] = @P__CanBeUsed,
            [LastModifiedBy] = @P__User,
            [LastModifiedDate] = @Now
        WHERE [Id] = @P__Id;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrSeverity INT = ERROR_SEVERITY();
        RAISERROR (@ErrMsg, @ErrSeverity, 1);
    END CATCH
END;
GO
This implementation adds the CanBeUsed column as the last column in your FileType management table, with the following behavior:

Initially disabled when viewing the table
Enabled when the user clicks the edit button
Saved along with other field changes when the user clicks save
New file types imported via the stored procedure will have CanBeUsed = 1 by default
In the usp_Insert_Into_All_Tables, i want to pass the value of CanBeUsed as parameter

Here's the modified usp_Insert_Into_All_Tables stored procedure to accept CanBeUsed as a parameter:

sql
ALTER PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250),
	@P__CanBeUsed BIT = 1  -- NEW PARAMETER with default value of 1
AS 
BEGIN 
    SET NOCOUNT ON
	SELECT 1;  
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @SystemUser NVARCHAR(250) = 'AlternaSystem'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],[Code],0,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Company] comp 
            WHERE comp.[Code] = input.[Code]
        );

        -- Temp tables to hold inserted IDs and link them back to input data 
        DECLARE @InsertedContainers TABLE( 
            RowId INT, 
            ContainerId INT 
        ); 

        DECLARE @InsertedFiles TABLE( 
            RowId INT, 
            FileId INT 
        ); 

        -- Insert Containers with unique Code + CompanyCode combination
        WITH UniqueContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate,
                   ROW_NUMBER() OVER (PARTITION BY BoxRef, Code ORDER BY RowId) as rn
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_Container] cont
                WHERE cont.[Code] = input.[BoxRef]
                AND cont.[CompanyCode] = input.[Code]
            )
        ),
        ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM UniqueContainerSource
            WHERE rn = 1
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @SystemUser, @Now, @SystemUser, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Capture existing container IDs for ALL rows (including duplicates within input)
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef]
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- For input rows with duplicate BoxRef + CompanyCode, map them to the inserted container
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, ic.ContainerId
        FROM @P__Old_Boxes input
        INNER JOIN @InsertedContainers ic ON EXISTS (
            SELECT 1 FROM @P__Old_Boxes input2 
            WHERE input2.RowId = ic.RowId 
            AND input2.BoxRef = input.BoxRef
            AND input2.Code = input.Code
        )
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- Insert new File Type with auto-incrementing FileTypeCode
        DECLARE @NewFileTypes TABLE (
            Description NVARCHAR(250),
            Entity NVARCHAR(50),
            ArchivingPeriod INT,
            NextCode INT
        );
        
        DECLARE @MaxFileTypeCode INT;
        SELECT @MaxFileTypeCode = ISNULL(MAX(CAST(Code AS INT)), 0) 
        FROM [dbo].[lkp_FileType] 
        WHERE ISNUMERIC(Code) = 1;
        
        -- Get distinct Entity+Description combinations, taking the first ArchivingPeriod encountered
        WITH UniqueNewFileTypes AS (
            SELECT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   MIN(input.[ArchivingPeriod]) as ArchivingPeriod
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
            GROUP BY input.[FileName], input.[Code]
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               @P__CanBeUsed,  -- MODIFIED: Use parameter value instead of hardcoded 1
               @SystemUser,
               @Now,
               @SystemUser,
               @Now
        FROM @NewFileTypes nft;

        -- Get the FileTypeCode for each file
        DECLARE @FileTypeCodes TABLE (
            RowId INT,
            FileTypeCode NVARCHAR(50)
        );
        
        INSERT INTO @FileTypeCodes (RowId, FileTypeCode)
        SELECT input.RowId, ft.Code
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.[Entity] = input.[Code] 
            AND ft.[Description] = input.[FileName];

        -- Insert Files
        INSERT INTO [dbo].[t_File] 
        ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
        OUTPUT inserted.Id
        INTO @InsertedFiles(FileId)
        SELECT null, 
               input.FileName, 
               ftc.FileTypeCode, 
               'FINAL', 
               input.Code, 
               null, 
               null, 
               input.AdditionalInfo, 
               0, 
               @SystemUser, 
               @Now, 
               @SystemUser, 
               @Now
        FROM @P__Old_Boxes input
        INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
        ORDER BY input.RowId;

        -- Map the inserted FileIds back to RowIds
        DECLARE @RowIdMapping TABLE (
            RowId INT,
            FileId INT,
            RowNum INT
        );

        INSERT INTO @RowIdMapping (RowId, RowNum)
        SELECT RowId, ROW_NUMBER() OVER (ORDER BY RowId) as RowNum
        FROM @P__Old_Boxes;

        WITH NumberedInsertedFiles AS (
            SELECT FileId, ROW_NUMBER() OVER (ORDER BY FileId) as RowNum
            FROM @InsertedFiles
        )
        UPDATE @InsertedFiles
        SET RowId = rm.RowId
        FROM @InsertedFiles if_target
        INNER JOIN NumberedInsertedFiles nif ON if_target.FileId = nif.FileId
        INNER JOIN @RowIdMapping rm ON nif.RowNum = rm.RowNum;

        -- Insert Container File Relationship
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @SystemUser, @Now, @SystemUser, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_CurrentContainerFileRelationship] rel
            WHERE rel.[FileId] = f.FileId AND rel.[ContainerId] = c.ContainerId
        );

        -- Insert new Sequence only if Owner doesn't already exist
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Sequence] seq
            WHERE seq.[Owner] = input.[Code]
        )
        AND input.[Code] NOT IN (
            SELECT i2.[Code] 
            FROM @P__Old_Boxes i2 
            WHERE i2.RowId < input.RowId
        );

        -- ========================================
        -- MODIFIED: Insert Container Status History
        -- ========================================
        
        -- 1. Insert SENT status for all containers
        -- HoldingEntityCode = Code from input (the branch/entity code)
        -- User = BoxSentBy if not empty, otherwise AlternaSystem
        -- Always inactive (isCurrentStatus = 0)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'SENT',
               i.[Code],  -- CHANGED: Use Code instead of 'WH'
               0,         -- Always inactive for SENT
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,  -- Use BoxSentBy or AlternaSystem
               i.[BoxSentDate],
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,
               i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'SENT'
        );

        -- 2. Insert RECEIVED status for containers with StatusCode RECEIVED, DESTROYED, or NOTFOUND
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Active only if final status is RECEIVED
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'RECEIVED',
               'WH',  -- CHANGED: Always 'WH' for RECEIVED
               CASE WHEN i.[StatusCode] = 'RECEIVED' THEN 1 ELSE 0 END,
               @SystemUser,  -- Always AlternaSystem for RECEIVED
               DATEADD(MINUTE, 1, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 1, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] IN ('RECEIVED', 'DESTROYED', 'NOTFOUND')  -- ADDED: NOTFOUND
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'RECEIVED'
        );

        -- 3. Insert DESTROYED status for containers with StatusCode DESTROYED only
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Always active (final status)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'DESTROYED',
               'WH',  -- Always 'WH' for DESTROYED
               1,     -- Always active as final status
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'DESTROYED'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'DESTROYED'
        );

        -- 4. NEW: Insert NOTFOUND status for containers with StatusCode NOTFOUND
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Always active (final status)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'NOTFOUND',
               'WH',  -- Always 'WH' for NOTFOUND
               1,     -- Always active as final status
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'NOTFOUND'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'NOTFOUND'
        );

        COMMIT TRANSACTION; 
    END TRY 
    BEGIN CATCH 
        ROLLBACK TRANSACTION; 
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(); 
        DECLARE @ErrSeverity INT = ERROR_SEVERITY(); 
        RAISERROR (@ErrMsg, @ErrSeverity, 1); 
    END CATCH 
END;
Key Changes:
Added new parameter: @P__CanBeUsed BIT = 1 with a default value of 1
Modified the INSERT statement: Changed 1 as CanBeUsed to @P__CanBeUsed in the lkp_FileType INSERT statement
Usage:
Now when calling this stored procedure, you can:

Option 1 - Use default value (CanBeUsed = 1):

sql
EXEC [dbo].[usp_Insert_Into_All_Tables] 
    @P__Old_Boxes = @OldBoxesData,
    @P__User = 'SomeUser'
Option 2 - Explicitly pass CanBeUsed value:

sql
EXEC [dbo].[usp_Insert_Into_All_Tables] 
    @P__Old_Boxes = @OldBoxesData,
    @P__User = 'SomeUser',
    @P__CanBeUsed = 0  -- For old file types
This gives you flexibility to control whether imported file types should be marked as usable or not, while maintaining backward compatibility with existing code that doesn't pass this parameter (it will default to 1).
