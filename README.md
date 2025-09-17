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

        -------------------------------------------------------------------
        -- 1. Insert into t_Company (no OUTPUT needed here)
        -------------------------------------------------------------------
        INSERT INTO [dbo].[t_Company]
           ([Code]
           ,[CompanyName]
           ,[NameAddress]
           ,[Mnemonic]
           ,[DisplayDescription]
           ,[isBranch]
           ,[IsActive]
           ,[CreatedBy]
           ,[CreatedDate]
           ,[LastModifiedBy]
           ,[LastModifiedDate])
        SELECT 
            [Code],[CompanyName],[CompanyName], [Mnemonic], null, 0, [IsActive], @User, @Now, @User, @Now
        FROM @MyInputTableType;


        -------------------------------------------------------------------
        -- 2. Temp tables to capture inserted IDs
        -------------------------------------------------------------------
        DECLARE @InsertedContainers TABLE(
            RowId INT,
            ContainerId INT
        );

        DECLARE @InsertedFiles TABLE(
            RowId INT,
            FileId INT
        );


        -------------------------------------------------------------------
        -- 3. Insert into t_Container with RowId tracking
        -------------------------------------------------------------------
        INSERT INTO [dbo].[t_Container]
            ([Code]
            ,[CompanyCode]
            ,[Entity]
            ,[CurrentLocation]
            ,[StatusCode]
            ,[ArchivingDate]
            ,[isDeleted]
            ,[CreatedBy]
            ,[CreatedDate]
            ,[LastModifiedBy]
            ,[LastModifiedDate]
            ,[isNotified])
        OUTPUT
            d.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId)
        SELECT 
            d.BoxRef, 
            d.Code,
            d.CompanyName,
            '', 
            d.StatusCode, 
            d.BoxSentDate, 
            0,
            @User, 
            @Now, 
            @User, 
            @Now, 
            1
        FROM (
            SELECT src.RowId, src.BoxRef, src.Code, src.CompanyName, src.StatusCode, src.BoxSentDate
            FROM @MyInputTableType src
        ) d;


        -------------------------------------------------------------------
        -- 4. Insert into t_File with RowId tracking
        -------------------------------------------------------------------
        INSERT INTO [dbo].[t_File]
            ([FileName]
            ,[FileTypeCode]
            ,[EntityName]
            ,[AdditionalInfo]
            ,[isDeleted]
            ,[CreatedBy]
            ,[CreatedDate]
            ,[LastModifiedBy]
            ,[LastModifiedDate])
        OUTPUT
            d.RowId, inserted.Id
        INTO @InsertedFiles(RowId, FileId)
        SELECT 
            d.FileName,
            @FileTypeCode,
            d.CompanyName,
            d.AdditionalInfo,
            0,
            @User,
            @Now,
            @User,
            @Now
        FROM (
            SELECT src.RowId, src.FileName, src.CompanyName, src.AdditionalInfo
            FROM @MyInputTableType src
        ) d;


        -------------------------------------------------------------------
        -- 5. Example: Insert into junction table t_ContainerFile
        -- (link Containers and Files by RowId)
        -------------------------------------------------------------------
        INSERT INTO [dbo].[t_ContainerFile]
            ([ContainerId]
            ,[FileId]
            ,[CreatedBy]
            ,[CreatedDate])
        SELECT 
            c.ContainerId,
            f.FileId,
            @User,
            @Now
        FROM @InsertedContainers c
        INNER JOIN @InsertedFiles f ON f.RowId = c.RowId;


        -------------------------------------------------------------------
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END
