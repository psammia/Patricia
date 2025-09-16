
CREATE TYPE dbo.MyInputTableType AS TABLE
(
[RowId] INT, -- Used to track each row uniquely

[CompanyCode] NVARCHAR(11), -- CompanyCode in t_Company
[CompanyName] NVARCHAR(22), --Entity Name t_Company & t_Container & lkp_File & t_File
[Mnemonic] NVARCHAR(50), --Mnemonic
[IsActive] BIT, --Active t_Company & t_Sequence

[BoxRef] NVARCHAR(50), --BoxRef t_Container
[FileName] NVARCHAR(250), --File Name lkp_File & t_File
[AdditionalInfo] NVARCHAR(1000), -- File additional Info t_File
[StatusCode] NVARCHAR(10), --Box Status t_Container
[ArchivingPeriod] INT, --ArchivingPeriod (yearly) lkp_FileType
[BoxSentBy] NVARCHAR(250), -- t_ContainerStatus
[BoxSentDate] DATETIME, -- t_Container & t_ContainerStatus
[LastIndex] BIGINT -- t_Sequence in case of a Non Active Entity
);


CREATE or ALTER PROCEDURE dbo.InsertIntoAllTables
    @InputData dbo.MyInputTableType READONLY
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @User NVARCHAR(50) = 'AlternaSystem';
    DECLARE @Now DATETIME = GETDATE();
	DECLARE @CompanyCode NVARCHAR(11)= 'ET000132'; --Get from PROD the latest one
	DECLARE @FileTypeCode NVARCHAR(50) = '205';	--Get from PROD the latest one

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Insert new Company
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
            @CompanyCode,[CompanyName], [CompanyName], [Mnemonic], null, 0, [IsActive], @User, @Now, @User, @Now
        FROM @InputData;
		
		--Temp tables to hold inserted IDs and link them back to input data
		DECLARE @InsertedContainers TABLE(
			RowId INT,
			ContainerId INT
		);
		DECLARE @InsertedFiles TABLE(
			RowId INT,
			FileId INT
		);

		-- Insert into t_Container and capture IDENTITY
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
			i.RowId, inserted.Id
		INTO @InsertedContainers(RowId, ContainerId)
		SELECT 
			i.BoxRef, i.CompanyCode, i.CompanyName, '', i.StatusCode, i.BoxSentDate, 0,@User, @Now, @User, @Now, 1
		FROM @InputData i;

		-- Insert new File Type if not already existing
		INSERT INTO [dbo].[lkp_FileType]
           ([Code]
           ,[Entity]
           ,[Category]
           ,[Description]
           ,[HasDate]
           ,[IsCustomer]
           ,[ArchivingPeriod]
           ,[CreatedBy]
           ,[CreatedDate]
           ,[LastModifiedBy]
           ,[LastModifiedDate])
		 SELECT 
            @FileTypeCode,[CompanyCode], 'Not Branch',[FileName], 0,0,[ArchivingPeriod], @User, @Now, @User, @Now
        FROM @InputData;


		-- Insert new File
		INSERT INTO [dbo].[t_File]
           ([CustomerId]
           ,[Name]
           ,[FileTypeCode]
           ,[StatusCode]
           ,[CompanyCode]
           ,[FromDate]
           ,[ToDate]
           ,[AdditionalInfo]
           ,[isDeleted]
           ,[CreatedBy]
           ,[CreatedDate]
           ,[LastModifiedBy]
           ,[LastModifiedDate])
		  SELECT 
            null,[FileName],@FileTypeCode,'FINAL',[CompanyCode],null,null,[AdditionalInfo],0,@User, @Now, @User, @Now
        FROM @InputData;

		-- Insert new Container File RelationShip
		INSERT INTO [dbo].[t_CurrentContainerFileRelationship]
           ([FileId]
           ,[ContainerId]
           ,[CreatedBy]
           ,[CreatedDate]
           ,[LastModifiedBy]
           ,[LastModifiedDate])
		 SELECT 
            t_File.Id,t_Container.Id,@User, @Now, @User, @Now
        FROM @InputData;

		-- Insert new Sequence in case of Non Active Entity
		INSERT INTO [dbo].[t_Sequence]
           ([Owner]
           ,[Prefix]
           ,[LastIndex]
           ,[Suffix]
           ,[IsActive]
           ,[CreatedBy]
           ,[CreatedDate]
           ,[LastModifiedBy]
           ,[LastModifiedDate])
		 SELECT 
            [CompanyCode],[CompanyCode]+'.',[LastIndex],null,[IsActive],@User, @Now, @User, @Now
        FROM @InputData;


		/* Insert Box Statuses history:
			1- If status = RECEIVED => 
				Insert 2 rows: 	1/ status='SENT' 2/ status='RECEIVED'
			2- If status = DESTROYED
				Insert 3 rows:  1/ status='SENT' 2/ status='RECEIVED' 3/ status='DESTROYED' */ 
		INSERT INTO [dbo].[t_ContainerStatus]
           ([ContainerId]
           ,[StatusCode]
           ,[HoldingEntityCode]
           ,[isCurrentStatus]
           ,[CreatedBy]
           ,[CreatedDate]
           ,[LastModifiedBy]
           ,[LastModifiedDate])
		  SELECT 
            t_Container.Id,'SENT','WH',0,[BoxSentBy], [BoxSentDate], [BoxSentBy], [BoxSentDate]
        FROM @InputData;


        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;

        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrSeverity INT = ERROR_SEVERITY();
        RAISERROR (@ErrMsg, @ErrSeverity, 1);
    END CATCH
END;
