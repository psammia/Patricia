@using Alterna.Archive.Core.Models.TableModel
@using Alterna.Archive.Core.Models
@model FileTypeModel

<table id="example" class="table table-striped table-bordered" style="width:100%">
    <thead>
        <tr>
            <th>Action</th>
            <th>Code Name</th>
            <th>Entity</th>
            <th>Description</th>
            <th>IsBranch</th>
            <th>IsCustomer</th>
            <th>ArchivingPeriod</th>
            <th>CanBeUsed</th>
        </tr>
    </thead>
    <tbody>
        @if (Model.FileTypeList.Count > 0)
        {
            StaticFileTypeModel.FileTypeList = Model.FileTypeList;
            for (int i = 0; i < Model.FileTypeList.Count; i++)
            {
                String entityTableTdId = "EntityTableTd" + Model.FileTypeList[i].Id;
                String hiddenClass = !Model.FileTypeList[i].IsBranch && !Model.FileTypeList[i].IsCustomer ? "" : "hidden";

                <tr>
                    <td>
                        <div style="text-align: center; cursor:pointer" onclick="editRow(this,@Model.FileTypeList[i].Id,@i)">
                            <span class="fa-regular fa-pen-to-square" title="Edit Details"></span>
                        </div>
                    </td>
                    @{
                        String CodeName = $"{Model.FileTypeList[i].Code} - {Model.FileTypeList[i].Description}";
                        String tdId = $"Code-{Model.FileTypeList[i].Id.ToString()}";
                    }

                    <td id="@tdId">@CodeName</td>

                    <td>
                        @Html.DropDownListFor(model => model.FileTypeList[i].Entity, @Model.EntityList, @Model.FileTypeList[i].Entity, new { @id = "Entity" + @Model.FileTypeList[i].Id, @disabled = "disabled", @class = hiddenClass })
                    </td>

                    <td>@Html.EditorFor(model => model.FileTypeList[i].Description, new { htmlAttributes = new { @id = "Description" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>

                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsBranch, new { @id = "IsBranch" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsCustomer, new { @id = "IsCustomer" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                    <td>@Html.EditorFor(model => model.FileTypeList[i].ArchivingPeriod, new { htmlAttributes = new { @type = "number", @min = "0", @step = "1", @id = "ArchivingPeriod" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>
                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].CanBeUsed, new { @id = "CanBeUsed" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                </tr>
            }
        }
    </tbody>
</table>

<script>
    $(document).ready(() => {
        $("#example").DataTable({
            pagingType: 'full_numbers'
        });
    });

    function editRow(element, id, index) {
        let row = $(element).closest('tr');
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        var prevData = {
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked  // NEW
        };
        sessionStorage.setItem('Ftt' + id, JSON.stringify(prevData))
        row.find('td:first-child').html('<div style="text-align: center;"><span style="cursor:pointer; color:green" class="fa-regular fa-floppy-disk" onclick="saveRow(this,' + id + ',' + index + ')" title="Save Details"></span>&nbsp;&nbsp;<span style="cursor:pointer; color:red" class="fa-solid fa-xmark" onclick="stopEdit(this,' + id + ')" title="Cancel"></span></div>');
        Fields_Switch(id, false)
    }

    function stopEdit(element, id) {
        var data = sessionStorage.getItem('Ftt' + id);
        var prevData = JSON.parse(data);
        let row = $(element).closest('tr');
        row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
        Fields_Switch(id, true)
        if (!prevData.IsCustomer && !prevData.IsBranch) {
            AddEntitySelectElementToDataTable(id);
        } else {
            RemoveEntitySelectElementToDataTable(id);
        }
        $('#Entity' + id)[0].value = prevData.Entity;
        $('#Description' + id)[0].value = prevData.Description;
        $('#IsBranch' + id)[0].checked = prevData.IsBranch,
        $('#IsCustomer' + id)[0].checked = prevData.IsCustomer;
        $('#ArchivingPeriod' + id)[0].value = prevData.ArchivingPeriod;
        $('#CanBeUsed' + id)[0].checked = prevData.CanBeUsed;  // NEW
    }

    function saveRow(element, id, Aindex) {
        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        let code = $("#Code-" + id).html();
        let match = code.match(/\d+/);
        if (match) {
            code = match[0];
        }
        var dat = {
            ModelId: id,
            code: code,
            Entity: entity,
            Description: $('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer: $('#IsCustomer' + id)[0].checked,
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value,
            CanBeUsed: $('#CanBeUsed' + id)[0].checked  // NEW
        };
        $.ajax({
            type: 'POST',
            url: '/Configuration/UpdateFileType/',
            data: dat,
            dataType: 'html',
            success: function (data) {
                let row = $(element).closest('tr');
                row.find('td:first-child').html('<div style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div>');
                Fields_Switch(id, true)
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }
        });
    }

    function Fields_Switch(id, x) {
        $('#Description' + id).prop('disabled', x);
        $('#IsCustomer' + id).prop('disabled', x);
        $('#ArchivingPeriod' + id).prop('disabled', x);
        $('#IsBranch' + id).prop('disabled', x);
        $('#Entity' + id).prop('disabled', x);
        $('#CanBeUsed' + id).prop('disabled', x);  // NEW
    }

    $('*[id*=IsBranch]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if (!$("#IsBranch" + index)[0].checked) {
            $('#IsCustomer' + index).prop('checked', false).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    $('*[id*=IsCustomer]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);
        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        } else {
            return;
        }
        if ($("#IsCustomer" + index)[0].checked) {
            $('#IsBranch' + index).prop('checked', true).change();
        }
        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        } else {
            RemoveEntitySelectElementToDataTable(index);
        }
    });

    function AddEntitySelectElementToDataTable(index) {
        $("#Entity" + index).removeClass("hidden");
    }

    function RemoveEntitySelectElementToDataTable(index) {
        $("#Entity" + index).addClass("hidden");
    }
</script>


===================
Configuration Controller
====================
[HttpPost]
public ActionResult UpdateFileType(int ModelId, string code, string Entity, string Description, bool IsBranch, bool IsCustomer, int ArchivingPeriod, bool CanBeUsed)
{
    try
    {
        String session = GetSession("ArchiveData");
        
        UpdateFileTypeReq request = new UpdateFileTypeReq()
        {
            BaseReq = new BaseRequest(HttpContext, session, false),
            FileTypeId = ModelId,
            Code = code,
            Entity = Entity,
            Description = Description,
            IsBranch = IsBranch,
            IsCustomer = IsCustomer,
            ArchivingPeriod = ArchivingPeriod,
            CanBeUsed = CanBeUsed  // NEW
        };

        UpdateFileTypeRes response = Common.ApiCall<UpdateFileTypeRes>(request, "UpdateFileType");

        if (response.WebResp.HttpResponseCode == HttpStatusCode.OK)
        {
            return Json(new { success = true, message = "File Type updated successfully" });
        }
        else
        {
            return Json(new { success = false, message = response.WebResp.ResponseMessage });
        }
    }
    catch (Exception ex)
    {
        return Json(new { success = false, message = ex.Message });
    }
}

=========================
Archiving Controller
=====================
#region UpdateFileType Controller
[HttpPost]
[Route("UpdateFileType")]
public UpdateFileTypeRes UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
{
    UpdateFileTypeRes response = new()
    {
        Req = updateFileTypeReq
    };

    CorrelationInfo correlationInfo = new()
    {
        CorrelationId = updateFileTypeReq.BaseReq.CorrelationId,
        RDirection = RequestDirection.Request,
        RequestURL = "UpdateFileType",
        UserName = updateFileTypeReq.BaseReq.CurrentUser
    };

    try
    {
        String CorrelationId = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : updateFileTypeReq.BaseReq.CorrelationId;
        String CurrentUser = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : updateFileTypeReq.BaseReq.CurrentUser;
        String CurrentEntity = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : updateFileTypeReq.BaseReq.CurrentEntity;
        String CurrentBranch = String.IsNullOrEmpty(updateFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : updateFileTypeReq.BaseReq.CurrentBranch;

        LogInfo("UpdateFileType Has been called with the following Request", correlationInfo);
        LogInfoJson(updateFileTypeReq, correlationInfo);

        correlationInfo.RDirection = RequestDirection.Processing;

        #region Data Guard Check
        using (BLL.BLL oBLL = new(CurrentUser))
        {
            LogInfo("Data guard checks have started", correlationInfo);

            Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
            {
                { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(updateFileTypeReq) },
            };

            oBLL.DataIntegrityCheck(DataGuardDictionnary);

            LogInfo("Data guard check successful", correlationInfo);

            LogInfo("Start of UpdateFileType call", correlationInfo);

            response.Success = oBLL.UpdateFileType(updateFileTypeReq);

            if (!response.Success)
            {
                throw new SGBLInternalServerException($"Failed to update File Type");
            }

            response.WebResp.CorrelationId = CorrelationId;
            response.WebResp.User = CurrentUser;
            response.WebResp.Entity = CurrentEntity;
            response.WebResp.Branch = CurrentBranch;
            response.WebResp.HttpResponseCode = HttpStatusCode.OK;

            correlationInfo.RDirection = RequestDirection.Response;

            LogInfo("UpdateFileType Has Replied with the Following response", correlationInfo);
            LogInfoJson(response, correlationInfo);
            LogInfo("Calling the UpdateFileType is completed", correlationInfo);
        }

        return response;
        #endregion
    }
    catch (SGBLBadRequestException ex)
    {
        response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : updateFileTypeReq.BaseReq.CorrelationId!;
        response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : updateFileTypeReq.BaseReq.CurrentUser!;
        response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : updateFileTypeReq.BaseReq.CurrentEntity!;
        response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : updateFileTypeReq.BaseReq.CurrentBranch!;
        response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
        response.WebResp.ResponseMessage = ex.StackTrace;
        response.Success = false;

        correlationInfo.CorrelationId = response.WebResp.CorrelationId;
        correlationInfo.UserName = response.WebResp.User;
        correlationInfo.StatusCode = HttpStatusCode.BadRequest;
        correlationInfo.RDirection = RequestDirection.Response;

        LogError(ex.Message, correlationInfo, ex);
        LogErrorJson(response, correlationInfo, ex);

        return response;
    }
    catch (Exception ex)
    {
        response.WebResp.CorrelationId = updateFileTypeReq.BaseReq.CorrelationId!;
        response.WebResp.User = updateFileTypeReq.BaseReq.CurrentUser!;
        response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
        response.WebResp.ResponseMessage = ex.StackTrace;
        response.Success = false;

        correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
        correlationInfo.RDirection = RequestDirection.Response;

        LogError(ex.StackTrace, correlationInfo);
        LogErrorJson(response, correlationInfo, ex);

        return response;
    }
}
#endregion

BLL.cs
==========
#region UpdateFileType

public bool UpdateFileType(UpdateFileTypeReq updateFileTypeReq)
{
    DAL.DAL iDAL = new();
    bool result = false;
    
    OnPreEventUpdateFileType?.Invoke(ref updateFileTypeReq);

    Dictionary<string, object> parameters = new Dictionary<string, object>
    {
        { "@P__Id", updateFileTypeReq.FileTypeId },
        { "@P__Code", updateFileTypeReq.Code },
        { "@P__Entity", updateFileTypeReq.Entity },
        { "@P__Description", updateFileTypeReq.Description },
        { "@P__Category", updateFileTypeReq.IsBranch ? "Branch" : "Not Branch" },
        { "@P__IsCustomer", updateFileTypeReq.IsCustomer },
        { "@P__ArchivingPeriod", updateFileTypeReq.ArchivingPeriod },
        { "@P__CanBeUsed", updateFileTypeReq.CanBeUsed },
        { "@P__User", this.CurrentUser }
    };

    int rowsAffected = iDAL.ExecuteNonQuery("usp_UpdateFileType", parameters, CommandType.StoredProcedure);
    result = rowsAffected > 0;
    
    OnPostEventUpdateFileType?.Invoke(ref result, ref updateFileTypeReq);
    
    return result;
}

#endregion

SQL SP
========
ALTER PROCEDURE [dbo].[usp_Insert_Into_All_Tables] 
	@P__Old_Boxes [dbo].[TVP_Old_Boxes] READONLY,
	@P__User NVARCHAR(250),
	@P__CanBeUsed BIT = 1  -- NEW PARAMETER with default value of 1
AS 
BEGIN 
    SET NOCOUNT ON
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
        
        -- Get distinct Entity+Description combinations, taking the first ArchivingPeriod encountered
        WITH UniqueNewFileTypes AS (
            SELECT 
                   input.[FileName] as Description,
                   input.[Code] as Entity,
                   MIN(input.[ArchivingPeriod]) as ArchivingPeriod
            FROM @P__Old_Boxes input
            WHERE NOT EXISTS (
                SELECT 1 FROM [dbo].[lkp_FileType] ft
                WHERE ft.[Description] = input.[FileName]
                AND ft.[Entity] = input.[Code]
            )
            GROUP BY input.[FileName], input.[Code]
        )
        INSERT INTO @NewFileTypes (Description, Entity, ArchivingPeriod, NextCode)
        SELECT Description, 
               Entity, 
               ArchivingPeriod,
               @MaxFileTypeCode + ROW_NUMBER() OVER (ORDER BY Entity, Description) as NextCode
        FROM UniqueNewFileTypes;

        INSERT INTO [dbo].[lkp_FileType] 
        ([Code],[Entity],[Category],[Description],[HasDate],[IsCustomer],[ArchivingPeriod],[CanBeUsed],[CreatedBy],[CreatedDate],[LastModifiedBy],[LastModifiedDate]) 
        SELECT CAST(nft.NextCode AS NVARCHAR(50)) as Code,
               nft.Entity,
               'Not Branch' as Category,
               nft.Description,
               0 as HasDate,
               0 as IsCustomer,
               nft.ArchivingPeriod,
               @P__CanBeUsed,  -- MODIFIED: Use parameter value instead of hardcoded 1
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
