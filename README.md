USE [Alterna.Archive.PRD2]
GO
/****** Object:  StoredProcedure [dbo].[usp_Insert_Into_All_Tables_Branch_OldBoxes]    Script Date: 20/02/2026 10:58:47 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER     PROCEDURE [dbo].[usp_Insert_Into_All_Tables_Branch_OldBoxes] 
	@P__Branch_Old_Boxes [dbo].[TVP_Branch_Old_Boxes] READONLY,
	@P__User NVARCHAR(250),
	@P__CanBeUsed BIT = 0
AS	
BEGIN 
    SET NOCOUNT ON
	SELECT 1;  
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @SystemUser NVARCHAR(250) = 'AlternaSystem'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

		-- ========================================================================
		-- Early Exit: Check if this data has already been processed
		-- if ANY container from the input already exists, reject the entire batch
		-- ========================================================================
		IF EXISTS (
			SELECT 1
			FROM @P__Branch_Old_Boxes input 
			INNER JOIN [dbo].[t_Container] cont
			ON cont.[code] = input.[BoxRef]
			AND cont.[CompanyCode] = input.[Code]
		)
		BEGIN
		RAISERROR('This data has already been uploaded. One or more containers already exist in the system.',16,1);
		ROLLBACK TRANSACTION;
		RETURN; 
		END
		 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],[Code],1,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Branch_Old_Boxes input
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
            FROM @P__Branch_Old_Boxes input
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
            VALUES (source.BoxRef, source.Code, 'RCA', '', source.StatusCode, source.BoxSentDate, 0, @SystemUser, @Now, @SystemUser, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Capture existing container IDs for ALL rows (including duplicates within input)
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @P__Branch_Old_Boxes input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef]
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- For input rows with duplicate BoxRef + CompanyCode, map them to the inserted container
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, ic.ContainerId
        FROM @P__Branch_Old_Boxes input
        INNER JOIN @InsertedContainers ic ON EXISTS (
            SELECT 1 FROM @P__Branch_Old_Boxes input2 
            WHERE input2.RowId = ic.RowId 
            AND input2.BoxRef = input.BoxRef
            AND input2.Code = input.Code
        )
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- ========================================
        -- MODIFIED: Insert new File Type - Uniqueness on Entity, Description, AND ArchivingPeriod
        -- ========================================
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
        
        -- Get distinct Entity+Description+ArchivingPeriod combinations
        WITH UniqueNewFileTypes AS (
            SELECT DISTINCT
                   input.[FileName] as Description,
                   'RCA' as Entity,
                   input.[ArchivingPeriod]
            FROM @P__Branch_Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
                AND ft.[ArchivingPeriod] = input.[ArchivingPeriod]  -- ADDED: Check ArchivingPeriod too
            )
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description, ArchivingPeriod) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Branch' as Category,
               nft.Description,
               1 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               @P__CanBeUsed,
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
        FROM @P__Branch_Old_Boxes input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.[Entity] = input.[Code] 
            AND ft.[Description] = input.[FileName]
            AND ft.[ArchivingPeriod] = input.[ArchivingPeriod];  -- ADDED: Match on ArchivingPeriod too

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
			   input.FromDate, 
               input.ToDate,  
               input.AdditionalInfo, 
               0, 
               @SystemUser, 
               @Now, 
               @SystemUser, 
               @Now
        FROM @P__Branch_Old_Boxes input
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
        FROM @P__Branch_Old_Boxes;

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

        -- Insert new Sequence (largest one) only if Owner doesn't already exist

		WITH RankedInputs AS (
    SELECT 
        [Code], 
        [LastIndex], 
        [IsActive],
        ROW_NUMBER() OVER (PARTITION BY [Code] ORDER BY [LastIndex] DESC) as rn
    FROM @P__Branch_Old_Boxes
)
INSERT INTO [dbo].[t_Sequence] 
    ([Owner], [Prefix], [LastIndex], [Suffix], [IsActive], 
     [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate]) 
SELECT 
    [Code], 
    [Code] + '.', 
    [LastIndex], 
    NULL, 
    [IsActive], 
    @SystemUser, 
    @Now, 
    @SystemUser, 
    @Now 
FROM RankedInputs
WHERE rn = 1 -- Only the row with the largest LastIndex per Code
AND NOT EXISTS (
    SELECT 1 FROM [dbo].[t_Sequence] seq
    WHERE seq.[Owner] = RankedInputs.[Code]
);
        --INSERT INTO [dbo].[t_Sequence] 
        --([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        --SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        --FROM @P__Branch_Old_Boxes input
        --WHERE NOT EXISTS (
        --    SELECT 1 FROM [dbo].[t_Sequence] seq
        --    WHERE seq.[Owner] = input.[Code]
        --)
        --AND input.[Code] NOT IN (
        --    SELECT i2.[Code] 
        --    FROM @P__Branch_Old_Boxes i2 
        --    WHERE i2.RowId < input.RowId
        --);



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
        INNER JOIN @P__Branch_Old_Boxes i ON c.RowId = i.RowId
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
        INNER JOIN @P__Branch_Old_Boxes i ON c.RowId = i.RowId
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
        INNER JOIN @P__Branch_Old_Boxes i ON c.RowId = i.RowId
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
        INNER JOIN @P__Branch_Old_Boxes i ON c.RowId = i.RowId
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



