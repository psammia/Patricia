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
