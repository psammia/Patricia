_FileTypeManagementTable.cshtml (Partial View)
===============================
@using Alterna.Archive.Core.Models.TableModel
@using Alterna.Archive.Core.Models
@model FileTypeModel


<table id="example" class="table table-striped table-bordered">
    <thead>
        <tr>
            <th></th>
            <th>Code Name</th>
            <th>Entity</th>
            <th>Description</th>
            <th>IsBranch</th>
            <th>IsCustomer</th>
            <th>ArchivingPeriod</th>
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
                    @* <td>@Html.EditorFor(model => model.FileTypeList[i].Code, new { htmlAttributes = new { @id = "Code" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td> *@
                    @{
                        String CodeName = $"{Model.FileTypeList[i].Code} - {Model.FileTypeList[i].Description}";
                        String tdId = $"Code-{Model.FileTypeList[i].Id.ToString()}";
                    }

                    <td id="@tdId">@CodeName</td>

                    <td>
                        @Html.DropDownListFor(model => model.FileTypeList[i].Entity, @Model.EntityList, @Model.FileTypeList[i].Entity, new { @id = "Entity" + @Model.FileTypeList[i].Id, @disabled = "disabled", @class = hiddenClass })
                    </td>
                    
                    <td>@Html.EditorFor(model => model.FileTypeList[i].Description, new { htmlAttributes = new { @id = "Description" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>

                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsBranch, new { @id = "IsBranch" + @Model.FileTypeList[i].Id, @disabled = "disabled" } )</td>
                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsCustomer, new {@id = "IsCustomer" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                    <td>@Html.EditorFor(model => model.FileTypeList[i].ArchivingPeriod, new { htmlAttributes = new { @type = "number", @min = "0", @step = "1", @id = "ArchivingPeriod" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>
                </tr>
            }
        }
    </tbody>
</table>

<script>
    $(document).ready(() => {
        $("#example").DataTable(
            {
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
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value
        };
        sessionStorage.setItem('Ftt'+id,JSON.stringify(prevData))
        row.find('td:first-child').html('<div><div  style="text-align: center; cursor:pointer" onclick="stopEdit(this,' + id + ')"><span class="fa-regular fa-trash-can" Title="Delete Row"></span></div><div  style="text-align: center; cursor:pointer" onclick="saveRow(this,' + id + ',' + index + ')"><span class="fa-solid fa-file-circle-check" Title="Confirm Changes"></span></div ></div> ');

        Fields_Switch(id,false)

    }
    function stopEdit(element, id) {
        var data = sessionStorage.getItem('Ftt' + id);
        var prevData = JSON.parse(data);
        let row = $(element).closest('tr');

        row.find('td:first-child').html('<div  style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div> ');
        Fields_Switch(id,true)

        if (!prevData.IsCustomer && !prevData.IsBranch) {
            AddEntitySelectElementToDataTable(id);
        }
        else {
            RemoveEntitySelectElementToDataTable(id);
        }

        $('#Entity' + id)[0].value = prevData.Entity;
        $('#Description' + id)[0].value = prevData.Description;
        $('#IsBranch' + id)[0].checked = prevData.IsBranch,
        $('#IsCustomer' + id)[0].checked = prevData.IsCustomer;
        $('#ArchivingPeriod' + id)[0].value = prevData.ArchivingPeriod;
    }

    function saveRow(element, id, Aindex) {

        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        let code = $("#Code-" + id).html();
        let match = code.match(/\d+/); // Match one or more digits

        if (match) {
            code = match[0]; // Parse the matched digits as an integer
        }

        var dat = {
            ModelId:id,
            code: code,
            Entity: entity,
            Description:$('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer:$('#IsCustomer' + id)[0].checked,
            ArchivingPeriod:$('#ArchivingPeriod' + id)[0].value
        };
        $.ajax({

            type: 'POST',
            url: '/Configuration/UpdateFileType/',
            data: dat,
            dataType: 'html',
            success: function (data) {
                let row = $(element).closest('tr');

                row.find('td:first-child').html('<div  style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ',' + Aindex + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div> ');
                Fields_Switch(id, true)
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }

        });

    }
    function Fields_Switch(id,x) {
        $('#Description' + id).prop('disabled', x);
        $('#IsCustomer' + id).prop('disabled', x);
        $('#ArchivingPeriod' + id).prop('disabled', x);
        $('#IsBranch' + id).prop('disabled', x);
        $('#Entity' + id).prop('disabled', x);
    }

    $('*[id*=IsBranch]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);

        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        }
        else {
            return;
        }

        if (!$("#IsBranch" + index)[0].checked) {
            $('#IsCustomer' + index).prop('checked', false).change();
        }

        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        }
        else{
            RemoveEntitySelectElementToDataTable(index);
        }

    });

    $('*[id*=IsCustomer]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);

        if(index !== null && index.length > 0){
            index = elementId.match(/\d+/)[0];
        }
        else{
            return;
        }

        if ($("#IsCustomer" + index)[0].checked) {
            $('#IsBranch' + index).prop('checked', true).change();
        }

        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        }
        else{
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
==============================================================

FileTypeManagement.cshtml (Main View)
=========================
@using Alterna.Archive.Core.Models.TableModel
@using Alterna.Archive.Core.Models
@model FileTypeModel


<table id="example" class="table table-striped table-bordered">
    <thead>
        <tr>
            <th></th>
            <th>Code Name</th>
            <th>Entity</th>
            <th>Description</th>
            <th>IsBranch</th>
            <th>IsCustomer</th>
            <th>ArchivingPeriod</th>
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
                    @* <td>@Html.EditorFor(model => model.FileTypeList[i].Code, new { htmlAttributes = new { @id = "Code" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td> *@
                    @{
                        String CodeName = $"{Model.FileTypeList[i].Code} - {Model.FileTypeList[i].Description}";
                        String tdId = $"Code-{Model.FileTypeList[i].Id.ToString()}";
                    }

                    <td id="@tdId">@CodeName</td>

                    <td>
                        @Html.DropDownListFor(model => model.FileTypeList[i].Entity, @Model.EntityList, @Model.FileTypeList[i].Entity, new { @id = "Entity" + @Model.FileTypeList[i].Id, @disabled = "disabled", @class = hiddenClass })
                    </td>
                    
                    <td>@Html.EditorFor(model => model.FileTypeList[i].Description, new { htmlAttributes = new { @id = "Description" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>

                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsBranch, new { @id = "IsBranch" + @Model.FileTypeList[i].Id, @disabled = "disabled" } )</td>
                    <td>@Html.CheckBoxFor(model => model.FileTypeList[i].IsCustomer, new {@id = "IsCustomer" + @Model.FileTypeList[i].Id, @disabled = "disabled" })</td>
                    <td>@Html.EditorFor(model => model.FileTypeList[i].ArchivingPeriod, new { htmlAttributes = new { @type = "number", @min = "0", @step = "1", @id = "ArchivingPeriod" + @Model.FileTypeList[i].Id, @disabled = "disabled" } })</td>
                </tr>
            }
        }
    </tbody>
</table>

<script>
    $(document).ready(() => {
        $("#example").DataTable(
            {
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
            ArchivingPeriod: $('#ArchivingPeriod' + id)[0].value
        };
        sessionStorage.setItem('Ftt'+id,JSON.stringify(prevData))
        row.find('td:first-child').html('<div><div  style="text-align: center; cursor:pointer" onclick="stopEdit(this,' + id + ')"><span class="fa-regular fa-trash-can" Title="Delete Row"></span></div><div  style="text-align: center; cursor:pointer" onclick="saveRow(this,' + id + ',' + index + ')"><span class="fa-solid fa-file-circle-check" Title="Confirm Changes"></span></div ></div> ');

        Fields_Switch(id,false)

    }
    function stopEdit(element, id) {
        var data = sessionStorage.getItem('Ftt' + id);
        var prevData = JSON.parse(data);
        let row = $(element).closest('tr');

        row.find('td:first-child').html('<div  style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div> ');
        Fields_Switch(id,true)

        if (!prevData.IsCustomer && !prevData.IsBranch) {
            AddEntitySelectElementToDataTable(id);
        }
        else {
            RemoveEntitySelectElementToDataTable(id);
        }

        $('#Entity' + id)[0].value = prevData.Entity;
        $('#Description' + id)[0].value = prevData.Description;
        $('#IsBranch' + id)[0].checked = prevData.IsBranch,
        $('#IsCustomer' + id)[0].checked = prevData.IsCustomer;
        $('#ArchivingPeriod' + id)[0].value = prevData.ArchivingPeriod;
    }

    function saveRow(element, id, Aindex) {

        let entity = ($('#IsBranch' + id)[0].checked || $('#IsCustomer' + id)[0].checked) ? '' : $('#Entity' + id).find(":selected").val();
        let code = $("#Code-" + id).html();
        let match = code.match(/\d+/); // Match one or more digits

        if (match) {
            code = match[0]; // Parse the matched digits as an integer
        }

        var dat = {
            ModelId:id,
            code: code,
            Entity: entity,
            Description:$('#Description' + id)[0].value,
            IsBranch: $('#IsBranch' + id)[0].checked,
            IsCustomer:$('#IsCustomer' + id)[0].checked,
            ArchivingPeriod:$('#ArchivingPeriod' + id)[0].value
        };
        $.ajax({

            type: 'POST',
            url: '/Configuration/UpdateFileType/',
            data: dat,
            dataType: 'html',
            success: function (data) {
                let row = $(element).closest('tr');

                row.find('td:first-child').html('<div  style="text-align: center; cursor:pointer" onclick="editRow(this,' + id + ',' + Aindex + ')"><span class="fa-regular fa-pen-to-square" title="Edit Details"></span></div> ');
                Fields_Switch(id, true)
            },
            error: function (xhr) {
                $('#MainRenderLocation').html(xhr.responseText);
            }

        });

    }
    function Fields_Switch(id,x) {
        $('#Description' + id).prop('disabled', x);
        $('#IsCustomer' + id).prop('disabled', x);
        $('#ArchivingPeriod' + id).prop('disabled', x);
        $('#IsBranch' + id).prop('disabled', x);
        $('#Entity' + id).prop('disabled', x);
    }

    $('*[id*=IsBranch]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);

        if (index !== null && index.length > 0) {
            index = elementId.match(/\d+/)[0];
        }
        else {
            return;
        }

        if (!$("#IsBranch" + index)[0].checked) {
            $('#IsCustomer' + index).prop('checked', false).change();
        }

        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        }
        else{
            RemoveEntitySelectElementToDataTable(index);
        }

    });

    $('*[id*=IsCustomer]').on('click', function () {
        let elementId = $(this).attr('id');
        let index = elementId.match(/\d+/);

        if(index !== null && index.length > 0){
            index = elementId.match(/\d+/)[0];
        }
        else{
            return;
        }

        if ($("#IsCustomer" + index)[0].checked) {
            $('#IsBranch' + index).prop('checked', true).change();
        }

        if (!$('#IsCustomer' + index)[0].checked && !$('#IsBranch' + index)[0].checked) {
            AddEntitySelectElementToDataTable(index);
        }
        else{
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
========================================================

ConfigurationController.cs (Front End)
============================
        public ActionResult FileTypeManagement()
        {
            String session = GetSession("ArchiveData");
            FileTypeModel model = new();
            GetActiveCompaniesOfUserRes getActiveCompaniesOfUserRes = Common.ApiCall<GetActiveCompaniesOfUserRes>(new GetActiveCompaniesOfUserReq() { BaseReq = new BaseRequest(HttpContext, session, false) }, "GetActiveCompaniesOfUser");

            foreach (Company comp in getActiveCompaniesOfUserRes.Resp)
            {
                model.EntityList.Add(new SelectListItem { Text = $"{comp.Code} - {comp.Mnemonic}", Value = comp.Code });
            }
            return View(model);
        }

        public ActionResult FileType(FileTypeModel fileTypeModel)
        {
            FileTypeModel model = new();
            GetFileTypeRes resp = new();
            resp = Common.ApiCall<GetFileTypeRes>(new GetEntityReq() { BaseReq = new BaseRequest(HttpContext, GetSession("ArchiveData")) }, "GetAllFileType");
            model.FileTypeList = resp.Resp;
            foreach (FileType ft in model.FileTypeList)
            {
                ft.IsBranch = ft.Category.ToLower().Equals("branch");
            }
            model.EntityList = fileTypeModel.EntityList;
            return PartialView("_FileTypeManagementTable", model);
        }

ArchvingController.cs (BackEnd)
==================================
        #region GetAllFileType Controller
        [HttpPost]
        [Route("GetAllFileType")]
        public GetAllFileTypeRes GetAllFileType(GetAllFileTypeReq GetAllFileTypeReq)
        {
            GetAllFileTypeRes response = new()
            {
                Req = GetAllFileTypeReq
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = GetAllFileTypeReq.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllFileType",
                UserName = GetAllFileTypeReq.BaseReq.CurrentUser
            };

            try
            {
                String CorrelationId = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CorrelationId) ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") : GetAllFileTypeReq.BaseReq.CorrelationId;
                String CurrentUser = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CurrentUser) ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") : GetAllFileTypeReq.BaseReq.CurrentUser;



                String CurrentEntity = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CurrentEntity) ? String.Empty : GetAllFileTypeReq.BaseReq.CurrentEntity;
                String CurrentBranch = String.IsNullOrEmpty(GetAllFileTypeReq.BaseReq.CurrentBranch) ? String.Empty : GetAllFileTypeReq.BaseReq.CurrentBranch;

                LogInfo("GetAllFileType Has been called with the following Request", correlationInfo);
                LogInfoJson(GetAllFileTypeReq, correlationInfo);

                correlationInfo.RDirection = RequestDirection.Processing;

                #region Data Guard Check
                using (BLL.BLL oBLL = new(CurrentUser))
                {
                    LogInfo("Data guard checks have started", correlationInfo);

                    Dictionary<DataIntegrityCheckFunctions, dynamic> DataGuardDictionnary = new()
            {
                { DataIntegrityCheckFunctions.CONTAINS_NULL, JsonConvert.SerializeObject(GetAllFileTypeReq) },
            };

                    oBLL.DataIntegrityCheck(DataGuardDictionnary);

                    LogInfo("Data guard check successful", correlationInfo);

                    LogInfo("Start of GetAllFileType call", correlationInfo);

                    response.Resp = oBLL.GetAllFileType(GetAllFileTypeReq);

                    if (response.Resp == null || response.Resp.Count == 0)
                    {
                        throw new SGBLNotFoundException($"No File Types have been found in our systems");
                    }

                    response.WebResp.CorrelationId = CorrelationId;
                    response.WebResp.User = CurrentUser;
                    response.WebResp.Entity = CurrentEntity;
                    response.WebResp.Branch = CurrentBranch;
                    response.WebResp.HttpResponseCode = HttpStatusCode.OK;

                    correlationInfo.RDirection = RequestDirection.Response;

                    LogInfo("GetAllFileType Has Replied with the Following response", correlationInfo);
                    LogInfoJson(response, correlationInfo);
                    LogInfo("Calling the GetAllFileType is completed", correlationInfo);

                }

                return response;
                #endregion
            }
            catch (SGBLBadRequestException ex)
            {
                response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") ? Guid.NewGuid().ToString() : GetAllFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = ex.Message.Contains("CurrentUser") ? "BadUser" : GetAllFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.Entity = ex.Message.Contains("CurrentEntity") ? "BadEntity" : GetAllFileTypeReq.BaseReq.CurrentEntity!;
                response.WebResp.Branch = ex.Message.Contains("CurrentBranch") ? "BadBranch" : GetAllFileTypeReq.BaseReq.CurrentBranch!;
                response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                //this was added in case correlation Id was invalid(null or Empty)
                correlationInfo.CorrelationId = response.WebResp.CorrelationId;
                //this was added in case Username was invalid(null or Empty)
                correlationInfo.UserName = response.WebResp.User;

                //don't forget to change status code in case of exception
                correlationInfo.StatusCode = HttpStatusCode.BadRequest;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo, ex);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (SGBLNotFoundException ex)
            {
                response.WebResp.CorrelationId = GetAllFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = GetAllFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.NoContent;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.NoContent;
                correlationInfo.RDirection = RequestDirection.Response;

                LogInfo(ex.Message, correlationInfo);

                return response;
            }
            catch (Exception ex)
            {
                response.WebResp.CorrelationId = GetAllFileTypeReq.BaseReq.CorrelationId!;
                response.WebResp.User = GetAllFileTypeReq.BaseReq.CurrentUser!;
                response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
                response.WebResp.ResponseMessage = ex.StackTrace;
                response.Resp = [];

                correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.StackTrace, correlationInfo);
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

BLL.cs
===========
        #region GetAllFileType

        public List<FileType> GetAllFileType(GetAllFileTypeReq getAllFileTypeReq)
        {
            DAL.DAL iDAL = new();
            List<FileType> Retlist = [];
            OnPreEventGetAllFileType?.Invoke(ref getAllFileTypeReq);

            Retlist = iDAL.ExecuteQuery<FileType>("usp_GetAllFileType", null, CommandType.StoredProcedure,
                CommandDirection.Select);
            OnPostEventGetAllFileType?.Invoke(ref Retlist, ref getAllFileTypeReq);
            return Retlist;
        }

        #endregion

===
SQL (Stored Procedure)
====
  ALTER PROCEDURE [dbo].[usp_GetAllFileType] AS BEGIN
SELECT
  Id,
  Code,
  Entity,
  Description,
  Category,
  HasDate,
  IsCustomer,
  ArchivingPeriod,
  CanBeUsed
FROM
  lkp_FileType
END

		
		
