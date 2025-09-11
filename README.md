CREATE OR ALTER PROCEDURE dbo.InsertIntoAllTables
    @InputData dbo.MyInputTableType READONLY
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @User NVARCHAR(50) = 'AlternaSystem';
    DECLARE @Now DATETIME = GETDATE();
    DECLARE @CompanyCode NVARCHAR(11) = 'ET000132';
    DECLARE @FileTypeCode NVARCHAR(50) = '205';

    -- Temp tables to hold IDs after insert
    DECLARE @InsertedContainers TABLE (
        RowId INT,
        ContainerId INT
    );

    DECLARE @InsertedFiles TABLE (
        RowId INT,
        FileId INT
    );

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Insert new Company (may duplicate if not checked; consider EXISTS logic)
        INSERT INTO [dbo].[t_Company]
            ([Code], [CompanyName], [NameAddress], [Mnemonic],
             [DisplayDescription], [isBranch], [IsActive],
             [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate])
        SELECT DISTINCT
            @CompanyCode, [CompanyName], [CompanyName], [Mnemonic],
            NULL, 0, [IsActive],
            @User, @Now, @User, @Now
        FROM @InputData;

        -- Insert into t_Container and capture inserted IDs
        INSERT INTO [dbo].[t_Container]
            ([Code], [CompanyCode], [Entity], [CurrentLocation],
             [StatusCode], [ArchivingDate], [isDeleted],
             [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate], [isNotified])
        OUTPUT i.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId)
        SELECT
            i.BoxRef, i.CompanyCode, i.CompanyName, '',
            i.StatusCode, i.BoxSentDate, 0,
            @User, @Now, @User, @Now, 1
        FROM @InputData i;

        -- Insert FileType (optionally check existence to avoid duplicates)
        INSERT INTO [dbo].[lkp_FileType]
            ([Code], [Entity], [Category], [Description],
             [HasDate], [IsCustomer], [ArchivingPeriod],
             [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate])
        SELECT DISTINCT
            @FileTypeCode, i.CompanyCode, 'Not Branch', i.FileName,
            0, 0, i.ArchivingPeriod,
            @User, @Now, @User, @Now
        FROM @InputData i;

        -- Insert into t_File and capture inserted IDs
        INSERT INTO [dbo].[t_File]
            ([CustomerId], [Name], [FileTypeCode], [StatusCode], [CompanyCode],
             [FromDate], [ToDate], [AdditionalInfo], [isDeleted],
             [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate])
        OUTPUT i.RowId, inserted.Id
        INTO @InsertedFiles(RowId, FileId)
        SELECT
            NULL, i.FileName, @FileTypeCode, 'FINAL', i.CompanyCode,
            NULL, NULL, i.AdditionalInfo, 0,
            @User, @Now, @User, @Now
        FROM @InputData i;

        -- Insert into relationship table
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship]
            ([FileId], [ContainerId], [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate])
        SELECT
            f.FileId, c.ContainerId,
            @User, @Now, @User, @Now
        FROM @InsertedFiles f
        JOIN @InsertedContainers c ON f.RowId = c.RowId;

        -- Insert into Sequence (only if Company is NOT Active)
        INSERT INTO [dbo].[t_Sequence]
            ([Owner], [Prefix], [LastIndex], [Suffix], [IsActive],
             [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate])
        SELECT
            i.CompanyCode, i.CompanyCode + '.', i.LastIndex, NULL, i.IsActive,
            @User, @Now, @User, @Now
        FROM @InputData i
        WHERE i.IsActive = 0;

        -- Insert into t_ContainerStatus (SENT status only here)
        INSERT INTO [dbo].[t_ContainerStatus]
            ([ContainerId], [StatusCode], [HoldingEntityCode], [isCurrentStatus],
             [CreatedBy], [CreatedDate], [LastModifiedBy], [LastModifiedDate])
        SELECT
            c.ContainerId, 'SENT', 'WH', 0,
            i.BoxSentBy, i.BoxSentDate, i.BoxSentBy, i.BoxSentDate
        FROM @InputData i
        JOIN @InsertedContainers c ON i.RowId = c.RowId;

        -- TODO: Add RECEIVED / DESTROYED logic here if needed (see below)

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;

        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrSeverity INT = ERROR_SEVERITY();
        RAISERROR (@ErrMsg, @ErrSeverity, 1);
    END CATCH
END;
