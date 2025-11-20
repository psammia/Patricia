USE [Alterna.Archive]
GO
/****** Object:  StoredProcedure [dbo].[usp_Insert_Into_All_Tables]    Script Date: 20/11/2025 9:16:39 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Patricia Sammia>
-- Create date: <2025/10/30>
-- Description:	<Insert Data of Old Boxes into all tables due to excel file upload>
-- =============================================
ALTER PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250)
AS 
BEGIN 
    SET NOCOUNT ON;
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
        
        WITH UniqueNewFileTypes AS (
            SELECT DISTINCT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   input.[ArchivingPeriod]
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
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
