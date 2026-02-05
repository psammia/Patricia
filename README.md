USE [Alterna_Telecom]
GO
/****** Object:  StoredProcedure [dbo].[usp_Bulk_Insert_File_Import_Content_Alfa]    Script Date: 05/02/2026 1:21:02 PM ******/
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

	DECLARE @StatusCode_AwaitingT24FileUpload NVARCHAR(50)=(SELECT RTRIM(LTRIM([Code])) FROM t_Lookup WHERE TableName = 'FileImportStatus' AND [Code]='Pending');
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
			@NewFileImportId,
			@P__FileImportContentAlfa,
			@P__Error OUTPUT,
			@P__User;
	
	if @P__Error IS NOT NULL		
	BEGIN
        RETURN;
    END

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

USE [Alterna_Telecom]
GO
/****** Object:  StoredProcedure [dbo].[usp_Edit_File_Import_Content_Alfa]    Script Date: 05/02/2026 1:21:48 PM ******/
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

    IF @StatusCode NOT IN ('AwaitingT24FileReturned', 'Pending')
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

    -- MERGE: Use MsisdnPrimaryContact + FileImportId as the business key
    MERGE [dbo].[t_File_Import_Content_Alfa] AS TARGET
    USING @P__FileImportContentAlfa AS SOURCE
    ON (TARGET.[FileImportId] = @NewFileImportId 
        AND TARGET.[MsisdnPrimaryContact] = SOURCE.[MsisdnPrimaryContact])
    
    -- UPDATE existing records 
    WHEN MATCHED THEN 
        UPDATE SET
		-- Keep existing BankCode if SOURCE is 0 or NULL

            TARGET.[BankCode] = CASE
			WHEN SOURCE.[BankCode] = 0 OR SOURCE.[BankCode] IS NULL 
			THEN TARGET.[Bankcode]
			ELSE SOURCE.[BankCode]
			END,

            TARGET.[BankName] = SOURCE.[BankName],
            TARGET.[BankBranch] = SOURCE.[BankBranch],
            TARGET.[BankAccountNumber] = SOURCE.[BankAccountNumber],
            TARGET.[CustomerName] = SOURCE.[CustomerName],
            TARGET.[ModifiedMsisdn] = SOURCE.[ModifiedMsisdn],
			TARGET.[ManuallyMarkedAsPaid] = SOURCE.[ManuallyMarkedAsPaid],
			TARGET.[PrimaryAccountNumber] = SOURCE.[PrimaryAccountNumber],
            TARGET.[AccountBalance] = SOURCE.[AccountBalance],
            TARGET.[InvoiceDate] = SOURCE.[InvoiceDate],

			TARGET.[AmountPaid] = CASE
			WHEN SOURCE.[AmountPaid] = 0 OR SOURCE.[AmountPaid] IS NULL 
			THEN TARGET.[AmountPaid]
			ELSE SOURCE.[AccountBalance]
			END,

			TARGET.[SayrafaRate] = CASE
			WHEN SOURCE.[SayrafaRate] = 0 OR SOURCE.[SayrafaRate] IS NULL 
			THEN TARGET.[SayrafaRate]
			ELSE SOURCE.[SayrafaRate]
			END,

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
			[ModifiedMsisdn],
			[ManuallyMarkedAsPaid],
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
			SOURCE.[ModifiedMsisdn],
			SOURCE.[ManuallyMarkedAsPaid],
            SOURCE.[AccountBalance],
            SOURCE.[InvoiceDate],
            SOURCE.[AmountPaid],
            SOURCE.[SayrafaRate],
            @CurrentDate,
            @P__User,
            @CurrentDate,
            @P__User
        );

    -- Insert into audit table
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
		[ModifiedMsisdn],
		[ManuallyMarkedAsPaid],
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
		[ModifiedMsisdn],
		[ManuallyMarkedAsPaid],
        [AccountBalance],
        [InvoiceDate],
        [AmountPaid],
        [SayrafaRate],
        [CreatedDate],          
        [CreatedBy],      
        [LastModifiedDate],  
        [LastModifiedBy]        
    FROM [dbo].[t_File_Import_Content_Alfa] 
    WHERE FileImportId = @NewFileImportId;
END





