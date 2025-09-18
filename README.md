
CREATE OR ALTER PROCEDURE dbo.InsertIntoAllTables 
@MyInputTableType dbo.MyInputTableType READONLY 
AS 
BEGIN 
    SET NOCOUNT ON; 
    DECLARE @User NVARCHAR(50) = 'AlternaSystem'; 
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @FileTypeCode NVARCHAR(50) = '205'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],null,0,[IsActive],@User,@Now,@User,@Now 
        FROM @MyInputTableType input
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

        -- Insert Containers only for combinations that don't already exist
        -- Assuming BoxRef + CompanyCode should be unique business key
        WITH ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM @MyInputTableType input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_Container] cont
                WHERE cont.[Code] = input.[BoxRef] 
                AND cont.[CompanyCode] = input.[Code]
            )
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0  -- Always insert, never match
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @User, @Now, @User, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Also capture existing container IDs for rows that weren't inserted
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @MyInputTableType input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef] 
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- Insert new File Type with auto-incrementing FileTypeCode (PK)
        -- Get the next available FileTypeCode for each unique Entity
        DECLARE @NextFileTypeCode TABLE (
            Entity NVARCHAR(50),
            NextCode INT
        );
        
        -- Calculate next FileTypeCode for each entity that needs a new FileType
        INSERT INTO @NextFileTypeCode (Entity, NextCode)
        SELECT DISTINCT input.[Code] as Entity,
               ISNULL((SELECT MAX(CAST(ft.Code AS INT)) FROM [dbo].[lkp_FileType] ft WHERE ISNUMERIC(ft.Code) = 1), 0) + 
               ROW_NUMBER() OVER (ORDER BY input.[Code]) as NextCode
        FROM @MyInputTableType input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[lkp_FileType] ft
            WHERE ft.Entity = input.[Code] 
            AND ft.[Description] = input.[FileName]
        );

        -- Insert new File Type records with incremented FileTypeCode
        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               input.[Code] as Entity,
               'Not Branch' as Category,
               input.[FileName] as Description,
               0 as HasDate,
               0 as IsCustomer,
               input.[ArchivingPeriod],
               @User,
               @Now,
               @User,
               @Now
        FROM @MyInputTableType input
        INNER JOIN @NextFileTypeCode nft ON nft.Entity = input.[Code]
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[lkp_FileType] ft
            WHERE ft.Entity = input.[Code] 
            AND ft.[Description] = input.[FileName]
        );

        -- Get the FileTypeCode for each file (either existing or newly created)
        DECLARE @FileTypeCodes TABLE (
            RowId INT,
            FileTypeCode NVARCHAR(50)
        );
        
        INSERT INTO @FileTypeCodes (RowId, FileTypeCode)
        SELECT input.RowId, ft.Code
        FROM @MyInputTableType input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.Entity = input.[Code] 
            AND ft.[Description] = input.[FileName];

        -- Insert Files using the correct FileTypeCode for each row
        WITH FileSource AS (
            SELECT input.RowId, input.FileName, input.Code, input.AdditionalInfo, ftc.FileTypeCode
            FROM @MyInputTableType input
            INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_File] f
                WHERE f.[Name] = input.[FileName] 
                AND f.[CompanyCode] = input.[Code]
                AND f.[FileTypeCode] = ftc.FileTypeCode
            )
        )
        MERGE [dbo].[t_File] AS target
        USING FileSource AS source ON 1=0  -- Always insert, never match
        WHEN NOT MATCHED THEN
            INSERT ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
            VALUES (null, source.FileName, source.FileTypeCode, 'FINAL', source.Code, null, null, source.AdditionalInfo, 0, @User, @Now, @User, @Now)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedFiles(RowId, FileId);

        -- Also capture existing file IDs for rows that weren't inserted
        INSERT INTO @InsertedFiles(RowId, FileId)
        SELECT input.RowId, f.Id
        FROM @MyInputTableType input
        INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
        INNER JOIN [dbo].[t_File] f ON f.[Name] = input.[FileName] 
            AND f.[CompanyCode] = input.[Code]
            AND f.[FileTypeCode] = ftc.FileTypeCode
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedFiles);

        -- Insert new Container File Relationship using the captured IDs
        -- Check if relationship already exists to avoid duplicates
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @User, @Now, @User, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_CurrentContainerFileRelationship] rel
            WHERE rel.[FileId] = f.FileId AND rel.[ContainerId] = c.ContainerId
        );

        -- Insert new Sequence only if Owner doesn't already exist
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@User,@Now,@User,@Now 
        FROM @MyInputTableType input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Sequence] seq
            WHERE seq.[Owner] = input.[Code]
        );

        -- Insert Box Statuses history using the captured Container IDs
        -- Only insert if this exact status doesn't already exist for the container
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT c.ContainerId,'SENT','WH',0,
               CASE WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @User ELSE i.[BoxSentBy] END,
               i.[BoxSentDate],
               CASE WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @User ELSE i.[BoxSentBy] END,
               i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @MyInputTableType i ON c.RowId = i.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'SENT'
            AND cs.[HoldingEntityCode] = 'WH'
            AND cs.[CreatedDate] = i.[BoxSentDate]
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
