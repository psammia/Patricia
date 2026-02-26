-------------
sql sp
---------------
USE [Alterna.Archive]
GO
/****** Object:  StoredProcedure [dbo].[usp_Insert_Into_All_Tables]    Script Date: 26/02/2026 1:13:41 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Patricia Sammia>
-- Create date: <2025/10/30>
-- Description:	<Insert Data of Old Boxes into all tables due to excel file upload>
-- =============================================
ALTER   PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250)
AS 
BEGIN 
    SET NOCOUNT ON;
	SELECT 1;  
    DECLARE @Now DATETIME = GETDATE(); 
    DECLARE @SystemUser NVARCHAR(250) = 'AlternaSystem'; 

    BEGIN TRY 
        BEGIN TRANSACTION; 

        -- Insert new Company (only if Code doesn't already exist)
        INSERT INTO [dbo].[t_Company] 
        ([Code],[CompanyName],[NameAddress],[Mnemonic],[DisplayDescription],[isBranch],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[CompanyName],[CompanyName],[Mnemonic],[Code],0,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Company] comp 
            WHERE comp.[Code] = input.[Code]
        );

        -- Temp tables to hold inserted IDs and link them back to input data 
        DECLARE @InsertedContainers TABLE( 
            RowId INT, 
            ContainerId INT 
        ); 

        DECLARE @InsertedFiles TABLE( 
            RowId INT, 
            FileId INT 
        ); 

        -- Insert Containers with unique Code + CompanyCode combination
        WITH UniqueContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate,
                   ROW_NUMBER() OVER (PARTITION BY BoxRef, Code ORDER BY RowId) as rn
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[t_Container] cont
                WHERE cont.[Code] = input.[BoxRef]
                AND cont.[CompanyCode] = input.[Code]
            )
        ),
        ContainerSource AS (
            SELECT RowId, BoxRef, Code, CompanyName, StatusCode, BoxSentDate
            FROM UniqueContainerSource
            WHERE rn = 1
        )
        MERGE [dbo].[t_Container] AS target
        USING ContainerSource AS source ON 1=0
        WHEN NOT MATCHED THEN
            INSERT ([Code],[CompanyCode],[Entity],[CurrentLocation],[StatusCode],[ArchivingDate],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate],[isNotified])
            VALUES (source.BoxRef, source.Code, source.CompanyName, '', source.StatusCode, source.BoxSentDate, 0, @SystemUser, @Now, @SystemUser, @Now, 1)
        OUTPUT source.RowId, inserted.Id
        INTO @InsertedContainers(RowId, ContainerId);

        -- Capture existing container IDs for ALL rows (including duplicates within input)
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, cont.Id
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[t_Container] cont ON cont.[Code] = input.[BoxRef]
            AND cont.[CompanyCode] = input.[Code]
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- For input rows with duplicate BoxRef + CompanyCode, map them to the inserted container
        INSERT INTO @InsertedContainers(RowId, ContainerId)
        SELECT input.RowId, ic.ContainerId
        FROM @P__Old_Boxes input
        INNER JOIN @InsertedContainers ic ON EXISTS (
            SELECT 1 FROM @P__Old_Boxes input2 
            WHERE input2.RowId = ic.RowId 
            AND input2.BoxRef = input.BoxRef
            AND input2.Code = input.Code
        )
        WHERE input.RowId NOT IN (SELECT RowId FROM @InsertedContainers);

        -- Insert new File Type with auto-incrementing FileTypeCode
        DECLARE @NewFileTypes TABLE (
            Description NVARCHAR(250),
            Entity NVARCHAR(50),
            ArchivingPeriod INT,
            NextCode INT
        );
        
        DECLARE @MaxFileTypeCode INT;
        SELECT @MaxFileTypeCode = ISNULL(MAX(CAST(Code AS INT)), 0) 
        FROM [dbo].[lkp_FileType] 
        WHERE ISNUMERIC(Code) = 1;
        
        WITH UniqueNewFileTypes AS (
            SELECT DISTINCT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   input.[ArchivingPeriod]
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               @SystemUser,
               @Now,
               @SystemUser,
               @Now
        FROM @NewFileTypes nft;

        -- Get the FileTypeCode for each file
        DECLARE @FileTypeCodes TABLE (
            RowId INT,
            FileTypeCode NVARCHAR(50)
        );
        
        INSERT INTO @FileTypeCodes (RowId, FileTypeCode)
        SELECT input.RowId, ft.Code
        FROM @P__Old_Boxes input
        INNER JOIN [dbo].[lkp_FileType] ft ON ft.[Entity] = input.[Code] 
            AND ft.[Description] = input.[FileName];

        -- Insert Files
        INSERT INTO [dbo].[t_File] 
        ([CustomerId],[Name],[FileTypeCode],[StatusCode],[CompanyCode],[FromDate],[ToDate],[AdditionalInfo],[isDeleted],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate])
        OUTPUT inserted.Id
        INTO @InsertedFiles(FileId)
        SELECT null, 
               input.FileName, 
               ftc.FileTypeCode, 
               'FINAL', 
               input.Code, 
               null, 
               null, 
               input.AdditionalInfo, 
               0, 
               @SystemUser, 
               @Now, 
               @SystemUser, 
               @Now
        FROM @P__Old_Boxes input
        INNER JOIN @FileTypeCodes ftc ON ftc.RowId = input.RowId
        ORDER BY input.RowId;

        -- Map the inserted FileIds back to RowIds
        DECLARE @RowIdMapping TABLE (
            RowId INT,
            FileId INT,
            RowNum INT
        );

        INSERT INTO @RowIdMapping (RowId, RowNum)
        SELECT RowId, ROW_NUMBER() OVER (ORDER BY RowId) as RowNum
        FROM @P__Old_Boxes;

        WITH NumberedInsertedFiles AS (
            SELECT FileId, ROW_NUMBER() OVER (ORDER BY FileId) as RowNum
            FROM @InsertedFiles
        )
        UPDATE @InsertedFiles
        SET RowId = rm.RowId
        FROM @InsertedFiles if_target
        INNER JOIN NumberedInsertedFiles nif ON if_target.FileId = nif.FileId
        INNER JOIN @RowIdMapping rm ON nif.RowNum = rm.RowNum;

        -- Insert Container File Relationship
        INSERT INTO [dbo].[t_CurrentContainerFileRelationship] 
        ([FileId],[ContainerId],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT f.FileId, c.ContainerId, @SystemUser, @Now, @SystemUser, @Now
        FROM @InsertedFiles f
        INNER JOIN @InsertedContainers c ON f.RowId = c.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_CurrentContainerFileRelationship] rel
            WHERE rel.[FileId] = f.FileId AND rel.[ContainerId] = c.ContainerId
        );

        -- Insert new Sequence only if Owner doesn't already exist
        INSERT INTO [dbo].[t_Sequence] 
        ([Owner],[Prefix],[LastIndex],[Suffix],[IsActive],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT [Code],[Code]+'.',[LastIndex],null,[IsActive],@SystemUser,@Now,@SystemUser,@Now 
        FROM @P__Old_Boxes input
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_Sequence] seq
            WHERE seq.[Owner] = input.[Code]
        )
        AND input.[Code] NOT IN (
            SELECT i2.[Code] 
            FROM @P__Old_Boxes i2 
            WHERE i2.RowId < input.RowId
        );

        -- ========================================
        -- MODIFIED: Insert Container Status History
        -- ========================================
        
        -- 1. Insert SENT status for all containers
        -- HoldingEntityCode = Code from input (the branch/entity code)
        -- User = BoxSentBy if not empty, otherwise AlternaSystem
        -- Always inactive (isCurrentStatus = 0)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'SENT',
               i.[Code],  -- CHANGED: Use Code instead of 'WH'
               0,         -- Always inactive for SENT
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,  -- Use BoxSentBy or AlternaSystem
               i.[BoxSentDate],
               CASE 
                   WHEN LTRIM(RTRIM(ISNULL(i.[BoxSentBy], ''))) = '' THEN @SystemUser 
                   ELSE i.[BoxSentBy] 
               END,
               i.[BoxSentDate]
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'SENT'
        );

        -- 2. Insert RECEIVED status for containers with StatusCode RECEIVED, DESTROYED, or NOTFOUND
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Active only if final status is RECEIVED
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'RECEIVED',
               'WH',  -- CHANGED: Always 'WH' for RECEIVED
               CASE WHEN i.[StatusCode] = 'RECEIVED' THEN 1 ELSE 0 END,
               @SystemUser,  -- Always AlternaSystem for RECEIVED
               DATEADD(MINUTE, 1, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 1, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] IN ('RECEIVED', 'DESTROYED', 'NOTFOUND')  -- ADDED: NOTFOUND
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'RECEIVED'
        );

        -- 3. Insert DESTROYED status for containers with StatusCode DESTROYED only
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Always active (final status)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'DESTROYED',
               'WH',  -- Always 'WH' for DESTROYED
               1,     -- Always active as final status
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'DESTROYED'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'DESTROYED'
        );

        -- 4. NEW: Insert NOTFOUND status for containers with StatusCode NOTFOUND
        -- HoldingEntityCode = 'WH'
        -- User = AlternaSystem always
        -- Always active (final status)
        INSERT INTO [dbo].[t_ContainerStatus] 
        ([ContainerId],[StatusCode],[HoldingEntityCode],[isCurrentStatus],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT DISTINCT 
               c.ContainerId,
               'NOTFOUND',
               'WH',  -- Always 'WH' for NOTFOUND
               1,     -- Always active as final status
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate]),
               @SystemUser,
               DATEADD(MINUTE, 2, i.[BoxSentDate])
        FROM @InsertedContainers c
        INNER JOIN @P__Old_Boxes i ON c.RowId = i.RowId
        WHERE i.[StatusCode] = 'NOTFOUND'
        AND NOT EXISTS (
            SELECT 1 FROM [dbo].[t_ContainerStatus] cs
            WHERE cs.[ContainerId] = c.ContainerId 
            AND cs.[StatusCode] = 'NOTFOUND'
        );

        COMMIT TRANSACTION; 
    END TRY 
    BEGIN CATCH 
        ROLLBACK TRANSACTION; 
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE(); 
        DECLARE @ErrSeverity INT = ERROR_SEVERITY(); 
        RAISERROR (@ErrMsg, @ErrSeverity, 1); 
    END CATCH 
END;
---------------
tvp
--------------
CREATE TYPE [dbo].[TVP_Old_Boxes] AS TABLE(
	[RowId] [int] NOT NULL,
	[Code] [nvarchar](11) NULL,
	[CompanyName] [nvarchar](22) NULL,
	[Mnemonic] [nvarchar](50) NULL,
	[IsActive] [bit] NULL,
	[BoxRef] [nvarchar](50) NULL,
	[FileName] [nvarchar](250) NULL,
	[AdditionalInfo] [nvarchar](1000) NULL,
	[StatusCode] [nvarchar](10) NULL,
	[ArchivingPeriod] [int] NULL,
	[BoxSentBy] [nvarchar](250) NULL,
	[BoxSentDate] [datetime] NULL,
	[LastIndex] [bigint] NULL
)
GO


------------------------
Front-End Controller
Configuration.cs
-----------------
        #region Entity Old Boxes Archiving
        public ActionResult OldBoxesArchive()
        {
            return View();
        }
        [HttpPost]
        public ActionResult ImportOldBoxesArchive(IFormFile excelFile)
        {
            string correlationId = Guid.NewGuid().ToString();

            try
            {
                //if (!ExcelValidationService(excelFile, out string message))
                //{
                //    return Json(new { isSuccess = false, message = message, correlationId = correlationId });
                //}
                // Configure validation for OldBoxes Excel file
                var validationConfig = new ExcelValidationConfig
                {
                    WorksheetIndex = 1,
                    HeaderRow = 1,
                    DataStartRow = 2,
                    CaseSensitiveHeaders = false,
                    MinimumDataRows = 1,
                    ExpectedHeaders = new List<ColumnConfig>
                {
                    new ColumnConfig
                    {
                        Name = "Code",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        MaxLength = 11
                    },
                    new ColumnConfig
                    {
                        Name = "CompanyName",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        MaxLength = 22
                    },
                    new ColumnConfig
                    {
                        Name = "IsActive",
                        DataType = CellDataType.Number,
                        IsRequired = true,
                        MinValue = 0,
                        MaxValue = 1,
                        MaxDecimalPlaces = 0
                    },
                    new ColumnConfig
                    {
                        Name = "BoxRef",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        //MustBeUnique = false,
                        MaxLength = 50
                    },
                    new ColumnConfig
                    {
                        Name = "FileName",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        MaxLength = 255
                    },
                    new ColumnConfig
                    {
                        Name = "FileAdditionalInfo",
                        DataType = CellDataType.Text,
                        IsRequired = false,
                        MaxLength = 1000
                    },
                    new ColumnConfig
                    {
                        Name = "BoxStatus",
                        DataType = CellDataType.Text,
                        IsRequired = true,
                        MaxLength = 10
                    },
                    new ColumnConfig
                    {
                        Name = "ArchivingPeriod",
                        DataType = CellDataType.Number,
                        IsRequired = true,
                        MinValue = -1,
                        MaxValue = 99,
                        MaxDecimalPlaces = 0
                    },
                    new ColumnConfig
                    {
                        Name = "BoxSentBy",
                        DataType = CellDataType.Text,
                        IsRequired = false,
                        MaxLength = 250
                    },
                    new ColumnConfig
                    {
                        Name = "BoxSentDate",
                        DataType = CellDataType.Date,
                        IsRequired = true,
                        DisallowFutureDates = true
                    },
                    new ColumnConfig
                    {
                        Name = "LastIndex",
                        DataType = CellDataType.Number,
                        IsRequired = true,
                        MinValue = 0,
                        MaxDecimalPlaces = 0
                    },
                    new ColumnConfig
                    {
                        Name = "CanBeUsed",
                        DataType = CellDataType.Number,
                        IsRequired = true,
                        MinValue = 0,
                        MaxValue = 1,
                        MaxDecimalPlaces = 0
                    }
                }
                };

                // Use the injected service -- Validate the file
                var validationResult = _validationService.ValidateExcelFile(excelFile, validationConfig);

                if (!validationResult.IsValid)
                {
                    var errorMessages = string.Join("\n", validationResult.Errors.Select(e => $"{e.Location}: {e.Message}"));
                    return Json(new
                    {
                        isSuccess = false,
                        message = $"File validation failed:\n{errorMessages}",
                        correlationId = correlationId
                    });
                }

                // IMPORTANT: Copy file to memory stream to allow re-reading
                byte[] fileBytes;
                using (var memoryStream = new MemoryStream())
                {
                    excelFile.CopyTo(memoryStream);
                    fileBytes = memoryStream.ToArray();
                }

                // Create new IFormFile from bytes for GetOldBoxes
                var reReadableFile = new FormFile(
                    new MemoryStream(fileBytes),
                    0,
                    fileBytes.Length,
                    excelFile.Name,
                    excelFile.FileName)
                {
                    Headers = excelFile.Headers,
                    ContentType = excelFile.ContentType
                };

                // Now read the data
                ImportOldBoxesReq req = new ImportOldBoxesReq()
                {
                    BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")),
                    OldBoxesList = GetOldBoxes(reReadableFile)
                };

                correlationId = req.BaseReq.CorrelationId;

                ImportOldBoxesRes resp = Common.ApiCall<ImportOldBoxesRes>(req, "ImportOldBoxes");

                if (resp.WebResp.HttpResponseCode != HttpStatusCode.OK)
                {
                    return Json(new
                    {
                        isSuccess = false,
                        message = resp.WebResp.ResponseMessage,
                        correlationId = req.BaseReq.CorrelationId
                    });
                }

                return Json(new
                {
                    isSuccess = true,
                    message = "File has been successfully processed!"
                });
            }
            catch (Exception ex)
            {
                Dictionary<string, object> dic = Common.GenerateVariables(HttpContext, GetSession("ArchiveData")!);

                LogError(ex.Message, new CorrelationInfo()
                {
                    CorrelationId = correlationId,
                    UserName = dic["user"].ToString(),
                    RequestURL = "File/Import",
                    StatusCode = HttpStatusCode.InternalServerError,
                    RDirection = RequestDirection.Response,
                    RType = RequestType.POST,
                }, ex);

                return Json(new
                {
                    isSuccess = false,
                    message = ex.Message, // Changed from ex.ToString() for cleaner message
                    correlationId = correlationId
                });
            }
        }
        private List<OldBox> GetOldBoxes(IFormFile file)
        {
            List<OldBox> result = new List<OldBox>();

            using (Stream stream = file.OpenReadStream())
            {
                using (XLWorkbook workbook = new XLWorkbook(stream))
                {
                    IXLWorksheet workSheet = workbook.Worksheets.First();

                    Dictionary<string, int> headers = workSheet.Row(1).Cells()
                        .Where(c => !string.IsNullOrWhiteSpace(c.GetString()))
                        .ToDictionary(c => c.GetString().Trim(), c => c.Address.ColumnNumber);

                    string[] requiredColumns = new[]
                    {
                    "Code",
                    "CompanyName",
                    "IsActive",
                    "BoxRef",
                    "FileName",
                    "FileAdditionalInfo",
                    "BoxStatus",
                    "ArchivingPeriod",
                    "BoxSentBy",
                    "BoxSentDate",
                    "LastIndex",
                    "CanBeUsed"
                };

                    foreach (string col in requiredColumns)
                    {
                        if (!headers.ContainsKey(col))
                        {
                            throw new Exception($"Missing required column: [{col}]");
                        }
                    }

                    foreach (IXLRow row in workSheet.RowsUsed().Skip(1))
                    {
                        OldBox record = new OldBox()
                        {
                            Code = row.Cell(headers["Code"]).GetString().Trim(),
                            CompanyName = row.Cell(headers["CompanyName"]).GetString().Trim(),
                            IsActive = int.Parse(row.Cell(headers["IsActive"]).GetString().Trim()),
                            BoxRef = row.Cell(headers["BoxRef"]).GetString().Trim(),
                            FileName = row.Cell(headers["FileName"]).GetString().Trim(),
                            FileAdditionalInfo = row.Cell(headers["FileAdditionalInfo"]).GetString().Trim(),
                            BoxStatus = row.Cell(headers["BoxStatus"]).GetString().Trim(),
                            ArchivingPeriod = int.Parse(row.Cell(headers["ArchivingPeriod"]).GetString().Trim()),
                            BoxSentBy = row.Cell(headers["BoxSentBy"]).GetString().Trim(),
                            BoxSentDate = row.Cell(headers["BoxSentDate"]).GetString().Trim(),
                            LastIndex = long.Parse(row.Cell(headers["LastIndex"]).GetString().Trim()),
                            CanBeUsed = int.Parse(row.Cell(headers["CanBeUsed"]).GetString().Trim())
                        };

                        result.Add(record);
                    }
                }
            }

            return result;
        }
        #endregion

---------------
BLL.cs
---------------
        #region ImportEntityOldBoxes
        public void ImportOldBoxes(ImportOldBoxesReq req)
        {
            DAL.DAL _dal = new();

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__Old_Boxes", GetOldBoxesDt(req.OldBoxesList).AsTableValuedParameter());
            parameters.Add("P__User", req.BaseReq.CurrentUser);

            _dal.ExecuteQuery<Container>(
                "usp_Insert_Into_All_Tables",
                parameters,
                CommandType.StoredProcedure,
               CommandDirection.Update);
        }

        public DataTable GetOldBoxesDt(List<OldBoxDto> oldBoxes)
        {
            int rowNbr = 1;

            DataTable dt = new DataTable("TVP_Old_Boxes");

            dt.Columns.Add("RowId");
            dt.Columns.Add("Code");
            dt.Columns.Add("CompanyName");
            dt.Columns.Add("Mnemonic");
            dt.Columns.Add("IsActive");
            dt.Columns.Add("BoxRef");
            dt.Columns.Add("FileName");
            dt.Columns.Add("AdditionalInfo");
            dt.Columns.Add("StatusCode");
            dt.Columns.Add("ArchivingPeriod");
            dt.Columns.Add("BoxSentBy");
            dt.Columns.Add("BoxSentDate");
            dt.Columns.Add("LastIndex");
            dt.Columns.Add("CanBeUsed");

            oldBoxes.ForEach(bx =>
            {
                DataRow dr = dt.NewRow();
                dr["RowId"] = rowNbr;
                dr["Code"] = bx.Code?.Trim();
                dr["CompanyName"] = bx.CompanyName?.Trim();
                dr["Mnemonic"] = bx.CompanyName?.Trim();
                dr["IsActive"] = bx.IsActive;
                dr["BoxRef"] = bx.BoxRef.Trim();
                dr["FileName"] = bx.FileName.Trim();
                dr["AdditionalInfo"] = bx.FileAdditionalInfo?.Trim();
                dr["StatusCode"] = bx.BoxStatus?.Trim();
                dr["ArchivingPeriod"] = bx.ArchivingPeriod;
                dr["BoxSentBy"] = bx.BoxSentBy?.Trim();
                dr["BoxSentDate"] = bx.BoxSentDate?.Trim();
                dr["LastIndex"] = bx.LastIndex;
                dr["CanBeUsed"] = bx.CanBeUsed;

                dt.Rows.Add(dr);

                rowNbr++;
            });

            return dt;
        }
        #endregion


------------------------------------
Back-end Controller
Archiving.cs
-----------------------------------
        #region ImportEntityOldBoxes
        [HttpPost]
        [Route("ImportOldBoxes")]
        public ImportOldBoxesRes ImportOldBoxes(ImportOldBoxesReq req)
        {
            string correlationId = Guid.NewGuid().ToString();

            ImportOldBoxesRes response = new ImportOldBoxesRes()
            {
                Req = req,
                WebResp = new BaseResponse()
                {
                    CorrelationId = req.BaseReq.CorrelationId ?? throw new ArgumentNullException(nameof(req.BaseReq.CorrelationId)),
                    HttpResponseCode = HttpStatusCode.OK,
                    ResponseMessage = string.Empty,
                    User = req.BaseReq.CurrentUser ?? throw new ArgumentNullException(nameof(req.BaseReq.CurrentUser)),
                    Branch = req.BaseReq.CurrentBranch ?? throw new ArgumentNullException(nameof(req.BaseReq.CurrentBranch)),
                    Entity = req.BaseReq.CurrentEntity ?? throw new ArgumentNullException(nameof(req.BaseReq.CurrentEntity))
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = req.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ImportOldBoxes",
                UserName = req.BaseReq.CurrentUser
            };

            try
            {
                correlationInfo.Reserved = "ImportOldBoxes has been called with the following Request";
                LogInfoJson(req, correlationInfo);
             
                using (BLL.BLL oBLL = new(req.BaseReq.CurrentUser))
                {
                    oBLL.ImportOldBoxes(req);

                    response.WebResp.CorrelationId = req.BaseReq.CorrelationId;
                    response.WebResp.User = req.BaseReq.CurrentUser;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;
                    correlationInfo.Reserved = "ImportOldBoxes Has Replied with the Following response";
                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfoJson(response, correlationInfo);

                    return response;
                }
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = string.IsNullOrWhiteSpace(req.BaseReq.CorrelationId) ? correlationId : req.BaseReq.CorrelationId;
                response.WebResp.User = !string.IsNullOrWhiteSpace(req.BaseReq.CurrentUser) ? req.BaseReq.CurrentUser : "BadUser";
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;

                correlationInfo.Reserved = ex.Message;
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;
                correlationInfo.RDirection = RequestDirection.Response;

                LogErrorJson(ex, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = string.IsNullOrWhiteSpace(req.BaseReq.CorrelationId) ? correlationId : req.BaseReq.CorrelationId;
                response.WebResp.User = !string.IsNullOrWhiteSpace(req.BaseReq.CurrentUser) ? req.BaseReq.CurrentUser : "BadUser";
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.Message;

                correlationInfo.Reserved = ex.Message;
                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogErrorJson(ex, correlationInfo);

                return response;
            }
        }
        #endregion
