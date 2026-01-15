/****** Object:  StoredProcedure [dbo].[usp_Update_File_Import_Content_Alfa_Msisdn]    Script Date: 14/01/2026 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[usp_Update_File_Import_Content_Alfa_Msisdn]
    @P__Id BIGINT,
    @P__MsisdnPrimaryContact NVARCHAR(255),
    @P__User NVARCHAR(255),
    @P__Error NVARCHAR(4000) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @CurrentDate DATETIME2(0) = GETDATE();
    DECLARE @TrimmedMsisdn NVARCHAR(255);
    DECLARE @FileImportId BIGINT;
    DECLARE @StatusCode NVARCHAR(50);

    SET @P__Error = '';
    SET @TrimmedMsisdn = LTRIM(RTRIM(@P__MsisdnPrimaryContact));

    -- Check if record exists and get FileImportId
    IF NOT EXISTS (SELECT 1 FROM [dbo].[t_File_Import_Content_Alfa] WHERE Id = @P__Id)
    BEGIN
        SET @P__Error = CONCAT('File Import Content with Id ', @P__Id, ' does not exist.');
        RETURN;
    END

    -- Get FileImportId from the content record
    SELECT @FileImportId = FileImportId 
    FROM [dbo].[t_File_Import_Content_Alfa] 
    WHERE Id = @P__Id;

    -- Check the status of the File Import
    SELECT @StatusCode = StatusCode 
    FROM [dbo].[t_File_Import] 
    WHERE Id = @FileImportId;

    -- Prevent editing if status is in the blocked list
    IF @StatusCode IN ('Completed', 'AwaitingT24FileReturned', 'Discarded')
    BEGIN
        SET @P__Error = CONCAT('Cannot edit MSISDN. The file import status is "', @StatusCode, '" and cannot be modified.');
        RETURN;
    END

    -- Validate MSISDN is not empty
    IF @TrimmedMsisdn IS NULL OR @TrimmedMsisdn = ''
    BEGIN
        SET @P__Error = 'MSISDN Primary Contact cannot be empty.';
        RETURN;
    END

    -- Validate MSISDN length (exactly 11 characters)
    IF LEN(@TrimmedMsisdn) != 11
    BEGIN
        SET @P__Error = 'MSISDN Primary Contact must be exactly 11 digits.';
        RETURN;
    END

    -- Validate MSISDN contains only digits (0-9)
    IF @TrimmedMsisdn LIKE '%[^0-9]%'
    BEGIN
        SET @P__Error = 'MSISDN Primary Contact must contain only digits (0-9).';
        RETURN;
    END

    -- Update only MsisdnPrimaryContact, LastModifiedDate, and LastModifiedBy
    UPDATE [dbo].[t_File_Import_Content_Alfa]
    SET 
        [MsisdnPrimaryContact] = @TrimmedMsisdn,
        [LastModifiedDate] = @CurrentDate,
        [LastModifiedBy] = @P__User
    WHERE Id = @P__Id;

    -- Insert the UPDATED record into audit table (after the update)
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
        [CreatedDate],
        [CreatedBy],
        [LastModifiedDate],
        [LastModifiedBy]
    FROM [dbo].[t_File_Import_Content_Alfa]
    WHERE Id = @P__Id;
END
GO

Easy Bal.cs
------------
#region UpdateFileImportContentMsisdn
public async Task UpdateFileImportContentMsisdn(UpdateFileImportContentMsisdnRequest request)
{
    // Validate MSISDN - will throw on FIRST error only
    Utils.ValidateMsisdnOrThrow(request.MsisdnPrimaryContact);

    // Let stored procedure handle status validation
    DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

    DynamicParameters parameters = new DynamicParameters();

    parameters.Add("@P__Id", request.Id);
    parameters.Add("@P__MsisdnPrimaryContact", request.MsisdnPrimaryContact.Trim());
    parameters.Add("@P__User", request.BaseReq.UserName);
    parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

    _ = await dal.ExecuteQueryAsync<dynamic>(
        "usp_Update_File_Import_Content_Alfa_Msisdn",
        parameters,
        CommandType.StoredProcedure,
        DapperDal.CommandDirection.Update);

    string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");

    if (!string.IsNullOrWhiteSpace(storedProcedureErrorMessage))
    {
        throw new SGBLBadRequestException(storedProcedureErrorMessage);
    }
}
#endregion

--------------------------
--------------------------
#region UpdateFileImportContentMsisdn
public async Task UpdateFileImportContentMsisdn(UpdateFileImportContentMsisdnRequest request)
{
    // Validate MSISDN - will throw on FIRST error only
    Utils.ValidateMsisdnOrThrow(request.MsisdnPrimaryContact);

    // Check status before attempting update
    await ValidateFileImportStatusForEdit(request.Id);

    // If we reach here, validation passed
    DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

    DynamicParameters parameters = new DynamicParameters();

    parameters.Add("@P__Id", request.Id);
    parameters.Add("@P__MsisdnPrimaryContact", request.MsisdnPrimaryContact.Trim());
    parameters.Add("@P__User", request.BaseReq.UserName);
    parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

    _ = await dal.ExecuteQueryAsync<dynamic>(
        "usp_Update_File_Import_Content_Alfa_Msisdn",
        parameters,
        CommandType.StoredProcedure,
        DapperDal.CommandDirection.Update);

    string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");

    if (!string.IsNullOrWhiteSpace(storedProcedureErrorMessage))
    {
        throw new SGBLBadRequestException(storedProcedureErrorMessage);
    }
}
#endregion

#region ValidateFileImportStatusForEdit
private async Task ValidateFileImportStatusForEdit(long fileImportContentId)
{
    DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

    DynamicParameters parameters = new DynamicParameters();

    parameters.Add("@P__FileImportContentId", fileImportContentId);

    string query = @"
        SELECT fi.StatusCode 
        FROM t_File_Import fi
        INNER JOIN t_File_Import_Content_Alfa fica ON fi.Id = fica.FileImportId
        WHERE fica.Id = @P__FileImportContentId";

    IEnumerable<string> result = await dal.ExecuteQueryAsync<string>(
        query,
        parameters,
        CommandType.Text,
        DapperDal.CommandDirection.Select);

    string? statusCode = result.FirstOrDefault();

    if (statusCode == null)
    {
        throw new SGBLBadRequestException("File Import Content record not found.");
    }

    if (statusCode.Equals("Completed", StringComparison.OrdinalIgnoreCase))
    {
        throw new SGBLBadRequestException("Cannot edit MSISDN. The file import status is Completed and cannot be modified.");
    }
}
#endregion

