USE [Alterna_Telecom]
GO
/****** Object:  StoredProcedure [dbo].[usp_Edit_File_Import_Content_Alfa]    Script Date: 27/01/2026 12:01:43 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[usp_Edit_File_Import_Content_Alfa]
	@NewFileImportId INT,
	@P__FileImportContentAlfa [dbo].[TVP_File_Import_Content_Alfa] READONLY,
	@P__Error NVARCHAR(4000) OUTPUT,
	@P__User NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @CurrentDate DATETIME2(0) = GETDATE();

	SET @P__Error = '';

    IF NOT EXISTS (SELECT 1 FROM [dbo].[t_File_Import_Content_Alfa] WHERE FileImportId = @NewFileImportId)
    BEGIN
        SET @P__Error = CONCAT('File Import with Id ', @NewFileImportId, ' does not exist.');
        RETURN;
    END

	BEGIN
	MERGE [dbo].[t_File_Import_Content_Alfa] AS TARGET
	USING @P__FileImportContentAlfa AS SOURCE
	ON (TARGET.[FileImportId] = @NewFileImportId)
	WHEN MATCHED THEN 
		UPDATE SET
		TARGET.BankCode= SOURCE.BankCode,
		TARGET.BankName= SOURCE.BankName,
		TARGET.BankBranch = SOURCE.BankBranch,
		TARGET.BankAccountNumber= SOURCE.BankAccountNumber,
		TARGET.CustomerName= SOURCE.CustomerName,
		TARGET.PrimaryAccountNumber= SOURCE.PrimaryAccountNumber,
		TARGET.MsisdnPrimaryContact= SOURCE.MsisdnPrimaryContact,
		TARGET.AccountBalance= SOURCE.AccountBalance,
		TARGET.InvoiceDate= SOURCE.InvoiceDate,
		TARGET.AmountPaid= SOURCE.AmountPaid,
		TARGET.SayrafaRate= SOURCE.SayrafaRate;

	INSERT INTO [dbo].[t_File_Import_Content_Alfa_Audit]
           ([Id]
		   ,[FileImportId]
           ,[BankCode]
           ,[BankName]
           ,[BankBranch]
           ,[BankAccountNumber]
           ,[CustomerName]
           ,[PrimaryAccountNumber]
           ,[MsisdnPrimaryContact]
           ,[AccountBalance]
           ,[InvoiceDate]
           ,[AmountPaid]
           ,[SayrafaRate]
           ,[CreatedDate]
           ,[CreatedBy]
           ,[LastModifiedDate]
           ,[LastModifiedBy])
	SELECT 
		   [Id]
		  ,[FileImportId]
		  ,[BankCode]
		  ,[BankName]
		  ,[BankBranch]
		  ,[BankAccountNumber]
		  ,[CustomerName]
		  ,[PrimaryAccountNumber]
		  ,[MsisdnPrimaryContact]
		  ,[AccountBalance]
		  ,[InvoiceDate]
		  ,[AmountPaid]
		  ,[SayrafaRate]
		  ,@CurrentDate
		  ,@P__User
		  ,@CurrentDate
		  ,@P__User
	FROM [dbo].[t_File_Import_Content_Alfa] 
	WHERE FileImportId=@NewFileImportId;
	END

END
