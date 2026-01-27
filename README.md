USE [Alterna_Telecom]
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

ALTER PROCEDURE [dbo].[usp_Edit_File_Import_Content_Alfa]
    @NewFileImportId BIGINT,
    @P__FileImportContentAlfa [dbo].[TVP_File_Import_Content_Alfa] READONLY,
    @P__Error NVARCHAR(4000) OUTPUT,
    @P__User NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @CurrentDate DATETIME2(0) = GETDATE();
    DECLARE @StatusCode NVARCHAR(50);

    SET @P__Error = '';

    -- Check if FileImport exists
    IF NOT EXISTS (SELECT 1 FROM [dbo].[t_File_Import] WHERE Id = @NewFileImportId)
    BEGIN
        SET @P__Error = CONCAT('File Import with Id ', @NewFileImportId, ' does not exist.');
        RETURN;
    END

    -- Check if status allows editing
    SELECT @StatusCode = StatusCode 
    FROM [dbo].[t_File_Import] 
    WHERE Id = @NewFileImportId;

    IF @StatusCode IN ('Completed', 'AwaitingT24FileReturned')
    BEGIN
        SET @P__Error = CONCAT('Cannot edit records. The file import status is "', @StatusCode, '" and cannot be modified.');
        RETURN;
    END

    -- Validate that we have data to process
    IF NOT EXISTS (SELECT 1 FROM @P__FileImportContentAlfa)
    BEGIN
        SET @P__Error = 'No data provided to update or insert.';
        RETURN;
    END

    -- Validate MSISDN uniqueness in the incoming data
    IF EXISTS (
        SELECT MsisdnPrimaryContact 
        FROM @P__FileImportContentAlfa 
        GROUP BY MsisdnPrimaryContact 
        HAVING COUNT(*) > 1
    )
    BEGIN
        SET @P__Error = 'Duplicate MSISDN found in the provided data. Each MSISDN must be unique.';
        RETURN;
    END

    -- MERGE: Use MsisdnPrimaryContact + FileImportId as the business key
    MERGE [dbo].[t_File_Import_Content_Alfa] AS TARGET
    USING @P__FileImportContentAlfa AS SOURCE
    ON (TARGET.[FileImportId] = @NewFileImportId 
        AND TARGET.[MsisdnPrimaryContact] = SOURCE.[MsisdnPrimaryContact])
    
    -- UPDATE existing records (when MSISDN already exists for this FileImportId)
    WHEN MATCHED THEN 
        UPDATE SET
            TARGET.[BankCode] = SOURCE.[BankCode],
            TARGET.[BankName] = SOURCE.[BankName],
            TARGET.[BankBranch] = SOURCE.[BankBranch],
            TARGET.[BankAccountNumber] = SOURCE.[BankAccountNumber],
            TARGET.[CustomerName] = SOURCE.[CustomerName],
            TARGET.[PrimaryAccountNumber] = SOURCE.[PrimaryAccountNumber],
            TARGET.[AccountBalance] = SOURCE.[AccountBalance],
            TARGET.[InvoiceDate] = SOURCE.[InvoiceDate],
            TARGET.[AmountPaid] = SOURCE.[AmountPaid],
            TARGET.[SayrafaRate] = SOURCE.[SayrafaRate],
            TARGET.[LastModifiedDate] = @CurrentDate,
            TARGET.[LastModifiedBy] = @P__User
    
    -- INSERT new records (when MSISDN doesn't exist for this FileImportId)
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (
            [FileImportId],
            [BankCode],
            [BankName],
            [BankBranch],
            [BankAccountNumber],
            [CustomerName],
            [PrimaryAccountNumber],
            [MsisdnPrimaryContact],
            [AccountBalance],
            [InvoiceDate],
            [AmountPaid],
            [SayrafaRate],
            [CreatedDate],
            [CreatedBy],
            [LastModifiedDate],
            [LastModifiedBy]
        )
        VALUES (
            @NewFileImportId,
            SOURCE.[BankCode],
            SOURCE.[BankName],
            SOURCE.[BankBranch],
            SOURCE.[BankAccountNumber],
            SOURCE.[CustomerName],
            SOURCE.[PrimaryAccountNumber],
            SOURCE.[MsisdnPrimaryContact],
            SOURCE.[AccountBalance],
            SOURCE.[InvoiceDate],
            SOURCE.[AmountPaid],
            SOURCE.[SayrafaRate],
            @CurrentDate,
            @P__User,
            @CurrentDate,
            @P__User
        );

    -- Insert into audit table (AFTER merge, captures the updated/inserted state)
    INSERT INTO [dbo].[t_File_Import_Content_Alfa_Audit]
    (
        [Id],
        [FileImportId],
        [BankCode],
        [BankName],
        [BankBranch],
        [BankAccountNumber],
        [CustomerName],
        [PrimaryAccountNumber],
        [MsisdnPrimaryContact],
        [AccountBalance],
        [InvoiceDate],
        [AmountPaid],
        [SayrafaRate],
        [CreatedDate],
        [CreatedBy],
        [LastModifiedDate],
        [LastModifiedBy]
    )
    SELECT 
        [Id],
        [FileImportId],
        [BankCode],
        [BankName],
        [BankBranch],
        [BankAccountNumber],
        [CustomerName],
        [PrimaryAccountNumber],
        [MsisdnPrimaryContact],
        [AccountBalance],
        [InvoiceDate],
        [AmountPaid],
        [SayrafaRate],
        [CreatedDate],          -- Preserve original CreatedDate
        [CreatedBy],            -- Preserve original CreatedBy
        [LastModifiedDate],     -- Use the updated LastModifiedDate
        [LastModifiedBy]        -- Use the updated LastModifiedBy
    FROM [dbo].[t_File_Import_Content_Alfa] 
    WHERE FileImportId = @NewFileImportId;
END
GO
