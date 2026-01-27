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

USE [Alterna_Telecom]
GO
/****** Object:  StoredProcedure [dbo].[usp_Bulk_Insert_File_Import_Content_Alfa]    Script Date: 27/01/2026 11:57:17 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[usp_Bulk_Insert_File_Import_Content_Alfa]
	@P__FileId INT,
	@P__CurrencyCode NVARCHAR(50),
	@P__Name NVARCHAR(255),
	@P__CheckSum NVARCHAR(255),
	@P__Directory NVARCHAR(255),
	@P__Cycle DATETIME2(0) NULL,
	@P__FileImportContentAlfa [dbo].[TVP_File_Import_Content_Alfa] READONLY,
	@P__Error NVARCHAR(4000) OUTPUT,
	@P__User NVARCHAR(255)
AS
BEGIN
	SET NOCOUNT ON;

	DECLARE @StatusCode_AwaitingT24FileUpload NVARCHAR(50)=(SELECT RTRIM(LTRIM([Code])) FROM t_Lookup WHERE TableName = 'FileImportStatus' AND [Code]='AwaitingT24FileUpload');
	DECLARE @StatusCode_Discarded NVARCHAR(50)=(SELECT RTRIM(LTRIM([Code])) FROM t_Lookup WHERE TableName = 'FileImportStatus' AND [Code]='Discarded');

	DECLARE @CurrentDate DATETIME2(0)=GETDATE();

	--Validate Lookups
	IF(@StatusCode_AwaitingT24FileUpload IS NULL OR @StatusCode_Discarded IS NULL)
	BEGIN
		SET @P__Error = 'Missing Status Code in the Lookup Tables';
		RETURN;
	END

	--Check for duplicates
	DECLARE @ExistingFile INT = (SELECT 1 FROM t_File_Import WHERE [CheckSum] = @P__CheckSum AND [StatusCode] != @StatusCode_Discarded);
	IF(@ExistingFile IS NOT NULL)
	BEGIN
		SET @P__Error = 'This file has already been imported. Duplicate files are not allowed.';
		RETURN;
	END

	BEGIN --Insert Attachment
	INSERT INTO [dbo].[t_Attachment]
           ([Name]
		   ,[Directory]
           ,[CreatedDate]
           ,[CreatedBy]
           ,[LastModifiedDate]
           ,[LastModifiedBy])
     VALUES
           (@P__Name
		   ,@P__Directory
           ,@CurrentDate
           ,@P__User
           ,@CurrentDate
           ,@P__User)
	END

	DECLARE @NewAttachmentId BIGINT=SCOPE_IDENTITY();

	BEGIN --Insert File Import
	INSERT INTO [dbo].[t_File_Import]
           ([FileId]
           ,[AttachmentId]
           ,[CurrencyCode]
           ,[StatusCode]
           ,[CheckSum]
           ,[T24FileCheckSum]
		   ,[Cycle]
           ,[CreatedDate]
           ,[CreatedBy]
           ,[LastModifiedDate]
           ,[LastModifiedBy])
     VALUES
           (@P__FileId
           ,@NewAttachmentId
           ,@P__CurrencyCode
           ,@StatusCode_AwaitingT24FileUpload
		   ,@P__CheckSum
		   ,NULL
           ,CAST(@P__Cycle AS date)
           ,@CurrentDate
           ,@P__User
           ,@CurrentDate
           ,@P__User)
	
	DECLARE @NewFileImportId BIGINT=SCOPE_IDENTITY();
	DECLARE @P__FileImportId BIGINT=@NewFileImportId;

	INSERT INTO [dbo].[t_File_Import_Audit]
           ([Id]
		   ,[FileId]
           ,[AttachmentId]
           ,[CurrencyCode]
           ,[StatusCode]
           ,[CheckSum]
           ,[T24FileCheckSum]
		   ,[Cycle]
           ,[CreatedDate]
           ,[CreatedBy]
           ,[LastModifiedDate]
           ,[LastModifiedBy])
	SELECT	
			[Id]
		   ,[FileId]
           ,[AttachmentId]
           ,[CurrencyCode]
           ,[StatusCode]
           ,[CheckSum]
           ,[T24FileCheckSum]
		   ,[Cycle]
           ,[CreatedDate]
           ,[CreatedBy]
           ,[LastModifiedDate]
           ,[LastModifiedBy]
	FROM t_File_Import
	WHERE Id=@NewFileImportId;
	END


	 EXEC [dbo].[usp_Edit_File_Import_Content_Alfa]
        @P__FileImportId,
        @P__FileImportContentAlfa,
        @P__Error, -- Don't forget OUTPUT if you want to capture it
        @P__User;

     --Optional: Check for errors returned in the output parameter
    IF @P__Error IS NOT NULL
        PRINT 'Error occurred: ' + @P__Error;

	--BEGIN --Insert File ImportContent / Audit
	--INSERT INTO [dbo].[t_File_Import_Content_Alfa]
 --          ([FileImportId]
 --          ,[BankCode]
 --          ,[BankName]
 --          ,[BankBranch]
 --          ,[BankAccountNumber]
 --          ,[CustomerName]
 --          ,[PrimaryAccountNumber]
 --          ,[MsisdnPrimaryContact]
 --          ,[AccountBalance]
 --          ,[InvoiceDate]
 --          ,[AmountPaid]
 --          ,[SayrafaRate]
 --          ,[CreatedDate]
 --          ,[CreatedBy]
 --          ,[LastModifiedDate]
 --          ,[LastModifiedBy])
	--SELECT 
	--	   @NewFileImportId
	--	  ,[BankCode]
	--	  ,[BankName]
	--	  ,[BankBranch]
	--	  ,[BankAccountNumber]
	--	  ,[CustomerName]
	--	  ,[PrimaryAccountNumber]
	--	  ,[MsisdnPrimaryContact]
	--	  ,[AccountBalance]
	--	  ,[InvoiceDate]
	--	  ,[AmountPaid]
	--	  ,[SayrafaRate]
	--	  ,@CurrentDate
	--	  ,@P__User
	--	  ,@CurrentDate
	--	  ,@P__User
	--FROM @P__FileImportContentAlfa

	--INSERT INTO [dbo].[t_File_Import_Content_Alfa_Audit]
 --          ([Id]
	--	   ,[FileImportId]
 --          ,[BankCode]
 --          ,[BankName]
 --          ,[BankBranch]
 --          ,[BankAccountNumber]
 --          ,[CustomerName]
 --          ,[PrimaryAccountNumber]
 --          ,[MsisdnPrimaryContact]
 --          ,[AccountBalance]
 --          ,[InvoiceDate]
 --          ,[AmountPaid]
 --          ,[SayrafaRate]
 --          ,[CreatedDate]
 --          ,[CreatedBy]
 --          ,[LastModifiedDate]
 --          ,[LastModifiedBy])
	--SELECT 
	--	   [Id]
	--	  ,[FileImportId]
	--	  ,[BankCode]
	--	  ,[BankName]
	--	  ,[BankBranch]
	--	  ,[BankAccountNumber]
	--	  ,[CustomerName]
	--	  ,[PrimaryAccountNumber]
	--	  ,[MsisdnPrimaryContact]
	--	  ,[AccountBalance]
	--	  ,[InvoiceDate]
	--	  ,[AmountPaid]
	--	  ,[SayrafaRate]
	--	  ,@CurrentDate
	--	  ,@P__User
	--	  ,@CurrentDate
	--	  ,@P__User
	--FROM [dbo].[t_File_Import_Content_Alfa] 
	--WHERE FileImportId=@NewFileImportId;
	--END
END


