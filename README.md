
CREATE TYPE dbo.MyInputTableType AS TABLE
(
[RowId] INT, -- Used to track each row uniquely

[Code] NVARCHAR(11), -- CompanyCode in t_Company
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

DECLARE @MyInputTableType MyInputTableType;
INSERT INTO @MyInputTableType ([RowId],[Code],[CompanyName],[Mnemonic],[IsActive],[BoxRef],[FileName],[AdditionalInfo],[StatusCode],[ArchivingPeriod],[BoxSentBy],[BoxSentDate],[LastIndex])
VALUES(1,'ET000132','AFFA','AFFA',1,'AFFA.1','File AFFA 1','Test','RECEIVED',5,'','2021-02-21',0),
	  (2,'ET000132','AFFA','AFFA',1,'AFFA.1','File AFFA 2','Test','RECEIVED',5,'','2021-02-21',0),
	  (3,'ET000132','AFFA','AFFA',1,'AFFA.1','File AFFA 3','Test','RECEIVED',5,'','2021-02-21',0),
	  (4,'ET000132','AFFA','AFFA',1,'AFFA.2','File AFFA 2','Test','RECEIVED',10,'','2021-02-22',0),

	  (5,'ET9900111','REES','REES',1,'REES.1','File REES 1','','RECEIVED',-1,'Clababidi','2021-02-21',65),
	  (6,'ET9900111','REES','REES',1,'REES.1','Contrats baux originaux','','RECEIVED',-1,'Clababidi','2021-02-21',65),
	  (7,'ET9900111','REES','REES',1,'REES.1','File REES 3','','RECEIVED',-1,'Clababidi','2021-02-21',65)
SELECT * FROM @MyInputTableType

CREATE or ALTER PROCEDURE dbo.InsertIntoAllTables
    @MyInputTableType dbo.MyInputTableType READONLY
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @User NVARCHAR(50) = 'AlternaSystem';
    DECLARE @Now DATETIME = GETDATE();
	--DECLARE @CompanyCode NVARCHAR(11)= 'ET000132';
	DECLARE @FileTypeCode NVARCHAR(50) = '205';

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
            [Code],[CompanyName],[CompanyName], [Mnemonic], null, 0, [IsActive], @User, @Now, @User, @Now
        FROM @MyInputTableType
		
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
			i.BoxRef, i.Code, i.CompanyName, '', i.StatusCode, i.BoxSentDate, 0,@User, @Now, @User, @Now, 1
		FROM @MyInputTableType i;
