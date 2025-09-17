CREATE or ALTER PROCEDURE dbo.InsertIntoAllTables 
@MyInputTableType dbo.MyInputTableType READONLY 
AS 
BEGIN 
    SET NOCOUNT ON; 
    DECLARE @User NVARCHAR(50) = 'AlternaSystem'; 
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @FileTypeCode NVARCHAR(50) = '205'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company 
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT [Code],[CompanyName],[CompanyName],[Mnemonic],null,0,[IsActive],@User,@Now,@User,@Now 
        FROM @MyInputTableType;

        -- Temp tables to hold inserted IDs and link them back to input data 
        DECLARE @InsertedContainers TABLE( 
            RowId INT, 
            ContainerId INT 
        ); 

        DECLARE @InsertedFiles TABLE( 
            RowId INT, 
            FileId INT 
        ); 

        -- Method 1: Use MERGE to capture both RowId and Container Id
        WITH ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM @MyInputTableType
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0  -- Always insert, never match
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @User, @Now, @User, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Insert new File Type if not already existing 
        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT @FileTypeCode,[Code],'Not Branch',[FileName],0,0,[ArchivingPeriod],@User,@Now,@User,@Now 
        FROM @MyInputTableType
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[lkp_FileType] 
            WHERE Code = @FileTypeCode AND Entity = [@MyInputTableType].[Code]
        );

        -- Insert new File with similar MERGE approach
        WITH FileSource AS (
            SELECT RowId, FileName, Code, AdditionalInfo
            FROM @MyInputTableType
        )
        MERGE [dbo].[t_File] AS target
        USING FileSource AS source ON 1=0  -- Always insert, never match
        WHEN NOT MATCHED THEN
            INSERT ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
            VALUES (null, source.FileName, @FileTypeCode, 'FINAL', source.Code, null, null, source.AdditionalInfo, 0, @User, @Now, @User, @Now)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedFiles(RowId, FileId);

        -- Insert new Container File Relationship using the captured IDs
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @User, @Now, @User, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId;

        -- Insert new Sequence in case of Non Active Entity 
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT [Code],[Code]+'.',[LastIndex],null,[IsActive],@User,@Now,@User,@Now 
        FROM @MyInputTableType; 

        -- Insert Box Statuses history using the captured Container IDs
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT c.ContainerId,'SENT','WH',0,i.[BoxSentBy],i.[BoxSentDate],i.[BoxSentBy],i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @MyInputTableType i ON c.RowId = i.RowId;

        COMMIT TRANSACTION; 
    END TRY 
    BEGIN CATCH 
        ROLLBACK TRANSACTION; 
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(); 
        DECLARE @ErrSeverity INT = ERROR_SEVERITY(); 
        RAISERROR (@ErrMsg, @ErrSeverity, 1); 
    END CATCH 
END;
